# Planlayıcı maliyet modeli: Path'ten Plan'a

*PostgreSQL **20devel** ağacı. Satır numaraları bu sürüme aittir
([meson.build:11](../meson.build#L11)).*

[SELECT-NASIL-CALISIR.md](SELECT-NASIL-CALISIR.md) sorgu boru hattını uçtan uca
anlatıyor ve `planner` adımında "maliyet tabanlı seçim" deyip geçiyor. Bu not o
kutunun içini açıyor: planlayıcı hangi alternatifleri üretiyor, her birine hangi
formülle fiyat biçiyor, ve yüzlercesinden hangisini saklayıp hangisini atıyor.
Örnek sorgu:

```sql
SELECT u.name, o.total FROM users u JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2024-01-01';
```

---

## 30 saniyelik özet

```
   Query → query_planner → make_one_rel
     [A] set_base_rel_sizes      rows = tuples × clauselist_selectivity(...)
     [B] set_base_rel_pathlists  seqscan + index + bitmap + tid
                                 → hepsi add_path turnuvasına girer
     [C] make_rel_from_joinlist  standard_join_search / GEQO / hook
           seviye 2: {A B}, {B C}, ...   ← dinamik programlama
           seviye 3: {A B C}, ...
           her çift → add_paths_to_joinrel
             nestloop / mergejoin / hashjoin denenir
             initial_cost_* → add_path_precheck → final_cost_*
                    │ en üst joinrel'in cheapest_total_path
                    ▼
   grouping_planner: GROUP BY / ORDER BY / LIMIT üstüne eklenir
                    ▼
   create_plan(root, best_path)  →  PlannedStmt
```

1. **İki ayrı ağaç var: Path ve Plan.** Path *ucuz* bir yapıdır; planlayıcı
   binlercesini üretip atar. Plan, kazananın executor'a verilecek biçimidir.
2. **Her ilişki kümesi için tek bir `RelOptInfo`.** `{A B C}` hangi sırayla
   kurulursa kurulsun aynı RelOptInfo'ya düşer; farklı kurma yolları onun
   `pathlist`'inde yarışan **Path**'lerdir.
3. **Maliyet birimi ardışık sayfa okumadır.** `seq_page_cost = 1.0` referanstır;
   sonuç saniye değil, birimsizdir.
4. **`add_path` tek boyutlu değildir.** Beş eksen birlikte bakılır: disabled node
   sayısı + maliyet, pathkey, parameterization, satır sayısı, parallel-safety.
5. **Join sırası dinamik programlama ile aranır.** Bir seviye bitmeden üstüne
   çıkılmaz; FROM öğesi `geqo_threshold`'u (12) aşarsa arama GEQO'ya devredilir.
6. **Kardinalite hatası maliyet hatasından tehlikelidir.** Yanlış katsayı planı
   kaydırır; yanlış satır tahmini plan şeklini değiştirir ve hata yukarıya
   çarpımsal taşınır.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/optimizer/README](../src/backend/optimizer/README) | **Önce bu okunur.** Path/Plan ayrımı, join ağacı kurulumu, parameterized path'ler |
| [../src/backend/optimizer/path/allpaths.c](../src/backend/optimizer/path/allpaths.c) | `make_one_rel`, `standard_join_search` — taban yollar + join aramasının başlatılması |
| [../src/backend/optimizer/path/costsize.c](../src/backend/optimizer/path/costsize.c) | **Bütün maliyet formülleri** ve boyut tahminleri |
| [../src/backend/optimizer/util/pathnode.c](../src/backend/optimizer/util/pathnode.c) | `add_path` turnuvası, `set_cheapest`, `create_*_path` fabrikaları |
| [../src/backend/optimizer/path/clausesel.c](../src/backend/optimizer/path/clausesel.c) | `clauselist_selectivity` — qual listesi → tek seçicilik sayısı |
| [../src/backend/utils/adt/selfuncs.c](../src/backend/utils/adt/selfuncs.c) | `eqsel`, `scalarltsel`, `eqjoinsel`; MCV + histogram okuma |
| [../src/backend/optimizer/path/joinpath.c](../src/backend/optimizer/path/joinpath.c) | `add_paths_to_joinrel` — üç join yönteminin denenmesi |
| [../src/backend/optimizer/path/joinrels.c](../src/backend/optimizer/path/joinrels.c) | `join_search_one_level` — DP'nin tek adımı; `make_join_rel`, `join_is_legal` |
| [../src/backend/optimizer/path/indxpath.c](../src/backend/optimizer/path/indxpath.c) | `create_index_paths` — düz ve parametrelenmiş index yolları |
| [../src/backend/optimizer/util/relnode.c](../src/backend/optimizer/util/relnode.c) | `build_simple_rel`, `build_join_rel`, `get_baserel_parampathinfo` |
| [../src/backend/optimizer/util/plancat.c](../src/backend/optimizer/util/plancat.c) | `estimate_rel_size`, `restriction_selectivity`, `add_function_cost` |
| [../src/backend/optimizer/geqo](../src/backend/optimizer/geqo) | Genetik join arama |
| [../src/backend/optimizer/plan/createplan.c](../src/backend/optimizer/plan/createplan.c) | `create_plan` — Path ağacı → Plan ağacı |
| [../src/include/nodes/pathnodes.h](../src/include/nodes/pathnodes.h) | `PlannerInfo`, `RelOptInfo`, `Path`, `ParamPathInfo` |

---

# 1. Neden iki temsil: Path ve Plan

```
During the planning/optimizing process, we build "Path" trees representing
the different ways of doing a query.  We select the cheapest Path that
generates the desired relation and turn it into a Plan to pass to the
executor.  (There is pretty nearly a one-to-one correspondence between the
Path and Plan trees, but Path nodes omit info that won't be needed during
planning, and include info needed for planning that won't be needed by the
executor.)
```
— [../src/backend/optimizer/README:20-26](../src/backend/optimizer/README#L20-L26)

`Path` ([../src/include/nodes/pathnodes.h:1964](../src/include/nodes/pathnodes.h#L1964))
küçüktür; karar verilen alanlar aşağıdakiler artı `pathkeys` ve `param_info`:

`rows`, `disabled_nodes`, `startup_cost`, `total_cost`, `pathkeys`, `param_info`
([2005-2011](../src/include/nodes/pathnodes.h#L2005-L2011)).

Düz seq scan için ek struct gerekmez; `Path`'in kendisi kullanılır. Index scan
`IndexPath`, join'ler `NestPath`/`MergePath`/`HashPath` gibi Path'i ilk alanı
olarak taşıyan büyük struct'lardır — tam liste
[../src/backend/optimizer/README:620-662](../src/backend/optimizer/README#L620-L662).
Path'ler acımasızca `pfree` edilir
([../src/backend/optimizer/util/pathnode.c:438-442](../src/backend/optimizer/util/pathnode.c#L438-L442));
bunun neden güvenli olduğunu Bölüm 7 anlatıyor.

Dönüşüm en sonda tek çağrıda olur
([createplan.c:345](../src/backend/optimizer/plan/createplan.c#L345),
[336-338](../src/backend/optimizer/plan/createplan.c#L336-L338)) ve Plan hâlâ
bitmiş değildir: Var'lar parser numaralandırmasındadır, `setrefs.c` onları
executor slot'larına bağlar.

---

# 2. Zemin: `PlannerInfo` ve `RelOptInfo`

Her `Query` seviyesi bir `PlannerInfo` (`root`) alır. Planlamayı yöneten alanlar:
`simple_rel_array` (range table index'iyle indekslenen baserel dizisi, 0. eleman
hep boş, [352-353](../src/include/nodes/pathnodes.h#L352-L353)); `join_rel_list`
/ `join_rel_hash` (üretilmiş bütün joinrel'ler; liste büyüyünce hash tablosu
kurulur, [402-403](../src/include/nodes/pathnodes.h#L402-L403));
`join_rel_level[k]` (tam k jointree öğesi içeren joinrel'ler — DP'nin seviye
dizisi, [415-418](../src/include/nodes/pathnodes.h#L415-L418)); ve
`total_table_pages` / `tuple_fraction` / `limit_tuples`
([614-620](../src/include/nodes/pathnodes.h#L614-L620)).

**`RelOptInfo`** ([pathnodes.h:1009](../src/include/nodes/pathnodes.h#L1009)) bir
ilişkiyi veya ilişki kümesini temsil eder. Kimliği `relids` bitmapset'idir:
baserel için tek bit, `{A B C}` için üç bit. README'nin vurguladığı sonuç şudur:
`{A B C}` için **tek bir** RelOptInfo vardır, hangi sırayla kurulduğu fark etmez;
farklı kurma yolları o RelOptInfo'nun `pathlist`'ine giren Path'lerdir
([README:32-38](../src/backend/optimizer/README#L32-L38)).

Baserel/joinrel ayrımı `reloptkind` alanındadır; altı değer vardır —
`RELOPT_BASEREL`, `RELOPT_JOINREL`, `RELOPT_OTHER_MEMBER_REL`,
`RELOPT_OTHER_JOINREL`, `RELOPT_UPPER_REL`, `RELOPT_OTHER_UPPER_REL`
([975-983](../src/include/nodes/pathnodes.h#L975-L983)).

`BASEREL` sorguda doğrudan geçen tablo/subquery; `JOINREL` iki veya daha fazla
baserel'in birleşimi; `OTHER_MEMBER_REL` partition çocukları; `UPPER_REL` GROUP
BY / DISTINCT / ORDER BY gibi scan-join üstü aşamalar. Yollar `pathlist`,
`partial_pathlist`, `cheapest_startup_path`, `cheapest_total_path` alanlarında
durur ([1050-1054](../src/include/nodes/pathnodes.h#L1050-L1054)).

Sadece baserel'de anlamlı olanlar: `pages` / `tuples`
([1095-1096](../src/include/nodes/pathnodes.h#L1095-L1096)), `baserestrictinfo`
(yalnız bu ilişkiye dokunan WHERE koşulları,
[1142](../src/include/nodes/pathnodes.h#L1142)) ve `joininfo` (bu ilişkiyi içeren
join koşulları — `o.user_id = u.id` her iki tarafta da görünür,
[1148](../src/include/nodes/pathnodes.h#L1148)).

`pages`/`tuples` `pg_class`'tan gelir ama körlemesine değil:
[plancat.c:1305](../src/backend/optimizer/util/plancat.c#L1305)
`estimate_rel_size` gerçek dosya boyutuna bakıp `relpages`/`reltuples`
yoğunluğunu ölçekler — son ANALYZE'dan sonra tablo büyüdüyse planlayıcı fark eder.

---

# 3. Maliyet birimleri

`costsize.c`'nin baş yorumu bütün modeli tanımlıyor:

```
 *	seq_page_cost		Cost of a sequential page fetch
 *	random_page_cost	Cost of a non-sequential page fetch
 *	cpu_tuple_cost		Cost of typical CPU time to process a tuple
 *	cpu_index_tuple_cost  Cost of typical CPU time to process an index tuple
 *	cpu_operator_cost	Cost of CPU time to execute an operator or function
 *	parallel_tuple_cost Cost of CPU time to pass a tuple from worker to leader backend
 *	parallel_setup_cost Cost of setting up shared memory for parallelism
```
— [../src/backend/optimizer/path/costsize.c:9-15](../src/backend/optimizer/path/costsize.c#L9-L15)

Yorumun kendi ifadesiyle bunlar "arbitrary units"
([costsize.c:6](../src/backend/optimizer/path/costsize.c#L6)) — tek anlamı
`seq_page_cost`'a göre orandır. Değişkenler
[131-137](../src/backend/optimizer/path/costsize.c#L131-L137), varsayılanlar
[../src/include/optimizer/cost.h:24-34](../src/include/optimizer/cost.h#L24-L34):

| GUC | Varsayılan | Anlamı |
|---|---|---|
| `seq_page_cost` | `1.0` | Ardışık sayfa okuma — referans birim |
| `random_page_cost` | `4.0` | Rastgele sayfa okuma |
| `cpu_tuple_cost` | `0.01` | Bir tuple'ı işleme |
| `cpu_index_tuple_cost` | `0.005` | Bir index girişini işleme |
| `cpu_operator_cost` | `0.0025` | Bir operatör/fonksiyon çağrısı |
| `parallel_tuple_cost` | `0.1` | Worker'dan leader'a bir tuple |
| `parallel_setup_cost` | `1000.0` | Paralel altyapıyı kurma |
| `effective_cache_size` | `524288` sayfa (4 GB) | Postgres + OS önbelleği tahmini |

`random_page_cost / seq_page_cost = 4` oranı dönen disk varsayımıdır; SSD'de
gerçek oran 1'e yakındır. İkisi tablespace başına da ayarlanabilir
([32-34](../src/backend/optimizer/path/costsize.c#L32-L34)).

Her yol iki maliyet taşır — `startup_cost` (ilk satırdan önce ödenen) ve
`total_cost`. Kısmi sonuç aralarında doğrusal interpolasyonla fiyatlanır:
`startup + (total − startup) × tuples_to_fetch / rows`
([43-45](../src/backend/optimizer/path/costsize.c#L43-L45)). `LIMIT 10` böyle
değerlendirilir; bu yüzden yüksek startup cost'lu düğümler (sort, hash build)
`LIMIT` altında yapısal olarak dezavantajlıdır.

Her satır tahmini
[costsize.c:215](../src/backend/optimizer/path/costsize.c#L215) `clamp_row_est`'ten
geçer: sonuç `MAXIMUM_ROWCOUNT` ile sınırlanır, **en az 1'e yuvarlanır** ve
tamsayıya çevrilir
([223-228](../src/backend/optimizer/path/costsize.c#L223-L228)).

`EXPLAIN`'de asla `rows=0` görmezsin; seçicilik çok düşükse tahmin 1'de takılır.
"1 satır bekliyordu, 400 bin geldi" patlamalarının klasik başlangıcı budur.

Qual'ın CPU maliyeti
[costsize.c:4923](../src/backend/optimizer/path/costsize.c#L4923)
`cost_qual_eval` ile hesaplanır; operatör başına ücret
`cost->per_tuple += procform->procost * cpu_operator_cost;` satırıdır
([plancat.c:2398](../src/backend/optimizer/util/plancat.c#L2398)).
`pg_proc.procost` varsayılanı 1'dir — `ALTER FUNCTION ... COST 10000` doğrudan bu
çarpanı değiştirir. Sonuç `RestrictInfo->eval_cost`'ta önbelleklenir
([4978](../src/backend/optimizer/path/costsize.c#L4978)).

---

# 4. Taban ilişki yolları — `make_one_rel`

Scan/join planlamasının tek giriş noktası
[allpaths.c:183](../src/backend/optimizer/path/allpaths.c#L183); `query_planner`
onu tek satırda çağırır
([planmain.c:297](../src/backend/optimizer/plan/planmain.c#L297)). Sırası:
`set_base_rel_sizes` ([195](../src/backend/optimizer/path/allpaths.c#L195)) →
`set_base_rel_pathlists` ([238](../src/backend/optimizer/path/allpaths.c#L238)) →
`make_rel_from_joinlist` ([244](../src/backend/optimizer/path/allpaths.c#L244)).
Boyut ve yol üretiminin ayrı geçişler olmasının sebebi rowcount tahminlerinin parametrelenmiş yol üretiminden önce hazır olması ve her
rel'in `consider_parallel` bayrağının doğru kurulmuş olmasıdır
([302-305](../src/backend/optimizer/path/allpaths.c#L302-L305)).

Yol üretimi tabloya iner
([allpaths.c:838](../src/backend/optimizer/path/allpaths.c#L838)):

```c
	/* Consider sequential scan */
	add_path(rel, create_seqscan_path(root, rel, required_outer, 0));
```
— [../src/backend/optimizer/path/allpaths.c:860-861](../src/backend/optimizer/path/allpaths.c#L860-L861) —
ardından paralel seq scan ve `create_index_paths`
([862-868](../src/backend/optimizer/path/allpaths.c#L862-L868)).

**Seq scan her zaman üretilir**, index yolu bulunsa bile — bir tabloyu okumanın
en az bir yolu olmalıdır, yoksa planlayıcı çuvallar.

---

# 5. Tarama maliyetleri

**`cost_seqscan`** ([costsize.c:271](../src/backend/optimizer/path/costsize.c#L271)):

```c
	disk_run_cost = spc_seq_page_cost * baserel->pages;

	/* CPU costs */
	get_restriction_qual_cost(root, baserel, param_info, &qpqual_cost);

	startup_cost += qpqual_cost.startup;
	cpu_per_tuple = cpu_tuple_cost + qpqual_cost.per_tuple;
	cpu_run_cost = cpu_per_tuple * baserel->tuples;
```
— [../src/backend/optimizer/path/costsize.c:300-307](../src/backend/optimizer/path/costsize.c#L300-L307)

Toplam bu ikisinin toplamıdır
([338-339](../src/backend/optimizer/path/costsize.c#L338-L339)), yani
`seq_page_cost × pages + (cpu_tuple_cost + qual) × tuples`. **`tuples`, `rows`
değil** — seq scan bütün tuple'ları okur ve her birine qual uygular; WHERE ne
kadar seçici olursa olsun maliyet düşmez, sadece `path->rows` düşer. Startup cost
sıfıra yakındır, bu yüzden `LIMIT 1` sorgularında seq scan sürpriz biçimde
kazanabilir. Paralellikte CPU bölünür, disk bölünmez
([313-325](../src/backend/optimizer/path/costsize.c#L313-L325)).

**`cost_index`** ([costsize.c:546](../src/backend/optimizer/path/costsize.c#L546)).
Index'in kendi maliyeti access method'a devredilir (`btcostestimate`,
`gincostestimate`, ...;
[617-621](../src/backend/optimizer/path/costsize.c#L617-L621)). Heap sıçramaları
için iki uç senaryo hesaplanır: korelasyonsuz durumda Mackert–Lohman yaklaşımı +
`random_page_cost`; tam korelasyonlu durumda ise okunacak sayfa sayısı doğrudan
`selectivity × table_size`'dır ve ilki dışında hepsi ardışık okumadır
([646-660](../src/backend/optimizer/path/costsize.c#L646-L660)).

Aradaki geçiş:

```c
	csquared = indexCorrelation * indexCorrelation;

	run_cost += max_IO_cost + csquared * (min_IO_cost - max_IO_cost);
```
— [../src/backend/optimizer/path/costsize.c:785-787](../src/backend/optimizer/path/costsize.c#L785-L787)

`indexCorrelation` `pg_stats.correlation`'dır ve **karesi alınır**: korelasyon
0.7 olsa bile ağırlık 0.49'dur — model kararsız korelasyonlara kuşkuyla yaklaşır.
Pratik sonuç: `CLUSTER` edilmiş veya sıralı yüklenmiş bir tabloda aynı index kat
kat ucuza gelir. Index-only scan indirimi
`pages_fetched * (1.0 - baserel->allvisfrac)` çarpanıdır
([725-726](../src/backend/optimizer/path/costsize.c#L725-L726)); `allvisfrac`
visibility map'ten gelir ve VACUUM çalışmamışsa 0'dır — "index-only scan neden
hâlâ heap fetch yapıyor" sorusunun cevabı hemen her zaman budur.

**`cost_sort`** ([costsize.c:2202](../src/backend/optimizer/path/costsize.c#L2202))
işi [1952](../src/backend/optimizer/path/costsize.c#L1952) `cost_tuplesort`'a
devreder. Üç senaryo var: bellekte quicksort `N log₂ N` karşılaştırmadır
([2024-2028](../src/backend/optimizer/path/costsize.c#L2024-L2028),
`comparison_cost` varsayılanı `2.0 * cpu_operator_cost`,
[1970](../src/backend/optimizer/path/costsize.c#L1970)); `LIMIT` altındaki top-N
heapsort `N log₂ K`'dır
([2017-2022](../src/backend/optimizer/path/costsize.c#L2017-L2022)); `work_mem`
aşılırsa disk maliyeti devreye girer:

```c
		if (nruns > mergeorder)
			log_runs = ceil(log(nruns) / log(mergeorder));
		else
			log_runs = 1.0;
		npageaccesses = 2.0 * npages * log_runs;
		/* Assume 3/4ths of accesses are sequential, 1/4th are not */
		*startup_cost += npageaccesses *
			(seq_page_cost * 0.75 + random_page_cost * 0.25);
```
— [../src/backend/optimizer/path/costsize.c:2005-2012](../src/backend/optimizer/path/costsize.c#L2005-L2012)

`work_mem`'i ikiye katlamak `nruns`'ı yarılar; `log_runs` bir adım düşerse disk
maliyeti yarılanır. Sort'un tüm maliyeti `startup_cost`'a yazılır
([2226](../src/backend/optimizer/path/costsize.c#L2226)): ilk satırı vermeden
önce her şeyin sıralanması gerekir.

---

# 6. Seçicilik ve kardinalite

## 6.1 `clauselist_selectivity` — bağımsızlık varsayımı

[clausesel.c:100](../src/backend/optimizer/path/clausesel.c#L100). Yorum
yaklaşımı ve zayıflığını birlikte itiraf ediyor:

```
 * The basic approach is to apply extended statistics first, on as many
 * clauses as possible, in order to capture cross-column dependencies etc.
 * The remaining clauses are then estimated by taking the product of their
 * selectivities, but that's only right if they have independent
 * probabilities, and in reality they are often NOT independent even if they
 * only refer to a single column.  So, we want to be smarter where we can.
```
— [../src/backend/optimizer/path/clausesel.c:66-71](../src/backend/optimizer/path/clausesel.c#L66-L71)

**Bu çarpım planlayıcının en büyük hata kaynağıdır.** `WHERE city = 'Ankara' AND
plate = '06'` için iki seçicilik çarpılır; oysa ikinci koşul hiçbir şey elemez.
Çözüm `CREATE STATISTICS` — kod önce onu dener
([144-155](../src/backend/optimizer/path/clausesel.c#L144-L155)), kalanları tek
tek hesaplayıp çarpar
([183-184](../src/backend/optimizer/path/clausesel.c#L183-L184)). Tek istisna
**range query**'dir: `x > 34 AND x < 42` çifti tanınıp `hisel + losel - 1` olarak
hesaplanır ([73-82](../src/backend/optimizer/path/clausesel.c#L73-L82)).

`clause_selectivity_ext` bir operatör ifadesini join mi restriction mı olduğuna
göre ayırıp `join_selectivity` veya `restriction_selectivity` çağırır
([836-852](../src/backend/optimizer/path/clausesel.c#L836-L852)). İkincisi
tahminciyi `pg_operator.oprrest`'ten bulur — `=` için `eqsel`, `<` için
`scalarltsel`; tahminci yoksa körlemesine 0.5 döner
([plancat.c:2225](../src/backend/optimizer/util/plancat.c#L2225),
[2235-2239](../src/backend/optimizer/util/plancat.c#L2235-L2239)). Kendi
operatörünü yazan extension buraya kendi tahmincisini takar.

## 6.2 `eqsel` ve `scalarltsel` — MCV + histogram

[selfuncs.c:302](../src/backend/utils/adt/selfuncs.c#L302) `eqsel` →
[370](../src/backend/utils/adt/selfuncs.c#L370) `var_eq_const`. Değer MCV
listesindeyse cevap kesindir:

```c
			/*
			 * Constant is "=" to this common value.  We know selectivity
			 * exactly (or as exactly as ANALYZE could calculate it, anyway).
			 */
			selec = sslot.numbers[i];
```
— [../src/backend/utils/adt/selfuncs.c:474-478](../src/backend/utils/adt/selfuncs.c#L474-L478)

Değilse kalan kütle `n_distinct`'e bölünür
([495-503](../src/backend/utils/adt/selfuncs.c#L495-L503)):

Yani `(1 − MCV toplamı − null oranı) / (n_distinct − MCV sayısı)`. `n_distinct`
örneklemeye dayalıdır ve büyük tablolarda sistematik olarak düşük tahmin edilir —
`ALTER TABLE ... SET (n_distinct = ...)` bunun için vardır.

Eşitsizlikler [1546](../src/backend/utils/adt/selfuncs.c#L1546) `scalarltsel` →
[655](../src/backend/utils/adt/selfuncs.c#L655) `scalarineqsel` yoluna gider:
`mcv_selectivity` ve `ineq_histogram_selectivity` sonuçları
`selec = (1 − stanullfrac − sumcommon) × hist_selec + mcv_selec` biçiminde
birleştirilir
([773-786](../src/backend/utils/adt/selfuncs.c#L773-L786)). Sütun üç parçaya
bölünmüştür — NULL'lar, MCV'ler, kalanlar — ve histogram sadece üçüncüsünü kapsar
([1109-1111](../src/backend/utils/adt/selfuncs.c#L1109-L1111)). Kutu sayısı
`default_statistics_target` (100) ile belirlenir; `SET STATISTICS 1000` hem MCV
listesini hem histogramı 10 kat büyütür — çarpık dağılımlarda en etkili tek
müdahaledir.

İstatistik hiç yoksa sabitler devreye girer
([selfuncs.h:34-56](../src/include/utils/selfuncs.h#L34-L56)): `DEFAULT_EQ_SEL` =
0.005, `DEFAULT_INEQ_SEL` = 1/3, `DEFAULT_RANGE_INEQ_SEL` = 0.005,
`DEFAULT_NUM_DISTINCT` = 200. `EXPLAIN`'de `rows` tablo satırının tam 1/200'ü
veya 1/3'ü çıkıyorsa o sütunda istatistik yok demektir — hızlı bir teşhis işareti.

Join tarafında [2387](../src/backend/utils/adt/selfuncs.c#L2387) `eqjoinsel`
çalışır; iki tarafta da MCV listesi yoksa:

```c
		selec = (1.0 - nullfrac1) * (1.0 - nullfrac2);
		if (nd1 > nd2)
			selec /= nd1;
		else
			selec /= nd2;
```
— [../src/backend/utils/adt/selfuncs.c:2730-2734](../src/backend/utils/adt/selfuncs.c#L2730-L2734)

`MIN(1/nd1, 1/nd2) × (1−null1) × (1−null2)`: büyük `n_distinct`'e sahip taraf
bölen olur çünkü bu bir üst sınır verir. **Join'in çıktı satır sayısı doğrudan
`n_distinct` tahminine bağlıdır** ve hata üst seviyelere çarpımsal taşınır.

## 6.3 Kardinalite: baserel ve joinrel

[costsize.c:5516](../src/backend/optimizer/path/costsize.c#L5516)
`set_baserel_size_estimates`'in tamamı `rows = tuples × selectivity`:

```c
	nrows = rel->tuples *
		clauselist_selectivity(root,
							   rel->baserestrictinfo,
							   0,
							   JOIN_INNER,
							   NULL);

	rel->rows = clamp_row_est(nrows);
```
— [../src/backend/optimizer/path/costsize.c:5523-5530](../src/backend/optimizer/path/costsize.c#L5523-L5530)

[5668](../src/backend/optimizer/path/costsize.c#L5668)
`calc_joinrel_size_estimate` "kartezyen çarpım × seçicilik" yapar, ama join
tipine göre taban/tavan uygular:

```c
		case JOIN_INNER:
			nrows = outer_rows * inner_rows * fkselec * jselec;
			/* pselec not used */
			break;
		case JOIN_LEFT:
			nrows = outer_rows * inner_rows * fkselec * jselec;
			if (nrows < outer_rows)
				nrows = outer_rows;
			nrows *= pselec;
			break;
```
— [../src/backend/optimizer/path/costsize.c:5766-5775](../src/backend/optimizer/path/costsize.c#L5766-L5775)
(`JOIN_FULL`, `JOIN_SEMI` ve `JOIN_ANTI` dalları
[5776-5791](../src/backend/optimizer/path/costsize.c#L5776-L5791))

- **LEFT JOIN çıktısı outer'dan küçük olamaz** — eşleşmeyen her outer satırı NULL
  genişletilmiş çıkar; `if (nrows < outer_rows)` kelepçesi budur.
- **SEMI JOIN inner ile çarpılmaz** (`nrows = outer_rows * fkselec * jselec`) —
  `EXISTS` "kaç outer satırın eşi var" sorusudur; **ANTI** onun tümleyenidir.
- **Outer join'de iki seçicilik var**: `jselec` (ON koşulları) ve `pselec`
  (dışarıdan itilmiş WHERE); pushed-down qual'lar join'den *sonra* çalıştığı için
  ayrı hesaplanır
  ([5709-5713](../src/backend/optimizer/path/costsize.c#L5709-L5713)).

`jselec`'ten önce foreign key'ler kontrol edilir
([5698-5702](../src/backend/optimizer/path/costsize.c#L5698-L5702)); gerekçe çok
sütunlu FK'lerde bağımsızlık varsayımının çökmesidir
([5810-5815](../src/backend/optimizer/path/costsize.c#L5810-L5815)). FK varlığı
"her child satırın tam bir parent'ı var" bilgisini kesin olarak verir — örnek
sorgumuzda `orders.user_id → users.id` FK'si varsa join kardinalitesi bu yoldan
gelir. FK tanımlamanın az bilinen performans etkisi budur.

---

# 7. `add_path` ve baskınlık (dominance)

Planlayıcının kalbi
[pathnode.c:459](../src/backend/optimizer/util/pathnode.c#L459).
Kural şudur: bir yol daha iyi sıralama, daha ucuz maliyet veya daha az satır
sunuyorsa saklanmaya değerdir; parallel-safe yollar ayrıca üstün sayılır
([391-394](../src/backend/optimizer/util/pathnode.c#L391-L394)). Baskınlık ise:

```
 *	  In addition to possibly adding new_path, we also remove from the rel's
 *    pathlist any old paths that are dominated by new_path --- that is,
 *    new_path is cheaper, at least as well ordered, generates no more rows,
 *    requires no outer rels not required by the old path, and is no less
 *    parallel-safe.
```
— [../src/backend/optimizer/util/pathnode.c:408-412](../src/backend/optimizer/util/pathnode.c#L408-L412)

Beş eksen: (1) `disabled_nodes` + startup/total cost
(`compare_path_costs_fuzzily`), (2) pathkey'ler (`compare_pathkeys`),
(3) `required_outer` (`bms_subset_compare`), (4) `rows`, (5) `parallel_safe`.
Hiçbirinde geride olmayıp en az birinde önde olan yol ötekini atar.
`COSTS_EQUAL` dalı beşliyi bir arada gösteriyor:

```c
						if (keyscmp == PATHKEYS_BETTER1)
						{
							if ((outercmp == BMS_EQUAL ||
								 outercmp == BMS_SUBSET1) &&
								new_path->rows <= old_path->rows &&
								new_path->parallel_safe >= old_path->parallel_safe)
								remove_old = true;	/* new dominates old */
						}
```
— [../src/backend/optimizer/util/pathnode.c:520-527](../src/backend/optimizer/util/pathnode.c#L520-L527)

**Daha pahalı bir yol saklanabilir.** Sıralı çıktı veren pahalı bir index scan,
üstteki merge join'de sort adımını atlatabileceği için hayatta kalır; daha az
satır üreten parametrelenmiş bir yol da öyle.

Maliyet karşılaştırması
([pathnode.c:181](../src/backend/optimizer/util/pathnode.c#L181))
maliyetle başlamaz: farklıysa `disabled_nodes` her şeyi ezer
([186-193](../src/backend/optimizer/util/pathnode.c#L186-L193)).

`disabled_nodes` maliyetin üst mertebeden bileşenidir
([costsize.c:53-59](../src/backend/optimizer/path/costsize.c#L53-L59)). Sonra
maliyet gelir, ama `STD_FUZZ_FACTOR = 1.01` — yani **%1 toleransla**
([pathnode.c:47](../src/backend/optimizer/util/pathnode.c#L47)). Startup ve total
birlikte bakılır; biri birinde diğeri ötekinde öndeyse sonuç `COSTS_DIFFERENT`
olur ve ikisi de saklanır
([163-170](../src/backend/optimizer/util/pathnode.c#L163-L170)).

**`pgs_mask`.** Bu ağaçta `enable_*` GUC'ları doğrudan okunmuyor; planlama başında
bir bit maskesine çevriliyor:

```c
	glob->default_pgs_mask = PGS_APPEND | PGS_MERGE_APPEND | PGS_FOREIGNJOIN |
		PGS_GATHER | PGS_CONSIDER_NONPARTIAL;
	if (enable_tidscan)
		glob->default_pgs_mask |= PGS_TIDSCAN;
	if (enable_seqscan)
		glob->default_pgs_mask |= PGS_SEQSCAN;
	if (enable_indexscan)
		glob->default_pgs_mask |= PGS_INDEXSCAN | PGS_INDEXONLYSCAN;
```
— [../src/backend/optimizer/plan/planner.c:493-500](../src/backend/optimizer/plan/planner.c#L493-L500)

Maske her `RelOptInfo`'ya kopyalanır
([relnode.c:234](../src/backend/optimizer/util/relnode.c#L234),
[840](../src/backend/optimizer/util/relnode.c#L840)) ve maliyet fonksiyonlarında
`path->disabled_nodes = (baserel->pgs_mask & enable_mask) == enable_mask ? 0 : 1;`
biçiminde kontrol edilir
([costsize.c:336-337](../src/backend/optimizer/path/costsize.c#L336-L337)). Bit
tanımları [pathnodes.h:66-84](../src/include/nodes/pathnodes.h#L66-L84); amaç
extension'lara *ilişki bazında* strateji kısıtlama imkânı vermek — GUC global,
`pgs_mask` yereldir ([29-33](../src/include/nodes/pathnodes.h#L29-L33)).

Join yolları pahalı hesaplandığı için önce kaba bir alt sınırla elenir
([pathnode.c:686](../src/backend/optimizer/util/pathnode.c#L686)
`add_path_precheck`); `pathlist`'in `disabled_nodes` + `total_cost` sırasında
tutulmasının asıl sebebi oradaki erken çıkıştır
([711-717](../src/backend/optimizer/util/pathnode.c#L711-L717)). Bütün yollar
üretilince [268](../src/backend/optimizer/util/pathnode.c#L268) `set_cheapest`
çağrılır; `cheapest_total_path` normalde en ucuz *parametrelenmemiş* yoldur, hiç
yoksa en az parametrelenmiş olan seçilir
([245-248](../src/backend/optimizer/util/pathnode.c#L245-L248)).

---

# 8. Join arama: dinamik programlama

[allpaths.c:3847](../src/backend/optimizer/path/allpaths.c#L3847)
`make_rel_from_joinlist` üç yola ayrılır:

```c
		if (join_search_hook)
			return (*join_search_hook) (root, levels_needed, initial_rels);
		else if (enable_geqo && levels_needed >= geqo_threshold)
			return geqo(root, levels_needed, initial_rels);
		else
			return standard_join_search(root, levels_needed, initial_rels);
```
— [../src/backend/optimizer/path/allpaths.c:3913-3918](../src/backend/optimizer/path/allpaths.c#L3913-L3918)

`geqo_threshold` varsayılanı **12**
([guc_parameters.dat:1187-1194](../src/backend/utils/misc/guc_parameters.dat#L1187-L1194)).
[allpaths.c:3952](../src/backend/optimizer/path/allpaths.c#L3952)
`standard_join_search` seviye seviye ilerler: `join_rel_level[1]`'i `initial_rels` yapıp `lev` 2'den
`levels_needed`'a kadar döner
([3974-3987](../src/backend/optimizer/path/allpaths.c#L3974-L3987)).

Her seviyede `join_search_one_level` çalışır, sonra o seviyedeki her joinrel için
`set_cheapest` çağrılır
([4023](../src/backend/optimizer/path/allpaths.c#L4023)). Bu sıra keyfî değil:

```
The dynamic-programming approach has an important property that's not
immediately obvious: we will finish constructing all paths for a given
relation before we construct any paths for relations containing that rel.
This means that we can reliably identify the "cheapest path" for each rel
before higher-level relations need to know that.  Also, we can safely
discard a path when we find that another path for the same rel is better,
```
— [../src/backend/optimizer/README:174-179](../src/backend/optimizer/README#L174-L179)

`add_path`'in Path'leri `pfree` edebilmesinin sebebi tam olarak budur.

**Tek seviyenin işi** — [joinrels.c:78](../src/backend/optimizer/path/joinrels.c#L78)
`join_search_one_level` üç aşamalıdır. (a) Sol/sağ dallı planlar: önceki
seviyenin rel'leri × tek öğeler, join clause'u olanlar için
([96-124](../src/backend/optimizer/path/joinrels.c#L96-L124)); clause'u olmayan
rel kartezyen çarpıma zorlanır
([125-142](../src/backend/optimizer/path/joinrels.c#L125-L142)). (b) Bushy
planlar: k öğeli × (level−k) öğeli, ama sadece kullanılabilir join clause varsa —
planlama süresini sınırlamak için bilinçli bir kısıt
([149-153](../src/backend/optimizer/path/joinrels.c#L149-L153)). (c) Son çare:
hiçbir birleşim çıkmadıysa kartezyen çarpım zorlanır
([224-236](../src/backend/optimizer/path/joinrels.c#L224-L236)).

Her aday çift [699](../src/backend/optimizer/path/joinrels.c#L699)
`make_join_rel`'e gider: `join_is_legal` outer join kısıtlarını denetler,
`build_join_rel` RelOptInfo'yu bulur ya da yaratır (ve
`set_joinrel_size_estimates` çağırır,
[relnode.c:956](../src/backend/optimizer/util/relnode.c#L956)), sonra
`populate_joinrel_with_paths` yolları üretir
([774](../src/backend/optimizer/path/joinrels.c#L774)).

**GEQO.** 12+ FROM öğesinde DP'nin arama uzayı patlar;
[geqo_main.c:75](../src/backend/optimizer/geqo/geqo_main.c#L75) `geqo` genetik
algoritma kullanır. Her kromozom bir join sırasıdır, uygunluk o sıranın en ucuz
path maliyetidir ([geqo_eval.c:57](../src/backend/optimizer/geqo/geqo_eval.c#L57),
[163](../src/backend/optimizer/geqo/geqo_eval.c#L163)). Kritik nokta: GEQO
yalnızca join *sırasını* yarı-rastgele arar; her aday çift yine `/path` koduna
verilir, yani join *yöntemi* seçimi her iki modda da tamdır
([README:9-14](../src/backend/optimizer/README#L9-L14)). GEQO rastgeledir — aynı
sorgu iki kez farklı plan alabilir.

---

# 9. Join yolları ve maliyetleri

[joinpath.c:123](../src/backend/optimizer/path/joinpath.c#L123)
`add_paths_to_joinrel` numaralı adımlarla ilerler:

```c
	/*
	 * 1. Consider mergejoin paths where both relations must be explicitly
	 * sorted.  Skip this if we can't mergejoin.
	 */
	if (mergejoin_allowed)
		sort_inner_and_outer(root, joinrel, outerrel, innerrel,
							 jointype, &extra);
```
— [../src/backend/optimizer/path/joinpath.c:310-316](../src/backend/optimizer/path/joinpath.c#L310-L316)

4. adım `hash_inner_and_outer`'dır ve `FULL JOIN` için `enable_hashjoin` kapalı
olsa bile denenir — hash ya da merge tek seçenektir
([348-355](../src/backend/optimizer/path/joinpath.c#L348-L355)). 2. adım
`match_unsorted_outer`
([1834](../src/backend/optimizer/path/joinpath.c#L1834)) hem sıralı outer'lı
mergejoin'leri hem **bütün nested loop'ları** üretir — outer yol başına beş adete
kadar ([1817-1822](../src/backend/optimizer/path/joinpath.c#L1817-L1822)).
`FULL JOIN` özeldir: hash veya merge tek seçenektir, GUC kapalı olsa bile
denenmelidir.

**İki fazlı maliyetlendirme.** `initial_cost_*` hızlıca bir **alt sınır** üretir;
bu sınırla eleyemezsek `final_cost_*` çağrılır
([3375-3385](../src/backend/optimizer/path/costsize.c#L3375-L3385)). Akış:
`initial_cost_*` → `add_path_precheck` (açıkça kötüyse elenir) → `final_cost_*`
(qual maliyeti + gerçek satır sayısı) → `add_path`.
Yürütücüsü `try_nestloop_path` / `try_mergejoin_path` / `try_hashjoin_path`
([joinpath.c:878](../src/backend/optimizer/path/joinpath.c#L878)).

**Nested loop** ([costsize.c:3396](../src/backend/optimizer/path/costsize.c#L3396)):

```c
	startup_cost += outer_path->startup_cost + inner_path->startup_cost;
	run_cost += outer_path->total_cost - outer_path->startup_cost;
	if (outer_path_rows > 1)
		run_cost += (outer_path_rows - 1) * inner_rescan_start_cost;
```
— [../src/backend/optimizer/path/costsize.c:3428-3431](../src/backend/optimizer/path/costsize.c#L3428-L3431)

Normal durumda inner, outer satır sayısı kadar taranır
([3451-3457](../src/backend/optimizer/path/costsize.c#L3451-L3457)); `cost_rescan`
([4808](../src/backend/optimizer/path/costsize.c#L4808)) sonraki taramaların
ucuzlayabileceğini modeller (Material düğümü veya index scan).
`SEMI`/`ANTI`'de ve inner unique olduğunda ilk eşleşmede durulur
([3436-3449](../src/backend/optimizer/path/costsize.c#L3436-L3449)). CPU maliyeti
`final_cost_nestloop`'ta **işlenen** tuple sayısı
(`ntuples = outer_rows × inner_rows`,
[3631](../src/backend/optimizer/path/costsize.c#L3631)) üzerinden eklenir. Nested
loop'un tek gerçek avantajı düşük startup cost'tur; `O(N×M)` terimi onu hızla
eler — **inner parametrelenmiş bir index scan değilse** (Bölüm 10).

**Hash join** ([costsize.c:4320](../src/backend/optimizer/path/costsize.c#L4320)):

```c
	/* cost of source data */
	startup_cost += outer_path->startup_cost;
	run_cost += outer_path->total_cost - outer_path->startup_cost;
	startup_cost += inner_path->total_cost;
```
— [../src/backend/optimizer/path/costsize.c:4348-4351](../src/backend/optimizer/path/costsize.c#L4348-L4351)

Son satır hash join'in karakteridir: **inner tarafın tamamı ilk çıktı satırından
önce ödenir.** Üstüne hash hesaplama ücreti gelir
([4363-4365](../src/backend/optimizer/path/costsize.c#L4363-L4365)). Batch sayısı
executor'ın kendi mantığından alınır (`ExecChooseHashTableSize`,
[4386-4394](../src/backend/optimizer/path/costsize.c#L4386-L4394)); birden
fazlaysa `seq_page_cost` üzerinden disk ücreti eklenir
([4403-4412](../src/backend/optimizer/path/costsize.c#L4403-L4412)).
`final_cost_hashjoin` bucket doluluğunu tahmin edip
([4525-4572](../src/backend/optimizer/path/costsize.c#L4525-L4572)) karşılaştırma
sayısını `outer_rows × inner_rows × innerbucketsize × 0.5` olarak fiyatlar
([4661-4663](../src/backend/optimizer/path/costsize.c#L4661-L4663)). Çarpık
dağılıma karşı güvenlik freni vardır: tek bir değerin bucket'ı `hash_mem`'i
aşacaksa `disable_cost` eklenir
([4578-4587](../src/backend/optimizer/path/costsize.c#L4578-L4587)).

**Merge join** ([costsize.c:3681](../src/backend/optimizer/path/costsize.c#L3681))
girdilerin sıralı olmasını gerektirir — ya hazır bir pathkey'den ya da açık bir
sort'tan. Ayırt edici ayrıntı: girdilerin tamamını okumak zorunda değildir.
`mergejoinscansel` iki tarafın örtüşen aralığını tahmin eder
([3746](../src/backend/optimizer/path/costsize.c#L3746)) ve satır sayılarına
çevirir — `outer_skip_rows`, `inner_skip_rows`, `outer_rows`, `inner_rows`
([3790-3793](../src/backend/optimizer/path/costsize.c#L3790-L3793)).

A'nın değerleri 1–100, B'ninkiler 5000–6000 ise iki taraf da erken durur; hash
join'de böyle bir avantaj yoktur. İkinci ayrıntı: outer'daki tekrar eden
anahtarlar inner'ın yeniden taranmasını gerektirir. Tahmin "join çıktısı − inner
satır sayısı"dır ([4058-4068](../src/backend/optimizer/path/costsize.c#L4058-L4068))
ve `rescanratio = 1.0 + (rescannedtuples / inner_rows)` oranına çevrilir
([4096](../src/backend/optimizer/path/costsize.c#L4096)). Oran yüksekse inner'a
Material düğümü eklemek ucuza gelebilir
([4136-4139](../src/backend/optimizer/path/costsize.c#L4136-L4139)) — `EXPLAIN`'de
merge join altındaki `Materialize` düğümünün açıklaması budur. CPU maliyeti
`N + M + rescan` karşılaştırmasına dayanır
([4205-4214](../src/backend/optimizer/path/costsize.c#L4205-L4214)).

| | Startup | Run | Sıralı çıktı | Kazandığı yer |
|---|---|---|---|---|
| Nested loop | Çok düşük | `O(N × M)` | Outer'ınki korunur | Küçük outer + parametrelenmiş inner |
| Hash join | Inner'ın tamamı | `O(N)` | Yok | Büyük eşitlik join'leri, bellek yeterli |
| Merge join | Sort gerekiyorsa yüksek | `O(N + M)` | Var | Girdiler zaten sıralı; büyük × büyük |

---

# 10. Parametrelenmiş yollar (parameterized paths)

Problemi README kuruyor: A küçük, B büyük ve `B.Y` üzerinde index varsa

```
something like this:

	NestLoop
		-> Seq Scan on A
		-> Index Scan using B_Y_IDX on B
			Index Condition: B.Y = A.X
```
— [../src/backend/optimizer/README:1073-1078](../src/backend/optimizer/README#L1073-L1078)

Nested loop'un tek gerçek kullanım alanı budur: inner'ın outer'dan gelen değeri
**index anahtarı** olarak kullanabilmesi. Temsil olarak Path'e `param_info`
eklenir ([pathnodes.h:1918](../src/include/nodes/pathnodes.h#L1918)):

```c
	Relids		ppi_req_outer;	/* rels supplying parameters used by path */
	Cardinality ppi_rows;		/* estimated number of result tuples */
	List	   *ppi_clauses;	/* join clauses available from outer rels */
```
— [../src/include/nodes/pathnodes.h:1924-1926](../src/include/nodes/pathnodes.h#L1924-L1926)

`ppi_req_outer` = "bu yol çalışmak için şu rel'lerden değer bekliyor"; erişim
makrosu `PATH_REQ_OUTER`
([2015-2016](../src/include/nodes/pathnodes.h#L2015-L2016)). Aynı parametrelenmedeki
bütün yollar tek `ParamPathInfo`'yu paylaşır, böylece satır tahmini hepsinde aynı
olur ([1901-1905](../src/include/nodes/pathnodes.h#L1901-L1905)).

**Satır sayısının neden ayrı bir ölçüt olduğu** burada anlaşılıyor:

```
While all ordinary paths for a particular relation generate the same set
of rows (since they must all apply the same set of restriction clauses),
parameterized paths typically generate fewer rows than less-parameterized
paths, since they have additional clauses to work with.  This means we
must consider the number of rows generated as an additional figure of
merit.  A path that costs more than another, but generates fewer rows,
```
— [../src/backend/optimizer/README:1154-1159](../src/backend/optimizer/README#L1154-L1159)

Tahmin [costsize.c:5546](../src/backend/optimizer/path/costsize.c#L5546)
`get_parameterized_baserel_size` ile yapılır; ekstra join clause'lar da
seçiciliğe katılır.

İki politika kararı var. **(1)** Parametrelenmiş yolun pathkey'i yok sayılır
(`new_path_pathkeys = new_path->param_info ? NIL : new_path->pathkeys;`,
[pathnode.c:473](../src/backend/optimizer/util/pathnode.c#L473)), çünkü yol zaten
nested loop'un iç tarafında kullanılacak ve orada sıralaması kimseyi
ilgilendirmiyor ([README:1190-1199](../src/backend/optimizer/README#L1190-L1199)).
**(2)** Aynı parametrelenmedeki bütün yollar aynı satır sayısını üretmelidir; her
biri o outer rel'lerle kullanılabilen *bütün* join clause'ları uygulamak
zorundadır ([README:1164-1168](../src/backend/optimizer/README#L1164-L1168)). Bu
kısıt `add_path_precheck`'in satır sayısını bilmeden çalışmasını sağlar.

Parametrelenmiş taban yolları sadece index scan'ler için üretilir
([README:1180-1184](../src/backend/optimizer/README#L1180-L1184)) —
[indxpath.c:239](../src/backend/optimizer/path/indxpath.c#L239)
`create_index_paths`:

```
 * join clauses (plus restriction clauses, if available) in its indexqual.
 * When joining such a scan to one of the relations supplying the other
 * variables used in its indexqual, the parameterized scan must appear as
 * the inner relation of a nestloop join; it can't be used on the outer side,
 * nor in a merge or hash join.  In that context, values for the other rels'
```
— [../src/backend/optimizer/path/indxpath.c:214-218](../src/backend/optimizer/path/indxpath.c#L214-L218)

`try_nestloop_path` parametrelenmenin makul olup olmadığını denetleyip
gereksizleri reddeder
([joinpath.c:929-938](../src/backend/optimizer/path/joinpath.c#L929-L938)).

---

# 11. Planlayıcı hook'ları

| Hook | Yer | Ne yapar |
|---|---|---|
| `planner_hook` | [planner.c:74](../src/backend/optimizer/plan/planner.c#L74) | Bütün planlamayı sarmalar (çağrı yeri [333-338](../src/backend/optimizer/plan/planner.c#L333-L338)); `pg_stat_statements` planlama süresini böyle ölçer |
| `planner_setup_hook` | [planner.c:77](../src/backend/optimizer/plan/planner.c#L77) | `PlannerGlobal` kurulunca; `default_pgs_mask`'i değiştirebilir |
| `planner_shutdown_hook` | [planner.c:80](../src/backend/optimizer/plan/planner.c#L80) | Planlama bitince |
| `set_rel_pathlist_hook` | [allpaths.c:89](../src/backend/optimizer/path/allpaths.c#L89) | Bir baserel'in yolları üretildikten sonra ([589-590](../src/backend/optimizer/path/allpaths.c#L589-L590)); custom scan buradan eklenir |
| `set_join_pathlist_hook` | [joinpath.c:31](../src/backend/optimizer/path/joinpath.c#L31) | Bir joinrel'in yolları üretildikten sonra |
| `join_search_hook` | [allpaths.c:92](../src/backend/optimizer/path/allpaths.c#L92) | Join arama algoritmasının tamamını değiştirir |
| `build_simple_rel_hook` | [relnode.c:51](../src/backend/optimizer/util/relnode.c#L51) | Bir baserel'in `RelOptInfo`'su kurulurken |

`join_search_hook` yazarlarına uyarı: `standard_join_search` sırasında çağrılan
fonksiyonlar `root->join_rel_list` ve `root->join_rel_hash`'i değiştirir; birden
fazla arama yapacaksan bu yapıların durumunu kaydedip geri yüklemen gerekir —
örnek `geqo_eval()`
([allpaths.c:3946-3949](../src/backend/optimizer/path/allpaths.c#L3946-L3949)).

---

# İzleme ve hata ayıklama

**EXPLAIN.** Maliyet biçimi
[explain.c:1811](../src/backend/commands/explain.c#L1811):

```
Seq Scan on users  (cost=0.00..1834.00 rows=1250 width=36)
                         │       │          │        └─ ortalama satır baytı
                         │       │          └─ tahmini çıktı satırı
                         │       └─ total_cost
                         └─ startup_cost
```

`EXPLAIN ANALYZE`'da `rows=` ile `actual rows=` arasındaki oran tahmin hatasıdır.
**10 katı aşan her fark araştırılmalıdır**, ve plan ağacında *en alttaki* hatalı
düğümden başlanır — hata yukarı doğru çarpımsal büyür. `Rows Removed by Filter`
seçiciliğin gerçek değerini verir. `Disabled: true` satırı
([explain.c:1893-1895](../src/backend/commands/explain.c#L1893-L1895)) bir
`enable_*` GUC'unun kapalı olduğunu ama planlayıcının başka seçenek bulamadığını
söyler.

**`debug_print_plan`**
([guc_parameters.dat:716-717](../src/backend/utils/misc/guc_parameters.dat#L716-L717))
`EXPLAIN`'in gizlediği her şeyi loga döker: targetlist Var'ları, qual ağaçları,
`plan_rows`/`plan_width`, `parallel_aware` bayrakları. `debug_pretty_print = on`
ve `client_min_messages = log` ile birlikte kullanılır; kardeşleri
`debug_print_parse` ve `debug_print_rewritten`. Path düzeyini görmek için kaynak
`OPTIMIZER_DEBUG` ile derlenmelidir
([allpaths.c:4042-4044](../src/backend/optimizer/path/allpaths.c#L4042-L4044)).

**`enable_*` GUC'ları teşhis aracıdır, ayar aracı değil.** Tam liste 27 adet
([guc_parameters.dat:866-1049](../src/backend/utils/misc/guc_parameters.dat#L866-L1049)).
Kullanım biçimi: sorguyu `EXPLAIN ANALYZE` ile ölç, `SET enable_hashjoin = off`
deyip tekrar ölç, `RESET` et. Alternatif daha hızlıysa **sorun maliyet modelinde
değil, tahmindedir** — doğru müdahale GUC'u kapalı bırakmak değil; ANALYZE,
`SET STATISTICS`, `CREATE STATISTICS` veya `random_page_cost` ayarıdır. Bu
GUC'lar yolu tamamen yasaklamaz, sadece `disabled_nodes` sayacını artırır.

**Tahmin hatası avlamak:**

```sql
SELECT attname, n_distinct, null_frac, correlation,
       array_length(most_common_vals, 1) AS n_mcv,
       array_length(histogram_bounds, 1) AS n_hist
FROM   pg_stats WHERE tablename = 'orders';
-- ayrıca: pg_class.relpages/reltuples/relallvisible ve
-- pg_stat_user_tables.last_analyze / n_mod_since_analyze
```

| Belirti | Muhtemel sebep | Müdahale |
|---|---|---|
| `rows` = tablo/200 | `n_distinct` yok, `DEFAULT_NUM_DISTINCT` | `ANALYZE` |
| `rows` = tablo/3 | Histogram yok, `DEFAULT_INEQ_SEL` | `ANALYZE` |
| `rows=1`, `actual rows=400000` | `clamp_row_est` tabanına çakıldı | `CREATE STATISTICS` |
| İki sütunlu AND'de büyük sapma | Bağımsızlık varsayımı | `CREATE STATISTICS (dependencies)` |
| Çarpık sütunda `=` hatası | MCV listesi kısa | `SET STATISTICS 1000` |
| Index scan seçilmiyor | `random_page_cost` SSD'ye göre yüksek | `random_page_cost = 1.1` |
| Index-only scan heap'e gidiyor | `allvisfrac` düşük | `VACUUM` |
| Aynı sorgu farklı planlar | GEQO rastgeleliği | `geqo_threshold` yükselt |

Planlama süresinin kendisi `EXPLAIN` çıktısındaki `Planning Time`'dır; çok
tablolu sorgularda yüksekse `join_collapse_limit` / `from_collapse_limit`
(varsayılan 8,
[guc_parameters.dat:1108](../src/backend/utils/misc/guc_parameters.dat#L1108),
[1536](../src/backend/utils/misc/guc_parameters.dat#L1536)) devrededir.

---

# Tek sayfalık özet

```
  İSTATİSTİK   pg_class.relpages/reltuples + pg_statistic (MCV, histogram,
               n_distinct, correlation) — ANALYZE toplar, estimate_rel_size
               gerçek dosya boyutuna göre ölçekler
        ▼
  SEÇİCİLİK    clauselist_selectivity → clause_selectivity
   clausesel.c   → restriction_selectivity (pg_operator.oprrest)
   selfuncs.c        ├─ eqsel       → MCV eşleşmesi / n_distinct
                     └─ scalarltsel → MCV + histogram
               ⚠ bağımsızlık varsayımı: s1 × s2 × ...
        ▼
  KARDİNALİTE  baserel: rows = tuples × selectivity
   costsize.c  joinrel: outer × inner × jselec × fkselec
                        + join tipine göre taban/tavan
               clamp_row_est: en az 1, tam sayı
        ▼
  MALİYET      seq:   pages×1.0 + tuples×(0.01+qual)
   costsize.c  index: amcostestimate + csquared interp. (Mackert–Lohman)
               sort:  N log N × 2×0.0025, work_mem'i aşarsa diske taşar
               NL / hash / merge: N×M  /  startup=inner + N  /  N+M×rescan
        ▼
  SEÇİM        add_path — 5 eksenli baskınlık: disabled_nodes →
   pathnode.c  startup/total (fuzz %1) → pathkeys → required_outer →
               rows → parallel_safe;  set_cheapest → cheapest_total_path
        ▼
  ARAMA        standard_join_search: seviye 2, 3, ..., N
   allpaths.c    join_search_one_level: sol/sağ dallı + bushy
   joinrels.c  ≥ geqo_threshold (12) → GEQO
   geqo/       join_search_hook → extension
        ▼
               create_plan → PlannedStmt
```

## Akılda kalması gereken 5 şey

1. **Path ucuz, Plan pahalı.** Planlayıcı binlerce Path üretip `pfree` eder;
   sadece kazanan Plan'a dönüşür. Bunu güvenli kılan şey DP'nin "alt seviye
   bitmeden üste çıkma" özelliğidir.

2. **`add_path` tek boyutlu değil.** Daha pahalı bir yol — daha iyi sıralama,
   daha az satır veya daha az parametrelenme sunuyorsa — saklanır. `EXPLAIN`
   bazen "daha pahalı" bir alt yolu seçmiş gibi görünür; üst seviyede kazandığı
   sort'u hesaba katmıyorsundur.

3. **Maliyet birimsizdir, `seq_page_cost = 1.0` referanstır.** Saniyeye çevirmeye
   çalışma. Ayarlanacak tek gerçek oran `random_page_cost / seq_page_cost`'tur ve
   SSD'de 4.0 fazla yüksektir.

4. **Kardinalite hatası maliyet hatasından tehlikelidir.** Satır tahmini 100 kat
   yanlışsa hangi formülün kullanıldığı önemsizdir. `EXPLAIN ANALYZE`'da en
   alttaki büyük `rows` / `actual rows` sapmasından başla.

5. **Bağımsızlık varsayımı en zayıf noktadır.** `clauselist_selectivity`
   seçicilikleri çarpar; korelasyonlu sütunlarda bu çarpım gerçeğin çok altında
   kalır. `CREATE STATISTICS` bunun için var.
