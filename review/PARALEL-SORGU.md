# Paralel sorgu PostgreSQL içinde nasıl çalışır?

*PostgreSQL **20devel** ağacı. Satır numaraları bu sürüme aittir
([meson.build:11](../meson.build#L11)).*

[SELECT-NASIL-CALISIR.md](SELECT-NASIL-CALISIR.md) tek bir backend'in sorguyu
nasıl sürdüğünü anlatıyor. Bu not o boru hattının çatallandığı yeri anlatıyor:
plan ayrı süreçlere nasıl gidiyor, tuple'lar nasıl geri geliyor, worker'da
patlayan hata leader'a nasıl ulaşıyor.

---

## 30 saniyelik özet

```
                       ┌──────────────────────────┐
  İstemci ◄────────────┤  LEADER (normal backend) │
                       │   Gather                 │ ← tuple'ları toplar
                       │     ├─ kendi de çalışır  │   need_to_scan_locally
                       └─────┼────────────────────┘
           ┌─────────────────┴──────────────────┐
           │  DSM segment (dsm_create)          │
           │   serialize PlannedStmt            │
           │   GUC / snapshot / xact / comboCID │
           │   error queue (16K) × N  ─┐        │
           │   tuple queue (64K) × N  ─┤ shm_mq │
           │   instrumentation, DSA area        │
           │   ParallelBlockTableScanDesc       │ ← paylaşılan tarama imleci
           └───┬──────────────┬─────────────┬───┘
         ┌─────▼────┐   ┌─────▼────┐  ┌─────▼────┐
         │ worker 0 │   │ worker 1 │  │ worker 2 │ bgworker süreçleri
         │ Parallel │   │ Parallel │  │ Parallel │ ParallelWorkerMain
         │ SeqScan  │   │ SeqScan  │  │ SeqScan  │   → ParallelQueryMain
         └──────────┘   └──────────┘  └──────────┘
               └──── tuple queue ────────┘
```

1. **Paralellik plan ağacının tabanında olur, tepesinde değil.** Alt kısım N
   kopya çalışır; `Gather` bu N akışı tek akışa indirir. Üstü tek süreçtir.
2. **Worker ayrı bir süreçtir, thread değil.** Leader'ın bütün durumu (GUC,
   snapshot, XID'ler, kütüphaneler, combo CID) DSM'e serialize edilir;
   kopyalanamayan her şey paralelliği yasaklama sebebidir.
3. **İki soru ayrıdır: "güvenli mi?" ve "değer mi?"** Güvenlik `proparallel` ile
   statik; değerlilik maliyet modeliyle (`parallel_setup_cost` 1000 peşin ceza,
   `parallel_tuple_cost` 0.1 tuple başına iletişim).
4. **Paralel mod = salt okunur mod.** Worker XID alamaz, DDL çalıştıramaz, GUC
   değiştiremez — yeni combo CID üretilse bildirmenin yolu yok.
5. **Worker sayısı bir istek, garanti değil.** *Workers Planned* planlayıcının
   istediği, *Workers Launched* postmaster'ın verdiğidir.
6. **Leader de çalışır, ve bu iki ucu keskin bıçaktır.** Boş beklemek yerine
   planın kopyasını sürer; uzun bir yerel işe saplanırsa worker'ların 64 KB'lık
   kuyrukları dolar ve onlar da bloke olur.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/access/transam/README.parallel](../src/backend/access/transam/README.parallel) | Tasarım belgesi. Önce bu okunur |
| [../src/backend/access/transam/parallel.c](../src/backend/access/transam/parallel.c) | Altyapı: `ParallelContext`, DSM, worker başlatma, hata iletimi. **En önemli dosya** |
| [../src/backend/executor/execParallel.c](../src/backend/executor/execParallel.c) | Executor kısmı: plan serileştirme, tuple queue, instrumentation |
| [../src/backend/executor/nodeGather.c](../src/backend/executor/nodeGather.c) + [nodeGatherMerge.c](../src/backend/executor/nodeGatherMerge.c) | Tuple toplama; ikincisi sırayı binary heap ile korur |
| [../src/backend/executor/tqueue.c](../src/backend/executor/tqueue.c) + [shm_mq.c](../src/backend/storage/ipc/shm_mq.c) | Tuple kuyruğu ve altındaki tek yazar/tek okuyucu halka tampon |
| [../src/backend/storage/ipc/dsm.c](../src/backend/storage/ipc/dsm.c) | Dinamik paylaşımlı bellek segmentleri |
| [../src/backend/optimizer/path/allpaths.c](../src/backend/optimizer/path/allpaths.c) | `consider_parallel`, partial path, `compute_parallel_worker` |
| [../src/backend/optimizer/util/clauses.c](../src/backend/optimizer/util/clauses.c) | `max_parallel_hazard` — sorgu paralel olabilir mi? |
| [../src/backend/optimizer/path/costsize.c](../src/backend/optimizer/path/costsize.c) | `cost_gather`, `get_parallel_divisor` |
| [../src/backend/access/table/tableam.c](../src/backend/access/table/tableam.c) | Paralel seq scan'in blok paylaştırma mantığı |
| [../src/backend/executor/nodeHash.c](../src/backend/executor/nodeHash.c) | Paralel hash table inşası, barrier'lar |
| [../src/backend/commands/vacuumparallel.c](../src/backend/commands/vacuumparallel.c) | Aynı altyapının VACUUM tarafındaki kullanıcısı |

---

# 1. Bu sorgu paralel olabilir mi?

```c
	if ((cursorOptions & CURSOR_OPT_PARALLEL_OK) != 0 &&
		IsUnderPostmaster &&
		parse->commandType == CMD_SELECT &&
		!parse->hasModifyingCTE &&
		max_parallel_workers_per_gather > 0 &&
		!IsParallelWorker())
	{
		/* all the cheap tests pass, so scan the query tree */
		glob->maxParallelHazard = max_parallel_hazard(parse);
		glob->parallelModeOK = (glob->maxParallelHazard != PROPARALLEL_UNSAFE);
	}
```
— [../src/backend/optimizer/plan/planner.c:413-423](../src/backend/optimizer/plan/planner.c#L413-L423)

Beş ucuz test: standalone backend değil, komut `SELECT`, veri değiştiren CTE
yok, GUC sıfır değil, ve **zaten bir worker içinde değiliz**. (`CREATE TABLE AS`
istisnası yorumda: yazılan tablo yeni ve worker'lar onu göremiyor,
[:399-406](../src/backend/optimizer/plan/planner.c#L399-L406).)

Testler geçilirse tüm ağaç gezilir
([clauses.c:762](../src/backend/optimizer/util/clauses.c#L762)).
`pg_proc.proparallel` üç değer alır
([pg_proc.h:177-179](../src/include/catalog/pg_proc.h#L177-L179)): **`s` safe**
— her yerde çalışır; **`r` restricted** — çalışır ama `Gather`'ın *altına*
itilemez, yukarıda kalır; **`u` unsafe** — tek tanesi tüm sorgunun paralelliğini
iptal eder, tarama ilk unsafe yapıda durur
([:838-841](../src/backend/optimizer/util/clauses.c#L838-L841)).
`CREATE FUNCTION` ile tanımlanan kullanıcı fonksiyonları varsayılan **unsafe**.
Walker fonksiyon OID'lerinin dışında belli node tiplerini de sabit etiketler:

| Node | Seviye | Gerekçe (koddaki yorumdan) | Satır |
|---|---|---|---|
| `CoerceToDomain` | restricted | Domain constraint'i unsafe fonksiyon içerebilir | [:879-883](../src/backend/optimizer/util/clauses.c#L879-L883) |
| `NextValueExpr` | **unsafe** | Sequence ilerletmek yazmadır | [:885-889](../src/backend/optimizer/util/clauses.c#L885-L889) |
| `WindowFunc` | restricted | Girdi satır sırası belirleyici değil | [:899-903](../src/backend/optimizer/util/clauses.c#L899-L903) |
| `Param` (PARAM_EXEC) | restricted | Worker'a param geçirilemiyor | [:960-974](../src/backend/optimizer/util/clauses.c#L960-L974) |
| `SubPlan` (unsafe ise) | restricted | Sadece güvenli subplan worker'a gider | [:931-951](../src/backend/optimizer/util/clauses.c#L931-L951) |
| `Query` + `rowMarks` | **unsafe** | `SELECT ... FOR UPDATE/SHARE` | [:985-990](../src/backend/optimizer/util/clauses.c#L985-L990) |

Sorgu güvenli olsa bile her range table girdisi ayrıca elenir
([allpaths.c:649](../src/backend/optimizer/path/allpaths.c#L649)): **geçici
tablo** — worker leader'ın local buffer'ına erişemez
([:679-680](../src/backend/optimizer/path/allpaths.c#L679-L680)); `LIMIT` içeren
subquery ([:736-748](../src/backend/optimizer/path/allpaths.c#L736-L748));
**CTE scan** — "CTE tuplestores aren't shared among parallel workers"
([:772-781](../src/backend/optimizer/path/allpaths.c#L772-L781));
`RTE_TABLEFUNC`; ve `IsForeignScanParallelSafe` demeyen FDW'ler
([:704-711](../src/backend/optimizer/path/allpaths.c#L704-L711)).

---

# 2. Partial path, worker sayısı, maliyet

```
 *	  A partial path is one which can be executed in any number of workers in
 *	  parallel such that each worker will generate a subset of the path's
 *	  overall result.
```
— [../src/backend/optimizer/util/pathnode.c:756-759](../src/backend/optimizer/util/pathnode.c#L756-L759)

Partial path tek başına **yanlış** cevap verir; doğru cevap ancak tüm kopyalar
birleşince çıkar. Aynı yorum parametreli partial path'in neden üretilmediğini de
açıklıyor: worker'ların aynı anda aynı parametre değeriyle çalıştığını garanti
edecek mekanizma yok
([:766-777](../src/backend/optimizer/util/pathnode.c#L766-L777)).

**Worker sayısı sayfa sayısının 3 tabanında logaritmasıdır:**

```c
			heap_parallel_threshold = Max(min_parallel_table_scan_size, 1);
			while (heap_pages >= (BlockNumber) (heap_parallel_threshold * 3))
			{
				heap_parallel_workers++;
				heap_parallel_threshold *= 3;
```
— [../src/backend/optimizer/path/allpaths.c:5011-5015](../src/backend/optimizer/path/allpaths.c#L5011-L5015)

`min_parallel_table_scan_size` 8 MB olduğuna göre: 8 MB → 1 worker, 24 MB → 2,
72 MB → 3, 216 MB → 4; sonunda `max_parallel_workers_per_gather` (varsayılan
**2**) ile kırpılır ([:5046](../src/backend/optimizer/path/allpaths.c#L5046)).
`ALTER TABLE ... SET (parallel_workers = N)` varsa logaritma hiç hesaplanmaz
([:4982-4983](../src/backend/optimizer/path/allpaths.c#L4982-L4983)). Index
scan'de heap ve index sayfa sayılarının **küçüğü** alınır
([:5038-5041](../src/backend/optimizer/path/allpaths.c#L5038-L5041); çağrı yeri
[costsize.c:765-778](../src/backend/optimizer/path/costsize.c#L765-L778)),
bitmap için `create_partial_bitmap_paths`
([:4936](../src/backend/optimizer/path/allpaths.c#L4936)).

`Gather` çıktısı sırasız olduğu için sadece **en ucuz** partial path ilginç
([:3276-3281](../src/backend/optimizer/path/allpaths.c#L3276-L3281)); sıralı
olanlar için `Gather Merge` denenir
([:3287-3299](../src/backend/optimizer/path/allpaths.c#L3287-L3299)), ve
`generate_useful_gather_paths` ([:3392](../src/backend/optimizer/path/allpaths.c#L3392))
partial path'in üstüne Sort ekleyip Gather Merge deniyor — sıralamayı
`Gather`'ın altına, yani paralele itmek için
([:3474-3493](../src/backend/optimizer/path/allpaths.c#L3474-L3493)). Alt plan
paralel-aware değilse `create_gather_path` **single-copy**'ye düşer:
`num_workers = 1`, leader hiç çalışmaz, sıralama korunur
([pathnode.c:1886-1891](../src/backend/optimizer/util/pathnode.c#L1886-L1891)).

**Maliyet iki GUC'la yazılır:**

```c
	/* Parallel setup and communication cost. */
	startup_cost += parallel_setup_cost;
	run_cost += parallel_tuple_cost * path->path.rows;
```
— [../src/backend/optimizer/path/costsize.c:450-452](../src/backend/optimizer/path/costsize.c#L450-L452)

Varsayılanlar 1000.0 ve 0.1
([cost.h:29-30](../src/include/optimizer/cost.h#L29-L30)). `seq_page_cost` 1.0
olduğuna göre paralel planın karlı çıkması için ~1000 sayfa okumadan tasarruf
etmesi gerekir — küçük tablolarda paralelliğin seçilmemesinin asıl sebebi budur.
`Gather Merge` her worker'dan tuple beklemek zorunda olduğu için %5 daha pahalı
sayılır ([:512-519](../src/backend/optimizer/path/costsize.c#L512-L519)).
Partial path'in satır/maliyet tahminleri ise bir bölene bölünür:

```c
		leader_contribution = 1.0 - (0.3 * path->parallel_workers);
		if (leader_contribution > 0)
			parallel_divisor += leader_contribution;
```
— [:6660-6662](../src/backend/optimizer/path/costsize.c#L6660-L6662)

Yorum modeli açıklıyor: leader her worker'a hizmet için zamanının %30'unu
harcıyor sayılır. 1 worker → bölen 1.7; 2 → 2.4; 3 → 3.1; **4'ten sonra leader
hiç sayılmaz** ([:6645-6655](../src/backend/optimizer/path/costsize.c#L6645-L6655)).
Ters yön `compute_gather_rows`
([:6792](../src/backend/optimizer/path/costsize.c#L6792)).

---

# 3. Altyapı: DSM segment ve `ParallelContext`

`README.parallel` DSM'in üç şey içerdiğini söylüyor: hata taşıyan bir `shm_mq`,
leader'ın private state'inin serialize hali, ve kullanıcıya özel veri
([:12-25](../src/backend/access/transam/README.parallel#L12-L25)); kullanım
kalıbı da orada, düz kod olarak
([:206-230](../src/backend/access/transam/README.parallel#L206-L230)). DSM tek
seferde ve sabit boyutta oluşturulur, bu yüzden **önce** herkes ne kadar yer
istediğini bildirir:

```c
	segsize = shm_toc_estimate(&pcxt->estimator);
	if (pcxt->nworkers > 0)
		pcxt->seg = dsm_create(segsize, DSM_CREATE_NULL_IF_MAXSEGMENTS);
	...
	else
	{
		pcxt->nworkers = 0;
		pcxt->private_memory = MemoryContextAlloc(TopMemoryContext, segsize);
```
— [../src/backend/access/transam/parallel.c:328-338](../src/backend/access/transam/parallel.c#L328-L338)

DSM oluşturulamazsa hata verilmez: paralellik sessizce iptal edilir, aynı bellek
backend-private ayrılır. Bu, altyapının genel felsefesi — **her başarısızlık
seri çalışmaya düşmekle çözülür**. Segment içi adresleme `shm_toc` ile, 64
bitlik `PARALLEL_KEY_*` anahtarlarıyla yapılır (GUC, combo CID, iki snapshot,
transaction state, relmapper, reindex, kütüphaneler, entrypoint …
[:67-81](../src/backend/access/transam/parallel.c#L67-L81)); serileştirme kodu
tek blok halinde [:384-456](../src/backend/access/transam/parallel.c#L384-L456).

Worker'a geçirilen **tek argüman DSM handle**'ıdır
([:613-615](../src/backend/access/transam/parallel.c#L613-L615)); gerisi
segmentten okunur. Fonksiyon işaretçisi geçirilmiyor — `EXEC_BACKEND`
derlemelerinde adresler farklı olabileceği için kütüphane ve fonksiyon *adı*
yazılıyor ([:484-496](../src/backend/access/transam/parallel.c#L484-L496)).
Kayıt döngüsü ilk başarısızlıktan sonra kalanları denemez ama yine de döngüden
geçer: o worker'lar için ayrılmış error queue'ları "unutmak" gerekiyor, yoksa
leader hiç gelmeyecek worker'ları beklerdi
([:639-651](../src/backend/access/transam/parallel.c#L639-L651)). Asıl sınır
postmaster'da: `max_parallel_workers` (8) **tüm sunucu genelinde** aktif paralel
worker sayısını sınırlar ve `max_worker_processes` (8) havuzundan harcanır
([bgworker.c:1103-1112](../src/backend/postmaster/bgworker.c#L1103-L1112)).

`LaunchParallelWorkers`'ın ilk işi `BecomeLockGroupLeader()`
([:595](../src/backend/access/transam/parallel.c#L595)): leader ve worker'lar
aynı **lock group**'a girer, birbirlerinin heavyweight kilitleriyle çakışmazlar
— bu olmadan tespit edilemeyen deadlock oluşurdu.

---

# 4. Hata iletimi: worker'daki `ERROR` leader'a nasıl gidiyor?

Worker attach olur olmaz **tüm protokol çıktısını** bir `shm_mq`'ya yönlendirir:

```c
	mq = (shm_mq *) (error_queue_space +
					 ParallelWorkerNumber * PARALLEL_ERROR_QUEUE_SIZE);
	shm_mq_set_sender(mq, MyProc);
	mqh = shm_mq_attach(mq, seg, NULL);
	pq_redirect_to_shm_mq(seg, mqh);
```
— [../src/backend/access/transam/parallel.c:1379-1383](../src/backend/access/transam/parallel.c#L1379-L1383)

Bundan sonra her `ereport` normal bir `ErrorResponse` üretir, ama soket yerine
kuyruğa yazılır. Kuyruk 16 KB
([:57](../src/backend/access/transam/parallel.c#L57)) — bir `ErrorResponse` tek
seferde sığsın, worker bloke olmadan yazıp ölebilsin diye. Bu noktadan **önce**
çıkan hatalar leader'a ulaşmaz; leader eksik worker'a zaten hazır olmak zorunda
([README.parallel:30-37](../src/backend/access/transam/README.parallel#L30-L37)).

```
worker: ereport(ERROR) → shm_mq → leader'a PROCSIG_PARALLEL_MESSAGE
leader: HandleParallelMessageInterrupt()  parallel.c:1046  (sadece bayrak)
   ↓ bir sonraki CHECK_FOR_INTERRUPTS()
        ProcessParallelMessages()         parallel.c:1057
          shm_mq_receive()                parallel.c:1113
          ProcessParallelMessage()        parallel.c:1146
            ThrowErrorData(&edata)        parallel.c:1199
```

Yeniden fırlatmada iki ince ayrıntı:

```c
				/* Death of a worker isn't enough justification for suicide. */
				edata.elevel = Min(edata.elevel, ERROR);
```
— [:1170-1171](../src/backend/access/transam/parallel.c#L1170-L1171)

Worker `FATAL` verdiyse leader bunu `ERROR`'a indirger. Ve mesaja bağlam
eklenir — **`CONTEXT: parallel worker` satırını gördüğünüzde hata başka bir
süreçte doğmuş** demektir
([:1181-1187](../src/backend/access/transam/parallel.c#L1181-L1187));
`debug_parallel_query = regress` modunda test çıktısı kararsız olmasın diye bu
satır eklenmez. Aynı kuyruk `NOTICE`, `NOTIFY` ve progress raporlarını da taşır
([:1207-1239](../src/backend/access/transam/parallel.c#L1207-L1239)); temiz
biten worker `PqMsg_Terminate` gönderir
([:1586](../src/backend/access/transam/parallel.c#L1586)).

Hiç mesaj vermeden ölen worker'ı `WaitForParallelWorkersToFinish` yakalar:
kuyruğa hiç sender bağlanmamışsa `"parallel worker failed to initialize"`
verilir ([:876-881](../src/backend/access/transam/parallel.c#L876-L881)). Hata
durumunda `DestroyParallelContext` kalanları `TerminateBackgroundWorker` ile
öldürür ve **kesintiye kapalı** biçimde çıkmalarını bekler
([:971-1014](../src/backend/access/transam/parallel.c#L971-L1014)) — çünkü
leader transaction'ı geri alırken worker hâlâ o transaction'ın oluşturduğu
tabloyu tarıyor olabilir
([README.parallel:176-181](../src/backend/access/transam/README.parallel#L176-L181)).

---

# 5. Plan serileştirme ve worker tarafı

`ExecInitParallelPlan`
([../src/backend/executor/execParallel.c:653](../src/backend/executor/execParallel.c#L653))
önce planı metne çevirir, sonra paralel context'i kurar
([:696-700](../src/backend/executor/execParallel.c#L696-L700)).
`ExecSerializePlan` ([:152](../src/backend/executor/execParallel.c#L152)) plan
ağacının `Gather` altındaki kısmını sahte bir `PlannedStmt`'e sarıp
`nodeToString` ile serialize eder, iki düzeltme yaparak. **Resjunk hilesi:**
worker'ın executor'ı resjunk sütun görürse junk filter takıp onları atardı, ama
tuple'lar kullanıcıya değil başka bir backend'e gidiyor ve orada gerekli
olabilirler — hepsi `resjunk = false` yapılıyor, yorum bunun hack olduğunu kabul
ediyor ([:169-174](../src/backend/executor/execParallel.c#L169-L174)).
**Güvensiz subplan'lar** yerine listede `NULL` bırakılıyor (indeksler korunsun
diye), böylece worker onları `ExecInitNode` bile edemez
([:206-214](../src/backend/executor/execParallel.c#L206-L214)).

| DSM anahtarı | İçerik | Satır |
|---|---|---|
| `EXECUTOR_FIXED` | `tuples_needed`, eflags, jit flags, DSA param pointer'ı | [:808-813](../src/backend/executor/execParallel.c#L808-L813) |
| `QUERY_TEXT` | Sorgu metni (worker `pg_stat_activity`'de görünsün) | [:816-818](../src/backend/executor/execParallel.c#L816-L818) |
| `PLANNEDSTMT` | `nodeToString` çıktısı | [:821-823](../src/backend/executor/execParallel.c#L821-L823) |
| `PARAMLISTINFO` | `$1`, `$2` … dış parametreler | [:826-828](../src/backend/executor/execParallel.c#L826-L828) |
| `BUFFER_USAGE` / `WAL_USAGE` | Worker başına sayaç dizisi | [:831-840](../src/backend/executor/execParallel.c#L831-L840) |
| `TUPLE_QUEUE` | Worker başına 64 KB `shm_mq` | [:843](../src/backend/executor/execParallel.c#L843) |
| `INSTRUMENTATION` | `EXPLAIN ANALYZE` için düğüm × worker matrisi | [:858-867](../src/backend/executor/execParallel.c#L858-L867) |
| `DSA` | `dsa_create_in_place` ile dinamik ayırıcı alanı | [:892-896](../src/backend/executor/execParallel.c#L892-L896) |

Son satır önemli: **DSA (dynamic shared area)** DSM'in içine yerleştirilmiş bir
`palloc` benzeri ayırıcıdır. Paralel hash join'in hash tablosu, paralel bitmap
scan'in iterator'ı gibi *çalışma zamanında boyutu bilinmeyen* yapılar buraya
konur. Paralel-aware düğümlerin talepleri iki geçişte toplanır:
`ExecParallelEstimate` ([:246](../src/backend/executor/execParallel.c#L246))
"ne kadar istersin" diye sorar, `ExecParallelInitializeDSM`
([:480](../src/backend/executor/execParallel.c#L480)) doldurur; aynı
`switch (nodeTag(planstate))` listesi paralel-aware olabilen düğümleri gösterir:
SeqScan, IndexScan, IndexOnlyScan, ForeignScan, TidRangeScan, Append,
CustomScan, BitmapHeapScan, HashJoin
([:254-343](../src/backend/executor/execParallel.c#L254-L343)).

Worker tarafında `ParallelWorkerMain`
([parallel.c:1301](../src/backend/access/transam/parallel.c#L1301)) durumu sıra
ile kurar ve sıra keyfî değil: DSM'e attach ve `FixedParallelState`
([:1352-1365](../src/backend/access/transam/parallel.c#L1352-L1365)) → error
queue → **lock group'a katıl** (heavyweight kilit almadan *önce*,
[:1401-1403](../src/backend/access/transam/parallel.c#L1401-L1403)) → user ID ve
DB bağlantısı (yetki kontrolü **yapılmaz**, leader zaten yapmıştı,
[:1429-1443](../src/backend/access/transam/parallel.c#L1429-L1443)) →
kütüphaneler, xact state, relmapper/reindex/comboCID, snapshot'lar,
`InvalidateSystemCaches()`, GUC'lar (DB'ye bağlandıktan *sonra*, katalog
erişimli check hook'lar için)
([:1460-1524](../src/backend/access/transam/parallel.c#L1460-L1524)) →
`EnterParallelMode()` ve giriş noktası
([:1566-1571](../src/backend/access/transam/parallel.c#L1566-L1571)).

Giriş noktası `ParallelQueryMain`
([execParallel.c:1514](../src/backend/executor/execParallel.c#L1514)).
`ExecParallelGetQueryDesc` ([:1326](../src/backend/executor/execParallel.c#L1326))
planı `stringToNode` ile geri kurup normal bir `QueryDesc` üretir; buradan
sonrası `ExecutorStart` / `ExecutorRun` / `ExecutorFinish`, yani **sıradan
executor**dur ([:1552-1588](../src/backend/executor/execParallel.c#L1552-L1588)).
Tek fark `DestReceiver`: `printtup` yerine tuple queue; her worker
`ParallelWorkerNumber` ile kendi 64 KB'lık dilimini seçer
([:1315-1319](../src/backend/executor/execParallel.c#L1315-L1319)). Bittiğinde
buffer/WAL sayaçları ve instrumentation DSM'e yazılır
([:1590-1607](../src/backend/executor/execParallel.c#L1590-L1607)); leader
bunları `ExecParallelFinish` / `ExecParallelCleanup`'ta toplar
([:1221](../src/backend/executor/execParallel.c#L1221),
[:1274](../src/backend/executor/execParallel.c#L1274)).

---

# 6. Snapshot, transaction, ve neden yazma yok

```c
	asnapshot = RestoreSnapshot(asnapspace);
	tsnapshot = tsnapspace ? RestoreSnapshot(tsnapspace) : asnapshot;
	RestoreTransactionSnapshot(tsnapshot,
							   fps->parallel_leader_pgproc);
	PushActiveSnapshot(asnapshot);
```
— [../src/backend/access/transam/parallel.c:1504-1508](../src/backend/access/transam/parallel.c#L1504-L1508)

Transaction snapshot yalnızca `REPEATABLE READ` / `SERIALIZABLE`'da serialize
edilir ([:403-409](../src/backend/access/transam/parallel.c#L403-L409));
`READ COMMITTED`'de active snapshot ikisinin yerine geçer. Sonuç aynı: **leader
ve tüm worker'lar tam olarak aynı tuple'ları görür.** Kopyalanan durumun tam
listesi `README.parallel`'in "State Sharing" bölümünde
([:88-130](../src/backend/access/transam/README.parallel#L88-L130)).

Yazma yasağının üç savunma katmanı var. **Planlayıcı**
`commandType == CMD_SELECT` değilse paralelliği düşünmez
([planner.c:415](../src/backend/optimizer/plan/planner.c#L415)). **XID ataması**
engellenir — yazma için XID gerekir, XID alınamıyorsa yazma da yok:

```c
	if (IsInParallelMode() || IsParallelWorker())
		ereport(ERROR,
				(errcode(ERRCODE_INVALID_TRANSACTION_STATE),
				 errmsg("cannot assign transaction IDs during a parallel operation")));
```
— [../src/backend/access/transam/xact.c:651-654](../src/backend/access/transam/xact.c#L651-L654)
(commit/abort/savepoint için de aynı yasak,
[:4219](../src/backend/access/transam/xact.c#L4219),
[:4441](../src/backend/access/transam/xact.c#L4441)). **Utility komutları**
`PreventCommandIfParallelMode` ile durdurulur
([utility.c:426-435](../src/backend/tcop/utility.c#L426-L435)); `SET` bile yasak
([guc.c:3374](../src/backend/utils/misc/guc.c#L3374)), çünkü GUC'lar worker
başlarken kopyalanıyor. Bu kısıtlar worker'a değil, **paralel modda olan
leader'a da** uygulanır — `EnterParallelMode` `ExecutePlan`'da çağrılıyor
([execMain.c:1758-1760](../src/backend/executor/execMain.c#L1758-L1760)).

Asıl gerekçe `README.parallel`'de:

```
 *  - The combo CID mappings. ... The need to synchronize this data structure is
 *    a major reason why we can't support writes in parallel mode: such writes
 *    might create new combo CIDs, and we have no way to let other workers
 *    (or the initiating backend) know about them.
```
— [README.parallel:108-112](../src/backend/access/transam/README.parallel#L108-L112)

Bir ek kısıt: `ExecutePlan` paralel modu yalnızca planın **tamamı**
çalıştırılacaksa açar
([execMain.c:1752-1755](../src/backend/executor/execMain.c#L1752-L1755)) — bu
yüzden bir cursor'dan tek tek `FETCH` yapmak paralelliği kapatır.

---

# 7. `Gather` ve `Gather Merge`

Worker'lar düğüm başlatılırken değil, **ilk tuple istendiğinde** başlatılır —
DSM segmenti büyük, gerekmedikçe ayrılmasın diye
([nodeGather.c:146-184](../src/backend/executor/nodeGather.c#L146-L184)). Hiç
worker gelmezse plan yine çalışır:

```c
		/* Run plan locally if no workers or enabled and not single-copy. */
		node->need_to_scan_locally = (node->nreaders == 0)
			|| (!gather->single_copy && parallel_leader_participation);
```
— [:213-215](../src/backend/executor/nodeGather.c#L213-L215)

Tuple toplamada sıra bellidir — **önce worker'lar, sonra kendisi**
([:271-295](../src/backend/executor/nodeGather.c#L271-L295)); böylece kuyruklar
boşaltılıp worker'ların tıkanması engelleniyor. `gather_readnext` round-robin
ama "yapışkan": bloke olmadan okunabildiği sürece aynı kuyruk okunur, çünkü
sürekli dönmenin verimsiz olduğu görülmüş
([:363-368](../src/backend/executor/nodeGather.c#L363-L368)). Tümü boşsa latch'te
uyunur (`WAIT_EVENT_EXECUTE_GATHER`,
[:386-387](../src/backend/executor/nodeGather.c#L386-L387)).

Kuyruk `shm_mq`: tek yazar / tek okuyucu halka tampon, sayaçlar mutex yerine
atomik 8 baytlık okuma/yazma ve bellek bariyerleriyle korunuyor
([shm_mq.c:42-46](../src/backend/storage/ipc/shm_mq.c#L42-L46)). Üstündeki ince
katman `tqueue.c`; tuple `MinimalTuple` olarak olduğu gibi kopyalanıyor, ayrı
bir serileştirme formatı yok
([tqueue.c:62-63](../src/backend/executor/tqueue.c#L62-L63)).

**`Gather Merge`** her kaynak (leader dahil) için bir slot tutar ve binary heap
ile en küçüğü seçer
([nodeGatherMerge.c:430-432](../src/backend/executor/nodeGatherMerge.c#L430-L432)).
Başlatmada **her kaynaktan en az bir tuple** gelmelidir — önce `nowait` modda
denenir, eksik kalırsa bloke ederek tekrarlanır
([:477-513](../src/backend/executor/nodeGatherMerge.c#L477-L513)); maliyet
modelindeki %5 farkın sebebi bu. Worker başına 10 tuple'lık tampon tutulur
(`MAX_TUPLE_STORE`, [:33](../src/backend/executor/nodeGatherMerge.c#L33)).

**Leader'ın katılımının dezavantajı** iki yönlü. *Tıkanma:* leader
`ExecProcNode(outerPlan)` içinde uzun kalırsa kuyrukları boşaltmaz, 64 KB
dolunca worker'lar `shm_mq_send`'de bloke olur. *Maliyet iyimserliği:*
`get_parallel_divisor` leader'ı kısmi bir worker sayıyor; tıkandığında bu
varsayım tutmaz. `SET parallel_leader_participation = off` yapıldığında leader
sadece toplar — küçük `LIMIT`'li sorgularda genelde iyileşme görülür.

---

# 8. Paralel seq scan: blok bazlı iş paylaşımı

Plan aynı; paylaşılan tek şey **bir sonraki okunacak blok numarası**dır:

```c
typedef struct ParallelBlockTableScanDescData
{
	ParallelTableScanDescData base;

	BlockNumber phs_nblocks;	/* # blocks in relation at start of scan */
	slock_t		phs_mutex;		/* mutual exclusion for setting startblock */
	BlockNumber phs_startblock; /* starting block number */
	BlockNumber phs_numblock;	/* # blocks to scan, or InvalidBlockNumber if
								 * no limit */
	pg_atomic_uint64 phs_nallocated;	/* number of blocks allocated to
										 * workers so far. */
}			ParallelBlockTableScanDescData;
```
— [../src/include/access/relscan.h:97-108](../src/include/access/relscan.h#L97-L108)

Yapı DSM'e `ExecSeqScanInitializeDSM` tarafından, düğümün `plan_node_id`'si
anahtar olarak konur
([nodeSeqscan.c:404-408](../src/backend/executor/nodeSeqscan.c#L404-L408)).
**Neden blok blok değil, chunk chunk?** İlk sürümlerde her worker sıradaki tek
bloğu alıyordu; kodda bunun neden terk edildiği yazıyor:

```
	 * result in workers never receiving consecutive block numbers.  Some
	 * operating systems would not detect the sequential I/O pattern due to
	 * each backend being a different process which could result in poor
	 * performance due to inefficient or no readahead.
```
— [../src/backend/access/table/tableam.c:561-564](../src/backend/access/table/tableam.c#L561-L564)

Çözüm ardışık blok **aralığı** (chunk). Boyut tablo ~1024-2048 parçaya
bölünecek şekilde seçilir (`PARALLEL_SEQSCAN_NCHUNKS` 2048,
[:42](../src/backend/access/table/tableam.c#L42)) ve üstten sınırlanır
([:526-536](../src/backend/access/table/tableam.c#L526-L536)); dağıtım kilitsiz,
tek `pg_atomic_fetch_add_u64` ile
([:632-634](../src/backend/access/table/tableam.c#L632-L634)). Sayaç 64 bit,
çünkü tarama bitse bile worker'lar artırmaya devam ediyor ve taşma olmamalı
([:586-591](../src/backend/access/table/tableam.c#L586-L591)); sonlara doğru
chunk boyutu yarılanıyor ki son işler eşit dağılsın
([:627-630](../src/backend/access/table/tableam.c#L627-L630)). Sonuç: **iş
paylaşımı statik değil, dinamik.** Yavaş worker az chunk alır, hızlı olan çok —
bu yüzden `EXPLAIN ANALYZE`'da worker satır sayıları eşit çıkmaz.

---

# 9. Paralel index scan ve bitmap heap scan

**B-tree.** Paylaşılan durum sayfa numarası + bir durum makinesi
(`BTPARALLEL_NOT_INITIALIZED` / `NEED_PRIMSCAN` / `ADVANCING` / `IDLE` / `DONE`,
[nbtree.c:56-63](../src/backend/access/nbtree/nbtree.c#L56-L63)); yapı
`btps_nextScanPage`, `btps_lastCurrPage`, `btps_pageStatus`, bir `LWLock` ve bir
`ConditionVariable` tutar
([:69-78](../src/backend/access/nbtree/nbtree.c#L69-L78)). Seq scan'den farkı:
index yaprakları **zincirle** bağlı, sonraki sayfa numarası ancak mevcut sayfa
okunduktan sonra bilinir. Atomik sayaç yetmiyor, sırayla "taramayı ele
geçirmek" gerekiyor:

```c
		else if (btscan->btps_pageStatus != BTPARALLEL_ADVANCING)
		{
			btscan->btps_pageStatus = BTPARALLEL_ADVANCING;
			*next_scan_page = btscan->btps_nextScanPage;
			...
		}
		LWLockRelease(&btscan->btps_lock);
		if (exit_loop || !status)
			break;
		ConditionVariableSleep(&btscan->btps_cv, WAIT_EVENT_BTREE_PAGE);
```
— [:966-981](../src/backend/access/nbtree/nbtree.c#L966-L981)

Ele geçiren worker sayfayı okur, sağ kardeş bağını `btps_nextScanPage`'e yazar
ve `_bt_parallel_release` ile sırayı bırakır
([:1011](../src/backend/access/nbtree/nbtree.c#L1011)); paylaşım birimi
**yaprak sayfa**dır.

**Bitmap heap scan** asimetriktir: bitmap **tek** süreçte kurulur
(`BM_INITIAL` → `BM_INPROGRESS` → `BM_FINISHED`,
[nodeBitmapHeapscan.c:62-69](../src/backend/executor/nodeBitmapHeapscan.c#L62-L69)),
sonra hep birlikte gezilir:

```c
	else if (BitmapShouldInitializeSharedState(pstate))
	{
		/*
		 * The leader will immediately come out of the function, but others
		 * will be blocked until leader populates the TBM and wakes them up.
		 */
		node->tbm = (TIDBitmap *) MultiExecProcNode(outerPlanState(node));
		...
		pstate->tbmiterator = tbm_prepare_shared_iterate(node->tbm);
```
— [:115-130](../src/backend/executor/nodeBitmapHeapscan.c#L115-L130)

Yani `Parallel Bitmap Heap Scan` planında **index taraması paralel değildir** —
biri bitmap'i kurarken diğerleri `wait_event = 'ParallelBitmapScan'` durumunda
bekler. Paralel olan kısım heap okumasıdır; iterator DSA'da paylaşılır.

---

# 10. Paralel hash join

Tasarımın tamamı `nodeHashjoin.c`'nin başındaki "PARALLELISM" bölümünde
([:58-158](../src/backend/executor/nodeHashjoin.c#L58-L158)). İki farklı şey
var: **parallel-oblivious** — her backend inner planın kopyasını çalıştırıp
kendi hash tablosunu kurar, bellek N kat harcanır (`Hash Join` + `Hash`); ve
**parallel-aware** — tek paylaşılan hash tablosu, hepsi birlikte kurar
(`Parallel Hash Join` + `Parallel Hash`). Senkronizasyon `Barrier` primitifiyle:
tüm katılımcılar varınca faz ilerler ve hepsi serbest kalır
([:70-76](../src/backend/executor/nodeHashjoin.c#L70-L76)).

```c
/* The phases for building batches, used by build_barrier. */
#define PHJ_BUILD_ELECT					0
#define PHJ_BUILD_ALLOCATE				1
#define PHJ_BUILD_HASH_INNER			2
#define PHJ_BUILD_HASH_OUTER			3
#define PHJ_BUILD_RUN					4
#define PHJ_BUILD_FREE					5
```
— [../src/include/executor/hashjoin.h:279-285](../src/include/executor/hashjoin.h#L279-L285)
(batch probe için ayrıca `PHJ_BATCH_ELECT` … `PHJ_BATCH_FREE`,
[:287-293](../src/include/executor/hashjoin.h#L287-L293))

Yıldızlı fazlar (`ALLOCATE`, `SCAN`, `FREE`) tek bir keyfî seçilmiş süreç
tarafından yapılır. Her batch'in **kendi bariyeri** vardır:
`ParallelHashJoinBatch` DSA pointer'ları, `batch_barrier` ve peşinden gelen
değişken boyutlu `SharedTuplestore` nesnelerini tutar
([:173-190](../src/include/executor/hashjoin.h#L173-L190)); üst seviye
`ParallelHashJoinState` batch dizisi, büyüme bayrağı ve üç bariyeri
(`build_barrier`, `grow_batches_barrier`, `grow_buckets_barrier`) barındırır
([:257-277](../src/include/executor/hashjoin.h#L257-L277)). Worker'lar aynı anda
başlamadığı için gelen her süreç önce nerede olunduğunu öğrenip doğru yere
atlar:

```c
	switch (BarrierPhase(build_barrier))
	{
		case PHJ_BUILD_ALLOCATE:
			BarrierArriveAndWait(build_barrier, WAIT_EVENT_HASH_BUILD_ALLOCATE);
			pg_fallthrough;

		case PHJ_BUILD_HASH_INNER:
			...
			if (PHJ_GROW_BATCHES_PHASE(BarrierAttach(&pstate->grow_batches_barrier)) !=
				PHJ_GROW_BATCHES_ELECT)
				ExecParallelHashIncreaseNumBatches(hashtable);
```
— [../src/backend/executor/nodeHash.c:266-291](../src/backend/executor/nodeHash.c#L266-L291)

Geç gelen worker devam eden bir batch büyütme işine yardım etmek zorunda, yoksa
bucket'lara güvenle erişemez. Bariyerlerin en tehlikeli tarafı tek cümlede:

```
 * To avoid deadlocks, we never wait for any barrier unless it is known that
 * all other backends attached to it are actively executing the node or have
 * finished.  Practically, that means that we never emit a tuple while attached
 * to a barrier, unless the barrier has reached a phase that means that no
 * process will wait on it again.
```
— [nodeHashjoin.c:146-150](../src/backend/executor/nodeHashjoin.c#L146-L150)

Sebep: tuple üreten bir süreç `Gather`'ın kuyruğunda bloke olabilir; o sırada
bariyerde bekleyen biri olsa kilitlenme olurdu. Çözüm `BarrierArriveAndDetach` /
`BarrierArriveAndDetachExceptLast`
([:152-158](../src/backend/executor/nodeHashjoin.c#L152-L158)).

---

# 11. Paralel Append ve paralel aggregate

**Append** paralel-aware olduğunda alt planlar worker'lara **dağıtılır**;
paylaşılan durum bir LWLock, `pa_next_plan` imleci ve `pa_finished[]` dizisidir
([nodeAppend.c:72-83](../src/backend/executor/nodeAppend.c#L72-L83)). Leader
**sondan** başlar — "Try to pick a plan which doesn't commit us to doing much
work locally... Cheapest subplans are at the end."
([:632-634](../src/backend/executor/nodeAppend.c#L632-L634)); worker'lar
**baştan** ilerler ve sona gelince ilk *partial* plana döner
([:713-716](../src/backend/executor/nodeAppend.c#L713-L716)). Kritik ayrım
`as_first_partial_plan`: listenin başındakiler **non-partial**dır (tek worker
tamamını çalıştırmalı, seçilir seçilmez `pa_finished` işaretlenir,
[:697-699](../src/backend/executor/nodeAppend.c#L697-L699)); sonrakiler
partial'dır ve birden fazla worker aynı plana girebilir.

**Aggregate** ikiye bölünür: worker'da Partial Aggregate, leader'da `Gather`'ın
üstünde Finalize Aggregate.

```c
	/* Initial phase of partial aggregation, with serialization: */
	AGGSPLIT_INITIAL_SERIAL = AGGSPLITOP_SKIPFINAL | AGGSPLITOP_SERIALIZE,
	/* Final phase of partial aggregation, with deserialization: */
	AGGSPLIT_FINAL_DESERIAL = AGGSPLITOP_COMBINE | AGGSPLITOP_DESERIALIZE,
```
— [../src/include/nodes/nodes.h:386-389](../src/include/nodes/nodes.h#L386-L389)

Worker normal `transfn` ile state biriktirir, `finalfn`'i **çalıştırmaz**,
`serialfn` ile aktarılabilir hale getirir. Leader `deserialfn` ile açar,
`transfn` yerine **`combinefn`** ile birleştirir, sonunda `finalfn`'i çalıştırır.
`AVG(x)`'te worker `(toplam, adet)` çifti üretir, leader çiftleri toplayıp böler.
Şema `pg_aggregate.aggcombinefn` yoksa çalışmaz — `combinefn_oid` geçersizse
`hasNonPartialAggs` set edilir
([prepagg.c:294-299](../src/backend/optimizer/prep/prepagg.c#L294-L299)); tipi
`internal` olan aggregate'lerde ayrıca serialize/deserialize fonksiyonları
gerekir, yoksa state süreçler arasında taşınamaz
([:301-312](../src/backend/optimizer/prep/prepagg.c#L301-L312)). Path üretimi
`create_partial_grouping_paths`
([planner.c:7607](../src/backend/optimizer/plan/planner.c#L7607); şemayı
özetleyen yorum [:7600-7602](../src/backend/optimizer/plan/planner.c#L7600-L7602)).
`string_agg(x ORDER BY y)` gibi sıralı/`DISTINCT` aggregate'ler bu şemaya
girmez — sıralama worker'lar arasında korunamaz.

---

# 12. Aynı altyapının diğer müşterisi: paralel VACUUM

```
 * In a parallel vacuum, we perform both index bulk deletion and index cleanup
 * with parallel worker processes.  Individual indexes are processed by one
 * vacuum process. ... Each time we process indexes in parallel,
 * the parallel context is re-initialized so that the same DSM can be used for
 * multiple passes of index bulk-deletion and index cleanup.
```
— [../src/backend/commands/vacuumparallel.c:11-19](../src/backend/commands/vacuumparallel.c#L11-L19)

Farklar: paylaşım birimi **index**tir (blok değil); aynı DSM birden fazla tur
için yeniden kullanılır (`maintenance_work_mem` dolduğunda Faz II tekrar
çalıştığı için); worker sayısı `max_parallel_maintenance_workers` ile
sınırlıdır. Bu ağaçta **paralel autovacuum** da var — leader'ın cost delay
parametreleri çalışırken değişebildiği için bir generation counter'la
worker'lara yayılıyor
([:21-26](../src/backend/commands/vacuumparallel.c#L21-L26)). Ayrıntısı
[VACUUM-NASIL-CALISIR.md](VACUUM-NASIL-CALISIR.md) notunda (Faz II). Paralel
index build de aynı altyapıyı kullanır; `plan_create_index_workers`
([planner.c:7317-7332](../src/backend/optimizer/plan/planner.c#L7317-L7332))
worker sayısını hesaplayıp `maintenance_work_mem`'e göre kırpar.

---

# İzleme ve hata ayıklama

```
Gather  (cost=1000.00..23041.66 rows=9701 width=8)
        (actual time=0.421..142.339 rows=9701 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on t  (cost=0.00..21071.56 rows=4042 width=8)
                              (actual time=0.032..131.204 rows=3234 loops=3)
        Filter: (x < 100)
```

- **`Workers Planned` ≠ `Workers Launched` ise sorun var.** Planlayıcı 2 istedi,
  postmaster 0 verdiyse plan seri çalışmış ama maliyet tahmini paralel
  varsaymıştı ([explain.c:2045-2055](../src/backend/commands/explain.c#L2045-L2055)).
- **`loops=3`** = 2 worker + 1 leader. `rows` **loop başına ortalama**dır;
  toplam için `rows × loops` (3234 × 3 ≈ 9701).
- **`Parallel ` öneki** düğümün `parallel_aware` olduğunu gösterir
  ([:1642-1643](../src/backend/commands/explain.c#L1642-L1643)). Öneksiz bir
  düğüm `Gather` altındaysa her worker onun **tam kopyasını** çalıştırıyor
  demektir — örneğin `Hash` her worker'da ayrı tablo kurar.
- `EXPLAIN (ANALYZE, VERBOSE)` her worker'ı ayrı yazdırır
  ([:5227-5231](../src/backend/commands/explain.c#L5227-L5231)); satır
  dengesizliği chunk dağıtımının doğal sonucudur. `Single Copy: true` = tek
  worker, leader katılmıyor
  ([:2057-2058](../src/backend/commands/explain.c#L2057-L2058)).

## GUC'lar

```sql
SET max_parallel_workers_per_gather = 0;   -- paralelliği kapat
SET min_parallel_table_scan_size = 0;      -- küçük tabloda paralele zorla
SET parallel_setup_cost = 0;
SET parallel_leader_participation = off;   -- tıkanma şüphesi varsa
ALTER TABLE büyük_tablo SET (parallel_workers = 6);
```

Varsayılanlar
[postgresql.conf.sample:233-238](../src/backend/utils/misc/postgresql.conf.sample#L233-L238)
ve [:462-465](../src/backend/utils/misc/postgresql.conf.sample#L462-L465):

| GUC | Varsayılan | Etkisi |
|---|---|---|
| `max_parallel_workers_per_gather` | 2 | Tek `Gather` başına üst sınır. 0 = kapalı |
| `max_parallel_workers` | 8 | Sunucu genelinde eşzamanlı paralel worker |
| `max_worker_processes` | 8 | Tüm bgworker havuzu (restart gerekir) |
| `parallel_setup_cost` | 1000 | Paralel plana peşin ceza |
| `parallel_tuple_cost` | 0.1 | Kuyruktan geçen tuple başına |
| `min_parallel_table_scan_size` | 8MB | Altında paralel seq scan yok |
| `min_parallel_index_scan_size` | 512kB | Altında paralel index scan yok |
| `parallel_leader_participation` | on | Leader alt planı da çalıştırsın mı |

Üç sınır iç içedir: `max_parallel_workers` ≤ `max_worker_processes` olmalı, ve
`max_parallel_workers_per_gather` tek sorgunun payıdır — eşzamanlı paralel
sorgular aynı havuzdan yer arar.

## Worker'ları canlı görmek

```sql
SELECT pid, leader_pid, backend_type, state, wait_event_type, wait_event,
       left(query, 60) AS query
FROM pg_stat_activity
WHERE leader_pid IS NOT NULL OR backend_type = 'parallel worker'
ORDER BY leader_pid NULLS FIRST, pid;
```

`leader_pid` yalnızca **aktif worker'lar** için dolar, leader'ın kendi satırında
`NULL`'dur; değer `PGPROC.lockGroupLeader`'dan geliyor
([pgstatfuncs.c:482-491](../src/backend/utils/adt/pgstatfuncs.c#L482-L491)).
`backend_type = 'parallel worker'` etiketi `bgw_type` olarak konuyor
([parallel.c:607](../src/backend/access/transam/parallel.c#L607)); worker'ın
`query` alanı leader'ın sorgu metnidir, DSM'e `PARALLEL_KEY_QUERY_TEXT` ile
geçirilip raporlanıyor
([execParallel.c:1541-1544](../src/backend/executor/execParallel.c#L1541-L1544)).
Anlamlı wait event'ler
([wait_event_names.txt](../src/backend/utils/activity/wait_event_names.txt)):
`ExecuteGather` ([:121](../src/backend/utils/activity/wait_event_names.txt#L121)),
`ParallelFinish` ([:148](../src/backend/utils/activity/wait_event_names.txt#L148)),
`BtreePage` ([:113](../src/backend/utils/activity/wait_event_names.txt#L113)),
`ParallelBitmapScan` ([:146](../src/backend/utils/activity/wait_event_names.txt#L146)),
`HashBuildHashInner` ([:127](../src/backend/utils/activity/wait_event_names.txt#L127)).

Kümülatif sayaçlar için
([system_views.sql:1182-1183](../src/backend/catalog/system_views.sql#L1182-L1183)):

```sql
SELECT parallel_workers_to_launch, parallel_workers_launched,
       parallel_workers_to_launch - parallel_workers_launched AS eksik
FROM pg_stat_database WHERE datname = current_database();
```

`eksik` sürekli büyüyorsa `max_parallel_workers` yetersiz demektir; sayaçlar
`ExecGather` içinde işleniyor
([nodeGather.c:190-191](../src/backend/executor/nodeGather.c#L190-L191)).

## `debug_parallel_query`

Geliştirici GUC'u, `GUC_NOT_IN_SAMPLE`
([guc_parameters.dat:695-702](../src/backend/utils/misc/guc_parameters.dat#L695-L702)).
`on` değeri plan güvenliyse **tepeye zorla bir `Gather` ekler**
(`num_workers = 1`, `single_copy = true`,
[planner.c:563-578](../src/backend/optimizer/plan/planner.c#L563-L578));
`regress` aynısını yapar ama `gather->invisible = true`
([:578](../src/backend/optimizer/plan/planner.c#L578)) — `Gather` `EXPLAIN`'de
görünmez, hata mesajlarına `parallel worker` bağlamı eklenmez. Ne işe yarar:
yazdığınız C fonksiyonunun **paralel modda patlayıp patlamadığını** test etmek;
sorgunuz normalde paralelleşmese bile paralel mod kısıtları devreye girer.
Üretimde açılmaz.

## Paralel plan üretilmiyorsa kontrol listesi

1. `max_parallel_workers_per_gather` sıfır mı?
2. Tablo `min_parallel_table_scan_size`'tan (8 MB) küçük mü?
3. `SELECT proname, proparallel FROM pg_proc WHERE proname = 'fonksiyonum';`
   — kendi fonksiyonlarınız varsayılan `u` gelir.
4. `SELECT ... FOR UPDATE`, `TEMPORARY` tablo, veya CTE üzerinden okuma var mı?
5. Cursor'dan `FETCH` ile mi sürülüyor?
   ([execMain.c:1752-1753](../src/backend/executor/execMain.c#L1752-L1753))
6. `parallel_setup_cost`'u geçici olarak 0 yapıp `EXPLAIN` farkına bakın.

---

# Tek sayfalık özet

```
PLANLAMA                          │  ÇALIŞTIRMA
──────────────────────────────────┼─────────────────────────────────────────
standard_planner    planner.c:413 │  ExecutePlan → EnterParallelMode()
  parallelModeOK?                 │  ExecGather            nodeGather.c:138
    ├ CMD_SELECT                  │    ├ ExecInitParallelPlan execParallel:653
    ├ !IsParallelWorker           │    │   ├ ExecSerializePlan (nodeToString)
    └ max_parallel_hazard(parse)  │    │   ├ CreateParallelContext
         proparallel: s / r / u   │    │   └ InitializeParallelDSM
              ↓                   │    │       dsm_create → shm_toc
set_rel_consider_parallel         │    │       GUC/snapshot/xact/comboCID
   temp? CTE? LIMIT'li subquery?  │    │       error queue 16K × N
              ↓                   │    │       tuple queue 64K × N
create_plain_partial_paths        │    ├ LaunchParallelWorkers parallel.c:583
   compute_parallel_worker        │    │    ∧ max_parallel_workers (8)
     log3(pages / 8MB)            │    └ need_to_scan_locally = ...
     ∧ max_parallel_workers_per_  │
       gather (2)                 │  ┌─ WORKER ───────────────────────────┐
              ↓                   │  │ ParallelWorkerMain parallel.c:1301 │
add_partial_path → Gather path    │  │  dsm_attach → error queue → redirect│
   cost_gather                    │  │  BecomeLockGroupMember             │
     + parallel_setup_cost (1000) │  │  GUC/snapshot/xact geri yükle      │
     + parallel_tuple_cost × rows │  │  EnterParallelMode                 │
   get_parallel_divisor           │  │ ParallelQueryMain execParallel:1514│
     N + (1 - 0.3N)               │  │  stringToNode(plan) → ExecutorRun   │
                                  │  │  DestReceiver = tqueue → shm_mq    │
                                  │  └────────────────────────────────────┘

İŞ PAYLAŞIMI (paylaşılan durum DSM'de)
  Seq Scan     → pg_atomic_fetch_add_u64(phs_nallocated, chunk_size)
  B-tree       → LWLock + ConditionVariable, sayfa sayfa "seize"
  Bitmap Heap  → biri bitmap'i kurar, hepsi DSA iterator'dan çeker
  Hash Join    → Barrier fazları (ELECT/ALLOCATE/HASH/RUN/FREE)
  Append       → LWLock + pa_finished[], leader sondan worker baştan
  Aggregate    → Partial (serialfn) → Gather → Finalize (combinefn)
```

## Akılda kalması gereken 5 şey

1. **`Gather` bir sınırdır.** Altı N kopya, üstü tek süreç. Bir işi
   hızlandırmak için onu `Gather`'ın altına indirmek gerekir — filtreyi, kısmi
   toplamayı, sıralamayı. Üstte kalan her şey tek çekirdeğe mahkumdur.

2. **Worker ayrı süreçtir; ne aktarılamıyorsa yasaktır.** Geçici tablo, CTE
   tuplestore'u, `FOR UPDATE`, `PARALLEL UNSAFE` fonksiyon, yeni XID — hepsi
   aynı sebepten dışarıda. Yazma yasağının teknik çekirdeği combo CID
   senkronizasyonudur.

3. **Sayı garanti değil, iki katman kırpar.** Planlayıcı
   `log3(pages / min_parallel_table_scan_size)` hesaplar,
   `max_parallel_workers_per_gather` (2) ile kırpar; sonra postmaster
   `max_parallel_workers` (8) havuzuna bakıp daha da azaltabilir. `EXPLAIN
   ANALYZE`'da **Planned ile Launched farkı** bu kırpmanın göstergesidir.

4. **İş paylaşımı dinamiktir.** Worker'lar önceden bölünmüş parça almaz;
   bitirdikçe atomik sayaçtan veya bariyerden yeni iş çekerler. Bu yüzden
   `EXPLAIN` çıktısındaki worker satır sayıları eşit olmaz ve olmaması
   normaldir.

5. **Hata iletimi `CHECK_FOR_INTERRUPTS` üzerinden yürür.** Worker'daki
   `ereport` bir `shm_mq`'ya yazılır, leader'a sinyal gider, leader bir sonraki
   `CHECK_FOR_INTERRUPTS`'ta mesajı okuyup `ThrowErrorData` ile yeniden
   fırlatır. `CONTEXT: parallel worker` satırı hatanın başka bir süreçte
   doğduğunun kanıtıdır.
