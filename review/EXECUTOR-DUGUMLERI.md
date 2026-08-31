# Executor düğümleri: Sort, Hash Join, Aggregate ve arkadaşları

*PostgreSQL **20devel** ağacı. Satır numaraları bu sürüme aittir ([meson.build:11](../meson.build#L11)).*

[SELECT-NASIL-CALISIR.md](SELECT-NASIL-CALISIR.md) tek bir `SeqScan`'in nasıl satır ürettiğini izliyor. Bu
not resmin geri kalanı — **satırı üretmeden önce iş yapmak zorunda olan** düğümler. Sort tüm girdiyi
okumadan tek satır veremez; Hash Join önce tablo kurar; Aggregate grup bitene kadar bekler. Hepsi aynı
`ExecProcNode` arayüzünün arkasında durur, ama içleri birer durum makinesidir ve `work_mem` bittiğinde diske
taşarlar.

---

## 30 saniyelik özet

```
   ExecutePlan ──ExecProcNode(planstate)──► her çağrı 1 tuple
        ┌──────────┐
        │ HashJoin │  durum makinesi (8 durum)
        └──┬────┬──┘
     outer─┘    └─inner
   ┌───▼───┐ ┌───▼───┐
   │ Sort  │ │ Hash  │ ← MultiExecProcNode (tuple değil, tablo döner)
   └───┬───┘ └───┬───┘
   ┌───▼───┐ ┌───▼─────┐
   │SeqScan│ │IndexScan│
   └───────┘ └─────────┘

   work_mem aşılırsa:
     Sort    → logical tape'lere run yazar, sonra k-yollu merge
     Hash    → batch dosyalarına (BufFile) taşar, nbatch ikiye katlanır
     HashAgg → partition'lara spill eder, batch batch geri okur
     Memoize → diske hiç taşmaz, LRU ile eleman atar
```

Zihinsel model — altı madde:

1. **Tek arayüz, iki tip düğüm.** Her düğüm `ExecProcNode` ile çağrılır ve *bir* `TupleTableSlot` döner. Ama
   düğümler ikiye ayrılır: **streaming** (SeqScan, NestLoop, Limit, Material — girdiyi okurken çıktı verir) ve
   **blocking** (Sort, Hash, Aggregate — tüm girdiyi tüketmeden ilk satırı veremez). Yüksek `actual startup
   time` blocking düğümün imzasıdır.
2. **Durum makineleri "ilk çağrı" sorununu çözer.** Pull modelinde düğüm her çağrıda kaldığı yerden devam
   etmek zorundadır. HashJoin (`hj_JoinState`), MergeJoin (`mj_JoinState`), Memoize (`mstatus`), Limit
   (`lstate`), IncrementalSort (`execution_status`) hepsi açık bir durum değişkeni tutar.
3. **`work_mem` bir sınır değil, bir eşiktir.** Aşıldığında sorgu hata vermez — düğüm stratejisini
   değiştirip diske yazmaya başlar. Hash tabanlı düğümler `work_mem` yerine
   `work_mem × hash_mem_multiplier` kullanır (varsayılan çarpan 2.0).
4. **Hash tabanlı düğümlerin ortak numarası: hash değerinin bitlerini bölmek.** Alt bitler bucket'ı, üst
   bitler batch/partition'ı seçer; batch sayısı ikiye katlandığında tuple ya yerinde kalır ya *ileri* gider.
5. **İfadeler ağaç değil, düz bir dizi olarak çalıştırılır.** `execExpr.c` ağacı `ExprState->steps[]`
   dizisine derler; `execExprInterp.c` bu diziyi computed goto ile yorumlar.
6. **`EXPLAIN ANALYZE` sayaçları düğümden değil, sarmalayıcıdan gelir.** `instrument` alanı doluysa
   `ExecProcNode` gerçek fonksiyon yerine `ExecProcNodeInstr`'a yönlenir.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/executor/README](../src/backend/executor/README) | Motorun kendi tarifi. **Önce bu okunur** |
| [../src/backend/executor/execProcnode.c](../src/backend/executor/execProcnode.c) | `ExecInitNode`/`ExecEndNode` dağıtıcıları, `ExecSetTupleBound` |
| [../src/include/executor/executor.h](../src/include/executor/executor.h) | `ExecProcNode` inline gövdesi |
| [../src/backend/executor/nodeSort.c](../src/backend/executor/nodeSort.c) | Sort düğümü — ince bir sarmalayıcı |
| [../src/backend/utils/sort/tuplesort.c](../src/backend/utils/sort/tuplesort.c) | Asıl sıralama motoru: quicksort → external merge |
| [../src/backend/utils/sort/tuplesortvariants.c](../src/backend/utils/sort/tuplesortvariants.c) | Tuple/datum/index varyantları, karşılaştırıcılar |
| [../src/backend/utils/sort/logtape.c](../src/backend/utils/sort/logtape.c) | Tek temp dosya içinde N mantıksal "tape", blok geri dönüşümü |
| [../src/backend/executor/nodeHash.c](../src/backend/executor/nodeHash.c) | Hash tablosu kurulumu, boyutlandırma, batch'e bölme |
| [../src/backend/executor/nodeHashjoin.c](../src/backend/executor/nodeHashjoin.c) | Hybrid hash join durum makinesi |
| [../src/backend/executor/nodeAgg.c](../src/backend/executor/nodeAgg.c) | Sorted + hashed aggregate, GROUPING SETS, spill. **En büyük dosya** |
| [../src/backend/executor/nodeMergejoin.c](../src/backend/executor/nodeMergejoin.c) | 11 durumlu merge join makinesi, mark/restore |
| [../src/backend/executor/nodeNestloop.c](../src/backend/executor/nodeNestloop.c) | Nested loop — parametre geçirme + rescan |
| [../src/backend/executor/nodeMemoize.c](../src/backend/executor/nodeMemoize.c) | Parametreli iç taraf için LRU cache |
| [../src/backend/executor/nodeMaterial.c](../src/backend/executor/nodeMaterial.c) | Tuplestore ile tamponlama |
| [../src/backend/executor/nodeLimit.c](../src/backend/executor/nodeLimit.c) | OFFSET/LIMIT/WITH TIES durum makinesi |
| [../src/backend/executor/nodeIncrementalSort.c](../src/backend/executor/nodeIncrementalSort.c) | Ön eki sıralı girdiyi grup grup sıralama |
| [../src/backend/executor/execExpr.c](../src/backend/executor/execExpr.c) | İfade **derleyicisi**: Expr ağacı → adım dizisi |
| [../src/backend/executor/execExprInterp.c](../src/backend/executor/execExprInterp.c) | İfade **yorumlayıcısı**: computed goto döngüsü |
| [../src/backend/executor/instrument.c](../src/backend/executor/instrument.c) | EXPLAIN ANALYZE sayaçları |

---

# 1. Ortak iskelet — Volcano (pull) modeli

README'nin ilk cümlesi tüm mimariyi anlatıyor:
```
 * The plan tree is essentially a demand-pull pipeline of tuple processing
 * operations.  Each node, when called, will produce the next tuple in its
 * output sequence, or NULL if no more tuples are available.
```
— [../src/backend/executor/README:6-9](../src/backend/executor/README#L6-L9) (kısaltılmış)

Çağrı noktası bir inline fonksiyon:
```c
static inline TupleTableSlot *
ExecProcNode(PlanState *node)
{
	if (node->chgParam != NULL) /* something changed? */
		ExecReScan(node);		/* let ReScan handle this */
	return node->ExecProcNode(node);
}
```
— [../src/include/executor/executor.h:321-328](../src/include/executor/executor.h#L321-L328)

İki şey dikkat çekici. **`chgParam` kontrolü burada**: bir parametre değiştiyse (nested loop'un dış satırı
değişti gibi) düğüm çağrılmadan önce otomatik rescan edilir. Ve **`node->ExecProcNode` bir fonksiyon
işaretçisi** — tip alanına bakan dev bir `switch` yok.

**İşaretçi neden iki tane?** `PlanState` hem `ExecProcNode` (çağrılan) hem `ExecProcNodeReal` (gerçek
gövde) tutar ([../src/include/nodes/execnodes.h:1207-1209](../src/include/nodes/execnodes.h#L1207-L1209)).
Kurulumda birincisi bir sarmalayıcıya ayarlanır
([../src/backend/executor/execProcnode.c:430-440](../src/backend/executor/execProcnode.c#L430-L440)); ilk
çağrıda `ExecProcNodeFirst` `check_stack_depth()` yapar ve kendini yerinden çıkarır — `instrument` doluysa
`ExecProcNodeInstr`, değilse doğrudan `ExecProcNodeReal` takılır
([../src/backend/executor/execProcnode.c:447-468](../src/backend/executor/execProcnode.c#L447-L468)). Stack
kontrolü **bir kez** yapılır ve `EXPLAIN ANALYZE` çalıştırmıyorsan ölçüm kodu hiç devreye girmez.

**Tuple değil, tablo döndürenler.** Hash düğümü tek tek tuple vermez, bir hash tablosu kurar; bunun için
`MultiExecProcNode` vardır
([../src/backend/executor/execProcnode.c:488](../src/backend/executor/execProcnode.c#L488)). `switch`
yalnızca dört tip tanır — `T_HashState`, `T_BitmapIndexScanState`, `T_BitmapAndState`, `T_BitmapOrState`
([../src/backend/executor/execProcnode.c:504-523](../src/backend/executor/execProcnode.c#L504-L523)). Kaç
tuple döndüğü belli olmadığı için instrumentation burada yapılmaz.

**TupleTableSlot** bir tuple'ı temsil eder ama *nerede* olduğunu soyutlar — `TTSOpsBufferHeapTuple` (disk
buffer sayfasında, pin tutulur), `TTSOpsHeapTuple` (palloc'lanmış fiziksel tuple), `TTSOpsMinimalTuple`
(header'ı kırpılmış; hash tablosu ve tuplestore formatı), `TTSOpsVirtual` (sadece `Datum`/`isnull`
dizileri) ([../src/include/executor/tuptable.h:32-36](../src/include/executor/tuptable.h#L32-L36),
[:248-251](../src/include/executor/tuptable.h#L248-L251)). Sort'un girdi slotu virtual, çıktı slotu minimal
tuple'dır ([../src/backend/executor/nodeSort.c:270-276](../src/backend/executor/nodeSort.c#L270-L276));
hash join'in diske yazdığı format da `MinimalTuple`'dır — en kompakt olan.

**Bellek: iki bağlam.** Sorgu ömrü boyunca yaşayan bir "per query" bağlam ve tuple başına sıfırlanan "per
tuple" bağlamlar ([../src/backend/executor/README:273-285](../src/backend/executor/README#L273-L285)). Her
düğümün gövdesi `ResetExprContext(econtext)` ile başlar — HashJoin
[:254](../src/backend/executor/nodeHashjoin.c#L254), MergeJoin
[:628](../src/backend/executor/nodeMergejoin.c#L628), NestLoop
[:92](../src/backend/executor/nodeNestloop.c#L92). Aggregate bunu kıramaz, çünkü transition değeri satırlar
arası yaşamalıdır; `tmpcontext`, `aggcontexts[]` (grouping set başına) ve `hashcontext` ayrı tutulur
([../src/backend/executor/nodeAgg.c:189-195](../src/backend/executor/nodeAgg.c#L189-L195)).

---

# 2. Sort

## 2.1 `nodeSort.c` ince bir sarmalayıcıdır

Tüm dosya 489 satır, çünkü iş `tuplesort.c`'de. `ExecSort` iki parçalı: ilk çağrıda **her şeyi** çek, sonra
tek tek geri ver.
```c
			for (;;)
			{
				slot = ExecProcNode(outerNode);
				if (TupIsNull(slot))
					break;
				tuplesort_puttupleslot(tuplesortstate, slot);
			}
		tuplesort_performsort(tuplesortstate);
```
— [../src/backend/executor/nodeSort.c:145-160](../src/backend/executor/nodeSort.c#L145-L160) (kısaltılmış)

`sort_Done` bayrağı düğümü blocking yapan şeydir: ilk `ExecProcNode(sortNode)` çağrısı alt ağacın tamamını
tüketir. Tek sütunluk çıktıda tuple yerine sadece `Datum` sıralanır — pass-by-value tiplerde belirgin biçimde
hızlıdır ([:285-288](../src/backend/executor/nodeSort.c#L285-L288)). Sort alt düğümden REWIND/BACKWARD/MARK
yeteneği de **istemez**, çünkü sonucu kendi saklar
([:263](../src/backend/executor/nodeSort.c#L263)).

## 2.2 `tuplesort.c` durum makinesi

```c
typedef enum
{
	TSS_INITIAL,				/* Loading tuples; still within memory limit */
	TSS_BOUNDED,				/* Loading tuples into bounded-size heap */
	TSS_BUILDRUNS,				/* Loading tuples; writing to tape */
	TSS_SORTEDINMEM,			/* Sort completed entirely in memory */
	TSS_SORTEDONTAPE,			/* Sort completed, final run is on tape */
	TSS_FINALMERGE,				/* Performing final merge on-the-fly */
} TupSortStatus;
```
— [../src/backend/utils/sort/tuplesort.c:153-161](../src/backend/utils/sort/tuplesort.c#L153-L161)

Bellek takibi tek sayaçla yapılır; negatife düşerse bellek bitmiştir — `LACKMEM`/`USEMEM`/`FREEMEM` ([:398-400](../src/backend/utils/sort/tuplesort.c#L398-L400)).

## 2.3 Kritik an: bellek dolduğunda

`tuplesort_puttuple_common` içindeki `TSS_INITIAL` dalı:
```c
			if (state->memtupcount < state->memtupsize && !LACKMEM(state))
				return;			/* hâlâ sığıyoruz */
			/* Nope; time to switch to tape-based operation. */
			inittapes(state, true);
			dumptuples(state, false);
```
— [../src/backend/utils/sort/tuplesort.c:1151-1167](../src/backend/utils/sort/tuplesort.c#L1151-L1167) (kısaltılmış)

`dumptuples` bellekteki tuple'ları **sıralayıp** tek bir "run" olarak tape'e yazar, sonra belleği boşaltır
([:2250-2268](../src/backend/utils/sort/tuplesort.c#L2250-L2268)). "External merge sort"un ilk yarısı budur:
girdi, her biri `work_mem` boyunda sıralı parçalara (run) bölünür; ikinci yarı bunları birleştirmektir.
Bellek içi sıralama üç yoldan biriyle yapılır — tamsayı karşılaştırıcısı ve ≥ 40 tuple varsa **radix sort**,
tek anahtar varsa `qsort_ssup`, aksi halde genel `qsort_tuple`
([:3010-3018](../src/backend/utils/sort/tuplesort.c#L3010-L3018)).

## 2.4 `tuplesort_performsort` — üç yol ayrımı

[../src/backend/utils/sort/tuplesort.c:1260](../src/backend/utils/sort/tuplesort.c#L1260)

| Giriş durumu | Yapılan | Çıkış durumu |
|---|---|---|
| `TSS_INITIAL` | `tuplesort_sort_memtuples()` — her şey bellekte | `TSS_SORTEDINMEM` |
| `TSS_BOUNDED` | `sort_bounded_heap()` — heap'i diziye çevir | `TSS_SORTEDINMEM` |
| `TSS_BUILDRUNS` | `dumptuples(alltuples=true)` + `mergeruns()` | `TSS_SORTEDONTAPE` / `TSS_FINALMERGE` |

Son satırdaki kod yorumu `external sort` ile `external merge` farkını açıklıyor: "*merge until we have a
single remaining run (or, if !randomAccess and !WORKER(), one run per tape)*"
([../src/backend/utils/sort/tuplesort.c:1324-1331](../src/backend/utils/sort/tuplesort.c#L1324-L1331)). Yani
çağıran random access istemiyorsa son birleştirme **yapılmaz**: tape'ler açık bırakılır ve birleştirme
`tuplesort_getXXX` çağrıları sırasında uçarak yapılır (`TSS_FINALMERGE`) — bir tam yazma+okuma turu
kazanılır.

## 2.5 `mergeruns` — dengeli k-yollu birleştirme

```
 * This implements the Balanced k-Way Merge Algorithm.  All input data has
 * already been written to initial runs on tape (see dumptuples).
```
— [../src/backend/utils/sort/tuplesort.c:1907-1912](../src/backend/utils/sort/tuplesort.c#L1907-L1912)

Kaç yollu? `work_mem`'e göre, `MINORDER` 6 ile `MAXORDER` 500 arasında sıkışır
([../src/backend/utils/sort/tuplesort.c:175-177](../src/backend/utils/sort/tuplesort.c#L175-L177)). Run
sayısı merge order'ı aşarsa **birden fazla merge pass** gerekir — dosya başındaki yorumun anlattığı senaryo
([../src/backend/utils/sort/tuplesort.c:37-42](../src/backend/utils/sort/tuplesort.c#L37-L42)).
Birleştirmede her tape'ten yalnızca en öndeki tuple bellekte tutulur, kalan bellek tape tamponlarına verilir
([:1987-1988](../src/backend/utils/sort/tuplesort.c#L1987-L1988)). `logtape.c` bunu tek temp dosyanın içinde
yapar ve okunan blokları anında geri dönüştürür — disk kullanımı veri hacminin iki katına çıkmaz
([../src/backend/utils/sort/logtape.c:6-20](../src/backend/utils/sort/logtape.c#L6-L20)).

## 2.6 top-N heapsort ve rescan

`ORDER BY ... LIMIT 10`'da tüm tabloyu sıralamak gereksizdir. Limit düğümü alt düğüme sınır bildirir
([../src/backend/executor/execProcnode.c:837-855](../src/backend/executor/execProcnode.c#L837-L855));
tuplesort yeterince satır biriktiğinde (`memtupcount > bound * 2`, ya da bellek dolduysa `> bound`)
`make_bounded_heap` ile heap moduna geçer
([../src/backend/utils/sort/tuplesort.c:1138-1147](../src/backend/utils/sort/tuplesort.c#L1138-L1147)). O
fonksiyon sıralama yönünü **tersine çevirir**: kökte *en büyük* eleman durur, yeni gelen küçük değerler onu
ucuzca dışarı atar
([:2474-2484](../src/backend/utils/sort/tuplesort.c#L2474-L2484)). Bellek `bound` kadar tuple ile
sınırlıdır — diske hiç inilmez.

**Rescan.** Parametre değişmediyse (`chgParam == NULL`), bound aynıysa ve random access varsa sıralanmış
veri **yerinde başa sarılır** (`tuplesort_rescan`); aksi halde her şey atılır ve yeniden sıralanır
([../src/backend/executor/nodeSort.c:384-401](../src/backend/executor/nodeSort.c#L384-L401)). README'nin
"moderately intelligent scheme" dediği şey budur
([../src/backend/executor/README:24-26](../src/backend/executor/README#L24-L26)).

---

# 3. Hash Join

Plan ağacında Hash Join'in iç tarafında her zaman bir `Hash` düğümü vardır. **`nodeHash.c`** tabloyu kurar,
boyutlandırır, batch'e böler, taşırır; **`nodeHashjoin.c`** dış tarafı tarar ve durum makinesini sürer.
Algoritma "hybrid hash join"dir
([../src/backend/executor/nodeHashjoin.c:15-18](../src/backend/executor/nodeHashjoin.c#L15-L18)).

## 3.1 Boyutlandırma

Planlayıcı ile executor **aynı fonksiyonu** kullanır (maliyet tahmini tutarlı olsun diye):
`ExecChooseHashTableSize`, hedef bucket yükü `NTUP_PER_BUCKET` = 1
([../src/backend/executor/nodeHash.c:679-684](../src/backend/executor/nodeHash.c#L679-L684)). Bellek
bütçesi `work_mem` değil
([../src/backend/executor/nodeHash.c:717](../src/backend/executor/nodeHash.c#L717)):
```c
	mem_limit = (double) work_mem * hash_mem_multiplier * 1024.0;
```
— [../src/backend/executor/nodeHash.c:3679-3686](../src/backend/executor/nodeHash.c#L3679-L3686)

Tahmini iç taraf bütçeye sığmıyorsa batch sayısı hesaplanır
([../src/backend/executor/nodeHash.c:819-822](../src/backend/executor/nodeHash.c#L819-L822)). `nbuckets` ve
`nbatch` **her zaman ikinin kuvvetidir** — yoksa bit maskeleme çalışmaz
([../src/backend/executor/nodeHash.c:794-796](../src/backend/executor/nodeHash.c#L794-L796)).

## 3.2 Hash değerinin bitleri nasıl bölünür

Tüm batching mekanizmasının kalbi:
```c
	if (nbatch > 1)
	{
		*bucketno = hashvalue & (nbuckets - 1);
		*batchno = pg_rotate_right32(hashvalue,
									 hashtable->log2_nbuckets) & (nbatch - 1);
	}
```
— [../src/backend/executor/nodeHash.c:1994-2005](../src/backend/executor/nodeHash.c#L1994-L2005)

Fonksiyonun üstündeki yorum invariantı açıkça yazıyor:
```
 * Note: on-the-fly increases of nbatch must not change the bucket number
 * for a given hash code (since we don't move tuples to different hash
 * chains), and must only cause the batch number to remain the same or
 * increase.
```
— [../src/backend/executor/nodeHash.c:1961-1964](../src/backend/executor/nodeHash.c#L1961-L1964)

Batch sayısı ikiye katlandığında `batchno`'nun tepesine bir bit eklenir; tuple ya yerinde kalır ya ileri gider.
Çok büyük join'lerde bit biteceği için hash değeri **rotate** edilir — batch, bucket'tan bit çalar
([:1978-1984](../src/backend/executor/nodeHash.c#L1978-L1984)).

## 3.3 Tablonun kurulması ve taşma

`MultiExecPrivateHash` alt düğümü sonuna kadar okur, her satırın hash değerini hesaplar ve
`ExecHashTableInsert` çağırır
([../src/backend/executor/nodeHash.c:165-197](../src/backend/executor/nodeHash.c#L165-L197)). NULL join
anahtarlı tuple'lar hash tablosuna **girmez** — hiçbir şeyle eşleşemezler; outer join için gerekiyorsa ayrı bir
tuplestore'a konur, değilse atılır ([:199-208](../src/backend/executor/nodeHash.c#L199-L208)).

`ExecHashTableInsert` iki yol ayırır. Bu batch'e girenler bucket'ın başındaki bağlı listeye eklenir ve
`spaceUsed` bütçeyi (`spaceAllowed`) aşınca hemen `ExecHashIncreaseNumBatches` çağrılır
([../src/backend/executor/nodeHash.c:1789-1815](../src/backend/executor/nodeHash.c#L1789-L1815),
[:1835-1842](../src/backend/executor/nodeHash.c#L1835-L1842)). Sığmayanlar `BufFile`'a yazılır — önce hash
değeri, sonra `MinimalTuple`
([../src/backend/executor/nodeHashjoin.c:1600-1601](../src/backend/executor/nodeHashjoin.c#L1600-L1601)); bu
sayede batch geri okunurken hash yeniden hesaplanmaz.

`ExecHashIncreaseNumBatches` ([:1055](../src/backend/executor/nodeHash.c#L1055)) `nbatch`'i ikiye katlar,
bucket dizisini sıfırlar, tüm chunk'ları gezip her tuple'ı yeniden sınıflandırır
([:1139-1189](../src/backend/executor/nodeHash.c#L1139-L1189)). En önemli kısım sonda — **büyümenin
kapatılması**:
```c
	/*
	 * If we dumped out either all or none of the tuples in the table, disable
	 * further expansion of nbatch. ... hope the server has enough RAM.
	 */
	if (nfreed == 0 || nfreed == ninmemory)
		hashtable->growEnabled = false;
```
— [../src/backend/executor/nodeHash.c:1199-1209](../src/backend/executor/nodeHash.c#L1199-L1209) (kısaltılmış)

Yani ağır skew'de (aynı anahtardan milyonlarca satır) hash join `work_mem`'i **aşabilir**. `EXPLAIN
ANALYZE`'da "Memory Usage" bütçeyi geçiyorsa sebebi budur.

## 3.4 Durum makinesi

Sekiz durum var — `HJ_BUILD_HASHTABLE`, `HJ_NEED_NEW_OUTER`, `HJ_SCAN_BUCKET`, `HJ_FILL_OUTER_TUPLE`,
`HJ_FILL_INNER_TUPLES`, `HJ_FILL_OUTER_NULL_TUPLES`, `HJ_FILL_INNER_NULL_TUPLES`, `HJ_NEED_NEW_BATCH`
([../src/backend/executor/nodeHashjoin.c:182-189](../src/backend/executor/nodeHashjoin.c#L182-L189)).
Motor paralel ve seri sürüm için ortak yazılıp `pg_always_inline` ile iki kez derletiliyor
([../src/backend/executor/nodeHashjoin.c:225-226](../src/backend/executor/nodeHashjoin.c#L225-L226)).

```
BUILD_HASHTABLE   ExecHashTableCreate + MultiExecProcNode(hashNode)        [:271]
      ▼
NEED_NEW_OUTER    dış tuple çek; bu batch değilse dosyaya yaz, döngüde kal [:435]
      ▼
SCAN_BUCKET       ExecScanHashBucket + joinqual → eşleşirse ExecProject    [:536]
      ▼ bitti
FILL_OUTER_TUPLE  LEFT/FULL ise NULL-doldurulmuş satır üret                [:625]
      ▼ (dış taraf bitti)
FILL_INNER_TUPLES RIGHT/FULL ise eşleşmemiş iç tuple'ları yay              [:650]
FILL_*_NULL       NULL anahtarlı tuple'ları yay                     [:683] [:729]
      ▼
NEED_NEW_BATCH    ExecHashJoinNewBatch; batch yoksa NULL                   [:770]
```
Etiketler: [:271](../src/backend/executor/nodeHashjoin.c#L271) · [:435](../src/backend/executor/nodeHashjoin.c#L435) ·
[:536](../src/backend/executor/nodeHashjoin.c#L536) · [:625](../src/backend/executor/nodeHashjoin.c#L625) ·
[:650](../src/backend/executor/nodeHashjoin.c#L650) · [:683](../src/backend/executor/nodeHashjoin.c#L683) ·
[:729](../src/backend/executor/nodeHashjoin.c#L729) · [:770](../src/backend/executor/nodeHashjoin.c#L770).
Bucket taraması önce hash değerini karşılaştırır (ucuz), sonra gerçek eşitlik ifadesini çalıştırır
(`ExecQualAndReset`, pahalı)
([../src/backend/executor/nodeHash.c:2040-2059](../src/backend/executor/nodeHash.c#L2040-L2059)).

## 3.5 Batch değiştirme

`ExecHashJoinNewBatch`
([../src/backend/executor/nodeHashjoin.c:1279](../src/backend/executor/nodeHashjoin.c#L1279)) iç batch
dosyasını baştan okur ve tabloyu yeniden kurar:
```c
		while ((slot = ExecHashJoinGetSavedTuple(...)))
		{
			/*
			 * NOTE: some tuples may be sent to future batches.  Also, it is
			 * possible for hashtable->nbatch to be increased here!
			 */
			ExecHashTableInsert(hashtable, slot, hashvalue);
		}
```
— [../src/backend/executor/nodeHashjoin.c:1381-1389](../src/backend/executor/nodeHashjoin.c#L1381-L1389) (argümanlar kısaltıldı)

Yorumdaki uyarı önemli: bir batch'i yüklerken **yeniden taşma** olabilir ve batch sayısı ortada artabilir; bu
yüzden `nbatch` `EXPLAIN`'de "originally N" ile gösterilir. Boş batch'ler atlanır
([:1330-1360](../src/backend/executor/nodeHashjoin.c#L1330-L1360)).

---

# 4. Aggregate

`nodeAgg.c` 4853 satırla executor'ın en karmaşık dosyası. Dört strateji var — `AGG_PLAIN` (tüm satırlar tek
grup), `AGG_SORTED` (girdi sıralı gelmeli), `AGG_HASHED` (iç hash tablosu), `AGG_MIXED` (ikisi birlikte)
([../src/include/nodes/nodes.h:360-366](../src/include/nodes/nodes.h#L360-L366)). `ExecAgg` bir dağıtıcıdır:
`AGG_HASHED`/`AGG_MIXED` → `agg_fill_hash_table` + `agg_retrieve_hash_table`; `AGG_PLAIN`/`AGG_SORTED` →
`agg_retrieve_direct`
([../src/backend/executor/nodeAgg.c:2246-2270](../src/backend/executor/nodeAgg.c#L2246-L2270)).

## 4.1 Sorted aggregate — grup sınırı tespiti

`agg_retrieve_direct` ([../src/backend/executor/nodeAgg.c:2283](../src/backend/executor/nodeAgg.c#L2283))
girdinin sıralı geldiğini varsayar; her satırda "hâlâ aynı grupta mıyız" diye sorar:
```c
						tmpcontext->ecxt_innertuple = firstSlot;
						if (!ExecQual(aggstate->phase->eqfunctions[node->numCols - 1],
									  tmpcontext))
						{
							aggstate->grp_firstTuple = ExecCopySlotHeapTuple(outerslot);
							break;   /* grup değişti */
						}
```
— [../src/backend/executor/nodeAgg.c:2576-2584](../src/backend/executor/nodeAgg.c#L2576-L2584) (dış `if` atıldı)

Grup değiştiğinde ilk satır `grp_firstTuple`'a saklanır — bir sonraki çağrının başlangıç noktası. Sorted
agg blocking değildir: ilk grubu bitirir bitirmez satır üretir, bellek kullanımı grup başına sabittir,
diske taşmaz. **`work_mem` sorted agg'i ilgilendirmez** — ama altındaki Sort'u ilgilendirir.

## 4.2 Hashed aggregate ve spill

`agg_fill_hash_table` alt düğümü sonuna kadar okur, her satır için hash girdisi arar/yaratır ve transition
fonksiyonlarını ilerletir
([../src/backend/executor/nodeAgg.c:2628-2652](../src/backend/executor/nodeAgg.c#L2628-L2652)). Hash agg
**blocking**tir, ama girdinin sıralı olmasını gerektirmez. Bellek yetmezse mekanizma dosyanın başında
anlatılıyor:
```
 *	  ... we enter "spill mode". In spill mode, we advance the transition
 *	  states only for groups already in the hash table. For tuples that would
 *	  need to create a new hash table entries (and initialize new transition
 *	  states), we instead spill them to disk to be processed later.
```
— [../src/backend/executor/nodeAgg.c:198-204](../src/backend/executor/nodeAgg.c#L198-L204) (kısaltılmış)

**Kritik ayrım:** spill modunda *tüm* satırlar diske gitmez. Zaten tabloda olan grupların transition değeri
güncellenmeye devam eder; sadece **yeni grup açacak** satırlar diske yazılır. `hash_agg_check_limits` iki
sınıra bakar — `total_mem > hash_mem_limit` ya da `ngroups > hash_ngroups_limit`
([../src/backend/executor/nodeAgg.c:1893-1898](../src/backend/executor/nodeAgg.c#L1893-L1898)). Grup sayısı
sınırı `array_agg` gibi transition değeri şişen aggregate'ler için var: bellek ölçümü geç kalabilir, grup
sayımı erken uyarır
([../src/backend/executor/nodeAgg.c:1798-1804](../src/backend/executor/nodeAgg.c#L1798-L1804)).

`lookup_hash_entries` spill modunda yeni giriş oluşturmayı reddeder (`p_isnew = NULL`) ve satırı
`hashagg_spill_tuple` ile diske yönlendirir
([../src/backend/executor/nodeAgg.c:2200-2229](../src/backend/executor/nodeAgg.c#L2200-L2229)). Hedef
partition, hash değerinin bir bit dilimidir ([:3064-3067](../src/backend/executor/nodeAgg.c#L3064-L3067));
partition sayısı `HASHAGG_MIN_PARTITIONS` 4 ile `HASHAGG_MAX_PARTITIONS` 1024 arasında sıkışır
([:298-300](../src/backend/executor/nodeAgg.c#L298-L300)) ve açık tape tamponları hash_mem'in çeyreğini
geçemez ([:2096-2102](../src/backend/executor/nodeAgg.c#L2096-L2102)).

Okuma tarafı bir yığın (stack) üzerinden döner: `agg_retrieve_hash_table` bellekteki grupları tüketir, bitince
`agg_refill_hash_table` ile bir sonraki batch'i yükler; o da yoksa `agg_done`
([../src/backend/executor/nodeAgg.c:2836-2854](../src/backend/executor/nodeAgg.c#L2836-L2854)).
`agg_refill_hash_table` ([:2683](../src/backend/executor/nodeAgg.c#L2683)) yığından bir batch alır, tabloyu
sıfırlar, o batch'i işler; batch de sığmazsa **özyinelemeli** olarak yeniden spill eder (bu yüzden
`used_bits` taşınır). `EXPLAIN`'deki `Batches: N` bu döngünün kaç kez döndüğüdür.

## 4.3 GROUPING SETS ve transition ifadesi

Planlayıcı tek bir "gerçek" Agg düğümü verir, ama yanında `chain` alanında ek Agg düğümleri asılıdır.
Hash'ler faz 0'a, her sıralı düğüm ayrı bir faza konur
([../src/backend/executor/nodeAgg.c:170-173](../src/backend/executor/nodeAgg.c#L170-L173)). ROLLUP gibi iç
içe geçmiş kümeler tek geçişte hesaplanır: her grouping set için ayrı transition değeri tutulur, grup
sınırında sadece *değişenler* sıfırlanır
([../src/backend/executor/nodeAgg.c:115-123](../src/backend/executor/nodeAgg.c#L115-L123)). Sıralama düzeni
farklı olan kümeler için ayrı faz açılır ve girdi fazlar arası bir tuplesort'a yazılır
([../src/backend/executor/nodeAgg.c:124-132](../src/backend/executor/nodeAgg.c#L124-L132)). `AGG_MIXED`
ikisini birleştirir: faz 1'de hem sorted gruplar ilerletilir hem hash tabloları doldurulur
([../src/backend/executor/nodeAgg.c:2536-2540](../src/backend/executor/nodeAgg.c#L2536-L2540)).

Transition fonksiyonları `nodeAgg.c`'den tek tek çağrılmaz: "*ExecBuildAggTrans() builds one large
expression that does both argument evaluation and transition function invocation*"
([../src/backend/executor/nodeAgg.c:230-235](../src/backend/executor/nodeAgg.c#L230-L235)).
`advance_aggregates` ([../src/backend/executor/nodeAgg.c:819](../src/backend/executor/nodeAgg.c#L819)) tek
bir devasa `ExprState` çalıştırır; `ExecBuildAggTrans`
([../src/backend/executor/execExpr.c:3671](../src/backend/executor/execExpr.c#L3671)) bunu derler. Sonuç:
JIT tüm aggregate döngüsünü tek native fonksiyona çevirebilir.

---

# 5. Merge Join

Onbir durum: `EXEC_MJ_INITIALIZE_OUTER`, `_INITIALIZE_INNER`, `_JOINTUPLES`, `_NEXTOUTER`, `_TESTOUTER`,
`_NEXTINNER`, `_SKIP_TEST`, `_SKIPOUTER_ADVANCE`, `_SKIPINNER_ADVANCE`, `_ENDOUTER`, `_ENDINNER`
([../src/backend/executor/nodeMergejoin.c:107-117](../src/backend/executor/nodeMergejoin.c#L107-L117)). Baş
yorum algoritmayı sözde kodla veriyor ([:61-85](../src/backend/executor/nodeMergejoin.c#L61-L85)): iki imleç
senkronize edilir, eşitlik bulununca iç taraf işaretlenir, dış taraf ilerledikçe işaret geri yüklenir.

**Neden merge clause'lar doğrudan çalıştırılmaz.** Merge join'in "büyük mü küçük mü" bilmesi gerekir,
sadece "eşit mi" değil; bu yüzden anahtarlar ayrı hesaplanıp btree karşılaştırıcısıyla
(`ApplySortComparator`) kıyaslanır
([../src/backend/executor/nodeMergejoin.c:387-408](../src/backend/executor/nodeMergejoin.c#L387-L408)).
NULL = NULL durumu **eşit sayılmaz**; sonuç +1'e zorlanır ki iç taraf ilerlesin
([../src/backend/executor/nodeMergejoin.c:425-435](../src/backend/executor/nodeMergejoin.c#L425-L435)).

**Mark / restore.** Eşitlik bulununca iç tarafın konumu işaretlenir (`ExecMarkPos`,
[:1190-1198](../src/backend/executor/nodeMergejoin.c#L1190-L1198)), dış taraf ilerleyip aynı değeri taşıyorsa
geri sarılır (`ExecRestrPos`, [:1078-1084](../src/backend/executor/nodeMergejoin.c#L1078-L1084)). **Bu yüzden
merge join'in altında sık sık `Materialize` görürsün** — iç taraf mark/restore desteklemiyorsa planlayıcı
araya Material koyar. `mj_SkipMarkRestore`, join `inner_unique` ise bu maliyeti tamamen kaldırır
([:1487](../src/backend/executor/nodeMergejoin.c#L1487)).

---

# 6. Nested Loop + Memoize

Nested loop iç tarafı elle çalıştırmaz; parametreyi yazar ve rescan tetikler:
```c
			foreach(lc, nl->nestParams)
			{
				prm = &(econtext->ecxt_param_exec_vals[paramno]);
				...
				innerPlan->chgParam = bms_add_member(innerPlan->chgParam, paramno);
			}
			ExecReScan(innerPlan);
```
— [../src/backend/executor/nodeNestloop.c:129-152](../src/backend/executor/nodeNestloop.c#L129-L152) (kısaltılmış)

Parametre `EState`'in paylaşılan `ecxt_param_exec_vals` dizisine yazılır; iç taraftaki IndexScan bunu tarama
anahtarı olarak okur — `Index Cond: (t2.id = t1.id)` satırının çalışma zamanı karşılığı budur. Sorun: **aynı
parametre değeri tekrar gelirse aynı tarama tekrar yapılır.** Memoize tam olarak bunu çözer.

Dosyanın kendi tarifi: "*When the cache fills, we never spill tuples to disk, instead, we choose to evict
the least recently used cache entry*"
([../src/backend/executor/nodeMemoize.c:20-23](../src/backend/executor/nodeMemoize.c#L20-L23)).

**Memoize diske taşmaz.** Beş durumu var — `MEMO_CACHE_LOOKUP`, `MEMO_CACHE_FETCH_NEXT_TUPLE`,
`MEMO_FILLING_CACHE`, `MEMO_CACHE_BYPASS_MODE`, `MEMO_END_OF_SCAN`
([../src/backend/executor/nodeMemoize.c:78-84](../src/backend/executor/nodeMemoize.c#L78-L84)). Cache hit
yolunda `entry->complete` doğruysa satırlar bağlı listeden okunur, alt düğüm hiç çalıştırılmaz
([../src/backend/executor/nodeMemoize.c:741-757](../src/backend/executor/nodeMemoize.c#L741-L757)).
`complete` ince bir nokta: semi join gibi çağıranlar taramayı yarıda bırakabilir, o zaman girdi eksiktir ve
kullanılamaz; unique join'de ise tek satırdan sonra "tamam" işaretlenebilir
([../src/backend/executor/nodeMemoize.c:29-36](../src/backend/executor/nodeMemoize.c#L29-L36)).

Bellek sınırı hash join ile aynı bütçeden gelir
([../src/backend/executor/nodeMemoize.c:1040](../src/backend/executor/nodeMemoize.c#L1040)); aşılınca
`cache_reduce_memory` ([:441](../src/backend/executor/nodeMemoize.c#L441)) LRU'dan atmaya başlar.
**Doldurulmakta olan** girdiyi de atmak zorunda kalırsa `cache_store_tuple` false döner, `cache_overflows`
artar ve düğüm `MEMO_CACHE_BYPASS_MODE`'a geçer
([:896-903](../src/backend/executor/nodeMemoize.c#L896-L903)) — satırlar artık cache'lenmeden geçirilir.
`EXPLAIN`'deki `Overflows: N` sıfırdan büyükse Memoize kâr etmiyor demektir.

---

# 7. Material, Limit ve Incremental Sort

**Material** `work_mem`'li bir tuplestore açar, MARK gerekiyorsa ikinci bir read pointer ayırır
([../src/backend/executor/nodeMaterial.c:63-77](../src/backend/executor/nodeMaterial.c#L63-L77)), ve
**streaming** çalışır: satırı alt düğümden çeker, tuplestore'a kopyalar ve aynı anda yukarı verir
([:135-146](../src/backend/executor/nodeMaterial.c#L135-L146)). `work_mem` aşılırsa tuplestore sessizce
diske geçer.

**Limit** sekiz durumlu bir makinedir
([../src/backend/executor/nodeLimit.c:40](../src/backend/executor/nodeLimit.c#L40),
[../src/include/nodes/execnodes.h:2786-2796](../src/include/nodes/execnodes.h#L2786-L2796)), ama asıl önemli
işi satır saymak değil, `ExecSetTupleBound(compute_tuples_needed(node), outerPlanState(node))` ile alt
düğüme haber vermektir
([../src/backend/executor/nodeLimit.c:423](../src/backend/executor/nodeLimit.c#L423)).
`ExecSetTupleBound` ([../src/backend/executor/execProcnode.c:829](../src/backend/executor/execProcnode.c#L829))
plan ağacında aşağı iner ve Sort/IncrementalSort'u "bounded" moda alır. `WITH TIES` varsa sınır
bildirilemez — kaç satır gerektiği önceden bilinemez.

**Incremental Sort**: girdi `(a, b)` sıralaması istenirken zaten `a`'ya göre sıralıysa, `a` değeri aynı
olan gruplar ayrı ayrı sıralanabilir. Avantajı iki yönlü — daha az veri aynı anda sıralandığı için
`work_mem`'e sığma şansı artar, *ve* "*it can start producing rows early, before sorting the whole dataset,
which is a significant benefit especially for queries with LIMIT*"
([../src/backend/executor/nodeIncrementalSort.c:51-56](../src/backend/executor/nodeIncrementalSort.c#L51-L56)).

Dört durum: `INCSORT_LOADFULLSORT`, `INCSORT_LOADPREFIXSORT`, `INCSORT_READFULLSORT`,
`INCSORT_READPREFIXSORT`
([../src/include/nodes/execnodes.h:2371-2377](../src/include/nodes/execnodes.h#L2371-L2377)). İki mod iki
sabitle yönetilir — `DEFAULT_MIN_GROUP_SIZE` = 32
([../src/backend/executor/nodeIncrementalSort.c:467](../src/backend/executor/nodeIncrementalSort.c#L467))
ve onun iki katı
([../src/backend/executor/nodeIncrementalSort.c:479](../src/backend/executor/nodeIncrementalSort.c#L479)):

- **Full sort modu**: ilk 32 satırda prefix kontrolü *yapılmaz*, hepsi tüm sütunlara göre sıralanır. Küçük
  gruplarda her grup için yeni tuplesort açma maliyetinden kaçınır.
- **Prefix sort modu**: 64 satırdan sonra grup hâlâ bitmediyse "bu büyük bir grup" varsayılır,
  `switchToPresortedPrefixMode`
  ([../src/backend/executor/nodeIncrementalSort.c:286](../src/backend/executor/nodeIncrementalSort.c#L286))
  çağrılır ve artık sadece *sıralı olmayan* sütunlara göre sıralanır. Karar noktası
  [:795-796](../src/backend/executor/nodeIncrementalSort.c#L795-L796).

Bu yüzden `EXPLAIN ANALYZE` çıktısında `Full-sort Groups:` ve `Presorted Groups:` diye iki satır vardır.

---

# 8. İfade değerlendirme

## 8.1 Derleyici: `execExpr.c`

Plan ağacı `PlanState` ağacına yansıtılır; ifade ağacı **yansıtılmaz** — tek bir `ExprState` içinde düz bir
`ExprEvalStep` dizisine derlenir. Gerekçe: bir Expr düğümünü hesaplamak o kadar ucuzdur ki ağaç yürüyüşü
baskın maliyet olur; düz gösterim ise özyinelemesiz çalıştırılabilir ve native koda derlenebilir
([../src/backend/executor/README:93-99](../src/backend/executor/README#L93-L99)).

```c
	/* Insert setup steps as needed */
	ExecCreateExprSetupSteps(state, (Node *) node);
	/* Compile the expression proper */
	ExecInitExprRec(node, state, &state->resvalue, &state->resnull);
	/* Finally, append a DONE step */
	scratch.opcode = EEOP_DONE_RETURN;
	ExprEvalPushStep(state, &scratch);
	ExecReadyExpr(state);
```
— [../src/backend/executor/execExpr.c:142-168](../src/backend/executor/execExpr.c#L142-L168) (boş satırlar atıldı)

Üç aşama: kurulum adımları, gövde, bitiş adımı. Son adım her zaman `EEOP_DONE_RETURN` veya
`EEOP_DONE_NO_RETURN`'dür — böylece yorumlayıcı dizi sonu kontrolü yapmaz
([../src/backend/executor/README:135-140](../src/backend/executor/README#L135-L140)). `a + b` üç adıma
derlenir: iki Var okuma + bir fonksiyon çağrısı; Var adımlarının `resvalue`'ları doğrudan fonksiyonun
`fcinfo->args[]` alanlarını gösterir — kopyalama yok
([../src/backend/executor/README:126-133](../src/backend/executor/README#L126-L133)). Hangi slottan
okunacağı da derleme zamanında sabitlenir: `EEOP_INNER_VAR`, `EEOP_OUTER_VAR`, `EEOP_SCAN_VAR` ayrı
opcode'lardır
([../src/backend/executor/execExpr.c:985-992](../src/backend/executor/execExpr.c#L985-L992)).

**Dallanma.** Kısa devre yapan ifadeler dizi içi atlama kullanır; hedef derleme sırasında bilinmediği için
önce `-1` konur, sonra düzeltilir:
```c
	foreach_ptr(Expr, node, qual)
	{
		ExecInitExprRec(node, state, &state->resvalue, &state->resnull);
		/* then emit EEOP_QUAL to detect if it's false (or null) */
		scratch.d.qualexpr.jumpdone = -1;
		ExprEvalPushStep(state, &scratch);
		adjust_jumps = lappend_int(adjust_jumps, state->steps_len - 1);
	}
	foreach_int(jump, adjust_jumps)		/* adjust jump targets */
		state->steps[jump].d.qualexpr.jumpdone = state->steps_len;
```
— [../src/backend/executor/execExpr.c:267-288](../src/backend/executor/execExpr.c#L267-L288) (Assert'ler ve ara değişken çıkarıldı)

`WHERE a > 1 AND b < 5` böyle derlenir: her koşuldan sonra bir `EEOP_QUAL` adımı, hepsi sona atlar. İlk
`false` bulunduğunda gerisi çalıştırılmaz.

## 8.2 Yorumlayıcı: `execExprInterp.c`

İki dağıtım stratejisi: computed goto varsa **direct threading**, yoksa **switch threading**
([../src/backend/executor/execExprInterp.c:16-26](../src/backend/executor/execExprInterp.c#L16-L26)).
İlerleme iki makroyla olur — `EEO_NEXT()` = `op++; EEO_DISPATCH();` ve `EEO_JUMP(stepno)` =
`op = &state->steps[stepno]; EEO_DISPATCH();`
([../src/backend/executor/execExprInterp.c:133-144](../src/backend/executor/execExprInterp.c#L133-L144)).
Direct threading'de `EEO_DISPATCH()` = `goto *((void *) op->opcode)`; her adımın sonunda ayrı bir dolaylı
sıçrama olduğu için CPU'nun dal tahmini belirgin biçimde iyileşir.

Adım gövdeleri çok incedir. Var okuma tek bir dizi erişimidir:
```c
		EEO_CASE(EEOP_OUTER_VAR)
		{
			int			attnum = op->d.var.attnum;
			*op->resvalue = outerslot->tts_values[attnum];
			*op->resnull = outerslot->tts_isnull[attnum];
			EEO_NEXT();
		}
```
— [../src/backend/executor/execExprInterp.c:706-717](../src/backend/executor/execExprInterp.c#L706-L717)

Bu kadar ucuz olmasının nedeni, dizinin başındaki `FETCHSOME` adımının `slot_getsomeattrs` ile tuple'ı
zaten açmış olmasıdır
([../src/backend/executor/execExprInterp.c:653-659](../src/backend/executor/execExprInterp.c#L653-L659)).
Ve `slot_getsomeattrs` tuple'ı **yalnızca gereken sütuna kadar** açar: 50 sütunluk bir tabloda 2. sütuna
bakan bir `WHERE`, kalan 48 sütunu hiç çözmez. Qual adımı kısa devreyi yapar — sonuç false ya da NULL ise
değeri `false`'a sabitleyip `EEO_JUMP(op->d.qualexpr.jumpdone)` ile ifadenin sonuna atlar
([../src/backend/executor/execExprInterp.c:1182-1195](../src/backend/executor/execExprInterp.c#L1182-L1195)).

**Fast path ve JIT.** Çok basit ifadeler için tüm yorumlayıcı döngüsü atlanır
([../src/backend/executor/execExprInterp.c:288-292](../src/backend/executor/execExprInterp.c#L288-L292)).
`ExecReadyInterpretedExpr` ([../src/backend/executor/execExprInterp.c:252](../src/backend/executor/execExprInterp.c#L252))
adım dizisinin uzunluğuna ve opcode desenine bakar: 4 adımlık
`OUTER_FETCHSOME + OUTER_VAR + HASHDATUM_FIRST + DONE` desenini görürse `ExecJustHashOuterVar`'ı doğrudan
`evalfunc` yapar; benzerleri `ExecJustInnerVar`, `ExecJustConst`, `ExecJustAssignOuterVar`. Bir seviye
yukarıda ise `ExecReadyExpr` önce `jit_compile_expr(state)` dener, başarısız olursa yorumlayıcıya düşer
([../src/backend/executor/execExpr.c:901-908](../src/backend/executor/execExpr.c#L901-L908)).

---

# İzleme ve hata ayıklama

## Sayaçlar nereden geliyor

`EXPLAIN ANALYZE` açıkken `ExecProcNode` gerçek fonksiyona değil `ExecProcNodeInstr`'a gider; o da
`InstrStartNode` → `ExecProcNodeReal` → `InstrStopNode(..., TupIsNull(result) ? 0.0 : 1.0)` sırasını işletir
([../src/backend/executor/instrument.c:181-192](../src/backend/executor/instrument.c#L181-L192)). `actual
time=X..Y (loops=N)` içindeki üç sayı:

- **X (startup)** = ilk tuple'a kadar geçen süre → `firsttuple`, ilk `InstrStopNode` çağrısında yakalanır
  ([../src/backend/executor/instrument.c:154-160](../src/backend/executor/instrument.c#L154-L160))
- **Y (total)** = `counter`, `InstrEndLoop`'ta `total`'a eklenir
- **loops** = `nloops`, `InstrEndLoop` ([../src/backend/executor/instrument.c:204](../src/backend/executor/instrument.c#L204))
  her rescan turunda bir artar

`BUFFERS`/`WAL` sayaçları bayrakla kontrol edilir
([../src/backend/executor/instrument.c:46-48](../src/backend/executor/instrument.c#L46-L48),
[../src/backend/executor/instrument.c:64-65](../src/backend/executor/instrument.c#L64-L65)); paralel
işçilerin sayaçları `InstrAggNode`
([../src/backend/executor/instrument.c:232](../src/backend/executor/instrument.c#L232)) ile lider sürece
toplanır.

## EXPLAIN satırlarının kod karşılığı

```
 HashAggregate  (actual time=812.4..905.1 rows=250000 loops=1)
   Batches: 5  Memory Usage: 8241kB  Disk Usage: 41552kB
   Buffers: shared hit=1204 read=88310, temp read=5120 written=5194
   ->  Hash Join  (actual time=201.8..612.9 rows=3000000)
         ->  Seq Scan on orders o  (actual time=0.0..98.2 rows=3000000)
         ->  Hash  (actual time=198.3..198.3 rows=500000 loops=1)
               Buckets: 65536 (originally 131072)  Batches: 4 (originally 1)
   ->  Sort   Sort Method: external merge  Disk: 78904kB
```

| EXPLAIN satırı | Kod kaynağı |
|---|---|
| `Sort Method: quicksort  Memory: NkB` | `maxSpaceStatus == TSS_SORTEDINMEM` [tuplesort.c:2417-2421](../src/backend/utils/sort/tuplesort.c#L2417-L2421) |
| `Sort Method: top-N heapsort` | aynı yer, `boundUsed == true` |
| `Sort Method: external merge  Disk: NkB` | `TSS_FINALMERGE` [tuplesort.c:2427-2429](../src/backend/utils/sort/tuplesort.c#L2427-L2429) — son merge uçarak yapıldı; `external sort` ise `TSS_SORTEDONTAPE` (random access istendi). Satırı yazan: [explain.c:3120-3124](../src/backend/commands/explain.c#L3120-L3124) |
| `Buckets: N (originally M)` | `nbuckets` ↔ `nbuckets_original`; `ExecHashIncreaseNumBuckets` [nodeHash.c:1612](../src/backend/executor/nodeHash.c#L1612) |
| `Batches: N (originally 1)` | `nbatch` ↔ `nbatch_original`; `ExecHashIncreaseNumBatches` [nodeHash.c:1055](../src/backend/executor/nodeHash.c#L1055) — **taşma göstergesi** |
| `Memory Usage: NkB` (Hash) | `hashtable->spacePeak` [nodeHash.c:1836-1837](../src/backend/executor/nodeHash.c#L1836-L1837); yazan [explain.c:3459-3474](../src/backend/commands/explain.c#L3459-L3474) |
| `Batches / Memory / Disk Usage` (HashAgg) | `hash_batches_used`, `hash_mem_peak`, `hash_disk_used` [nodeAgg.c:1978-1984](../src/backend/executor/nodeAgg.c#L1978-L1984); yazan [explain.c:3805-3814](../src/backend/commands/explain.c#L3805-L3814) |
| `Hits/Misses/Evictions/Overflows` | `MemoizeState.stats` [nodeMemoize.c:745](../src/backend/executor/nodeMemoize.c#L745), [:773](../src/backend/executor/nodeMemoize.c#L773), [:900](../src/backend/executor/nodeMemoize.c#L900); yazan [explain.c:3683](../src/backend/commands/explain.c#L3683) |
| `Full-sort / Presorted Groups` | `INSTRUMENT_SORT_GROUP` [nodeIncrementalSort.c:97-116](../src/backend/executor/nodeIncrementalSort.c#L97-L116) |
| `temp read=... written=...` | `pgBufferUsage`, `BUFFERS` bayrağıyla [instrument.c:64-65](../src/backend/executor/instrument.c#L64-L65) |

Okuma sırası — yavaş bir sorguda: (1) **`Batches` > 1 mi?** Evetse hash düğümü diske taşmış.
(2) **`Sort Method` `external` mi?** Evetse sort diske taşmış. (3) **`originally` değeri farklı mı?**
Farklıysa **planlayıcının tahmini yanlıştı** — `ANALYZE` çalıştır veya istatistik hedefini artır.
(4) **`loops` büyük mü?** Nested loop'un iç tarafındasın; Memoize veya farklı bir join stratejisi düşün.
(5) **`Overflows` > 0 mı?** Memoize bütçeye sığmıyor, kâr etmiyor.

## `work_mem` ayarı

Varsayılanlar: `work_mem` 4096 kB
([../src/backend/utils/misc/guc_parameters.dat:3637-3645](../src/backend/utils/misc/guc_parameters.dat#L3637-L3645)),
`hash_mem_multiplier` 2.0
([../src/backend/utils/misc/guc_parameters.dat:1220-1227](../src/backend/utils/misc/guc_parameters.dat#L1220-L1227)).

| Düğüm | Bütçe | Nerede |
|---|---|---|
| Sort, IncrementalSort | `work_mem` | [nodeSort.c:110](../src/backend/executor/nodeSort.c#L110) |
| Material, aggregate'in DISTINCT tuplesort'u | `work_mem` | [nodeMaterial.c:65](../src/backend/executor/nodeMaterial.c#L65) |
| Hash (join) | `work_mem × hash_mem_multiplier` | [nodeHash.c:717](../src/backend/executor/nodeHash.c#L717) |
| HashAggregate | aynı | [nodeAgg.c:1814](../src/backend/executor/nodeAgg.c#L1814) |
| Memoize | aynı | [nodeMemoize.c:1040](../src/backend/executor/nodeMemoize.c#L1040) |

Üç uyarı: (1) **`work_mem` sorgu başına değil, düğüm başınadır** — beş Sort içeren bir plan beş kez
isteyebilir, paralel planda her işçi için ayrıca. (2) **Aşılabilir**: hash join skew yüzünden `growEnabled`'ı
kapatırsa sınırı geçer ([nodeHash.c:1199-1209](../src/backend/executor/nodeHash.c#L1199-L1209)), HashAgg'de
şişen transition değerleri de sınırı zorlar
([nodeAgg.c:222-227](../src/backend/executor/nodeAgg.c#L222-L227)). (3) **Oturum bazında ayarlanabilir** —
tek bir ağır rapor için `SET work_mem = '256MB'` yapıp geri almak, `postgresql.conf`'u global şişirmekten
güvenlidir.

## `log_temp_files` ve `trace_sort`

```c
	if (log_temp_files >= 0)
		if ((size / 1024) >= log_temp_files)
			ereport(LOG, (errmsg("temporary file: path \"%s\", size %lld", path, size)));
```
— [../src/backend/storage/file/fd.c:1520-1526](../src/backend/storage/file/fd.c#L1520-L1526) (girinti ve cast sadeleştirildi)

Varsayılan `-1` (kapalı), `0` her temp dosyayı loglar
([../src/backend/utils/misc/guc_parameters.dat:1904-1912](../src/backend/utils/misc/guc_parameters.dat#L1904-L1912)).
`SET log_temp_files = 0;` (ya da `1024` — yalnızca 1 MB'den büyükler). Kayıt **dosya silinirken** yazılır,
yani sorgu bittikten sonra; faydası `EXPLAIN ANALYZE` çalıştıramadığın üretim sorgularının taştığını
görebilmek.

`SET trace_sort = on; SET client_min_messages = log;` ile tuplesort her önemli anda `elog(LOG, ...)` atar:
`switching to bounded heapsort at N tuples` ([tuplesort.c:1143-1146](../src/backend/utils/sort/tuplesort.c#L1143-L1146)),
`starting quicksort of run N` ([:2245-2247](../src/backend/utils/sort/tuplesort.c#L2245-L2247)) /
`finished writing run N to tape M` ([:2288-2291](../src/backend/utils/sort/tuplesort.c#L2288-L2291)),
`using N KB of memory for tape buffers` ([:1987-1989](../src/backend/utils/sort/tuplesort.c#L1987-L1989)),
`performsort ... done (except N-way final merge)` ([:1345-1347](../src/backend/utils/sort/tuplesort.c#L1345-L1347)).
Kaç run üretildiğini ve kaç merge pass gerektiğini görmenin en doğrudan yolu.

```sql
SELECT datname, temp_files, pg_size_pretty(temp_bytes)   -- ne kadar temp üretildi?
FROM pg_stat_database WHERE temp_files > 0 ORDER BY temp_bytes DESC;

SELECT name, setting, unit FROM pg_settings              -- mevcut ayarlar
WHERE name IN ('work_mem', 'hash_mem_multiplier', 'log_temp_files',
               'enable_hashagg', 'enable_hashjoin');

SET enable_hashjoin = off;   -- merge veya nested loop'a zorlayıp planı karşılaştır
SET enable_sort = off;       -- index tabanlı sıralamaya zorla
```

`enable_*` ayarları düğümü gerçekten kapatmaz — maliyetine devasa bir sabit ekler; alternatifi yoksa plan yine o düğümü kullanır.

---

# Tek sayfalık özet

```
┌───────────────────────────────────────────────────────────────────────┐
│ ORTAK İSKELET                                                         │
│  ExecProcNode(node) → chgParam varsa ExecReScan → node->ExecProcNode   │
│   ilk çağrı ExecProcNodeFirst: instrument? → ExecProcNodeInstr, yoksa  │
│   doğrudan ExecProcNodeReal.  Dönen değer: TupleTableSlot* (NULL=bitti)│
│  STREAMING                     BLOCKING                               │
│  SeqScan, IndexScan            Sort          → tüm girdi + tuplesort   │
│  NestLoop, MergeJoin           Hash          → tüm iç taraf + tablo    │
│  Material, Limit, Memoize      HashAggregate → tüm girdi + tablo       │
│                          (HashJoin: iç blocking, dış streaming)        │
├───────────────────────────────────────────────────────────────────────┤
│ work_mem AŞILDIĞINDA                     │ DURUM MAKİNELERİ           │
│ Sort   TSS_INITIAL ─► TSS_BUILDRUNS      │ HashJoin   8  hj_JoinState │
│          quicksort    run'lar tape'e     │ MergeJoin 11  mj_JoinState │
│          ─performsort─► mergeruns (k-yol)│ Memoize    5  mstatus      │
│ Hash   spaceUsed > spaceAllowed          │ Limit      8  lstate       │
│          ─► nbatch *= 2, yeniden dağıt   │ IncSort    4  exec_status  │
│          nfreed 0 veya hepsi → growEnabled=false (!)                   │
│ HashAgg hash_agg_check_limits ─► spill mode                            │
│          mevcut gruplar güncellenir, YENİ gruplar diske               │
│          partition'lar batch olur, yığından geri okunur (özyineli)     │
│ Memoize mem_used > mem_limit ─► LRU'dan at (diske ASLA yazmaz)         │
│          doldurulan girdi de atılırsa → BYPASS MODE                    │
├───────────────────────────────────────────────────────────────────────┤
│ İFADE DEĞERLENDİRME                                                   │
│  Expr ağacı ─ExecInitExpr─► ExprState{steps[]} ─ExecReadyExpr─►        │
│               execExpr.c     düz dizi, ağaç yok   JIT? / interp        │
│  steps: [OUTER_FETCHSOME][OUTER_VAR][CONST][FUNCEXPR][QUAL][DONE]      │
│  Yorumlayıcı: computed goto (EEO_NEXT / EEO_JUMP); basit desenler      │
│  ExecJustInnerVar / ExecJustConst ile döngü hiç kurulmaz               │
└───────────────────────────────────────────────────────────────────────┘
```

## Akılda kalması gereken 5 şey

1. **Her düğüm `ExecProcNode` ile bir tuple döndürür — ama bu tuple'ı üretmek için ne kadar iş yaptığı düğüme
   göre değişir.** Sort ilk çağrıda tüm alt ağacı tüketir; SeqScan bir sayfa okur. `actual time=X..Y`
   içindeki **X** bu farkı gösterir.
2. **`work_mem` bir hata sınırı değil, strateji değiştirme eşiğidir.** Aşıldığında sorgu başarısız olmaz;
   Sort external merge'e, Hash Join batch'lere, HashAgg spill moduna geçer. Bedeli disk I/O'sudur. Hash
   tabanlı düğümler bu bütçenin `hash_mem_multiplier` katını kullanır.
3. **`EXPLAIN`'de `originally` kelimesini gördüysen planlayıcı yanılmıştır.** `Batches: 8 (originally 1)`:
   planlayıcı iç tarafın belleğe sığacağını sandı, executor sığmadığını çalışırken keşfetti ve üç kez yeniden
   bölmek zorunda kaldı. Çözüm `work_mem` artırmaktan önce `ANALYZE`'dır.
4. **Blocking düğüm hep aynı yerde durmaz.** Hash Join'in *iç* tarafı blocking'dir, *dış* tarafı streaming.
   Bu yüzden Hash Join `LIMIT` ile iyi çalışır, Sort çalışmaz — ama Sort'a `LIMIT` bildirilirse top-N
   heapsort'a geçip diski tamamen atlar.
5. **İfadeler çalışma zamanında ağaç değildir.** `execExpr.c` onları düz bir adım dizisine derler,
   `execExprInterp.c` computed goto ile yürütür, `slot_getsomeattrs` yalnızca gereken sütuna kadar tuple'ı
   açar. Bir sorgunun CPU zamanının büyük kısmı bu döngüde geçer — ve JIT tam olarak burada devreye girer.
