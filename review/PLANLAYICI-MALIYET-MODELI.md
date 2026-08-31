# Planlayıcı maliyet modeli: Path'ten Plan'a

*PostgreSQL **20devel** ağacı. Satır numaraları bu sürüme aittir
([meson.build:11](../meson.build#L11)).*

[SELECT-NASIL-CALISIR.md](SELECT-NASIL-CALISIR.md) sorgu boru hattını uçtan uca
anlatıyor ve `planner` adımında "maliyet tabanlı seçim" deyip geçiyor. Bu not o
kutunun içini açıyor: planlayıcı hangi alternatifleri üretiyor, her birine hangi
formülle fiyat biçiyor, ve yüzlerce alternatiften hangisini saklayıp hangisini
atıyor.

Örnek sorgu — dosya boyunca bunu takip ediyoruz:

```sql
SELECT u.name, o.total
FROM   users u JOIN orders o ON o.user_id = u.id
WHERE  u.created_at > '2024-01-01';
```

---

## 30 saniyelik özet

```
   Query (rewrite sonrası)
        │
        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ query_planner → make_one_rel                                 │
   │                                                              │
   │  [A] set_base_rel_sizes     her baserel için rows / width    │
   │        rows = tuples × clauselist_selectivity(...)           │
   │                                                              │
   │  [B] set_base_rel_pathlists  her baserel için Path'ler       │
   │        seqscan + index + bitmap + tid → add_path turnuvası   │
   │                                                              │
   │  [C] make_rel_from_joinlist                                  │
   │        standard_join_search (veya GEQO / join_search_hook)   │
   │          seviye 2: {A B}, {B C}, ...                         │
   │          seviye 3: {A B C}, ...   ← dinamik programlama      │
   │          her çift → add_paths_to_joinrel                     │
   │            nestloop / mergejoin / hashjoin denenir           │
   │            initial_cost_* → add_path_precheck → final_cost_* │
   └──────────────────────────┬───────────────────────────────────┘
                              │ en üst joinrel'in cheapest_total_path
                              ▼
       grouping_planner: GROUP BY / ORDER BY / LIMIT üstüne eklenir
                              ▼
              create_plan(root, best_path)  →  PlannedStmt
```

Zihinsel model — altı madde:

1. **İki ayrı ağaç var: Path ve Plan.** Path *ucuz* bir yapıdır; planlayıcı
   binlercesini üretip atar. Plan, tek bir kazananın executor'a verilecek
   biçimidir.
2. **Her ilişki kümesi için tek bir `RelOptInfo`.** `{A B C}` hangi sırayla
   kurulursa kurulsun aynı RelOptInfo'ya düşer; farklı kurma yolları onun
   `pathlist`'inde yarışan **Path**'lerdir.
3. **Maliyet birimi ardışık sayfa okumadır.** `seq_page_cost = 1.0` referanstır;
   sonuç saniye değil, birimsizdir.
4. **`add_path` tek boyutlu bir karşılaştırma değildir.** Beş eksen birlikte
   bakılır: disabled node sayısı + maliyet, pathkey, parameterization, satır
   sayısı, parallel-safety.
5. **Join sırası dinamik programlama ile aranır.** Seviye seviye ilerlenir; bir
   seviye bitmeden üstüne çıkılmaz. FROM öğesi `geqo_threshold`'u (12) aşarsa
   arama genetik algoritmaya devredilir.
6. **Kardinalite hatası maliyet hatasından tehlikelidir.** Yanlış katsayı planı
   biraz kaydırır; yanlış satır tahmini plan şeklini değiştirir ve hata yukarıya
   çarpımsal taşınır.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/optimizer/README](../src/backend/optimizer/README) | **Önce bu okunur.** Path/Plan ayrımı, join ağacı kurulumu, parameterized path'ler — 1623 satırlık tasarım belgesi |
| [../src/backend/optimizer/path/allpaths.c](../src/backend/optimizer/path/allpaths.c) | `make_one_rel`, `standard_join_search` — taban yollar + join aramasının başlatılması |
| [../src/backend/optimizer/path/costsize.c](../src/backend/optimizer/path/costsize.c) | **Bütün maliyet formülleri** ve boyut tahminleri |
| [../src/backend/optimizer/util/pathnode.c](../src/backend/optimizer/util/pathnode.c) | `add_path` turnuvası, `set_cheapest`, `create_*_path` fabrikaları |
| [../src/backend/optimizer/path/clausesel.c](../src/backend/optimizer/path/clausesel.c) | `clauselist_selectivity` — qual listesi → tek seçicilik sayısı |
| [../src/backend/utils/adt/selfuncs.c](../src/backend/utils/adt/selfuncs.c) | Operatör başına seçicilik: `eqsel`, `scalarltsel`, `eqjoinsel`; MCV + histogram |
| [../src/backend/optimizer/path/joinpath.c](../src/backend/optimizer/path/joinpath.c) | `add_paths_to_joinrel` — bir joinrel için üç join yönteminin denenmesi |
| [../src/backend/optimizer/path/joinrels.c](../src/backend/optimizer/path/joinrels.c) | `join_search_one_level` — DP'nin tek adımı; `make_join_rel`, `join_is_legal` |
| [../src/backend/optimizer/path/indxpath.c](../src/backend/optimizer/path/indxpath.c) | `create_index_paths` — düz ve parametrelenmiş index yolları |
| [../src/backend/optimizer/util/relnode.c](../src/backend/optimizer/util/relnode.c) | `build_simple_rel`, `build_join_rel`, `get_baserel_parampathinfo` |
| [../src/backend/optimizer/util/plancat.c](../src/backend/optimizer/util/plancat.c) | Katalog arayüzü: `estimate_rel_size`, `restriction_selectivity`, `add_function_cost` |
| [../src/backend/optimizer/geqo](../src/backend/optimizer/geqo) | Genetik join arama — 12'den fazla FROM öğesinde |
| [../src/backend/optimizer/plan/createplan.c](../src/backend/optimizer/plan/createplan.c) | `create_plan` — kazanan Path ağacı → Plan ağacı |
| [../src/include/nodes/pathnodes.h](../src/include/nodes/pathnodes.h) | `PlannerInfo`, `RelOptInfo`, `Path`, `ParamPathInfo` |

---

# 1. Neden iki temsil: Path ve Plan

Gerekçe README'nin ilk sayfasında:

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

`Path` yapısı ([../src/include/nodes/pathnodes.h:1964](../src/include/nodes/pathnodes.h#L1964))
küçüktür; karar verilen alanlar şunlardır:

```c
	/* estimated size/costs for path (see costsize.c for more info) */
	Cardinality rows;			/* estimated number of result tuples */
	int			disabled_nodes; /* count of disabled nodes */
	Cost		startup_cost;	/* cost expended before fetching any tuples */
	Cost		total_cost;		/* total cost (assuming all tuples fetched) */

	/* sort ordering of path's output; a List of PathKey nodes; see above */
	List	   *pathkeys;
```
— [../src/include/nodes/pathnodes.h:2004-2011](../src/include/nodes/pathnodes.h#L2004-L2011)

Bunlara `param_info` eklenir (Bölüm 11). `add_path` kararlarını *yalnızca* bu
altı şeye bakarak verir.

Düz seq scan için ek struct gerekmez; `Path`'in kendisi kullanılır. Index scan
`IndexPath`, join'ler `NestPath`/`MergePath`/`HashPath` gibi Path'i ilk alanı
olarak taşıyan büyük struct'lardır — tam liste
[../src/backend/optimizer/README:620-662](../src/backend/optimizer/README#L620-L662).

Path'ler acımasızca `pfree` edilir
([../src/backend/optimizer/util/pathnode.c:438-442](../src/backend/optimizer/util/pathnode.c#L438-L442));
bunun neden güvenli olduğunu Bölüm 9 anlatıyor. Path → Plan dönüşümü en sonda,
tek çağrıda olur:
[../src/backend/optimizer/plan/createplan.c:345](../src/backend/optimizer/plan/createplan.c#L345).

---

# 2. Zemin: `PlannerInfo` ve `RelOptInfo`

Her `Query` seviyesi bir `PlannerInfo` (`root`) alır. Planlamayı yöneten üç alan:

```c
	struct RelOptInfo **simple_rel_array pg_node_attr(array_size(simple_rel_array_size));
	/* allocated size of array */
```
— [../src/include/nodes/pathnodes.h:352-353](../src/include/nodes/pathnodes.h#L352-L353) —
range table index'iyle indekslenen baserel dizisi (0. eleman hep boş).

```c
	List	   *join_rel_list;
	struct HTAB *join_rel_hash pg_node_attr(read_write_ignore);
```
— [../src/include/nodes/pathnodes.h:402-403](../src/include/nodes/pathnodes.h#L402-L403) —
şimdiye kadar üretilmiş bütün joinrel'ler; liste büyüyünce hash tablosu kurulur.

```c
	/* lists of join-relation RelOptInfos */
	List	  **join_rel_level pg_node_attr(read_write_ignore);
	/* index of list being extended */
	int			join_cur_level;
```
— [../src/include/nodes/pathnodes.h:415-418](../src/include/nodes/pathnodes.h#L415-L418) —
dinamik programlamanın seviye dizisi (Bölüm 9).

Maliyeti doğrudan etkileyen iki alan daha: `total_table_pages` (index önbellek
etkisi) ve `tuple_fraction` / `limit_tuples` — `LIMIT` varlığında startup cost'a
verilecek ağırlık
([../src/include/nodes/pathnodes.h:618-620](../src/include/nodes/pathnodes.h#L618-L620)).

## 2.1 `RelOptInfo` — baserel ve joinrel

[../src/include/nodes/pathnodes.h:1009](../src/include/nodes/pathnodes.h#L1009).
Kimliği `relids` bitmapset'idir: baserel için tek bit, `{A B C}` için üç bit.
README bunun sonucunu vurguluyor:

```
planning.  A join rel is simply a combination of base rels.  There is only
one join RelOptInfo for any given set of baserels --- for example, the join
{A B C} is represented by the same RelOptInfo no matter whether we build it
by joining A and B first and then adding C, or joining B and C first and
then adding A, etc.  These different means of building the joinrel are
represented as Paths.  For each RelOptInfo we build a list of Paths that
represent plausible ways to implement the scan or join of that relation.
```
— [../src/backend/optimizer/README:32-38](../src/backend/optimizer/README#L32-L38)

Ayrım `reloptkind` alanındadır:

```c
typedef enum RelOptKind
{
	RELOPT_BASEREL,
	RELOPT_JOINREL,
	RELOPT_OTHER_MEMBER_REL,
	RELOPT_OTHER_JOINREL,
	RELOPT_UPPER_REL,
	RELOPT_OTHER_UPPER_REL,
} RelOptKind;
```
— [../src/include/nodes/pathnodes.h:975-983](../src/include/nodes/pathnodes.h#L975-L983)

`BASEREL` sorguda doğrudan geçen tablo/subquery; `JOINREL` iki veya daha fazla
baserel'in birleşimi; `OTHER_MEMBER_REL` partition/inheritance çocukları;
`UPPER_REL` GROUP BY / DISTINCT / ORDER BY gibi scan-join üstü aşamalar.

Yol depoları:

```c
	List	   *pathlist;		/* Path structures */
	List	   *ppilist;		/* ParamPathInfos used in pathlist */
	List	   *partial_pathlist;	/* partial Paths */
	struct Path *cheapest_startup_path;
	struct Path *cheapest_total_path;
```
— [../src/include/nodes/pathnodes.h:1050-1054](../src/include/nodes/pathnodes.h#L1050-L1054)

Baserel'e özgü istatistikler `pages` / `tuples`
([../src/include/nodes/pathnodes.h:1095-1096](../src/include/nodes/pathnodes.h#L1095-L1096));
qual listeleri `baserestrictinfo`
([../src/include/nodes/pathnodes.h:1142](../src/include/nodes/pathnodes.h#L1142) —
sadece bu ilişkiye dokunan koşullar) ve `joininfo`
([../src/include/nodes/pathnodes.h:1148](../src/include/nodes/pathnodes.h#L1148) —
bu ilişkiyi içeren join koşulları; `o.user_id = u.id` her iki tarafta da görünür).

`pages`/`tuples` `pg_class`'tan gelir ama körlemesine değil:
[../src/backend/optimizer/util/plancat.c:1305](../src/backend/optimizer/util/plancat.c#L1305)
`estimate_rel_size` gerçek dosya boyutuna bakıp `relpages`/`reltuples`
yoğunluğunu ölçekler — son ANALYZE'dan sonra tablo büyüdüyse planlayıcı bunu
fark eder.

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
([../src/backend/optimizer/path/costsize.c:6](../src/backend/optimizer/path/costsize.c#L6));
tek anlamı `seq_page_cost`'a göre orandır. Değişkenler
[../src/backend/optimizer/path/costsize.c:131-137](../src/backend/optimizer/path/costsize.c#L131-L137),
varsayılanları
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
([../src/backend/optimizer/path/costsize.c:32-34](../src/backend/optimizer/path/costsize.c#L32-L34)).

## 3.1 İki maliyet, tek doğru

```
 * We compute two separate costs for each path:
 *		total_cost: total estimated cost to fetch all tuples
 *		startup_cost: cost that is expended before first tuple is fetched
```
— [../src/backend/optimizer/path/costsize.c:37-39](../src/backend/optimizer/path/costsize.c#L37-L39)

Kısmi sonuç bu ikisi arasında doğrusal interpolasyonla fiyatlanır:

```
 * result by interpolating between startup_cost and total_cost.  In detail:
 *		actual_cost = startup_cost +
 *			(total_cost - startup_cost) * tuples_to_fetch / path->rows;
```
— [../src/backend/optimizer/path/costsize.c:43-45](../src/backend/optimizer/path/costsize.c#L43-L45)

`LIMIT 10` böyle değerlendirilir. Bu yüzden yüksek startup cost'lu düğümler
(sort, hash build) `LIMIT` altında yapısal olarak dezavantajlıdır.

## 3.2 `clamp_row_est` — sıfır satır diye bir şey yok

[../src/backend/optimizer/path/costsize.c:215](../src/backend/optimizer/path/costsize.c#L215):

```c
	if (nrows > MAXIMUM_ROWCOUNT || isnan(nrows))
		nrows = MAXIMUM_ROWCOUNT;
	else if (nrows <= 1.0)
		nrows = 1.0;
	else
		nrows = rint(nrows);

	return nrows;
```
— [../src/backend/optimizer/path/costsize.c:223-230](../src/backend/optimizer/path/costsize.c#L223-L230)

Her satır tahmini buradan geçer. `EXPLAIN`'de asla `rows=0` görmezsin — ve
seçicilik çok düşükse tahmin 1'de takılır. "1 satır bekliyordu, 400 bin geldi"
patlamalarının klasik başlangıcı budur.

## 3.3 Qual maliyeti

[../src/backend/optimizer/path/costsize.c:4923](../src/backend/optimizer/path/costsize.c#L4923)
`cost_qual_eval`. Operatör başına ücret
[../src/backend/optimizer/util/plancat.c:2398](../src/backend/optimizer/util/plancat.c#L2398):

```c
	cost->per_tuple += procform->procost * cpu_operator_cost;
```

`pg_proc.procost` varsayılanı 1'dir. `ALTER FUNCTION ... COST 10000` doğrudan bu
çarpanı değiştirir ve planlayıcının o fonksiyonu az satırda çağırmasını sağlar.
Sonuç `RestrictInfo->eval_cost`'ta önbelleklenir
([../src/backend/optimizer/path/costsize.c:4978](../src/backend/optimizer/path/costsize.c#L4978)).

---

# 4. Taban ilişki yolları — `make_one_rel`

Scan/join planlamasının tek giriş noktası
[../src/backend/optimizer/path/allpaths.c:183](../src/backend/optimizer/path/allpaths.c#L183);
`query_planner` onu tek satırda çağırır
([../src/backend/optimizer/plan/planmain.c:297](../src/backend/optimizer/plan/planmain.c#L297)).

```c
	set_base_rel_consider_startup(root);

	/*
	 * Compute size estimates and consider_parallel flags for each base rel.
	 */
	set_base_rel_sizes(root);
```
— [../src/backend/optimizer/path/allpaths.c:190-195](../src/backend/optimizer/path/allpaths.c#L190-L195)

Sonra `set_base_rel_pathlists`
([../src/backend/optimizer/path/allpaths.c:238](../src/backend/optimizer/path/allpaths.c#L238))
ve:

```c
	/*
	 * Generate access paths for the entire join tree.
	 */
	rel = make_rel_from_joinlist(root, joinlist);
```
— [../src/backend/optimizer/path/allpaths.c:241-244](../src/backend/optimizer/path/allpaths.c#L241-L244)

**Boyut ve yol üretimi neden iki ayrı geçiş?**

```
 * We do this in a separate pass over the base rels so that rowcount
 * estimates are available for parameterized path generation, and also so
 * that each rel's consider_parallel flag is set correctly before we begin to
 * generate paths.
```
— [../src/backend/optimizer/path/allpaths.c:302-305](../src/backend/optimizer/path/allpaths.c#L302-L305)

Yol üretimi tabloya iner
([../src/backend/optimizer/path/allpaths.c:838](../src/backend/optimizer/path/allpaths.c#L838)):

```c
	/* Consider sequential scan */
	add_path(rel, create_seqscan_path(root, rel, required_outer, 0));

	/* If appropriate, consider parallel sequential scan */
	if (rel->consider_parallel && required_outer == NULL)
		create_plain_partial_paths(root, rel);

	/* Consider index scans */
	create_index_paths(root, rel);
```
— [../src/backend/optimizer/path/allpaths.c:860-868](../src/backend/optimizer/path/allpaths.c#L860-L868)

**Seq scan her zaman üretilir**, index yolu bulunsa bile — bir tabloyu okumanın
en az bir yolu olmalıdır, yoksa planlayıcı çuvallar.

---

# 5. Tarama maliyetleri

## 5.1 `cost_seqscan`

[../src/backend/optimizer/path/costsize.c:271](../src/backend/optimizer/path/costsize.c#L271):

```c
	disk_run_cost = spc_seq_page_cost * baserel->pages;

	/* CPU costs */
	get_restriction_qual_cost(root, baserel, param_info, &qpqual_cost);

	startup_cost += qpqual_cost.startup;
	cpu_per_tuple = cpu_tuple_cost + qpqual_cost.per_tuple;
	cpu_run_cost = cpu_per_tuple * baserel->tuples;
```
— [../src/backend/optimizer/path/costsize.c:300-307](../src/backend/optimizer/path/costsize.c#L300-L307)

```c
	path->startup_cost = startup_cost;
	path->total_cost = startup_cost + cpu_run_cost + disk_run_cost;
```
— [../src/backend/optimizer/path/costsize.c:338-339](../src/backend/optimizer/path/costsize.c#L338-L339)

Tek satırda:
`total = seq_page_cost × pages + (cpu_tuple_cost + qual) × tuples`.

**`tuples`, `rows` değil.** Seq scan bütün tuple'ları okur ve her birine qual
uygular; WHERE ne kadar seçici olursa olsun maliyet düşmez, sadece `path->rows`
düşer. Startup cost sıfıra yakındır — `LIMIT 1` sorgularında seq scan sürpriz
biçimde kazanabilir.

Paralellikte CPU bölünür, disk bölünmez:

```c
	if (path->parallel_workers > 0)
	{
		double		parallel_divisor = get_parallel_divisor(path);

		/* The CPU cost is divided among all the workers. */
		cpu_run_cost /= parallel_divisor;
```
— [../src/backend/optimizer/path/costsize.c:313-318](../src/backend/optimizer/path/costsize.c#L313-L318)

## 5.2 `cost_index` — korelasyon ve Mackert–Lohman

[../src/backend/optimizer/path/costsize.c:546](../src/backend/optimizer/path/costsize.c#L546).
Index'in kendi maliyeti access method'a devredilir (`btcostestimate`,
`gincostestimate`, ...):

```c
	amcostestimate = (amcostestimate_function) index->amcostestimate;
	amcostestimate(root, path, loop_count,
				   &indexStartupCost, &indexTotalCost,
				   &indexSelectivity, &indexCorrelation,
				   &index_pages);
```
— [../src/backend/optimizer/path/costsize.c:617-621](../src/backend/optimizer/path/costsize.c#L617-L621)

Heap sıçramaları için iki uç senaryo hesaplanır: korelasyonsuz durumda
Mackert–Lohman yaklaşımı + `random_page_cost`; tam korelasyonlu durumda ise

```
	 * When the index ordering is exactly correlated with the table ordering
	 * (just after a CLUSTER, for example), the number of pages fetched should
	 * be exactly selectivity * table_size.  What's more, all but the first
	 * will be sequential fetches, not the random fetches that occur in the
	 * uncorrelated case.  So if the number of pages is more than 1, we
```
— [../src/backend/optimizer/path/costsize.c:651-655](../src/backend/optimizer/path/costsize.c#L651-L655)

Aradaki geçiş:

```c
	csquared = indexCorrelation * indexCorrelation;

	run_cost += max_IO_cost + csquared * (min_IO_cost - max_IO_cost);
```
— [../src/backend/optimizer/path/costsize.c:785-787](../src/backend/optimizer/path/costsize.c#L785-L787)

`indexCorrelation` `pg_stats.correlation`'dır. **Karesi alınır**: korelasyon 0.7
olsa bile ağırlık 0.49'dur — model kararsız korelasyonlara kuşkuyla yaklaşır.
Pratik sonuç: `CLUSTER` edilmiş veya sıralı yüklenmiş bir tabloda aynı index kat
kat ucuza gelir.

Index-only scan indirimi:

```c
		if (indexonly)
			pages_fetched = ceil(pages_fetched * (1.0 - baserel->allvisfrac));
```
— [../src/backend/optimizer/path/costsize.c:725-726](../src/backend/optimizer/path/costsize.c#L725-L726)

`allvisfrac` visibility map'ten gelir. VACUUM çalışmamışsa 0'dır ve index-only
scan hiçbir avantaj kazanamaz — "index-only scan neden heap fetch yapıyor"
sorusunun cevabı hemen her zaman budur.

## 5.3 `cost_sort` / `cost_tuplesort`

[../src/backend/optimizer/path/costsize.c:2202](../src/backend/optimizer/path/costsize.c#L2202)
işi [../src/backend/optimizer/path/costsize.c:1952](../src/backend/optimizer/path/costsize.c#L1952)
`cost_tuplesort`'a devreder. Üç senaryo:

**Bellekte quicksort** — `N log₂ N` karşılaştırma:

```c
	else
	{
		/* We'll use plain quicksort on all the input tuples */
		*startup_cost = comparison_cost * tuples * LOG2(tuples);
	}
```
— [../src/backend/optimizer/path/costsize.c:2024-2028](../src/backend/optimizer/path/costsize.c#L2024-L2028)

`comparison_cost` varsayılanı `2.0 * cpu_operator_cost`
([../src/backend/optimizer/path/costsize.c:1970](../src/backend/optimizer/path/costsize.c#L1970)).

**Diske taşan sort** — `work_mem`'in maliyete girdiği yer:

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
maliyeti yarılanır.

**Sınırlı (top-N) heapsort** — `LIMIT` varsa `N log₂ K`:

```c
		 * We'll use a bounded heap-sort keeping just K tuples in memory, for
		 * a total number of tuple comparisons of N log2 K; but the constant
		 * factor is a bit higher than for quicksort.  Tweak it so that the
		 * cost curve is continuous at the crossover point.
		 */
		*startup_cost = comparison_cost * tuples * LOG2(2.0 * output_tuples);
```
— [../src/backend/optimizer/path/costsize.c:2017-2022](../src/backend/optimizer/path/costsize.c#L2017-L2022)

Sort'un tüm maliyeti `startup_cost`'a yazılır
([../src/backend/optimizer/path/costsize.c:2226](../src/backend/optimizer/path/costsize.c#L2226)) —
ilk satırı vermeden önce her şeyin sıralanması gerekir.

---

# 6. Seçicilik (selectivity)

## 6.1 `clauselist_selectivity` — bağımsızlık varsayımı

[../src/backend/optimizer/path/clausesel.c:100](../src/backend/optimizer/path/clausesel.c#L100).
Yorum yaklaşımı ve zayıflığını birlikte itiraf ediyor:

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
Çözüm `CREATE STATISTICS`; kod önce onu dener
([../src/backend/optimizer/path/clausesel.c:144-155](../src/backend/optimizer/path/clausesel.c#L144-L155)),
kalanları tek tek hesaplayıp çarpar
([../src/backend/optimizer/path/clausesel.c:183-184](../src/backend/optimizer/path/clausesel.c#L183-L184)).

Tek özel durum **range query**'dir — `x > 34 AND x < 42` çifti yakalanır:

```
 * We also recognize "range queries", such as "x > 34 AND x < 42".  Clauses
 * are recognized as possible range query components if they are restriction
 * opclauses whose operators have scalarltsel or a related function as their
 * restriction selectivity estimator.  We pair up clauses of this form that
 * refer to the same variable.  An unpairable clause of this kind is simply
 * multiplied into the selectivity product in the normal way.  But when we
 * find a pair, we know that the selectivities represent the relative
 * positions of the low and high bounds within the column's range, so instead
 * of figuring the selectivity as hisel * losel, we can figure it as hisel +
 * losel - 1.  (To visualize this, see that hisel is the fraction of the range
```
— [../src/backend/optimizer/path/clausesel.c:73-82](../src/backend/optimizer/path/clausesel.c#L73-L82)

## 6.2 `restriction_selectivity` — operatörün kendi tahmincisi

`clause_selectivity_ext` bir operatör ifadesini join mi restriction mı olduğuna
göre ayırıp `join_selectivity` veya `restriction_selectivity` çağırır
([../src/backend/optimizer/path/clausesel.c:836-852](../src/backend/optimizer/path/clausesel.c#L836-L852)).
İkincisi tahminciyi katalogdan bulur
([../src/backend/optimizer/util/plancat.c:2225](../src/backend/optimizer/util/plancat.c#L2225)):

```c
	RegProcedure oprrest = get_oprrest(operatorid);
	float8		result;

	/*
	 * if the oprrest procedure is missing for whatever reason, use a
	 * selectivity of 0.5
	 */
	if (!oprrest)
		return (Selectivity) 0.5;
```
— [../src/backend/optimizer/util/plancat.c:2231-2239](../src/backend/optimizer/util/plancat.c#L2231-L2239)

`pg_operator.oprrest`: `=` için `eqsel`, `<` için `scalarltsel`, `>` için
`scalargtsel`. Kendi operatörünü yazan extension buraya kendi tahmincisini takar.

## 6.3 `eqsel` → `var_eq_const` — MCV listesi

[../src/backend/utils/adt/selfuncs.c:302](../src/backend/utils/adt/selfuncs.c#L302) →
[../src/backend/utils/adt/selfuncs.c:370](../src/backend/utils/adt/selfuncs.c#L370).
Değer MCV listesindeyse cevap kesindir:

```c
			/*
			 * Constant is "=" to this common value.  We know selectivity
			 * exactly (or as exactly as ANALYZE could calculate it, anyway).
			 */
			selec = sslot.numbers[i];
```
— [../src/backend/utils/adt/selfuncs.c:474-478](../src/backend/utils/adt/selfuncs.c#L474-L478)

Değilse kalan kütle eşit dağıtılır:

```c
			/*
			 * and in fact it's probably a good deal less. We approximate that
			 * all the not-common values share this remaining fraction
			 * equally, so we divide by the number of other distinct values.
			 */
			otherdistinct = get_variable_numdistinct(vardata, &isdefault) -
				sslot.nnumbers;
			if (otherdistinct > 1)
				selec /= otherdistinct;
```
— [../src/backend/utils/adt/selfuncs.c:495-503](../src/backend/utils/adt/selfuncs.c#L495-L503)

Yani `(1 − MCV toplamı − null oranı) / (n_distinct − MCV sayısı)`. Buradaki
`n_distinct` örneklemeye dayalıdır ve büyük tablolarda sistematik olarak düşük
tahmin edilir — `ALTER TABLE ... SET (n_distinct = ...)` bunun için vardır.

## 6.4 `scalarltsel` → `scalarineqsel` — MCV + histogram

[../src/backend/utils/adt/selfuncs.c:1546](../src/backend/utils/adt/selfuncs.c#L1546) →
[../src/backend/utils/adt/selfuncs.c:655](../src/backend/utils/adt/selfuncs.c#L655).
`mcv_selectivity` ve `ineq_histogram_selectivity` sonuçları şöyle birleştirilir:

```c
	selec = 1.0 - stats->stanullfrac - sumcommon;

	if (hist_selec >= 0.0)
		selec *= hist_selec;
```
— [../src/backend/utils/adt/selfuncs.c:773-776](../src/backend/utils/adt/selfuncs.c#L773-L776)

ve sonuna `selec += mcv_selec` eklenir
([../src/backend/utils/adt/selfuncs.c:786](../src/backend/utils/adt/selfuncs.c#L786)).
Sütun üç parçaya bölünmüştür — NULL'lar, MCV'ler, kalanlar — ve histogram
**sadece üçüncüsünü** kapsar:

```
 * Note that the result disregards both the most-common-values (if any) and
 * null entries.  The caller is expected to combine this result with
 * statistics for those portions of the column population.
```
— [../src/backend/utils/adt/selfuncs.c:1109-1111](../src/backend/utils/adt/selfuncs.c#L1109-L1111)

Histogram kutu sayısı `default_statistics_target` (100) ile belirlenir.
`ALTER TABLE ... SET STATISTICS 1000` hem MCV listesini hem histogramı 10 kat
büyütür — çarpık dağılımlarda en etkili tek müdahaledir.

## 6.5 İstatistik yoksa: varsayılanlar

[../src/include/utils/selfuncs.h:34-56](../src/include/utils/selfuncs.h#L34-L56):

| Sabit | Değer | Kullanım |
|---|---|---|
| `DEFAULT_EQ_SEL` | `0.005` | `=`, istatistik yok |
| `DEFAULT_INEQ_SEL` | `0.3333...` | `<`, `>`, istatistik yok |
| `DEFAULT_RANGE_INEQ_SEL` | `0.005` | Eşleşen range query çifti |
| `DEFAULT_NUM_DISTINCT` | `200` | `n_distinct` bilinmiyor |

`EXPLAIN`'de `rows` tablo satırının tam 1/200'ü veya 1/3'ü çıkıyorsa o sütunda
istatistik yok demektir. Hızlı bir teşhis işareti.

## 6.6 Join seçiciliği — `eqjoinsel`

[../src/backend/utils/adt/selfuncs.c:2387](../src/backend/utils/adt/selfuncs.c#L2387).
İki tarafta da MCV listesi yoksa:

```c
		selec = (1.0 - nullfrac1) * (1.0 - nullfrac2);
		if (nd1 > nd2)
			selec /= nd1;
		else
			selec /= nd2;
```
— [../src/backend/utils/adt/selfuncs.c:2730-2734](../src/backend/utils/adt/selfuncs.c#L2730-L2734)

`MIN(1/nd1, 1/nd2) × (1−null1) × (1−null2)`. Büyük `n_distinct`'e sahip taraf
bölen olur, çünkü bu bir üst sınır verir. Pratik sonuç: **join'in çıktı satır
sayısı doğrudan `n_distinct` tahminine bağlıdır** ve hata üst seviyelere
çarpımsal taşınır.

---

# 7. Kardinalite tahmini

## 7.1 Taban ilişki

[../src/backend/optimizer/path/costsize.c:5516](../src/backend/optimizer/path/costsize.c#L5516).
Bütün fonksiyon `rows = tuples × selectivity`:

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

## 7.2 Join ilişkisi

[../src/backend/optimizer/path/costsize.c:5668](../src/backend/optimizer/path/costsize.c#L5668)
`calc_joinrel_size_estimate`. Temel fikir "kartezyen çarpım × seçicilik", ama
join tipine göre taban/tavan uygulanır:

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
		...
			/* pselec not used */
			break;
		case JOIN_ANTI:
			nrows = outer_rows * (1.0 - fkselec * jselec);
			nrows *= pselec;
			break;
```
— [../src/backend/optimizer/path/costsize.c:5766-5791](../src/backend/optimizer/path/costsize.c#L5766-L5791)

Okunacak şeyler:

- **LEFT JOIN çıktısı outer'dan küçük olamaz** — eşleşmeyen her outer satırı
  NULL genişletilmiş çıkar. `if (nrows < outer_rows)` kelepçesi budur.
- **SEMI JOIN inner ile çarpılmaz** (`nrows = outer_rows * fkselec * jselec`) —
  `EXISTS` "kaç outer satırın eşi var" sorusudur.
- **ANTI JOIN tamamlayıcıdır** — `1.0 - selectivity`.
- **Outer join'de iki seçicilik var**: `jselec` (ON koşulları) ve `pselec`
  (dışarıdan itilmiş WHERE); pushed-down qual'lar join'den *sonra* çalıştığı için
  ayrı hesaplanır
  ([../src/backend/optimizer/path/costsize.c:5709-5713](../src/backend/optimizer/path/costsize.c#L5709-L5713)).

## 7.3 `fkselec` — foreign key kısayolu

`jselec`'ten önce FK'ler kontrol edilir
([../src/backend/optimizer/path/costsize.c:5698-5702](../src/backend/optimizer/path/costsize.c#L5698-L5702)):

```
 * The reason for treating such clauses specially is that we can get better
 * estimates this way than by relying on clauselist_selectivity(), especially
 * for multi-column FKs where that function's assumption that the clauses are
 * independent falls down badly.  But even with single-column FKs, we may be
 * able to get a better answer when the pg_statistic stats are missing or out
 * of date.
```
— [../src/backend/optimizer/path/costsize.c:5810-5815](../src/backend/optimizer/path/costsize.c#L5810-L5815)

FK varlığı "her child satırın tam bir parent'ı var" bilgisini kesin olarak verir.
Bu, FK tanımlamanın az bilinen performans etkisidir. Örnek sorgumuzda
`orders.user_id → users.id` FK'si varsa join kardinalitesi bu yoldan gelir.

---

# 8. `add_path` ve baskınlık (dominance)

Planlayıcının kalbi:
[../src/backend/optimizer/util/pathnode.c:459](../src/backend/optimizer/util/pathnode.c#L459).

## 8.1 Kural

```
 *	  A path is worthy if it has a better sort order (better pathkeys) or
 *	  cheaper cost (as defined below), or generates fewer rows, than any
 *    existing path that has the same or superset parameterization rels.  We
 *    also consider parallel-safe paths more worthy than others.
```
— [../src/backend/optimizer/util/pathnode.c:391-394](../src/backend/optimizer/util/pathnode.c#L391-L394)

```
 *	  In addition to possibly adding new_path, we also remove from the rel's
 *    pathlist any old paths that are dominated by new_path --- that is,
 *    new_path is cheaper, at least as well ordered, generates no more rows,
 *    requires no outer rels not required by the old path, and is no less
 *    parallel-safe.
```
— [../src/backend/optimizer/util/pathnode.c:408-412](../src/backend/optimizer/util/pathnode.c#L408-L412)

Beş eksen birden:

```
                new_path  vs  old_path
   ┌──────────────────────────────────────────────┐
   │ 1. disabled_nodes + startup/total cost       │  compare_path_costs_fuzzily
   │ 2. pathkeys (sıralama)                       │  compare_pathkeys
   │ 3. required_outer (parameterization)         │  bms_subset_compare
   │ 4. rows (satır sayısı)                       │  düz karşılaştırma
   │ 5. parallel_safe                             │  bool
   └──────────────────────────────────────────────┘
   Hiçbirinde geride değil + en az birinde önde → domine eder, ötekini atar
```

Kodun `COSTS_EQUAL` dalı beşliyi bir arada gösteriyor:

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

## 8.2 Bulanık maliyet karşılaştırması

[../src/backend/optimizer/util/pathnode.c:181](../src/backend/optimizer/util/pathnode.c#L181)
`compare_path_costs_fuzzily`. İlk kontrol maliyet değil:

```c
	/* Number of disabled nodes, if different, trumps all else. */
	if (unlikely(path1->disabled_nodes != path2->disabled_nodes))
	{
		if (path1->disabled_nodes < path2->disabled_nodes)
			return COSTS_BETTER1;
```
— [../src/backend/optimizer/util/pathnode.c:186-190](../src/backend/optimizer/util/pathnode.c#L186-L190)

`disabled_nodes` maliyetin üst mertebeden bileşenidir:

```
 * Each path stores the total number of disabled nodes that exist at or
 * below that point in the plan tree. This is regarded as a component of
 * the cost, and paths with fewer disabled nodes should be regarded as
 * cheaper than those with more. Disabled nodes occur when the user sets
```
— [../src/backend/optimizer/path/costsize.c:53-56](../src/backend/optimizer/path/costsize.c#L53-L56)

Sonra maliyet, ama **%1 toleransla**:

```c
#define STD_FUZZ_FACTOR 1.01
```
— [../src/backend/optimizer/util/pathnode.c:47](../src/backend/optimizer/util/pathnode.c#L47)

Startup ve total birlikte bakılır; biri birinde diğeri ötekinde öndeyse sonuç
`COSTS_DIFFERENT` olur ve **ikisi de saklanır**
([../src/backend/optimizer/util/pathnode.c:163-170](../src/backend/optimizer/util/pathnode.c#L163-L170)).

## 8.3 `disabled_nodes` nasıl işaretlenir — `pgs_mask`

Bu ağaçta `enable_*` GUC'ları doğrudan okunmuyor; planlama başında bit maskesine
çevriliyor:

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
([../src/backend/optimizer/util/relnode.c:234](../src/backend/optimizer/util/relnode.c#L234),
[840](../src/backend/optimizer/util/relnode.c#L840)) ve maliyet fonksiyonlarında
kontrol edilir:

```c
	path->disabled_nodes =
		(baserel->pgs_mask & enable_mask) == enable_mask ? 0 : 1;
```
— [../src/backend/optimizer/path/costsize.c:336-337](../src/backend/optimizer/path/costsize.c#L336-L337)

Bit tanımları
[../src/include/nodes/pathnodes.h:66-84](../src/include/nodes/pathnodes.h#L66-L84).
Amaç extension'lara *ilişki bazında* strateji kısıtlama imkânı vermek — GUC
global, `pgs_mask` yereldir
([../src/include/nodes/pathnodes.h:29-33](../src/include/nodes/pathnodes.h#L29-L33)).

## 8.4 `add_path_precheck` ve `set_cheapest`

Join yolları pahalı hesaplandığı için önce kaba bir alt sınırla elenir
([../src/backend/optimizer/util/pathnode.c:686](../src/backend/optimizer/util/pathnode.c#L686)):

```c
		if (unlikely(old_path->disabled_nodes != disabled_nodes))
		{
			if (disabled_nodes < old_path->disabled_nodes)
				break;
		}
		else if (total_cost <= old_path->total_cost * STD_FUZZ_FACTOR)
			break;
```
— [../src/backend/optimizer/util/pathnode.c:711-717](../src/backend/optimizer/util/pathnode.c#L711-L717)

`pathlist`'in sıralı tutulmasının asıl sebebi bu erken çıkıştır.

Bütün yollar üretilince
[../src/backend/optimizer/util/pathnode.c:268](../src/backend/optimizer/util/pathnode.c#L268)
`set_cheapest` çağrılır; `cheapest_total_path` normalde en ucuz
*parametrelenmemiş* yoldur, hiç yoksa en az parametrelenmiş olan seçilir
([../src/backend/optimizer/util/pathnode.c:245-248](../src/backend/optimizer/util/pathnode.c#L245-L248)).

---

# 9. Join arama: dinamik programlama

## 9.1 Üç yollu dallanma

[../src/backend/optimizer/path/allpaths.c:3847](../src/backend/optimizer/path/allpaths.c#L3847)
`make_rel_from_joinlist`:

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
([../src/backend/utils/misc/guc_parameters.dat:1187-1194](../src/backend/utils/misc/guc_parameters.dat#L1187-L1194)).

## 9.2 `standard_join_search`

[../src/backend/optimizer/path/allpaths.c:3952](../src/backend/optimizer/path/allpaths.c#L3952):

```c
	root->join_rel_level = (List **) palloc0((levels_needed + 1) * sizeof(List *));

	root->join_rel_level[1] = initial_rels;

	for (lev = 2; lev <= levels_needed; lev++)
```
— [../src/backend/optimizer/path/allpaths.c:3974-3978](../src/backend/optimizer/path/allpaths.c#L3974-L3978)

Her seviyede `join_search_one_level` çalışır, sonra o seviyedeki her joinrel için
`set_cheapest` çağrılır
([../src/backend/optimizer/path/allpaths.c:4023](../src/backend/optimizer/path/allpaths.c#L4023)).
Bu sıra keyfî değil:

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

## 9.3 `join_search_one_level`

[../src/backend/optimizer/path/joinrels.c:78](../src/backend/optimizer/path/joinrels.c#L78).
Üç aşama.

**(a) Sol/sağ dallı planlar** — önceki seviyenin rel'leri × tek öğeler:

```c
	foreach(r, joinrels[level - 1])
	{
		RelOptInfo *old_rel = (RelOptInfo *) lfirst(r);

		if (old_rel->joininfo != NIL || old_rel->has_eclass_joins ||
			has_join_restriction(root, old_rel))
```
— [../src/backend/optimizer/path/joinrels.c:96-101](../src/backend/optimizer/path/joinrels.c#L96-L101)

Join clause'u olmayan rel `make_rels_by_clauseless_joins` ile kartezyen çarpıma
zorlanır
([../src/backend/optimizer/path/joinrels.c:125-142](../src/backend/optimizer/path/joinrels.c#L125-L142)).

**(b) Bushy planlar** — k öğeli × (level−k) öğeli:

```c
	 * We only consider bushy-plan joins for pairs of rels where there is a
	 * suitable join clause (or join order restriction), in order to avoid
	 * unreasonable growth of planning time.
	 */
	for (k = 2;; k++)
```
— [../src/backend/optimizer/path/joinrels.c:149-153](../src/backend/optimizer/path/joinrels.c#L149-L153)

**(c) Son çare** — hiçbir birleşim çıkmadıysa kartezyen çarpım zorlanır
([../src/backend/optimizer/path/joinrels.c:224-236](../src/backend/optimizer/path/joinrels.c#L224-L236)).

Her aday çift
[../src/backend/optimizer/path/joinrels.c:699](../src/backend/optimizer/path/joinrels.c#L699)
`make_join_rel`'e gider: `join_is_legal` outer join kısıtlarını kontrol eder,
`build_join_rel` RelOptInfo'yu bulur ya da yaratır (ve
`set_joinrel_size_estimates` çağırır,
[../src/backend/optimizer/util/relnode.c:956](../src/backend/optimizer/util/relnode.c#L956)),
sonra `populate_joinrel_with_paths` yolları üretir
([../src/backend/optimizer/path/joinrels.c:774](../src/backend/optimizer/path/joinrels.c#L774)).

## 9.4 GEQO

12+ FROM öğesinde DP'nin arama uzayı patlar; `geqo/` dizini genetik algoritma
kullanır
([../src/backend/optimizer/geqo/geqo_main.c:75](../src/backend/optimizer/geqo/geqo_main.c#L75)).
Her kromozom bir join sırasıdır, uygunluk o sıranın en ucuz path maliyetidir
([../src/backend/optimizer/geqo/geqo_eval.c:57](../src/backend/optimizer/geqo/geqo_eval.c#L57)
`geqo_eval`,
[../src/backend/optimizer/geqo/geqo_eval.c:163](../src/backend/optimizer/geqo/geqo_eval.c#L163)
`gimme_tree`). Kritik nokta:

```
/util is utility stuff.  /geqo is the separate "genetic optimization" planner
--- it does a semi-random search through the join tree space, rather than
exhaustively considering all possible join trees.  (But each join considered
by /geqo is given to /path to create paths for, so we consider all possible
implementation paths for each specific join pair even in GEQO mode.)
```
— [../src/backend/optimizer/README:10-14](../src/backend/optimizer/README#L10-L14)

Join *yöntemi* seçimi her iki modda aynıdır; farklı olan hangi *sıraların*
denendiğidir. GEQO rastgeledir — aynı sorgu iki kez farklı plan alabilir.

---

# 10. Join yolları ve maliyetleri

## 10.1 `add_paths_to_joinrel`

[../src/backend/optimizer/path/joinpath.c:123](../src/backend/optimizer/path/joinpath.c#L123):

```c
	/*
	 * 1. Consider mergejoin paths where both relations must be explicitly
	 * sorted.  Skip this if we can't mergejoin.
	 */
	if (mergejoin_allowed)
		sort_inner_and_outer(root, joinrel, outerrel, innerrel,
							 jointype, &extra);
	...
	/*
	 * 4. Consider paths where both outer and inner relations must be hashed
	 * before being joined.  As above, when it's a full join, we must try this
	 * even when the path type is disabled, because it may be our only option.
	 */
	if ((extra.pgs_mask & PGS_HASHJOIN) != 0 || jointype == JOIN_FULL)
		hash_inner_and_outer(root, joinrel, outerrel, innerrel,
							 jointype, &extra);
```
— [../src/backend/optimizer/path/joinpath.c:310-355](../src/backend/optimizer/path/joinpath.c#L310-L355)

Aradaki 2. adım `match_unsorted_outer`
([../src/backend/optimizer/path/joinpath.c:1834](../src/backend/optimizer/path/joinpath.c#L1834))
hem sıralı outer'lı mergejoin'leri hem **bütün nested loop'ları** üretir:

```
 * We always generate a nestloop path for each available outer path.
 * In fact we may generate as many as five: one on the cheapest-total-cost
 * inner path, one on the same with materialization, one on the
 * cheapest-startup-cost inner path (if different), one on the
 * cheapest-total inner-indexscan path (if any), and one on the
 * cheapest-startup inner-indexscan path (if different).
```
— [../src/backend/optimizer/path/joinpath.c:1817-1822](../src/backend/optimizer/path/joinpath.c#L1817-L1822)

`FULL JOIN` özeldir: hash veya merge tek seçenektir, o yüzden GUC kapalı olsa
bile denenmelidir.

## 10.2 İki fazlı maliyetlendirme

Her join yöntemi bir `initial_cost_*` / `final_cost_*` çiftine sahiptir:

```
 * This must quickly produce lower-bound estimates of the path's startup and
 * total costs.  If we are unable to eliminate the proposed path from
 * consideration using the lower bounds, final_cost_nestloop will be called
 * to obtain the final estimates.
```
— [../src/backend/optimizer/path/costsize.c:3375-3378](../src/backend/optimizer/path/costsize.c#L3375-L3378)

```
   initial_cost_*      →  ucuz alt sınır (join qual'lara bakmadan)
   add_path_precheck   →  açıkça kötüyse burada elenir
   final_cost_*        →  qual maliyeti + gerçek satır sayısı
   add_path            →  turnuva
```

Akışı `try_nestloop_path` / `try_mergejoin_path` / `try_hashjoin_path` yürütür
([../src/backend/optimizer/path/joinpath.c:878](../src/backend/optimizer/path/joinpath.c#L878)).

## 10.3 Nested loop

[../src/backend/optimizer/path/costsize.c:3396](../src/backend/optimizer/path/costsize.c#L3396):

```c
	startup_cost += outer_path->startup_cost + inner_path->startup_cost;
	run_cost += outer_path->total_cost - outer_path->startup_cost;
	if (outer_path_rows > 1)
		run_cost += (outer_path_rows - 1) * inner_rescan_start_cost;
```
— [../src/backend/optimizer/path/costsize.c:3428-3431](../src/backend/optimizer/path/costsize.c#L3428-L3431)

```c
		/* Normal case; we'll scan whole input rel for each outer row */
		run_cost += inner_run_cost;
		if (outer_path_rows > 1)
			run_cost += (outer_path_rows - 1) * inner_rescan_run_cost;
```
— [../src/backend/optimizer/path/costsize.c:3453-3456](../src/backend/optimizer/path/costsize.c#L3453-L3456)

Inner, outer satır sayısı kadar taranır. `cost_rescan`
([../src/backend/optimizer/path/costsize.c:4808](../src/backend/optimizer/path/costsize.c#L4808))
sonraki taramaların ucuzlayabileceğini modeller (Material düğümü veya index
scan). `SEMI`/`ANTI`'de ve inner unique olduğunda ilk eşleşmede durulur
([../src/backend/optimizer/path/costsize.c:3436-3449](../src/backend/optimizer/path/costsize.c#L3436-L3449)).

CPU maliyeti `final_cost_nestloop`'ta, **işlenen** tuple sayısı üzerinden eklenir
— `ntuples = outer_rows × inner_rows`
([../src/backend/optimizer/path/costsize.c:3631](../src/backend/optimizer/path/costsize.c#L3631),
[3635-3638](../src/backend/optimizer/path/costsize.c#L3635-L3638)). Nested
loop'un tek gerçek avantajı düşük startup cost'tur; `O(N×M)` terimi onu hızla
eler — **inner parametrelenmiş bir index scan değilse** (Bölüm 11).

## 10.4 Hash join

[../src/backend/optimizer/path/costsize.c:4320](../src/backend/optimizer/path/costsize.c#L4320):

```c
	/* cost of source data */
	startup_cost += outer_path->startup_cost;
	run_cost += outer_path->total_cost - outer_path->startup_cost;
	startup_cost += inner_path->total_cost;
```
— [../src/backend/optimizer/path/costsize.c:4348-4351](../src/backend/optimizer/path/costsize.c#L4348-L4351)

Son satır hash join'in karakteridir: **inner tarafın tamamı ilk çıktı satırından
önce ödenir.** Hash hesaplama ücreti:

```c
	startup_cost += (cpu_operator_cost * num_hashclauses + cpu_tuple_cost)
		* inner_path_rows;
	run_cost += cpu_operator_cost * num_hashclauses * outer_path_rows;
```
— [../src/backend/optimizer/path/costsize.c:4363-4365](../src/backend/optimizer/path/costsize.c#L4363-L4365)

Batch sayısı executor'ın kendi mantığından alınır (`ExecChooseHashTableSize`,
[../src/backend/optimizer/path/costsize.c:4386-4394](../src/backend/optimizer/path/costsize.c#L4386-L4394));
birden fazlaysa disk ücreti eklenir:

```c
		startup_cost += seq_page_cost * innerpages;
		run_cost += seq_page_cost * (innerpages + 2 * outerpages);
```
— [../src/backend/optimizer/path/costsize.c:4410-4411](../src/backend/optimizer/path/costsize.c#L4410-L4411)

`final_cost_hashjoin` bucket doluluğunu tahmin eder
([../src/backend/optimizer/path/costsize.c:4525-4572](../src/backend/optimizer/path/costsize.c#L4525-L4572))
ve karşılaştırma sayısını `outer_rows × inner_rows × innerbucketsize × 0.5`
olarak fiyatlar
([../src/backend/optimizer/path/costsize.c:4661-4663](../src/backend/optimizer/path/costsize.c#L4661-L4663)).
Çarpık dağılıma karşı bir güvenlik freni vardır — tek bir değerin bucket'ı
`hash_mem`'i aşacaksa:

```c
	if (relation_byte_size(clamp_row_est(inner_path_rows * innermcvfreq),
						   inner_path->pathtarget->width) > get_hash_memory_limit())
		startup_cost += disable_cost;
```
— [../src/backend/optimizer/path/costsize.c:4585-4587](../src/backend/optimizer/path/costsize.c#L4585-L4587)

## 10.5 Merge join

[../src/backend/optimizer/path/costsize.c:3681](../src/backend/optimizer/path/costsize.c#L3681).
Girdilerin sıralı olmasını gerektirir; sıralama ya hazır pathkey'den gelir ya da
açık bir sort eklenir.

Ayırt edici ayrıntı: merge join girdilerin tamamını okumak zorunda değildir.
`mergejoinscansel` iki tarafın örtüşen aralığını tahmin eder
([../src/backend/optimizer/path/costsize.c:3746](../src/backend/optimizer/path/costsize.c#L3746)),
sonuç satır sayılarına çevrilir:

```c
	outer_skip_rows = rint(outer_path_rows * outerstartsel);
	inner_skip_rows = rint(inner_path_rows * innerstartsel);
	outer_rows = clamp_row_est(outer_path_rows * outerendsel);
	inner_rows = clamp_row_est(inner_path_rows * innerendsel);
```
— [../src/backend/optimizer/path/costsize.c:3790-3793](../src/backend/optimizer/path/costsize.c#L3790-L3793)

A'nın değerleri 1–100, B'ninkiler 5000–6000 ise iki taraf da erken durur — hash
join'de böyle bir avantaj yoktur.

İkinci ayrıntı: outer'daki tekrar eden anahtarlar inner'ın yeniden taranmasını
gerektirir. Tahmin "join çıktısı − inner satır sayısı" olarak yapılır
([../src/backend/optimizer/path/costsize.c:4058-4068](../src/backend/optimizer/path/costsize.c#L4058-L4068))
ve tek orana çevrilir:

```c
	 * We'll inflate various costs this much to account for rescanning.  Note
	 * that this is to be multiplied by something involving inner_rows, or
	 * another number related to the portion of the inner rel we'll scan.
	 */
	rescanratio = 1.0 + (rescannedtuples / inner_rows);
```
— [../src/backend/optimizer/path/costsize.c:4092-4096](../src/backend/optimizer/path/costsize.c#L4092-L4096)

Oran yüksekse inner'a Material düğümü eklemek ucuza gelebilir:

```c
	else if ((extra->pgs_mask & PGS_MERGEJOIN_MATERIALIZE) != 0 &&
			 (mat_inner_cost < bare_inner_cost ||
			  (extra->pgs_mask & PGS_MERGEJOIN_PLAIN) == 0))
		path->materialize_inner = true;
```
— [../src/backend/optimizer/path/costsize.c:4136-4139](../src/backend/optimizer/path/costsize.c#L4136-L4139)

`EXPLAIN`'de merge join altındaki `Materialize` düğümünün açıklaması budur. CPU
maliyeti `N + M + rescan` karşılaştırmasına dayanır:

```c
	startup_cost += merge_qual_cost.per_tuple *
		(outer_skip_rows + inner_skip_rows * rescanratio);
	run_cost += merge_qual_cost.per_tuple *
		((outer_rows - outer_skip_rows) +
		 (inner_rows - inner_skip_rows) * rescanratio);
```
— [../src/backend/optimizer/path/costsize.c:4210-4214](../src/backend/optimizer/path/costsize.c#L4210-L4214)

**Üç yöntemin özeti:**

| | Startup | Run | Sıralı çıktı | Kazandığı yer |
|---|---|---|---|---|
| Nested loop | Çok düşük | `O(N × M)` | Outer'ınki korunur | Küçük outer + parametrelenmiş inner |
| Hash join | Inner'ın tamamı | `O(N)` | Yok | Büyük eşitlik join'leri, bellek yeterli |
| Merge join | Sort gerekiyorsa yüksek | `O(N + M)` | Var | Girdiler zaten sıralı; büyük × büyük |

---

# 11. Parametrelenmiş yollar (parameterized paths)

## 11.1 Problem

```
We can make this better by using a merge or hash join, but it still
requires scanning all of both input relations.  If A is very small and B is
very large, but there is an index on B.Y, it can be enormously better to do
something like this:

	NestLoop
		-> Seq Scan on A
		-> Index Scan using B_Y_IDX on B
			Index Condition: B.Y = A.X
```
— [../src/backend/optimizer/README:1070-1078](../src/backend/optimizer/README#L1070-L1078)

Nested loop'un tek gerçek kullanım alanı budur: inner'ın outer'dan gelen değeri
**index anahtarı** olarak kullanabilmesi.

## 11.2 Temsil

Path'e `param_info` alanı eklenir
([../src/include/nodes/pathnodes.h:1918](../src/include/nodes/pathnodes.h#L1918)):

```c
	Relids		ppi_req_outer;	/* rels supplying parameters used by path */
	Cardinality ppi_rows;		/* estimated number of result tuples */
	List	   *ppi_clauses;	/* join clauses available from outer rels */
```
— [../src/include/nodes/pathnodes.h:1924-1926](../src/include/nodes/pathnodes.h#L1924-L1926)

`ppi_req_outer` = "bu yol çalışmak için şu rel'lerden değer bekliyor". Erişim
makrosu:

```c
#define PATH_REQ_OUTER(path)  \
	((path)->param_info ? (path)->param_info->ppi_req_outer : (Relids) NULL)
```
— [../src/include/nodes/pathnodes.h:2015-2016](../src/include/nodes/pathnodes.h#L2015-L2016)

Aynı parametrelenmeye sahip bütün yollar tek `ParamPathInfo`'yu paylaşır — böylece
satır tahmini hepsinde aynı olur
([../src/include/nodes/pathnodes.h:1901-1905](../src/include/nodes/pathnodes.h#L1901-L1905)).

## 11.3 Neden `rows` ayrı bir ölçüt

```
While all ordinary paths for a particular relation generate the same set
of rows (since they must all apply the same set of restriction clauses),
parameterized paths typically generate fewer rows than less-parameterized
paths, since they have additional clauses to work with.  This means we
must consider the number of rows generated as an additional figure of
merit.  A path that costs more than another, but generates fewer rows,
must be kept since the smaller number of rows might save work at some
```
— [../src/backend/optimizer/README:1154-1160](../src/backend/optimizer/README#L1154-L1160)

`add_path`'in dördüncü ekseninin varlık sebebi budur. Satır tahmini
[../src/backend/optimizer/path/costsize.c:5546](../src/backend/optimizer/path/costsize.c#L5546)
`get_parameterized_baserel_size` ile yapılır: ekstra join clause'lar da
seçiciliğe katılır.

## 11.4 İki politika kararı

**(1) Parametrelenmiş yolun pathkey'i yok sayılır:**

```c
	new_path_pathkeys = new_path->param_info ? NIL : new_path->pathkeys;
```
— [../src/backend/optimizer/util/pathnode.c:473](../src/backend/optimizer/util/pathnode.c#L473)

Gerekçe: yol zaten nested loop'un iç tarafında kullanılacak, orada sıralaması
kimseyi ilgilendirmiyor
([../src/backend/optimizer/README:1190-1199](../src/backend/optimizer/README#L1190-L1199)).

**(2) Aynı parametrelenmedeki yollar aynı satır sayısını üretmelidir:**

```
To keep cost estimation rules relatively simple, we make an implementation
restriction that all paths for a given relation of the same parameterization
(i.e., the same set of outer relations supplying parameters) must have the
same rowcount estimate.  This is justified by insisting that each such path
apply *all* join clauses that are available with the named outer relations.
```
— [../src/backend/optimizer/README:1164-1168](../src/backend/optimizer/README#L1164-L1168)

Bu kısıt `add_path_precheck`'in satır sayısını bilmeden çalışmasını sağlar.

## 11.5 Üretim ve kullanım

Parametrelenmiş taban yolları sadece index scan'ler için üretilir
([../src/backend/optimizer/README:1180-1184](../src/backend/optimizer/README#L1180-L1184)) —
[../src/backend/optimizer/path/indxpath.c:239](../src/backend/optimizer/path/indxpath.c#L239)
`create_index_paths`:

```
 * so it can be applied in any context.  A "parameterized" index scan uses
 * join clauses (plus restriction clauses, if available) in its indexqual.
 * When joining such a scan to one of the relations supplying the other
 * variables used in its indexqual, the parameterized scan must appear as
 * the inner relation of a nestloop join; it can't be used on the outer side,
 * nor in a merge or hash join.  In that context, values for the other rels'
```
— [../src/backend/optimizer/path/indxpath.c:213-218](../src/backend/optimizer/path/indxpath.c#L213-L218)

`try_nestloop_path` parametrelenmenin makul olup olmadığını kontrol edip
gereksizleri reddeder
([../src/backend/optimizer/path/joinpath.c:929-938](../src/backend/optimizer/path/joinpath.c#L929-L938)).

---

# 12. Path → Plan

Kazanan yol `create_plan` ile Plan ağacına çevrilir
([../src/backend/optimizer/plan/createplan.c:345](../src/backend/optimizer/plan/createplan.c#L345),
gövde [357](../src/backend/optimizer/plan/createplan.c#L357)):

```
 *	  The tlists and quals in the plan tree are still in planner format,
 *	  ie, Vars still correspond to the parser's numbering.  This will be
 *	  fixed later by setrefs.c.
```
— [../src/backend/optimizer/plan/createplan.c:336-338](../src/backend/optimizer/plan/createplan.c#L336-L338)

Yani üç aşama: Path (maliyet + yapı) → Plan (executor düğümleri) → `setrefs.c`
sonrası Plan (Var referansları executor slot'larına bağlanmış).

---

# 13. Planlayıcı hook'ları

| Hook | Yer | Ne yapar |
|---|---|---|
| `planner_hook` | [../src/backend/optimizer/plan/planner.c:74](../src/backend/optimizer/plan/planner.c#L74) | Bütün planlamayı sarmalar; `pg_stat_statements` planlama süresini böyle ölçer |
| `planner_setup_hook` | [../src/backend/optimizer/plan/planner.c:77](../src/backend/optimizer/plan/planner.c#L77) | `PlannerGlobal` kurulunca; `default_pgs_mask`'i değiştirebilir |
| `planner_shutdown_hook` | [../src/backend/optimizer/plan/planner.c:80](../src/backend/optimizer/plan/planner.c#L80) | Planlama bitince |
| `set_rel_pathlist_hook` | [../src/backend/optimizer/path/allpaths.c:89](../src/backend/optimizer/path/allpaths.c#L89) | Bir baserel'in yolları üretildikten sonra; custom scan buradan eklenir |
| `set_join_pathlist_hook` | [../src/backend/optimizer/path/joinpath.c:31](../src/backend/optimizer/path/joinpath.c#L31) | Bir joinrel'in yolları üretildikten sonra |
| `join_search_hook` | [../src/backend/optimizer/path/allpaths.c:92](../src/backend/optimizer/path/allpaths.c#L92) | Join arama algoritmasının tamamını değiştirir |
| `build_simple_rel_hook` | [../src/backend/optimizer/util/relnode.c:51](../src/backend/optimizer/util/relnode.c#L51) | Bir baserel'in `RelOptInfo`'su kurulurken |

```c
	if (planner_hook)
		result = (*planner_hook) (parse, query_string, cursorOptions,
								  boundParams, es);
	else
		result = standard_planner(parse, query_string, cursorOptions,
								  boundParams, es);
```
— [../src/backend/optimizer/plan/planner.c:333-338](../src/backend/optimizer/plan/planner.c#L333-L338)

```c
	if (set_rel_pathlist_hook)
		(*set_rel_pathlist_hook) (root, rel, rti, rte);
```
— [../src/backend/optimizer/path/allpaths.c:589-590](../src/backend/optimizer/path/allpaths.c#L589-L590)

`join_search_hook` yazarlarına uyarı:

```
 * Note to plugin authors: the functions invoked during standard_join_search()
 * modify root->join_rel_list and root->join_rel_hash.  If you want to do more
 * than one join-order search, you'll probably need to save and restore the
 * original states of those data structures.  See geqo_eval() for an example.
```
— [../src/backend/optimizer/path/allpaths.c:3946-3949](../src/backend/optimizer/path/allpaths.c#L3946-L3949)

---

# İzleme ve hata ayıklama

## EXPLAIN çıktısını okumak

Biçim
[../src/backend/commands/explain.c:1811](../src/backend/commands/explain.c#L1811):

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
seçiciliğin gerçek değerini verir.

Devre dışı bırakılmış düğüm ayrıca işaretlenir
([../src/backend/commands/explain.c:1893-1895](../src/backend/commands/explain.c#L1893-L1895)):
`Disabled: true` görüyorsan bir `enable_*` GUC'u kapalıydı ama planlayıcı başka
seçenek bulamadı.

## `debug_print_plan`

[../src/backend/utils/misc/guc_parameters.dat:716-717](../src/backend/utils/misc/guc_parameters.dat#L716-L717):

```
{ name => 'debug_print_plan', type => 'bool', context => 'PGC_USERSET', group => 'LOGGING_WHAT',
  short_desc => 'Logs each query\'s execution plan.',
```

```sql
SET debug_print_plan = on;
SET debug_pretty_print = on;
SET client_min_messages = log;
```

`EXPLAIN`'in gizlediği her şeyi döker: targetlist Var'ları, qual ağaçları,
`plan_rows`/`plan_width`, `parallel_aware` bayrakları. Kardeşleri
`debug_print_parse` ve `debug_print_rewritten`. Path düzeyini görmek için kaynak
`OPTIMIZER_DEBUG` ile derlenmelidir
([../src/backend/optimizer/path/allpaths.c:4042-4044](../src/backend/optimizer/path/allpaths.c#L4042-L4044)).

## `enable_*` GUC'ları — teşhis aracı, ayar aracı değil

Tam liste (27 adet)
[../src/backend/utils/misc/guc_parameters.dat:866-1049](../src/backend/utils/misc/guc_parameters.dat#L866-L1049).
Kullanım biçimi:

```sql
EXPLAIN ANALYZE SELECT ...;   -- planlayicinin secimi
SET enable_hashjoin = off;
EXPLAIN ANALYZE SELECT ...;   -- alternatif gercekten daha mi hizli?
RESET enable_hashjoin;
```

Alternatif daha hızlıysa **sorun maliyet modelinde değil, tahmindedir.** Doğru
müdahale GUC'u kapalı bırakmak değil; ANALYZE, `SET STATISTICS`,
`CREATE STATISTICS` veya `random_page_cost` ayarıdır. Bu GUC'lar yolu tamamen
yasaklamaz, sadece `disabled_nodes` sayacını artırır (Bölüm 8.3).

## Tahmin hatası avlamak

```sql
-- 1. Istatistik ne diyor?
SELECT attname, n_distinct, null_frac, correlation,
       array_length(most_common_vals, 1) AS n_mcv,
       array_length(histogram_bounds, 1) AS n_hist
FROM   pg_stats WHERE tablename = 'orders';

-- 2. Planlayicinin gordugu boyut
SELECT relname, relpages, reltuples, relallvisible
FROM   pg_class WHERE relname = 'orders';

-- 3. Son ANALYZE ne zamandi?
SELECT relname, last_analyze, last_autoanalyze, n_mod_since_analyze
FROM   pg_stat_user_tables WHERE relname = 'orders';
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

Planlama süresinin kendisi `EXPLAIN` çıktısındaki `Planning Time`'dır. Çok
tablolu sorgularda yüksekse `join_collapse_limit` / `from_collapse_limit`
(varsayılan 8,
[../src/backend/utils/misc/guc_parameters.dat:1108](../src/backend/utils/misc/guc_parameters.dat#L1108),
[1536](../src/backend/utils/misc/guc_parameters.dat#L1536)) devrededir.

---

# Tek sayfalık özet

```
  ┌───────────────────────────────────────────────────────────────────────┐
  │ İSTATİSTİK   pg_class.relpages/reltuples + pg_statistic (MCV,         │
  │              histogram, n_distinct, correlation) — ANALYZE toplar,    │
  │              estimate_rel_size gerçek dosya boyutuna göre ölçekler    │
  └───────────────────────────────┬───────────────────────────────────────┘
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │ SEÇİCİLİK    clauselist_selectivity → clause_selectivity              │
  │  clausesel.c   → restriction_selectivity (pg_operator.oprrest)        │
  │  selfuncs.c        ├─ eqsel       → MCV eşleşmesi / n_distinct        │
  │                    └─ scalarltsel → MCV + histogram                   │
  │  ⚠ bağımsızlık varsayımı: s1 × s2 × ...  (CREATE STATISTICS düzeltir) │
  └───────────────────────────────┬───────────────────────────────────────┘
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │ KARDİNALİTE  baserel: rows = tuples × selectivity                     │
  │  costsize.c  joinrel: outer × inner × jselec × fkselec                │
  │                       + join tipine göre taban/tavan                  │
  │              clamp_row_est: en az 1, tam sayı                         │
  └───────────────────────────────┬───────────────────────────────────────┘
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │ MALİYET      seq:   pages×1.0 + tuples×(0.01+qual)                    │
  │  costsize.c  index: amcostestimate + csquared interpolasyonu          │
  │                     (min_IO ↔ max_IO, Mackert–Lohman)                 │
  │              sort:  N log N × 2×0.0025, work_mem'i aşarsa diske taşar │
  │              NL:    outer_rows × inner_run_cost                       │
  │              hash:  startup = inner'ın tamamı; run = N                │
  │              merge: N + M, rescanratio ile şişirilir                  │
  └───────────────────────────────┬───────────────────────────────────────┘
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │ SEÇİM        add_path — 5 eksenli baskınlık:                          │
  │  pathnode.c    disabled_nodes → startup/total (fuzz %1)               │
  │                → pathkeys → required_outer → rows → parallel_safe     │
  │              set_cheapest → cheapest_total_path                       │
  └───────────────────────────────┬───────────────────────────────────────┘
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │ ARAMA        standard_join_search: seviye 2, 3, ..., N                │
  │  allpaths.c    join_search_one_level: sol/sağ dallı + bushy           │
  │  joinrels.c  ≥ geqo_threshold (12) → GEQO                             │
  │  geqo/       join_search_hook → extension                             │
  └───────────────────────────────┬───────────────────────────────────────┘
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
