# İnceleme konu önerileri — sırada ne var?

*PostgreSQL 20devel ağacı için. Satır numaraları bu ağaçtan doğrulandı
([meson.build:11](../meson.build#L11) `version: '20devel'`).*

Bu dosya bir inceleme notu değil, **yol haritası**. Mevcut dört notun bıraktığı
boşlukları listeler, her konu için giriş noktalarını verir ve hangi sırayla
okunacağını söyler.

---

## Durum (2026-08-31)

Yol haritasındaki 21 konudan **11'i yazıldı** (ayrıca haritada olmayan 4 not daha
eklendi: `WHERE-VE-HAVING-FARKI`, `SILINEN-VERI-KURTARILABILIR-MI`,
`SAYFA-SISTEMI-NASIL-CALISIR`, `ISTATISTIKLER-NASIL-CALISIR`).

| Durum | Not |
|---|---|
| ✅ | `VACUUM-NASIL-CALISIR.md` (#1) |
| ✅ | `WAL-VE-KURTARMA.md` (#2) |
| ✅ | `BTREE-INDEX-NASIL-CALISIR.md` (#3) |
| ✅ | `BUFFER-VE-AIO.md` (#4) |
| ✅ | `PLANLAYICI-MALIYET-MODELI.md` (#5) |
| ✅ | `EXECUTOR-DUGUMLERI.md` (#6) |
| ✅ | `TRANSACTION-YONETIMI.md` (#7) |
| ✅ | `SNAPSHOT-VE-GORUNURLUK.md` (#8) |
| ✅ | `INSERT-DELETE-NASIL-CALISIR.md` (#9) |
| ✅ | `PARALEL-SORGU.md` (#10) |
| ✅ | `TOAST-NASIL-CALISIR.md` (#11) |
| ✅ | `ISTATISTIKLER-NASIL-CALISIR.md` (#17, ayrı oturumda yazıldı) |
| ⬜ | #12 KATALOG-ONBELLEKLERI |
| ⬜ | #13 BELLEK-BAGLAMLARI |
| ⬜ | #14 REPLIKASYON-NASIL-CALISIR |
| ⬜ | #15 PARTITIONING-NASIL-CALISIR |
| ⬜ | #16 ISTEMCI-PROTOKOLU |
| ⬜ | #18 GENISLETILEBILIRLIK |
| ⬜ | #19 REPACK-NASIL-CALISIR |
| ⬜ | #20 PROPERTY-GRAPH |
| ⬜ | #21 DIGER-INDEX-TURLERI |

Sıradaki parti: **#12, #13, #14, #15, #16**.

---

## 30 saniyelik özet

```
   YAZILMIŞ (4 not)                    BOŞLUK (öneriler)
   ───────────────                     ─────────────────

   POSTGRES-NASIL-CALISIR ──┬────────> VACUUM / autovacuum / freeze
   (genel tur)              │          WAL yazma + crash recovery
                            ├────────> Buffer manager + AIO (yeni)
                            └────────> Bellek bağlamları, katalog cache

   SELECT-NASIL-CALISIR ────┬────────> Planlayıcı maliyet modeli + join sırası
   (boru hattı)             ├────────> Executor düğümleri (sort/hash/agg)
                            └────────> Paralel sorgu

   UPDATE-NASIL-CALISIR ────┬────────> INSERT / DELETE + trigger makinesi
   (DML + HOT)              ├────────> TOAST
                            └────────> B-tree index içi

   KILIT-MEKANIZMALARI ─────┬────────> Transaction yöneticisi (xid, 2PC, SLRU)
   (4 kilit türü)           └────────> Snapshot / procarray / xmin horizon
```

Üç cümlelik model:

- Mevcut dört not **yatay** kesitler: bir komutun yolculuğu. Eksik olan
  **dikey** kesitler: tek bir alt sistemin kendi içindeki tam mantığı.
- Dört notun hepsi VACUUM'a, WAL'a ve buffer manager'a *değinip geçiyor* —
  o üçü ilk yazılacak notlar.
- Bu ağaçta upstream'de olmayan üç konu var (REPACK, property graph, AIO);
  bunlar en özgün ama en son yazılacak notlar, çünkü temel gerekiyor.

---

## Kaynak haritası — önerilen konular nerede yaşıyor

| Dizin | Kaç konu | Ne var |
|---|---|---|
| `src/backend/access/heap/` | 3 | VACUUM, freeze, TOAST, visibility map |
| `src/backend/access/transam/` | 3 | WAL, recovery, xact, CLOG/SLRU, MultiXact |
| `src/backend/access/nbtree`, `gin`, `gist`, `brin` | 2 | index access method'lar |
| `src/backend/optimizer/` | 1 | maliyet modeli, join arama, GEQO |
| `src/backend/executor/` | 3 | düğümler, paralellik, INSERT/DELETE |
| `src/backend/storage/buffer`, `aio`, `smgr` | 1 | buffer pool, asenkron I/O, dosya katmanı |
| `src/backend/replication/` | 2 | fiziksel + mantıksal replikasyon, REPACK plugin |
| `src/backend/utils/cache`, `mmgr`, `activity` | 3 | syscache, relcache, bellek, pgstat |
| `src/backend/commands/` | 2 | REPACK, property graph, COPY |

---

## Öncelik tablosu

| # | Önerilen dosya | Neden | Süre | Önkoşul |
|---|---|---|---|---|
| 1 | `VACUUM-NASIL-CALISIR.md` | Dört notun dördü de "VACUUM temizler" deyip geçiyor | 1 gün | UPDATE notu |
| 2 | `WAL-VE-KURTARMA.md` | WAL yazma yarım anlatıldı, replay hiç anlatılmadı | 1 gün | genel tur |
| 3 | `BTREE-INDEX-NASIL-CALISIR.md` | Index "güncellenir" deniyor, içi hiç açılmadı | 1 gün | UPDATE notu |
| 4 | `BUFFER-VE-AIO.md` | PG 18+ AIO altyapısı; genel turdaki anlatım eskimiş | 1 gün | genel tur |
| 5 | `PLANLAYICI-MALIYET-MODELI.md` | SELECT notu `make_one_rel`'de duruyor | 1,5 gün | SELECT notu |
| 6 | `EXECUTOR-DUGUMLERI.md` | Sadece SeqScan anlatıldı; sort/hash/agg yok | 1 gün | SELECT notu |
| 7 | `TRANSACTION-YONETIMI.md` | xid tahsisi, subtransaction, 2PC, SLRU | 1 gün | kilit notu |
| 8 | `SNAPSHOT-VE-GORUNURLUK.md` | `GetSnapshotData` ve xmin horizon derinliği | yarım gün | genel tur |
| 9 | `INSERT-DELETE-NASIL-CALISIR.md` | DML üçlüsünü tamamlar | yarım gün | UPDATE notu |
| 10 | `PARALEL-SORGU.md` | Paralellik her yerde geçiyor, hiç açılmadı | 1 gün | 5 + 6 |
| 11 | `TOAST-NASIL-CALISIR.md` | UPDATE notunda tek paragraf | yarım gün | UPDATE notu |
| 12 | `KATALOG-ONBELLEKLERI.md` | syscache / relcache / invalidation üçlüsü | 1 gün | genel tur |
| 13 | `BELLEK-BAGLAMLARI.md` | Genel turda 25 satır; tek başına not olmalı | yarım gün | yok |
| 14 | `REPLIKASYON-NASIL-CALISIR.md` | Hiç dokunulmadı; en büyük eksik alt sistem | 2 gün | 2 |
| 15 | `PARTITIONING-NASIL-CALISIR.md` | UPDATE notu partition'a değinip bıraktı | 1 gün | 5 |
| 16 | `ISTEMCI-PROTOKOLU.md` | Extended query, prepared statement, COPY | 1 gün | SELECT notu |
| 17 | `ISTATISTIKLER.md` | ANALYZE + pgstat; tahmin nereden geliyor | 1 gün | 5 |
| 18 | `GENISLETILEBILIRLIK.md` | Hook, extension, AM, FDW, JIT | 1 gün | genel tur |
| 19 | `REPACK-NASIL-CALISIR.md` | **Bu ağaca özel** — upstream'de yok | 1,5 gün | 1, 14 |
| 20 | `PROPERTY-GRAPH.md` | **Bu ağaca özel** — SQL/PGQ | 1 gün | SELECT notu |
| 21 | `DIGER-INDEX-TURLERI.md` | GIN, GiST, BRIN, SP-GiST karşılaştırması | 1,5 gün | 3 |

---

# 1. Sıra — mevcut notların açıkta bıraktığı dört konu

Bu dördü yazılmadan diğerleri havada kalıyor; çünkü var olan dört not bu
konulara *isim vererek* atıfta bulunuyor ama açıklamıyor.

## 1.1 `VACUUM-NASIL-CALISIR.md`

**Neden:** [UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md) sayfa içi budamayı
anlatıyor ama asıl temizlik VACUUM'da. Wraparound, freeze ve autovacuum
eşikleri üretimde en çok başa bela olan konular.

**Kapsam:**

- İki fazlı tarama: heap → index → heap; `maintenance_work_mem` sınırının etkisi
- Freeze kararı ve wraparound: `FrozenTransactionId`, `vacuum_freeze_min_age`
- Visibility map: index-only scan'i mümkün kılan bit
- Autovacuum launcher/worker ikilisi ve eşik formülü
- `VACUUM FULL` neden farklı (yeni heap yazar, ACCESS EXCLUSIVE alır)

**Giriş noktaları:**

- [../src/backend/access/heap/vacuumlazy.c:624](../src/backend/access/heap/vacuumlazy.c#L624) — `heap_vacuum_rel`
- [../src/backend/access/heap/vacuumlazy.c:1279](../src/backend/access/heap/vacuumlazy.c#L1279) — `lazy_scan_heap`
- [../src/backend/access/heap/heapam.c:7267](../src/backend/access/heap/heapam.c#L7267) — `heap_prepare_freeze_tuple`
- [../src/backend/access/heap/visibilitymap.c:260](../src/backend/access/heap/visibilitymap.c#L260) — `visibilitymap_set`
- [../src/backend/postmaster/autovacuum.c:411](../src/backend/postmaster/autovacuum.c#L411) — `AutoVacLauncherMain`
- [../src/backend/commands/vacuumparallel.c](../src/backend/commands/vacuumparallel.c) — paralel index temizliği

## 1.2 `WAL-VE-KURTARMA.md`

**Neden:** [POSTGRES-NASIL-CALISIR.md](POSTGRES-NASIL-CALISIR.md) WAL *yazmayı*
anlatıyor; okuma tarafı — yani çökme sonrası replay — hiç yok. Checkpoint'in ne
işe yaradığı ancak replay anlatılınca oturuyor.

**Kapsam:**

- Kayıt formatı: `XLogRecord`, blok referansları, full-page write
- `XLogInsertRecord` içindeki WAL insert lock yarışı
- Checkpoint: redo noktası seçimi, neden `CHECKPOINT` uzun sürer
- Recovery döngüsü: redo → rmgr dispatch → sayfa LSN karşılaştırması
- Restart point, timeline, `pg_waldump` ile kayıt okuma

**Giriş noktaları:**

- [../src/backend/access/transam/xlog.c:784](../src/backend/access/transam/xlog.c#L784) — `XLogInsertRecord`
- [../src/backend/access/transam/xlog.c:5845](../src/backend/access/transam/xlog.c#L5845) — `StartupXLOG`
- [../src/backend/access/transam/xlog.c:7400](../src/backend/access/transam/xlog.c#L7400) — `CreateCheckPoint`
- [../src/backend/access/transam/xlogrecovery.c:1617](../src/backend/access/transam/xlogrecovery.c#L1617) — `PerformWalRecovery`
- [../src/backend/access/transam/README](../src/backend/access/transam/README)
- [../src/backend/access/rmgrdesc/](../src/backend/access/rmgrdesc/) — her rmgr'ın kayıt açıklayıcısı

## 1.3 `BTREE-INDEX-NASIL-CALISIR.md`

**Neden:** UPDATE notu "index'e yeni giriş eklenir" diyor, SELECT notu index
scan'i atlıyor. B-tree PostgreSQL'in en çok kullanılan yapısı ve içinde
öğretici mekanizmalar var (sayfa bölme, right link, ölü giriş işaretleme).

**Kapsam:**

- Sayfa yapısı: meta sayfa, iç düğüm, yaprak, high key, right link
- Arama: `_bt_search` ve kök→yaprak yolu, Lehman-Yao kilit protokolü
- Ekleme ve bölme: `_bt_insertonpg` → `_bt_split`, sağa yayılan bölünme
- Unique kısıt kontrolü neden ekleme yolunda yapılıyor
- Deduplication ve posting list
- Index-only scan: visibility map ile bağ

**Giriş noktaları:**

- [../src/backend/access/nbtree/README](../src/backend/access/nbtree/README) — önce bunu oku
- [../src/backend/access/nbtree/nbtsearch.c:102](../src/backend/access/nbtree/nbtsearch.c#L102) — `_bt_search`
- [../src/backend/access/nbtree/nbtree.c:206](../src/backend/access/nbtree/nbtree.c#L206) — `btinsert`
- [../src/backend/access/nbtree/nbtinsert.c:1119](../src/backend/access/nbtree/nbtinsert.c#L1119) — `_bt_insertonpg`
- [../src/backend/access/nbtree/nbtinsert.c:1489](../src/backend/access/nbtree/nbtinsert.c#L1489) — `_bt_split`
- [../src/backend/executor/nodeIndexonlyscan.c:63](../src/backend/executor/nodeIndexonlyscan.c#L63) — `IndexOnlyNext`

## 1.4 `BUFFER-VE-AIO.md`

**Neden:** Genel turdaki buffer anlatımı klasik `ReadBuffer` modelini anlatıyor.
Bu ağaçta okuma yolu **read stream + asenkron I/O** üzerinden geçiyor
([../src/backend/storage/aio/README.md](../src/backend/storage/aio/README.md)).
Yeni ve iyi belgelenmiş bir alt sistem.

**Kapsam:**

- Buffer descriptor, pin/unpin, clock-sweep (kısa tekrar)
- `StartReadBuffers` / `WaitReadBuffers` ikilisi: batch okuma
- Read stream: ardışık blokları önceden isteme
- Üç I/O yöntemi: `sync`, `worker`, `io_uring` — `io_method` ayarı
- `smgr` katmanı ve `md.c`: 1 GB'lık segment dosyaları

**Giriş noktaları:**

- [../src/backend/storage/aio/README.md](../src/backend/storage/aio/README.md)
- [../src/backend/storage/buffer/bufmgr.c:1276](../src/backend/storage/buffer/bufmgr.c#L1276) — `ReadBuffer_common`
- [../src/backend/storage/buffer/bufmgr.c:1618](../src/backend/storage/buffer/bufmgr.c#L1618) — `StartReadBuffers`
- [../src/backend/storage/aio/read_stream.c:1030](../src/backend/storage/aio/read_stream.c#L1030) — `read_stream_next_buffer`
- [../src/backend/storage/aio/aio_io.c:78](../src/backend/storage/aio/aio_io.c#L78) — `pgaio_io_start_readv`
- [../src/backend/storage/smgr/md.c:858](../src/backend/storage/smgr/md.c#L858) — `mdreadv`

---

# 2. Sıra — sorgu boru hattının derinleşmesi

## 2.1 `PLANLAYICI-MALIYET-MODELI.md`

[SELECT-NASIL-CALISIR.md](SELECT-NASIL-CALISIR.md) `make_one_rel`'e kadar geliyor
ve orada duruyor. Planlayıcı tek başına bir not hak ediyor.

- Path ile Plan ayrımı, `add_path` ve baskınlık (dominance) kuralı
- Maliyet formülleri: `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`
- Seçicilik tahmini: `clauselist_selectivity`, histogram + MCV
- Join arama: dinamik programlama ile GEQO eşiği (`geqo_threshold`)
- Üç join stratejisi ve hangisi ne zaman kazanır

Giriş noktaları:
[../src/backend/optimizer/README](../src/backend/optimizer/README) ·
[../src/backend/optimizer/path/allpaths.c:183](../src/backend/optimizer/path/allpaths.c#L183) `make_one_rel` ·
[../src/backend/optimizer/path/allpaths.c:3952](../src/backend/optimizer/path/allpaths.c#L3952) `standard_join_search` ·
[../src/backend/optimizer/path/joinpath.c:123](../src/backend/optimizer/path/joinpath.c#L123) `add_paths_to_joinrel` ·
[../src/backend/optimizer/path/costsize.c:271](../src/backend/optimizer/path/costsize.c#L271) `cost_seqscan` ·
[../src/backend/optimizer/path/clausesel.c:100](../src/backend/optimizer/path/clausesel.c#L100) `clauselist_selectivity`

## 2.2 `EXECUTOR-DUGUMLERI.md`

SELECT notu sadece `SeqScan`'i izliyor. Asıl ilginç düğümler diske taşabilen
düğümler.

- Sort: quicksort → external merge sort geçişi, `work_mem` sınırı
- Hash join: batch'lere bölme, `work_mem` aşımında diske taşma
- Agg: hash agg ile sorted agg, hash agg'ın disk taşması
- Memoize, Material, Gather — ne zaman plana girerler
- `EXPLAIN (ANALYZE, BUFFERS)` çıktısının kod karşılığı

Giriş noktaları:
[../src/backend/executor/README](../src/backend/executor/README) ·
[../src/backend/executor/nodeHashjoin.c:225](../src/backend/executor/nodeHashjoin.c#L225) `ExecHashJoinImpl` ·
[../src/backend/executor/nodeAgg.c:2247](../src/backend/executor/nodeAgg.c#L2247) `ExecAgg` ·
[../src/backend/utils/sort/tuplesort.c:1260](../src/backend/utils/sort/tuplesort.c#L1260) `tuplesort_performsort`

## 2.3 `INSERT-DELETE-NASIL-CALISIR.md`

UPDATE notunun kardeşi; DML üçlüsünü tamamlar. Kısa tutulabilir, çünkü ortak
altyapının çoğu zaten anlatıldı.

- `ExecInsert`: `heap_insert`, sayfa seçimi, FSM
- `ExecDelete`: `heap_delete`, sadece `xmax` yazma
- `ON CONFLICT` (speculative insertion) — asıl ilginç kısım
- Index bakımı: `ExecInsertIndexTuples` ve unique kontrol
- Trigger makinesi: before/after, statement/row, ertelenmiş kuyruk

Giriş noktaları:
[../src/backend/executor/nodeModifyTable.c:874](../src/backend/executor/nodeModifyTable.c#L874) `ExecInsert` ·
[../src/backend/executor/execIndexing.c:311](../src/backend/executor/execIndexing.c#L311) `ExecInsertIndexTuples` ·
[../src/backend/commands/trigger.c:2336](../src/backend/commands/trigger.c#L2336) `ExecCallTriggerFunc` ·
[../src/backend/commands/trigger.c:5186](../src/backend/commands/trigger.c#L5186) `AfterTriggerEndQuery` ·
[../src/backend/storage/freespace/freespace.c:154](../src/backend/storage/freespace/freespace.c#L154) `RecordAndGetPageWithFreeSpace`

## 2.4 `PARALEL-SORGU.md`

Dört notta da "paralel" kelimesi geçiyor, hiçbirinde açıklanmıyor.

- Dynamic shared memory segmenti ve plan serileştirme
- Worker başlatma, `parallel_setup_cost` neden var
- Paralel-güvenli fonksiyon işaretlemesi (`proparallel`)
- Gather ile Gather Merge farkı; leader'ın da çalışması
- Paralel bitmap heap scan ve paralel hash join

Giriş noktaları:
[../src/backend/executor/execParallel.c:653](../src/backend/executor/execParallel.c#L653) `ExecInitParallelPlan` ·
[../src/backend/executor/nodeGather.c:138](../src/backend/executor/nodeGather.c#L138) `ExecGather` ·
[../src/backend/access/transam/parallel.c](../src/backend/access/transam/parallel.c)

---

# 3. Sıra — transaction ve katalog altyapısı

## 3.1 `TRANSACTION-YONETIMI.md`

[KILIT-MEKANIZMALARI.md](KILIT-MEKANIZMALARI.md) kilitleri anlatıyor ama
transaction'ın kendisini değil.

- xid tahsisi tembeldir: salt-okunur transaction xid almaz
- Subtransaction: `SAVEPOINT`, SUBTRANS SLRU, xid ağacı
- Commit protokolü: CLOG yazımı → WAL flush → kilit bırakma sırası
- İki fazlı commit (`PREPARE TRANSACTION`) ve `pg_twophase`
- SLRU: CLOG, SUBTRANS ve MultiXact aynı altyapıyı paylaşır
- MultiXact — çoklu satır kilitçisi (UPDATE notunda değinilmişti)

Giriş noktaları:
[../src/backend/access/transam/xact.c:2559](../src/backend/access/transam/xact.c#L2559) `PrepareTransaction` ·
[../src/backend/access/transam/slru.c:550](../src/backend/access/transam/slru.c#L550) `SimpleLruReadPage` ·
[../src/backend/access/transam/multixact.c:358](../src/backend/access/transam/multixact.c#L358) `MultiXactIdCreate`

## 3.2 `SNAPSHOT-VE-GORUNURLUK.md`

Genel turda snapshot'a bir sayfa ayrılmış; asıl ilginç kısım `procarray.c`.

- `GetSnapshotData`'nın maliyeti ve snapshot scalability iyileştirmesi
- xmin horizon: VACUUM'un neyi silebileceğini kim belirler
- Hot standby feedback ve `max_standby_streaming_delay` bağlantısı
- `pg_stat_activity.backend_xmin` ile uzun transaction avı

Giriş noktaları:
[../src/backend/storage/ipc/procarray.c:2114](../src/backend/storage/ipc/procarray.c#L2114) `GetSnapshotData`

## 3.3 `KATALOG-ONBELLEKLERI.md`

Her sorgu katalog okur ama diske gitmez. Nasıl?

- syscache: `SearchCatCache` ve negatif giriş kavramı
- relcache: `RelationBuildDesc`, init file (`pg_internal.init`)
- Invalidation: commit'te yayınlanan mesajlar, `sinval` kuyruğu, overflow
- DDL sırasında cache'in nasıl tutarlı kaldığı

Giriş noktaları:
[../src/backend/utils/cache/catcache.c:1374](../src/backend/utils/cache/catcache.c#L1374) `SearchCatCache` ·
[../src/backend/utils/cache/relcache.c:1059](../src/backend/utils/cache/relcache.c#L1059) `RelationBuildDesc` ·
[../src/backend/utils/cache/inval.c:823](../src/backend/utils/cache/inval.c#L823) `LocalExecuteInvalidationMessage`

## 3.4 `BELLEK-BAGLAMLARI.md` (kısa not)

Genel turdaki 25 satır bu konuyu hakkıyla anlatmıyor. Yarım günlük, tatmin
edici bir not olur.

- Bağlam ağacı ve `MemoryContextReset` ile toplu serbest bırakma
- Dört ayırıcı: aset, generation, slab, bump — hangisi nerede
- `palloc` çağrı yolu ve chunk başlığı
- `MemoryContextStats` ile bellek şişmesi avı

Giriş noktaları:
[../src/backend/utils/mmgr/README](../src/backend/utils/mmgr/README) ·
[../src/backend/utils/mmgr/mcxt.c:1235](../src/backend/utils/mmgr/mcxt.c#L1235) `MemoryContextAlloc` ·
[../src/backend/utils/mmgr/aset.c:735](../src/backend/utils/mmgr/aset.c#L735) `AllocSetAllocLarge`

## 3.5 `TOAST-NASIL-CALISIR.md` (kısa not)

UPDATE notunda tek paragraf var; genişletilmeye değer.

- 2 KB eşiği ve dört strateji (plain / extended / external / main)
- Sıkıştırma: pglz ile LZ4 (`default_toast_compression`)
- Chunk'lara bölme ve TOAST tablosunun kendi index'i
- `detoast.c` ve dilim (slice) okuma — `substr` neden hızlı olabilir

Giriş noktaları:
[../src/backend/access/heap/heaptoast.c:96](../src/backend/access/heap/heaptoast.c#L96) `heap_toast_insert_or_update` ·
[../src/backend/access/common/toast_internals.c:119](../src/backend/access/common/toast_internals.c#L119) `toast_save_datum` ·
[../src/backend/access/common/detoast.c](../src/backend/access/common/detoast.c)

---

# 4. Sıra — dışa dönük alt sistemler

## 4.1 `REPLIKASYON-NASIL-CALISIR.md`

En büyük eksik alt sistem. İki gün ayrılmalı, hatta ikiye bölünebilir
(fiziksel / mantıksal).

- Walsender ile walreceiver protokolü, keepalive ve feedback mesajları
- Replication slot: WAL'ı tutan mekanizma ve slot'un tehlikesi
- Senkron replikasyon: commit'in beklediği yer
- Mantıksal kod çözme: WAL → ReorderBuffer → output plugin
- Snapshot builder — mantıksal decoding'in en zor parçası
- `pgoutput` ve logical replication worker'ları

Giriş noktaları:
[../src/backend/replication/README](../src/backend/replication/README) ·
[../src/backend/replication/walsender.c:3048](../src/backend/replication/walsender.c#L3048) `WalSndLoop` ·
[../src/backend/replication/walreceiver.c:155](../src/backend/replication/walreceiver.c#L155) `WalReceiverMain` ·
[../src/backend/replication/syncrep.c:149](../src/backend/replication/syncrep.c#L149) `SyncRepWaitForLSN` ·
[../src/backend/replication/logical/decode.c:89](../src/backend/replication/logical/decode.c#L89) `LogicalDecodingProcessRecord` ·
[../src/backend/replication/logical/reorderbuffer.c:2880](../src/backend/replication/logical/reorderbuffer.c#L2880) `ReorderBufferCommit`

## 4.2 `PARTITIONING-NASIL-CALISIR.md`

UPDATE notu partition sınırını aşan UPDATE'i anlatıyor; asıl mekanizma
anlatılmadı.

- Plan zamanı pruning ile çalışma zamanı pruning
- Partitionwise join ve partitionwise aggregate
- `ATTACH PARTITION` neden artık tabloyu tam taramayabiliyor
- Tuple routing: INSERT hangi partition'a gider

Giriş noktaları:
[../src/backend/partitioning/partprune.c:780](../src/backend/partitioning/partprune.c#L780) `prune_append_rel_partitions` ·
[../src/backend/executor/execPartition.c](../src/backend/executor/execPartition.c)

## 4.3 `ISTEMCI-PROTOKOLU.md`

SELECT notu **simple query** yolunu anlatıyor. Gerçek uygulamalar **extended
query** kullanıyor — farklı bir kod yolu.

- Parse / Bind / Execute / Describe mesajları ve named statement
- Generic plan ile custom plan kararı (beşinci çalıştırma kuralı)
- Portal'ın yeniden kullanımı, `FETCH` ve cursor
- COPY protokolü: neden INSERT'ten kat kat hızlı

Giriş noktaları:
[../src/backend/tcop/postgres.c:1421](../src/backend/tcop/postgres.c#L1421) `exec_parse_message` ·
[../src/backend/tcop/postgres.c:1662](../src/backend/tcop/postgres.c#L1662) `exec_bind_message` ·
[../src/backend/utils/cache/plancache.c](../src/backend/utils/cache/plancache.c) ·
[../src/backend/commands/copyfrom.c:781](../src/backend/commands/copyfrom.c#L781) `CopyFrom`

## 4.4 `ISTATISTIKLER.md`

İki ayrı istatistik dünyası var ve sürekli karıştırılıyor: **planlayıcı
istatistikleri** (`pg_statistic`) ve **çalışma istatistikleri** (`pg_stat_*`).

- ANALYZE örnekleme algoritması, `default_statistics_target`
- MCV listesi, histogram, korelasyon; genişletilmiş istatistikler
- Kümülatif istatistik sistemi artık paylaşımlı bellekte
- `pgstat_report_stat` ne zaman diske yazar

Giriş noktaları:
[../src/backend/commands/analyze.c](../src/backend/commands/analyze.c) ·
[../src/backend/statistics/](../src/backend/statistics/) ·
[../src/backend/utils/activity/pgstat.c:723](../src/backend/utils/activity/pgstat.c#L723) `pgstat_report_stat`

## 4.5 `GENISLETILEBILIRLIK.md`

Genel turda dört başlık altında özetlenmiş; ayrı not olmayı hak ediyor.

- Hook mekanizması: `ProcessUtility_hook` ve arkadaşları
- `CREATE EXTENSION` ne yapar; control dosyası, upgrade script'leri
- Table AM ve Index AM arayüzleri (`*_handler` fonksiyonları)
- FDW: `GetForeignPaths` → `IterateForeignScan`
- JIT: hangi ifadeler derlenir, `jit_above_cost`

Giriş noktaları:
[../src/backend/tcop/utility.c:72](../src/backend/tcop/utility.c#L72) `ProcessUtility_hook` ·
[../src/backend/commands/extension.c:2149](../src/backend/commands/extension.c#L2149) `CreateExtension` ·
[../src/backend/access/table/tableamapi.c](../src/backend/access/table/tableamapi.c) ·
[../src/backend/jit/llvm/llvmjit_expr.c:80](../src/backend/jit/llvm/llvmjit_expr.c#L80) `llvm_compile_expr`

---

# 5. Bu ağaca özel konular — 20devel'de olup eski sürümlerde olmayan

Bu üçü en özgün notlar olur, çünkü haklarında hazır kaynak yok. Ama temel
notlar bittikten sonra yazılmalı.

## 5.1 `REPACK-NASIL-CALISIR.md`

`REPACK` komutu bu ağaçta `CLUSTER` ve `VACUUM FULL`'un işini devralıyor ve
**eşzamanlı** (concurrent) bir varyantı var. Eşzamanlı varyant, tabloyu
kopyalarken gelen değişiklikleri **mantıksal kod çözmeyle** yakalıyor — yani
replikasyon altyapısını DDL için kullanan ilginç bir tasarım.

- `REPACK` ile eski `CLUSTER` / `VACUUM FULL` farkı: komut yüzeyi
- Yeni heap oluşturma → veri kopyalama → dosya takası (üç aşama)
- CONCURRENTLY: replication slot açılıyor, değişiklikler decode edilip yeni
  heap'e uygulanıyor
- Neden identity index gerekiyor
- Kilit seviyesi farkı: `RepackLockLevel`

Giriş noktaları:
[../src/backend/commands/repack.c:250](../src/backend/commands/repack.c#L250) `ExecRepack` ·
[../src/backend/commands/repack.c:449](../src/backend/commands/repack.c#L449) `RepackLockLevel` ·
[../src/backend/commands/repack.c:858](../src/backend/commands/repack.c#L858) `check_concurrent_repack_requirements` ·
[../src/backend/commands/repack.c:1253](../src/backend/commands/repack.c#L1253) `copy_table_data` ·
[../src/backend/commands/repack.c:1499](../src/backend/commands/repack.c#L1499) `swap_relation_files` ·
[../src/backend/replication/pgrepack/pgrepack.c](../src/backend/replication/pgrepack/pgrepack.c) — REPACK'in output plugin'i

## 5.2 `PROPERTY-GRAPH.md`

Bu ağaçta SQL/PGQ property graph desteği var: `CREATE PROPERTY GRAPH`,
`GRAPH_TABLE`. Standart SQL'in graf sorgulama uzantısı.

- Katalog: property graph nasıl saklanıyor
- `GRAPH_TABLE` sorgusunun normal SQL'e çevrilmesi
- Vertex ve edge tanımlarının tablolarla eşleşmesi

Giriş noktaları:
[../src/backend/commands/propgraphcmds.c:104](../src/backend/commands/propgraphcmds.c#L104) `CreatePropGraph` ·
[../src/backend/parser/parse_graphtable.c](../src/backend/parser/parse_graphtable.c) ·
[../src/backend/parser/gram.y](../src/backend/parser/gram.y) — `PropertyGraph` kuralları

## 5.3 `DIGER-INDEX-TURLERI.md`

B-tree notu yazıldıktan sonra karşılaştırmalı bir not: aynı problemi dört
farklı yapı nasıl çözüyor.

- GIN: ters index, posting list, pending list (`fastupdate`)
- GiST: genelleştirilmiş arama ağacı, `penalty` / `picksplit` operatörleri
- BRIN: blok aralığı özeti — devasa tabloda küçücük index
- SP-GiST: alan bölmeli ağaçlar
- Hangisi hangi sorgu tipinde kazanır

Giriş noktaları:
[../src/backend/access/gin/README](../src/backend/access/gin/README) ·
[../src/backend/access/gin/gininsert.c:865](../src/backend/access/gin/gininsert.c#L865) `gininsert` ·
[../src/backend/access/gist/gist.c:639](../src/backend/access/gist/gist.c#L639) `gistdoinsert` ·
[../src/backend/access/brin/brin.c:349](../src/backend/access/brin/brin.c#L349) `brininsert`

---

# Tek sayfalık özet — okuma sırası

```
                      POSTGRES-NASIL-CALISIR  (yazıldı)
                                 │
        ┌────────────────┬───────┴────────┬─────────────────┐
        ▼                ▼                ▼                 ▼
   SELECT (yazıldı)  UPDATE (yazıldı)  KILIT (yazıldı)   [yeni temel]
        │                │                │                 │
        │                │                │        ┌────────┼────────┐
        ▼                ▼                ▼        ▼        ▼        ▼
  ① PLANLAYICI     ① VACUUM         ① TRANSACTION  WAL   BUFFER   BELLEK
  ② EXECUTOR       ② BTREE          ② SNAPSHOT      +     +AIO     BAGLAMLARI
  ③ PARALEL        ③ INSERT/DELETE     KATALOG-    KURTARMA
     PROTOKOL         TOAST            ONBELLEK
     ISTATISTIK       PARTITIONING
        │                │                │
        └────────────────┴────────────────┘
                         │
                         ▼
              REPLIKASYON ──> REPACK (bu ağaca özel)
              GENISLETILEBILIRLIK ──> PROPERTY-GRAPH, DIGER-INDEX-TURLERI

   ①②③ = o daldaki önerilen yazım sırası
```

**Üç notluk minimum paket:** VACUUM → WAL-VE-KURTARMA → BTREE.
Bu üçü bittiğinde mevcut dört notun içindeki "bunu sonra anlatacağız" boşlukları
büyük ölçüde kapanmış olur.

---

## Konu seçme ölçütü

Bir konu şu üç sorunun ikisine "evet" diyorsa yazılmaya değer:

1. Mevcut notlardan biri bu konuya isim verip geçiyor mu?
2. Kodda o konuyu anlatan bir `README` var mı? (varsa iş yarı yarıya biter)
3. Üretimde bu konu insanın başına dert oluyor mu?

`src/backend` altındaki `README` dosyaları, hangi alt sistemin yazılı belgeye
sahip olduğunu gösterir — not yazarken ilk okunacak yer orasıdır:

```
src/backend/access/nbtree/README        src/backend/executor/README
src/backend/access/gin/README           src/backend/optimizer/README
src/backend/access/gist/README          src/backend/storage/aio/README.md
src/backend/access/transam/README       src/backend/replication/README
src/backend/utils/mmgr/README           src/backend/storage/lmgr/README
```
