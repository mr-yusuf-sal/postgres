# VACUUM PostgreSQL içinde nasıl çalışır?

*PostgreSQL **20devel** ağacı. Satır numaraları bu sürüme aittir
([meson.build:11](../meson.build#L11)).*

[UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md) sayfa içi budamayı (HOT pruning)
anlatıyor: bir sayfaya dokunulduğunda ölü tuple'ın *gövdesi* atılabiliyor, ama
line pointer duruyor ve index girişleri hâlâ oraya işaret ediyor. Bu notu asıl
temizlik anlatıyor — line pointer'ları geri kazanmak, index girişlerini silmek
ve XID'leri dondurmak (freeze) için tabloyu baştan sona tarayan iş.

---

## 30 saniyelik özet

```
   VACUUM tablo
        │
        ▼
   ┌────────────────────────────────────────────────────────────┐
   │ FAZ I  — heap taraması (lazy_scan_heap)                    │
   │   her sayfa için:  prune → freeze → ölü TID'leri topla     │
   │   ölü TID'ler TidStore'da birikir (maintenance_work_mem)   │
   └───────────────────────┬────────────────────────────────────┘
                           │  TidStore dolarsa VEYA tarama bitince
                           ▼
   ┌────────────────────────────────────────────────────────────┐
   │ FAZ II — index temizliği (lazy_vacuum_all_indexes)         │
   │   her index taranır, TidStore'daki TID'ler silinir         │
   │   birden fazla index varsa paralel olabilir                │
   └───────────────────────┬────────────────────────────────────┘
                           ▼
   ┌────────────────────────────────────────────────────────────┐
   │ FAZ III — heap temizliği (lazy_vacuum_heap_rel)            │
   │   LP_DEAD line pointer'lar LP_UNUSED yapılır → yer açılır  │
   └───────────────────────┬────────────────────────────────────┘
                           │  TidStore boşaldı → FAZ I kaldığı yerden
                           ▼
        sonda: FSM güncelle → sondaki boş sayfaları kırp →
               pg_class.relfrozenxid / reltuples güncelle
```

Zihinsel model — altı madde:

1. **Sıra keyfî değil, zorunlu.** Index girişi heap'teki line pointer'a işaret
   eder. Line pointer'ı önce boşaltıp yeniden kullanırsan, eski index girişi
   yeni bir satırı gösterir. Bu yüzden *önce* index girişleri silinir, *sonra*
   line pointer serbest bırakılır.
2. **Bellek sınırı fazları döngüye çevirir.** Ölü TID'ler `maintenance_work_mem`
   kadar yer tutabilir. Dolarsa VACUUM Faz I'i askıya alır, II ve III'ü çalıştırır,
   listeyi boşaltır ve kaldığı yerden devam eder. Bu yüzden `VACUUM VERBOSE`
   çıktısında "index scans: 3" görürsün — bellek üç kez dolmuş demektir.
3. **İki mod var: normal ve aggressive.** Normal VACUUM visibility map'e güvenip
   sayfa atlar. Aggressive VACUUM her donmamış tuple'a bakmak zorundadır, çünkü
   `relfrozenxid`'i ilerletmesi gerekir.
4. **Freeze, wraparound'a karşı sigortadır.** XID 32 bitlik ve döngüseldir. Çok
   eski bir XID taşıyan tuple donduruluyor; donmuş tuple "her zaman görünür"
   sayılır ve artık XID karşılaştırmasına girmez.
5. **Failsafe her şeyi iptal edebilir.** `relfrozenxid` tehlikeli derecede
   geride kalırsa VACUUM index temizliğini, heap temizliğini ve kırpmayı bırakır;
   sadece dondurmaya odaklanır.
6. **Autovacuum ayrı bir dünya.** Launcher süreci hangi veritabanına worker
   göndereceğine karar verir; worker tabloları eşiklere göre seçer. Bu ağaçta
   tablolar ayrıca **skor**lanıp sıralanıyor — wraparound'a yaklaşan tablo öne
   geçiyor.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/commands/vacuum.c](../src/backend/commands/vacuum.c) | Komut yüzeyi, tablo listesi, cutoff hesabı, `pg_class` güncellemesi, gecikme (delay) |
| [../src/backend/access/heap/vacuumlazy.c](../src/backend/access/heap/vacuumlazy.c) | Asıl motor: üç faz, freeze kararı, kırpma. **En önemli dosya** |
| [../src/backend/access/heap/pruneheap.c](../src/backend/access/heap/pruneheap.c) | Sayfa içi budama + dondurma (`heap_page_prune_and_freeze`) |
| [../src/backend/access/heap/heapam.c](../src/backend/access/heap/heapam.c) | Tuple bazında freeze planı (`heap_prepare_freeze_tuple`) |
| [../src/backend/access/heap/visibilitymap.c](../src/backend/access/heap/visibilitymap.c) | İki bitlik harita: all-visible / all-frozen |
| [../src/backend/commands/vacuumparallel.c](../src/backend/commands/vacuumparallel.c) | Faz II'nin paralel çalıştırılması |
| [../src/backend/postmaster/autovacuum.c](../src/backend/postmaster/autovacuum.c) | Launcher + worker, eşikler, skorlama |
| [../src/backend/access/common/tidstore.c](../src/backend/access/common/tidstore.c) | Ölü TID'lerin sıkıştırılmış deposu (radix tree) |
| [../src/backend/commands/repack.c](../src/backend/commands/repack.c) | `VACUUM FULL` buraya yönlendirilir — ayrı bir motor |

---

# 1. Giriş: komuttan motora

`VACUUM` bir utility komutu. Yol şöyle:

[../src/backend/commands/vacuum.c:163](../src/backend/commands/vacuum.c#L163) —
`ExecVacuum` seçenekleri ayrıştırır, sonra
[../src/backend/commands/vacuum.c:494](../src/backend/commands/vacuum.c#L494)
`vacuum()` tablo listesini gezer ve her biri için
[../src/backend/commands/vacuum.c:2030](../src/backend/commands/vacuum.c#L2030)
`vacuum_rel()` çağrılır.

`vacuum_rel` içinde kritik ayrım şu:

```c
		if (params.options & VACOPT_FULL)
		{
			...
			cluster_rel(REPACK_COMMAND_VACUUMFULL, rel, InvalidOid,
```
— [../src/backend/commands/vacuum.c:2314-2322](../src/backend/commands/vacuum.c#L2314-L2322)

**`VACUUM FULL` bu notun konusu değil.** Bu ağaçta `VACUUM FULL` artık
`repack.c` motoruna yönlendiriliyor: tabloyu yeni bir heap'e kopyalar ve
dosyaları takas eder, `ACCESS EXCLUSIVE` kilit tutar. Ayrıntısı
`REPACK-NASIL-CALISIR.md` notunda.

Normal (lazy) VACUUM ise tablo access method üzerinden
[../src/backend/access/heap/vacuumlazy.c:624](../src/backend/access/heap/vacuumlazy.c#L624)
`heap_vacuum_rel`'e iner. Alınan kilit `ShareUpdateExclusiveLock`'tur — okuma ve
yazma engellenmez, sadece başka bir VACUUM/ANALYZE/DDL engellenir.

---

# 2. Üç faz — dosyanın kendi tarifi

`vacuumlazy.c`'nin baş yorumu algoritmayı tek paragrafta anlatıyor; bu notun
iskeleti de odur:

```
 * Heap relations are vacuumed in three main phases. In phase I, vacuum scans
 * relation pages, pruning and freezing tuples and saving dead tuples' TIDs in
 * a TID store. If that TID store fills up or vacuum finishes scanning the
 * relation, it progresses to phase II: index vacuuming. Index vacuuming
 * deletes the dead index entries referenced in the TID store. In phase III,
 * vacuum scans the blocks of the relation referred to by the TIDs in the TID
 * store and reaps the corresponding dead items, freeing that space for future
 * tuples.
```
— [../src/backend/access/heap/vacuumlazy.c:6-13](../src/backend/access/heap/vacuumlazy.c#L6-L13)

Devamında iki önemli kaçış yolu tarif ediliyor:

```
 * If there are no indexes or index scanning is disabled, phase II may be
 * skipped. If phase I identified very few dead index entries or if vacuum's
 * failsafe mechanism has triggered (to avoid transaction ID wraparound),
 * vacuum may skip phases II and III.
```
— [../src/backend/access/heap/vacuumlazy.c:15-18](../src/backend/access/heap/vacuumlazy.c#L15-L18)

Yorumun kendisi bu "faz" kelimesinin aslında yanlış olduğunu da itiraf ediyor:
bunlar bir durum makinesinin durumları, çünkü Faz I ↔ II ↔ III arasında
gidip gelinebiliyor ([satır 24-26](../src/backend/access/heap/vacuumlazy.c#L24-L26)).

---

# 3. Cutoff hesabı — VACUUM ne siler, ne dondurur?

Her şey dört sayıya bağlı. Bunlar
[../src/backend/commands/vacuum.c:1124](../src/backend/commands/vacuum.c#L1124)
`vacuum_get_cutoffs` içinde hesaplanır ve `VacuumCutoffs` yapısında döner.

| Alan | Anlamı | Kullanımı |
|---|---|---|
| `OldestXmin` | Sistemdeki en eski canlı snapshot'ın gördüğü XID | Bundan önce ölmüş tuple **silinebilir** |
| `FreezeLimit` | `nextXID - vacuum_freeze_min_age` | Bundan eski XID'ler **dondurulmalı** |
| `OldestMxact` | En eski canlı MultiXactId | MultiXact silme sınırı |
| `MultiXactCutoff` | `nextMXID - vacuum_multixact_freeze_min_age` | MultiXact dondurma sınırı |

`OldestXmin` şuradan gelir:

```c
	cutoffs->OldestXmin = GetOldestNonRemovableTransactionId(rel);
```
— [../src/backend/commands/vacuum.c:1160](../src/backend/commands/vacuum.c#L1160)

**Bu tek satır, üretimdeki en yaygın VACUUM şikâyetinin sebebidir.** Açık kalmış
tek bir uzun transaction (ya da unutulmuş bir replication slot, ya da
`hot_standby_feedback` ile geri bildirim yapan bir standby) `OldestXmin`'i
geride tutar; VACUUM tabloyu tarar, çalışır, ama hiçbir şey silemez.

`FreezeLimit` hesabı, kullanıcının ayarını kendi başına sınırlıyor:

```c
	if (freeze_min_age < 0)
		freeze_min_age = vacuum_freeze_min_age;
	freeze_min_age = Min(freeze_min_age, autovacuum_freeze_max_age / 2);
```
— [../src/backend/commands/vacuum.c:1206-1208](../src/backend/commands/vacuum.c#L1206-L1208)

`vacuum_freeze_min_age`'i ne kadar büyük ayarlarsan ayarla, tavan
`autovacuum_freeze_max_age / 2`. Sebep yorumda yazıyor: aksi halde
anti-wraparound autovacuum'lar çok sık tetiklenirdi.

## 3.1 Aggressive kararı aynı fonksiyonda veriliyor

`vacuum_get_cutoffs`'un dönüş değeri `bool aggressive`:

```c
	aggressiveXIDCutoff = nextXID - freeze_table_age;
	if (!TransactionIdIsNormal(aggressiveXIDCutoff))
		aggressiveXIDCutoff = FirstNormalTransactionId;
	if (TransactionIdPrecedesOrEquals(cutoffs->relfrozenxid,
									  aggressiveXIDCutoff))
		return true;
```
— [../src/backend/commands/vacuum.c:1252-1257](../src/backend/commands/vacuum.c#L1252-L1257)

Yani: tablonun `relfrozenxid`'i `vacuum_freeze_table_age` kadar geride kaldıysa,
bu VACUUM aggressive olur. Çağıran taraf:

```c
	vacrel->aggressive = vacuum_get_cutoffs(rel, params, &vacrel->cutoffs);
```
— [../src/backend/access/heap/vacuumlazy.c:802](../src/backend/access/heap/vacuumlazy.c#L802)

`DISABLE_PAGE_SKIPPING` seçeneği verildiğinde de mod zorla aggressive'e çekilir
([satır 820-823](../src/backend/access/heap/vacuumlazy.c#L820-L823)).

---

# 4. Faz I — heap taraması

[../src/backend/access/heap/vacuumlazy.c:1279](../src/backend/access/heap/vacuumlazy.c#L1279)
`lazy_scan_heap`. Bu ağaçta tarama artık **read stream** üzerinden yapılıyor —
klasik "blok blok `ReadBuffer`" döngüsü değil:

```c
	stream = read_stream_begin_relation(READ_STREAM_MAINTENANCE,
										vacrel->bstrategy,
										vacrel->rel,
										MAIN_FORKNUM,
										heap_vac_scan_next_block,
										vacrel,
										sizeof(bool));
```
— [../src/backend/access/heap/vacuumlazy.c:1313-1319](../src/backend/access/heap/vacuumlazy.c#L1313-L1319)

Hangi bloğun okunacağına
[../src/backend/access/heap/vacuumlazy.c:1648](../src/backend/access/heap/vacuumlazy.c#L1648)
`heap_vac_scan_next_block` callback'i karar veriyor; stream bu callback'i önden
çağırıp okumaları birleştirebiliyor. (Read stream ve AIO altyapısı için
`BUFFER-VE-AIO.md`.)

## 4.1 Sayfa atlama — visibility map

Visibility map sayfa başına **iki bit** tutar:

```
 * The visibility map is a bitmap with two bits (all-visible and all-frozen)
 * per heap page. A set all-visible bit means that all tuples on the page are
 * ... A set all-frozen bit means that all tuples on the page are
 * ...
 * The all-frozen bit must be set only when the page is already all-visible.
```
— [../src/backend/access/heap/visibilitymap.c:25-31](../src/backend/access/heap/visibilitymap.c#L25-L31)

Normal VACUUM all-visible sayfaları atlayabilir; aggressive VACUUM sadece
all-frozen sayfaları atlayabilir. Atlanacak blokları
[../src/backend/access/heap/vacuumlazy.c:1748](../src/backend/access/heap/vacuumlazy.c#L1748)
`find_next_unskippable_block` bulur.

Atlamanın iki istisnası var:

**(a) Kısa atlama yapma.** Çekirdeğin readahead'ini bozmamak için 32 sayfadan
kısa atlama aralıkları atlanmaz:

```c
#define SKIP_PAGES_THRESHOLD	((BlockNumber) 32)
```
— [../src/backend/access/heap/vacuumlazy.c:209](../src/backend/access/heap/vacuumlazy.c#L209)

**(b) Eager scanning — bu ağacın yeni davranışı.** Normal VACUUM, atlayabileceği
bazı all-visible sayfaları yine de okuyup dondurmaya çalışır; amaç bir sonraki
aggressive VACUUM'un yükünü azaltmak. Ama bu iş sınırlandırılmış:

```c
#define MAX_EAGER_FREEZE_SUCCESS_RATE 0.2
```
— [../src/backend/access/heap/vacuumlazy.c:241](../src/backend/access/heap/vacuumlazy.c#L241)

```c
#define EAGER_SCAN_REGION_SIZE 4096
```
— [../src/backend/access/heap/vacuumlazy.c:250](../src/backend/access/heap/vacuumlazy.c#L250)

Mantık: başarı (gerçekten donan sayfa) **global** olarak, all-visible-ama-frozen-
olmayan sayfaların %20'siyle sınırlı; başarısızlık ise 4096 bloklu **bölgeler**
bazında sayılıyor ve `vacuum_max_eager_freeze_failure_rate` oranı aşılınca o
bölgede eager tarama askıya alınıyor. Kurulumu
[../src/backend/access/heap/vacuumlazy.c:497](../src/backend/access/heap/vacuumlazy.c#L497)
`heap_vacuum_eager_scan_setup` yapıyor. Dosyanın baş yorumu bu tasarımı
[satır 53-88](../src/backend/access/heap/vacuumlazy.c#L53-L88) arasında
gerekçesiyle anlatıyor.

## 4.2 Sayfa işleme — cleanup lock var mı?

Prune ve defragment için **cleanup lock** gerekir (sayfayı sadece sen pinlemiş
olmalısın). Alınamazsa VACUUM beklemez:

- alınırsa → [../src/backend/access/heap/vacuumlazy.c:2021](../src/backend/access/heap/vacuumlazy.c#L2021) `lazy_scan_prune`
- alınamazsa → [../src/backend/access/heap/vacuumlazy.c:2158](../src/backend/access/heap/vacuumlazy.c#L2158) `lazy_scan_noprune`

`lazy_scan_noprune` sayfayı sadece inceler: dondurma gerekiyor mu diye bakar,
mevcut LP_DEAD item'ları toplayabilir, ama budama yapmaz. Aggressive VACUUM ise
cleanup lock'ı beklemek zorundadır — çünkü `relfrozenxid`'i ilerletebilmek için
sayfadaki her tuple'ı işlemesi gerekir.

Asıl iş `lazy_scan_prune` içinde tek bir çağrıya devrediliyor:

```c
	heap_page_prune_and_freeze(&params,
							   &presult,
							   &vacrel->offnum,
							   &vacrel->NewRelfrozenXid, &vacrel->NewRelminMxid);
```
— [../src/backend/access/heap/vacuumlazy.c:2071-2074](../src/backend/access/heap/vacuumlazy.c#L2071-L2074)

[../src/backend/access/heap/pruneheap.c:1113](../src/backend/access/heap/pruneheap.c#L1113)
`heap_page_prune_and_freeze` tek geçişte hem budar, hem dondurma planı çıkarır,
hem de ölü offset'leri toplar. Bu birleşme önemli: sayfa bir kez kilitlenir, bir
kez WAL'lanır.

Dönen ölü offset'ler sıralanıp depoya eklenir:

```c
		qsort(presult.deadoffsets, presult.lpdead_items, sizeof(OffsetNumber),
			  cmpOffsetNumbers);

		dead_items_add(vacrel, blkno, presult.deadoffsets, presult.lpdead_items);
```
— [../src/backend/access/heap/vacuumlazy.c:2103-2106](../src/backend/access/heap/vacuumlazy.c#L2103-L2106)

## 4.3 Ölü TID deposu ve `maintenance_work_mem`

Depo bir TidStore (radix tree tabanlı, sıkıştırılmış). Bütçesi:

```c
	int			vac_work_mem = AmAutoVacuumWorkerProcess() &&
		autovacuum_work_mem != -1 ?
		autovacuum_work_mem : maintenance_work_mem;
```
— [../src/backend/access/heap/vacuumlazy.c:3419-3421](../src/backend/access/heap/vacuumlazy.c#L3419-L3421)

Yani autovacuum worker'ı `autovacuum_work_mem`, elle çalıştırılan VACUUM
`maintenance_work_mem` kullanır. Tarama döngüsünde bütçe kontrolü:

```c
		if (vacrel->dead_items_info->num_items > 0 &&
			TidStoreMemoryUsage(vacrel->dead_items) > vacrel->dead_items_info->max_bytes)
```
— [../src/backend/access/heap/vacuumlazy.c:1355-1356](../src/backend/access/heap/vacuumlazy.c#L1355-L1356)

Doluysa `lazy_vacuum(vacrel)` çağrılır — Faz II ve III burada araya giriyor,
sonra tarama devam ediyor. `VACUUM VERBOSE` çıktısındaki *index scans* sayısı
bu döngünün kaç kez döndüğüdür ve **1'den büyükse `maintenance_work_mem` düşük**
demektir.

---

# 5. Faz II — index temizliği

[../src/backend/access/heap/vacuumlazy.c:2369](../src/backend/access/heap/vacuumlazy.c#L2369)
`lazy_vacuum` önce ilginç bir soru soruyor: *index temizliğini tamamen atlasak mı?*

```c
		threshold = (double) vacrel->rel_pages * BYPASS_THRESHOLD_PAGES;
		bypass = (vacrel->lpdead_item_pages < threshold &&
				  TidStoreMemoryUsage(vacrel->dead_items) < 32 * 1024 * 1024);
```
— [../src/backend/access/heap/vacuumlazy.c:2435-2437](../src/backend/access/heap/vacuumlazy.c#L2435-L2437)

`BYPASS_THRESHOLD_PAGES` %2'dir
([satır 187](../src/backend/access/heap/vacuumlazy.c#L187)). Yani ölü item taşıyan
sayfa sayısı tablonun %2'sinden azsa ve depo 32 MB'ı geçmiyorsa, index taraması
hiç yapılmaz. Gerekçe yorumda çok net:

```
 * This is likely to be helpful with a table that is continually affected
 * by UPDATEs that can mostly apply the HOT optimization, but occasionally
 * have small aberrations that lead to just a few heap pages retaining
 * only one or two LP_DEAD items.
```
— [../src/backend/access/heap/vacuumlazy.c:2394-2397](../src/backend/access/heap/vacuumlazy.c#L2394-L2397)

Bir index'i tam taramak, iki ölü giriş için ödenecek bedelden çok daha pahalı.

Atlanmıyorsa
[../src/backend/access/heap/vacuumlazy.c:2494](../src/backend/access/heap/vacuumlazy.c#L2494)
`lazy_vacuum_all_indexes` her index için `ambulkdelete` çağırır
([../src/backend/access/heap/vacuumlazy.c:3013](../src/backend/access/heap/vacuumlazy.c#L3013)
`lazy_vacuum_one_index`). Index AM'i indexi baştan sona tarar ve TidStore'da
bulunan her TID'i siler — B-tree için bu tam bir index taramasıdır.

## 5.1 Paralel index temizliği

İki koşul: en az iki index olacak ve tablo geçici olmayacak.

```c
	if (nworkers >= 0 && vacrel->nindexes > 1 && vacrel->do_index_vacuuming)
```
— [../src/backend/access/heap/vacuumlazy.c:3428](../src/backend/access/heap/vacuumlazy.c#L3428)

Bir index'i tek bir worker işler; paralellik index sayısı kadardır, index içi
paralellik yoktur. Bu ağaçta paralellik autovacuum'a da açılmış:

```
 * For parallel autovacuum, we need to propagate cost-based vacuum delay
 * parameters from the leader to its workers, as the leader's parameters can
 * change even while processing a table (e.g., due to a config reload).
```
— [../src/backend/commands/vacuumparallel.c:21-23](../src/backend/commands/vacuumparallel.c#L21-L23)

---

# 6. Faz III — heap temizliği

[../src/backend/access/heap/vacuumlazy.c:2640](../src/backend/access/heap/vacuumlazy.c#L2640)
`lazy_vacuum_heap_rel`, TidStore'daki blokları ikinci kez ziyaret eder ve her
sayfada
[../src/backend/access/heap/vacuumlazy.c:2758](../src/backend/access/heap/vacuumlazy.c#L2758)
`lazy_vacuum_heap_page` çağırır. Burada LP_DEAD line pointer'lar LP_UNUSED
yapılır, sayfa defragment edilir, WAL kaydı yazılır ve sayfa artık gerçekten
all-visible olduysa visibility map biti
[../src/backend/access/heap/visibilitymap.c:260](../src/backend/access/heap/visibilitymap.c#L260)
`visibilitymap_set` ile işaretlenir.

**Yer bu fazda açılır.** Faz I'in sonunda tablo hâlâ şişkindir; Faz III bitmeden
`pg_freespacemap` yeni yeri göstermez.

---

# 7. Freeze — wraparound'a karşı

XID 32 bit; yaklaşık 4 milyar transaction sonra başa döner. Karşılaştırmalar
modüler yapıldığı için "geçmiş" ve "gelecek" ancak yarım tur içinde anlamlıdır.
Çözüm: yeterince eski XID taşıyan tuple'ları **dondurmak** — donmuş tuple
"herkes tarafından görülebilir" kabul edilir ve XID'sine bir daha bakılmaz.

Karar tuple bazında verilir:
[../src/backend/access/heap/heapam.c:7267](../src/backend/access/heap/heapam.c#L7267)
`heap_prepare_freeze_tuple`. Fonksiyonun yorumu iki kritik noktayı söylüyor:

```
 * VACUUM caller decides on whether or not to freeze the page as a whole.
 * We'll often prepare freeze plans for a page that caller just discards.
 * However, VACUUM doesn't always get to make a choice; it must freeze when
 * pagefrz.freeze_required is set, to ensure that any XIDs < FreezeLimit (and
 * MXIDs < MultiXactCutoff) can never be left behind.
```
— [../src/backend/access/heap/heapam.c:7242-7246](../src/backend/access/heap/heapam.c#L7242-L7246)

Yani iki ayrı karar var:

- **Zorunlu freeze:** `FreezeLimit`'ten eski XID varsa seçenek yok, sayfa donacak.
- **İsteğe bağlı freeze:** sayfa zaten yazılacaksa, "hazır buradayken" dondurmak
  ucuza gelir; VACUUM bu planı kabul veya reddedebilir.

Bir başka uyarı da orada:

```
 * NB: This function has side effects: it might allocate a new MultiXactId.
```
— [../src/backend/access/heap/heapam.c:7261](../src/backend/access/heap/heapam.c#L7261)

Dondurma, `xmax`'ta MultiXact varsa yeni bir MultiXact üretebiliyor — freeze'in
her zaman "sadeleştirme" olmadığının kanıtı.

## 7.1 Sonuç: `relfrozenxid` ilerlemesi

Tarama boyunca `vacrel->NewRelfrozenXid` biriktirilir; sonda
[../src/backend/commands/vacuum.c:1450](../src/backend/commands/vacuum.c#L1450)
`vac_update_relstats` ile `pg_class`'a yazılır. Aggressive VACUUM `relfrozenxid`'i
en az `FreezeLimit`'e taşımak zorundadır; normal VACUUM istediği kadar taşır ya
da hiç taşımaz ([../src/backend/access/heap/vacuumlazy.c:926-932](../src/backend/access/heap/vacuumlazy.c#L926-L932)).

Ardından veritabanı seviyesine çıkılır:
[../src/backend/commands/vacuum.c:1632](../src/backend/commands/vacuum.c#L1632)
`vac_update_datfrozenxid` → [../src/backend/commands/vacuum.c:1853](../src/backend/commands/vacuum.c#L1853)
`vac_truncate_clog`. **CLOG dosyaları ancak burada silinir** — yani tek bir
ihmal edilmiş tablo, tüm kümenin CLOG'unu şişirebilir.

---

# 8. Failsafe — acil durum freni

Wraparound'a gerçekten yaklaşıldıysa VACUUM'un temizlik yapacak vakti yoktur.
Kontrol tarama sırasında düzenli aralıklarla yapılır:

```c
#define FAILSAFE_EVERY_PAGES \
	((BlockNumber) (((uint64) 4 * 1024 * 1024 * 1024) / BLCKSZ))
```
— [../src/backend/access/heap/vacuumlazy.c:193-194](../src/backend/access/heap/vacuumlazy.c#L193-L194)

Her 4 GB'da bir
[../src/backend/access/heap/vacuumlazy.c:1345](../src/backend/access/heap/vacuumlazy.c#L1345)
`lazy_check_wraparound_failsafe` çağrılır. Tetiklenirse
([../src/backend/access/heap/vacuumlazy.c:2890](../src/backend/access/heap/vacuumlazy.c#L2890)):

```c
		vacrel->bstrategy = NULL;

		/* Disable index vacuuming, index cleanup, and heap rel truncation */
		vacrel->do_index_vacuuming = false;
		vacrel->do_index_cleanup = false;
		vacrel->do_rel_truncate = false;
```
— [../src/backend/access/heap/vacuumlazy.c:2912-2916](../src/backend/access/heap/vacuumlazy.c#L2912-L2916)

Üç şey birden olur: **(1)** ring buffer bırakılır, VACUUM tüm `shared_buffers`'ı
kullanabilir; **(2)** index ve heap temizliği ile kırpma iptal edilir; **(3)**
maliyet gecikmesi kapatılır:

```c
		/* Stop applying cost limits from this point on */
		VacuumCostActive = false;
		VacuumCostBalance = 0;
```
— [../src/backend/access/heap/vacuumlazy.c:2930-2932](../src/backend/access/heap/vacuumlazy.c#L2930-L2932)

Log'da şu uyarı görünür: *"bypassing nonessential maintenance of table ... as a
failsafe"*. Bu mesajı gördüysen kümede bir sorun var — VACUUM artık sadece
dondurmaya çalışıyor.

Eşik [../src/backend/commands/vacuum.c:1292](../src/backend/commands/vacuum.c#L1292)
`vacuum_xid_failsafe_check` içinde; `vacuum_failsafe_age` /
`vacuum_multixact_failsafe_age` GUC'larıyla ayarlanır.

---

# 9. Sonlandırma: kırpma ve istatistik

## 9.1 Sondaki boş sayfaları kırpma

Kırpma ancak kazanç yeterliyse denenir:

```c
	possibly_freeable = vacrel->rel_pages - vacrel->nonempty_pages;
	if (possibly_freeable > 0 &&
		(possibly_freeable >= REL_TRUNCATE_MINIMUM ||
		 possibly_freeable >= vacrel->rel_pages / REL_TRUNCATE_FRACTION))
		return true;
```
— [../src/backend/access/heap/vacuumlazy.c:3129-3132](../src/backend/access/heap/vacuumlazy.c#L3129-L3132)

`REL_TRUNCATE_MINIMUM` = 1000 sayfa, `REL_TRUNCATE_FRACTION` = 16 (yani %6,25)
— [satır 169-170](../src/backend/access/heap/vacuumlazy.c#L169-L170).

Kırpmanın maliyeti ise
[../src/backend/access/heap/vacuumlazy.c:3142](../src/backend/access/heap/vacuumlazy.c#L3142)
`lazy_truncate_heap` içinde: **`AccessExclusiveLock` gerekir.** VACUUM bu kilidi
almaya çalışır ama başkasını bekletmemek için 5 saniyede vazgeçer
(`VACUUM_TRUNCATE_LOCK_TIMEOUT`, [satır 181](../src/backend/access/heap/vacuumlazy.c#L181)).
Bu yüzden yoğun bir tabloda VACUUM sık sık tabloyu küçültemez.

## 9.2 FSM ve istatistikler

FSM, fazlar arasında ve indexsiz tablolarda periyodik olarak güncellenir:

```c
#define VACUUM_FSM_EVERY_PAGES \
	((BlockNumber) (((uint64) 8 * 1024 * 1024 * 1024) / BLCKSZ))
```
— [../src/backend/access/heap/vacuumlazy.c:202-203](../src/backend/access/heap/vacuumlazy.c#L202-L203)

Sonda `pg_class.reltuples` / `relpages` / `relallvisible` / `relallfrozen`
güncellenir. `reltuples` **tahmindir**, çünkü VACUUM sayfa atlamış olabilir;
karışım [../src/backend/commands/vacuum.c:1354](../src/backend/commands/vacuum.c#L1354)
`vac_estimate_reltuples` içinde yapılır.

---

# 10. Autovacuum

## 10.1 İki tür süreç

**Launcher** — [../src/backend/postmaster/autovacuum.c:411](../src/backend/postmaster/autovacuum.c#L411)
`AutoVacLauncherMain`. Veri okumaz; sadece hangi veritabanına ne zaman worker
göndereceğine karar verir. Uyku süresi
[../src/backend/postmaster/autovacuum.c:847](../src/backend/postmaster/autovacuum.c#L847)
`launcher_determine_sleep` ile hesaplanır: hedef, `autovacuum_naptime` içinde her
veritabanına bir kez uğramaktır.

**Worker** — [../src/backend/postmaster/autovacuum.c:1418](../src/backend/postmaster/autovacuum.c#L1418)
`AutoVacWorkerMain` → [../src/backend/postmaster/autovacuum.c:1928](../src/backend/postmaster/autovacuum.c#L1928)
`do_autovacuum`. Bir veritabanına bağlanır, tüm tabloları gezer, eşikleri aşanları
işler.

## 10.2 Eşik formülü

[../src/backend/postmaster/autovacuum.c:3074](../src/backend/postmaster/autovacuum.c#L3074)
`relation_needs_vacanalyze` içinde, üç eşik:

```c
	vacthresh = (float4) vac_base_thresh + vac_scale_factor * reltuples;
	if (vac_max_thresh >= 0 && vacthresh > (float4) vac_max_thresh)
		vacthresh = (float4) vac_max_thresh;

	vacinsthresh = (float4) vac_ins_base_thresh +
		vac_ins_scale_factor * reltuples * pcnt_unfrozen;
	anlthresh = (float4) anl_base_thresh + anl_scale_factor * reltuples;
```
— [../src/backend/postmaster/autovacuum.c:3286-3292](../src/backend/postmaster/autovacuum.c#L3286-L3292)

Üç ayrıntı:

- **`vac_max_thresh` bir tavandır** (`autovacuum_vacuum_max_threshold`). Klasik
  formülün büyük tablolarda ürettiği devasa eşik böylece sınırlanıyor — büyük
  tabloların "hiç vacuum'lanmama" sorununun çözümü.
- **Insert eşiği donmuş oranıyla ölçekleniyor:** `pcnt_unfrozen` çarpanı,
  tablonun sadece "aktif" (donmamış) kısmını dikkate alıyor
  ([satır 3274-3283](../src/backend/postmaster/autovacuum.c#L3275-L3283)).
- Wraparound zorlaması eşiklerden bağımsızdır; `autovacuum = off` olsa bile
  çalışır ([satır 3185-3198](../src/backend/postmaster/autovacuum.c#L3185-L3198)).

## 10.3 Skorlama — bu ağacın yeni davranışı

Eski autovacuum tabloları bulduğu sırayla işlerdi. Bu ağaçta her tabloya bir
**skor** veriliyor ve liste skora göre sıralanıyor:

```c
	scores->vac = (double) vactuples / Max(vacthresh, 1);
	scores->vac *= autovacuum_vacuum_score_weight;
	scores->max = Max(scores->max, scores->vac);
```
— [../src/backend/postmaster/autovacuum.c:3295-3297](../src/backend/postmaster/autovacuum.c#L3295-L3297)

Skor, tablonun eşiğini kaç kat aştığıdır. Beş bileşen var (vacuum, insert,
analyze, XID yaşı, MultiXact yaşı) ve her birinin ağırlığı ayarlanabiliyor:
`autovacuum_freeze_score_weight`, `autovacuum_vacuum_score_weight`,
`autovacuum_analyze_score_weight` vb.
([satır 3062-3068](../src/backend/postmaster/autovacuum.c#L3062-L3068)).

Wraparound'a yaklaşan tablo öne çıksın diye XID skoru üstel olarak büyütülüyor:

```c
	if (xid_age >= effective_xid_failsafe_age)
		scores->xid = pow(scores->xid, Max(1.0, (double) xid_age / 100000000));
```
— [../src/backend/postmaster/autovacuum.c:3237-3238](../src/backend/postmaster/autovacuum.c#L3237-L3238)

Sıralama [../src/backend/postmaster/autovacuum.c:1913](../src/backend/postmaster/autovacuum.c#L1913)
`TableToProcessComparator` ile yapılıyor. Skorları dışarıdan görmek için
[../src/backend/postmaster/autovacuum.c:3650](../src/backend/postmaster/autovacuum.c#L3650)
`pg_stat_get_autovacuum_scores` fonksiyonu var.

Yorumda tüm ağırlıkları 0.0 yapmanın eski davranışa döndürdüğü de yazıyor
([satır 3060](../src/backend/postmaster/autovacuum.c#L3060)).

## 10.4 Maliyet gecikmesi

Autovacuum'un I/O'yu boğmaması için her sayfa işleminden sonra bir "borç"
birikir ve limit aşılınca uyunur:
[../src/backend/commands/vacuum.c:2457](../src/backend/commands/vacuum.c#L2457)
`vacuum_delay_point`. Limit çalışan worker sayısına bölünerek dağıtılır
([../src/backend/postmaster/autovacuum.c:1753](../src/backend/postmaster/autovacuum.c#L1753)
`AutoVacuumUpdateCostLimit`) — yani `autovacuum_max_workers`'ı artırmak tek
başına hızlanma getirmez, aynı pastayı böler.

---

# 11. İzleme ve hata ayıklama

**Şişkinlik ve son vacuum zamanı:**

```sql
SELECT relname,
       n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_vacuum, last_autovacuum, autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

**Wraparound'a ne kadar kaldı:**

```sql
SELECT c.relname,
       age(c.relfrozenxid) AS xid_age,
       mxid_age(c.relminmxid) AS mxid_age,
       current_setting('autovacuum_freeze_max_age')::int - age(c.relfrozenxid) AS kalan
FROM pg_class c
WHERE c.relkind IN ('r','m','t')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 20;
```

**`OldestXmin`'i kim geride tutuyor** (VACUUM çalışıp da hiçbir şey silmiyorsa):

```sql
-- 1) uzun transaction'lar
SELECT pid, state, backend_xmin, now() - xact_start AS sure, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC;

-- 2) unutulmuş replication slot'lar
SELECT slot_name, active, xmin, catalog_xmin,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS tutulan_wal
FROM pg_replication_slots;

-- 3) hazırlanmış (prepared) transaction'lar
SELECT * FROM pg_prepared_xacts;
```

**Çalışan VACUUM'u izleme** — faz isimleri
[../src/include/commands/progress.h:36](../src/include/commands/progress.h#L36)
civarındaki sabitlerden gelir:

```sql
SELECT p.pid, p.relid::regclass, p.phase,
       p.heap_blks_scanned, p.heap_blks_total,
       p.index_vacuum_count,
       pg_size_pretty(p.max_dead_tuple_bytes) AS butce,
       pg_size_pretty(p.dead_tuple_bytes)     AS kullanilan
FROM pg_stat_progress_vacuum p;
```

`phase` sütunundaki değerler doğrudan bu notun fazlarıdır:
*scanning heap* (Faz I), *vacuuming indexes* (Faz II), *vacuuming heap* (Faz III),
*truncating heap* (bölüm 9.1).

**Autovacuum sıra skorları** (bu ağaca özel):

```sql
SELECT * FROM pg_stat_get_autovacuum_scores();
```

**Ayar önerileri — hangi belirti, hangi düğme:**

| Belirti | Bakılacak yer |
|---|---|
| `VERBOSE`'da "index scans: 3" | `maintenance_work_mem` / `autovacuum_work_mem` düşük |
| "bypassing nonessential maintenance" uyarısı | Failsafe tetiklendi; wraparound acil |
| Tablo şişiyor ama `n_dead_tup` düşük | Ölü tuple'lar `OldestXmin` yüzünden silinemiyor |
| Autovacuum sürekli çalışıyor, bitmiyor | `autovacuum_vacuum_cost_delay` / `cost_limit` |
| Büyük tablo hiç vacuum'lanmıyor | `autovacuum_vacuum_max_threshold` (tavan) |
| Sondaki boş alan geri verilmiyor | Kırpma `AccessExclusiveLock` alamıyor (5 sn timeout) |

**Elle inceleme:**

```sql
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstattuple('tablom');          -- gerçek şişkinlik
SELECT * FROM pg_visibility_map_summary('tablom');  -- pg_visibility eklentisi
```

---

# 12. Tek sayfalık özet

```
 VACUUM tablo                                 autovacuum
      │                                            │
      │                             launcher ──> worker (naptime/db)
      ▼                                            │
 vacuum_rel ──── VACOPT_FULL? ──> repack.c         │  relation_needs_vacanalyze
      │ hayır         (ayrı motor)                 │  vacthresh = base + factor*reltuples
      ▼                                            │  (tavan: vacuum_max_threshold)
 heap_vacuum_rel                                   │  skor = eşiğin kaç katı
      │  ShareUpdateExclusiveLock                  │  → liste skora göre sıralanır
      ▼
 vacuum_get_cutoffs ──> OldestXmin, FreezeLimit, aggressive?
      │
      ▼
 ┌─ FAZ I ── lazy_scan_heap ─────────────────────────────────────┐
 │  read stream ──> blok                                         │
 │     atla?  (visibility map + SKIP_PAGES_THRESHOLD + eager)    │
 │     cleanup lock var mı?                                      │
 │        evet ──> lazy_scan_prune ──> heap_page_prune_and_freeze│
 │        hayır ─> lazy_scan_noprune (budama yok)                │
 │     ölü TID'ler ──> TidStore                                  │
 │     her 4 GB'da failsafe kontrolü                             │
 └───────────────┬───────────────────────────────────────────────┘
                 │ TidStore doldu ya da tarama bitti
                 ▼
        lazy_vacuum:  ölü sayfa < %2 ve depo < 32 MB  ──> ATLA
                 │ değilse
                 ▼
 ┌─ FAZ II ─ lazy_vacuum_all_indexes ── her index: ambulkdelete ─┐
 │           (>1 index ve kalıcı tablo ise paralel olabilir)     │
 └───────────────┬───────────────────────────────────────────────┘
                 ▼
 ┌─ FAZ III ─ lazy_vacuum_heap_rel ── LP_DEAD → LP_UNUSED ───────┐
 │            sayfa all-visible/all-frozen ise VM biti kur       │
 └───────────────┬───────────────────────────────────────────────┘
                 │ TidStore boşaldı → FAZ I devam
                 ▼
        FSM güncelle → index cleanup (amvacuumcleanup)
                     → kırpma (AccessExclusiveLock, 5 sn dene)
                     → pg_class: relfrozenxid, reltuples, relallvisible
                     → vac_update_datfrozenxid → vac_truncate_clog

 FAILSAFE tetiklenirse: index temizliği YOK, heap temizliği YOK,
                        kırpma YOK, maliyet gecikmesi YOK — sadece freeze.
```

**Akılda kalması gereken beş şey:**

1. Fazların sırası zorunludur: index girişleri silinmeden line pointer geri
   verilemez.
2. `maintenance_work_mem` dolarsa VACUUM index'leri birden fazla kez tarar —
   `VERBOSE`'daki "index scans" sayısı bunun ölçüsüdür.
3. VACUUM'un neyi silebileceğini `OldestXmin` belirler; uzun transaction, açık
   slot ve standby feedback bunu geride tutar.
4. Aggressive VACUUM `relfrozenxid`'i ilerletmek zorundadır ve bunun için
   cleanup lock'ı bekler; normal VACUUM beklemez.
5. Failsafe bir optimizasyon değil, alarmdır: tetiklendiğinde temizlik tamamen
   bırakılır.
