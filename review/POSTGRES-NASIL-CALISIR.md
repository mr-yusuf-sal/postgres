# PostgreSQL nasıl çalışır? — kodun içinden tam tur

> Bu depo: PostgreSQL **20devel** ([meson.build](../meson.build) `version: '20devel'`).
> Sorgu boru hattının (parse → plan → execute) ayrıntısı ayrı dosyada: [SELECT-NASIL-CALISIR.md](SELECT-NASIL-CALISIR.md).
> Bu dosya **motorun geri kalanını** anlatır: süreçler, bellek, disk, MVCC, WAL, kilitler, bakım.

---

## 30 saniyelik özet

```
                        ┌─────────────────────────────────────────┐
   istemci ──TCP──►     │  postmaster  (tek süreç, veri okumaz)   │
                        │  ServerLoop → accept() → fork()         │
                        └───────────────┬─────────────────────────┘
                                        │ her bağlantıya 1 süreç
        ┌───────────────┬───────────────┼───────────────┬──────────────┐
        ▼               ▼               ▼               ▼              ▼
    backend #1      backend #2     checkpointer     bgwriter      autovacuum
        │               │               │               │              │
        └───────────────┴───── PAYLAŞIMLI BELLEK ────────┴──────────────┘
                    shared_buffers │ WAL buffers │ ProcArray │ lock table
                                   │
                                   ▼
                          disk: base/ , pg_wal/ , pg_xact/
```

**Beş cümlelik model:**

1. **Süreç başına bağlantı.** Thread yok. `fork()` (Windows'ta `CreateProcess`).
2. **Tüm süreçler tek bir paylaşımlı bellek bloğunu** `mmap` ile paylaşır — disk sayfaları orada tutulur.
3. **Hiçbir şey diske yazılmadan önce WAL'a yazılır.** Kalıcılık buradan gelir, veri dosyalarından değil.
4. **Satırlar yerinde güncellenmez.** Her `UPDATE` yeni bir satır sürümü yazar → MVCC.
5. **Ölü satırları `VACUUM` toplar.** Bu, 3 ve 4'ün kaçınılmaz bedelidir.

---

## Kaynak ağacı haritası — nereye bakacağınız

| Dizin | Ne yapar |
|---|---|
| [src/backend/postmaster/](../src/backend/postmaster/) | Süreç yönetimi: postmaster, checkpointer, bgwriter, autovacuum |
| [src/backend/tcop/](../src/backend/tcop/) | "Traffic cop" — protokol döngüsü, sorgu orkestrasyonu |
| [src/backend/parser/](../src/backend/parser/) | flex/bison + anlamsal analiz |
| [src/backend/optimizer/](../src/backend/optimizer/) | Maliyet tabanlı planlayıcı |
| [src/backend/executor/](../src/backend/executor/) | Plan yürütme (volcano/pull modeli) |
| [src/backend/access/heap/](../src/backend/access/heap/) | Tablo depolama, MVCC görünürlüğü, VACUUM |
| [src/backend/access/transam/](../src/backend/access/transam/) | Transaction + WAL |
| [src/backend/storage/buffer/](../src/backend/storage/buffer/) | Shared buffer pool |
| [src/backend/storage/lmgr/](../src/backend/storage/lmgr/) | Kilitler (heavyweight, LWLock, spinlock) |
| [src/backend/storage/smgr/](../src/backend/storage/smgr/) | Disk I/O soyutlaması |
| [src/backend/utils/mmgr/](../src/backend/utils/mmgr/) | Memory context'ler (`palloc`) |
| [src/backend/catalog/](../src/backend/catalog/) | Sistem katalogları (`pg_class`, `pg_attribute`, …) |

**Kritik ipucu:** Neredeyse her alt sistemin bir `README` dosyası var ve bunlar dokümantasyondan daha derindir:

```
src/backend/access/transam/README     ← WAL + MVCC tasarımı (en önemlisi)
src/backend/storage/buffer/README     ← buffer değiştirme algoritması
src/backend/storage/lmgr/README       ← kilit hiyerarşisi, deadlock tespiti
src/backend/optimizer/README          ← planlayıcı iç yapısı
src/backend/executor/README           ← yürütme modeli
src/backend/access/nbtree/README      ← B-tree (Lehman-Yao)
```

---

# 1. Süreç mimarisi

## 1.1 Giriş noktası

[src/backend/main/main.c:71](../src/backend/main/main.c#L71)

Tek bir `postgres` ikili dosyası vardır; hangi rolde çalışacağına argümanlarla karar verir:

```c
main(int argc, char *argv[])
{
    ...
    if (do_check_root)
        check_root(progname);      /* root olarak çalışmayı reddeder */
```

Sonra rol seçimi yapılır → `PostmasterMain` (normal sunucu), `PostgresSingleUserMain` (`--single`), `BootstrapModeMain` (`initdb` içinden), veya `SubPostmasterMain` (Windows'ta yeniden exec edilen çocuk).

## 1.2 Postmaster — kabul eden ama veriye dokunmayan süreç

[src/backend/postmaster/postmaster.c:497](../src/backend/postmaster/postmaster.c#L497) — `PostmasterMain`

Yaptıkları sırayla:

1. `postgresql.conf` okunur, GUC'lar kurulur
2. Dinleme soketleri açılır
3. **Paylaşımlı bellek + semaforlar oluşturulur** (`CreateSharedMemoryAndSemaphores`)
4. Yardımcı süreçler başlatılır (startup, checkpointer, bgwriter, walwriter, autovacuum launcher)
5. `ServerLoop()`'a girilir ve bir daha çıkılmaz

**Postmaster asla shared buffer'lara dokunmaz, asla transaction çalıştırmaz.** Nedeni: bir backend segfault ederse paylaşımlı bellek tutarsız olabilir; postmaster ona bulaşmamış olmalı ki tüm çocukları öldürüp temiz bir kurtarma başlatabilsin. Bu, mimarinin en bilinçli tasarım kararlarından biri.

## 1.3 Ana döngü

[src/backend/postmaster/postmaster.c:1678](../src/backend/postmaster/postmaster.c#L1678)

```c
ServerLoop(void)
{
    ConfigurePostmasterWaitSet(true);
    last_lockfile_recheck_time = last_touch_time = time(NULL);

    for (;;)
    {
        nevents = WaitEventSetWait(pm_wait_set,
                                   DetermineSleepTime(),
                                   events,
                                   lengthof(events),
                                   0);
        /* Latch (sinyal) mi geldi, yeni bağlantı mı?
           Yeni bağlantıysa: ilgilenmesi için bir çocuk süreç fork et. */
```

`WaitEventSetWait` = `epoll`/`kqueue`/`WaitForMultipleObjects` üzerine platform soyutlaması ([src/backend/storage/ipc/waiteventset.c](../src/backend/storage/ipc/waiteventset.c)). Postmaster busy-wait yapmaz; uyanma sebepleri: yeni bağlantı, çocuk ölümü sinyali, ya da zamanlayıcı.

## 1.4 Çocuk süreç doğuşu

[src/backend/postmaster/postmaster.c:3576](../src/backend/postmaster/postmaster.c#L3576) — `BackendStartup`

```c
cac = canAcceptConnections(B_BACKEND);
if (cac == CAC_OK)
{
    bn = AssignPostmasterChildSlot(B_BACKEND);
    if (!bn)
        cac = CAC_TOOMANY;          /* max_connections doldu */
}
if (!bn)
    bn = AllocDeadEndChild();       /* sadece hata mesajı yazıp ölecek çocuk */

startup_data.canAcceptConnections = cac;

pid = postmaster_child_launch(bn->bkend_type, bn->child_slot,
                              &startup_data, sizeof(startup_data),
                              client_sock);
```

**"Dead-end child" fikri:** bağlantı reddedilecekse bile bir süreç fork edilir. Neden? Böylece "sorry, too many clients" mesajını yazma işi postmaster'ı bloklamaz. Postmaster'ın hiçbir şey için beklememesi gerekir.

## 1.5 fork vs. exec — Windows farkı

[src/backend/postmaster/launch_backend.c:205](../src/backend/postmaster/launch_backend.c#L205)

```c
#ifdef EXEC_BACKEND
    pid = internal_forkexec(child_type, child_slot,
                            startup_data, startup_data_len, client_sock);
    /* çocuk süreç SubPostmasterMain'e varacak */
#else
    pid = fork_process();
    if (pid == 0)               /* çocuk */
    {
        MyBackendType = child_type;
        ...
```

| | Unix (`fork`) | Windows (`EXEC_BACKEND`) |
|---|---|---|
| Nasıl | Adres alanı kopyalanır | `CreateProcess` + `postgres --forkchild` |
| Global değişkenler | Otomatik miras alınır | **Elle serileştirilip aktarılır** |
| Varış noktası | `fork_process` sonrası | `SubPostmasterMain` |
| Maliyet | Ucuz (COW) | Pahalı |

Windows'ta çalışıyorsanız her bağlantı tam bir süreç başlatma maliyeti öder. Connection pooler (PgBouncer) kullanmanın önemi burada büyür.

## 1.6 Süreç tipleri

[src/include/miscadmin.h:340](../src/include/miscadmin.h#L340)

```c
typedef enum BackendType
{
    B_INVALID = 0,

    /* Backend ve backend benzeri süreçler */
    B_BACKEND,              /* normal istemci bağlantısı */
    B_DEAD_END_BACKEND,     /* sadece hata yazıp ölecek */
    B_AUTOVAC_LAUNCHER,
    B_AUTOVAC_WORKER,
    B_BG_WORKER,            /* extension'ların başlattığı işçiler */
    B_WAL_SENDER,           /* replikasyon */
    B_SLOTSYNC_WORKER,

    B_STANDALONE_BACKEND,   /* --single modu */

    /* Yardımcı süreçler: PGPROC'ları var ama veritabanına bağlı değiller,
       transaction çalıştıramaz, heavyweight kilit alamazlar.
       Her birinden aynı anda sadece bir tane (IO worker hariç). */
    B_ARCHIVER,
    B_BG_WRITER,
    B_CHECKPOINTER,
    B_IO_WORKER,            /* PG 18+ asenkron I/O */
    B_STARTUP,              /* WAL replay */
    B_WAL_RECEIVER,
    B_WAL_SUMMARIZER,
    ...
```

Bunu canlıda görmek için:

```sql
SELECT pid, backend_type, state, query FROM pg_stat_activity;
```

---

# 2. Bir bağlantının doğuşu — accept'ten ilk sorguya

```
postmaster: accept()                    postmaster.c:1678 ServerLoop
   └─ BackendStartup                    postmaster.c:3576
      └─ postmaster_child_launch         launch_backend.c:205    ◄── fork burada
         │
         ▼ (artık çocuk süreçteyiz)
      BackendMain                        backend_startup.c:76
       ├─ BackendInitialize              backend_startup.c:141
       │   ├─ soketi devral, sinyalleri kur
       │   ├─ startup paketini oku (db adı, kullanıcı, seçenekler)
       │   └─ ClientAuthentication        libpq/auth.c:376        ◄── pg_hba.conf
       │
       ├─ InitPostgres                    utils/init/postinit.c:722
       │   ├─ paylaşımlı belleğe bağlan, PGPROC slotu al
       │   ├─ veritabanına bağlan (pg_database'de ara)
       │   ├─ relcache / syscache'i başlat
       │   └─ MyDatabaseId, MyProcPid gibi globalleri kur
       │
       └─ PostgresMain                    tcop/postgres.c:4364
           └─ for(;;) { mesaj oku; işle; ReadyForQuery gönder; }
```

## 2.1 Kimlik doğrulama

[src/backend/libpq/auth.c:376](../src/backend/libpq/auth.c#L376) — `ClientAuthentication`

`pg_hba.conf` satır satır taranır (ilk eşleşen kazanır: tip, veritabanı, kullanıcı, adres). Sonuç bir `UserAuth` yöntemidir: `trust`, `scram-sha-256`, `md5`, `peer`, `ldap`, `cert`, …

**Önemli davranış:** eşleşen satır bulunamazsa bağlantı reddedilir — "sonraki satıra bak" yoktur. `pg_hba.conf` düzenlerken en sık yapılan hata budur.

## 2.2 `InitPostgres` — pahalı kısım

[src/backend/utils/init/postinit.c:722](../src/backend/utils/init/postinit.c#L722)

Burada her yeni bağlantı için yapılan işler, connection pooling'in neden bu kadar fark yarattığını açıklar:

- Paylaşımlı belleğe bağlanma, `PGPROC` slotu alma
- `pg_database` katalog taraması
- **Relcache ve syscache'in sıfırdan doldurulması** — bu süreç-yerel önbellekler paylaşılmaz
- Bağlantı başına birkaç MB özel bellek

Yani her `psql` bağlantısı kendi katalog önbelleğini yeniden ısıtır.

---

# 3. Paylaşımlı bellek — süreçlerin buluştuğu yer

[src/backend/storage/ipc/ipci.c:57](../src/backend/storage/ipc/ipci.c#L57) — `CalculateShmemSize`
[src/backend/storage/ipc/ipci.c:120](../src/backend/storage/ipc/ipci.c#L120) — `CreateSharedMemoryAndSemaphores`

Postmaster **tek seferde** tüm paylaşımlı belleği ayırır. Çalışma anında büyütülemez — `shared_buffers` değiştirmek için restart gerekmesinin sebebi budur.

İçindekiler:

| Bölge | Ne tutar | GUC |
|---|---|---|
| **Buffer pool** | Disk sayfalarının kopyaları | `shared_buffers` |
| **WAL buffers** | Henüz diske yazılmamış WAL | `wal_buffers` |
| **ProcArray** | Her canlı backend'in `PGPROC`'u — snapshot üretimi buradan | `max_connections` |
| **Lock table** | Heavyweight kilit hash tablosu | `max_locks_per_transaction` |
| **CLOG / SLRU** | Transaction commit durumları | — |

`fork()` kullanıldığı için çocuk süreçler bu bloğu adres alanlarında **aynı adreste** miras alır. Bu yüzden shared memory içindeki işaretçiler süreçler arasında geçerlidir — PostgreSQL'in bir sürü yerde ham pointer kullanabilmesinin sebebi budur.

---

# 4. Diskte veri nasıl duruyor

## 4.1 Üç katman

```
Tablo (Relation)
  └─ Fork'lar:  main (veri) | fsm (boş alan) | vm (görünürlük haritası) | init
       └─ Segment dosyaları: 1 GB'lık parçalar  →  16384, 16384.1, 16384.2 …
            └─ Sayfa (Page/Block): 8 KB sabit
                 └─ Tuple (satır sürümü)
```

Dosya yolu: `base/<db_oid>/<relfilenode>`. Örnek:

```sql
SELECT pg_relation_filepath('users');   -- base/16384/16401
```

## 4.2 Sayfa düzeni

[src/include/storage/bufpage.h:184](../src/include/storage/bufpage.h#L184)

```c
typedef struct PageHeaderData
{
    PageXLogRecPtr pd_lsn;      /* bu sayfayı en son değiştiren WAL kaydının sonu */
    uint16      pd_checksum;
    uint16      pd_flags;
    LocationIndex pd_lower;     /* boş alanın başlangıcı */
    LocationIndex pd_upper;     /* boş alanın sonu */
    LocationIndex pd_special;   /* özel alanın başlangıcı (index AM'ler kullanır) */
    uint16      pd_pagesize_version;
    TransactionId pd_prune_xid; /* budanabilir en eski XID, yoksa 0 */
    ItemIdData  pd_linp[];      /* satır işaretçisi dizisi */
} PageHeaderData;
```

Görsel olarak — **başlık ve satırlar birbirine doğru büyür:**

```
0                                                                   8192
├──────────┬───────────────┬─────────────────┬──────────────────────────┤
│ 24 byte  │ pd_linp[]     │   BOŞ ALAN      │  tuple'lar (sondan başa) │
│ header   │ →→→→→→        │                 │        ←←←←←←            │
└──────────┴───────────────┴─────────────────┴──────────────────────────┘
           ▲               ▲                 ▲
           |            pd_lower          pd_upper
     satır işaretçileri
```

`pd_lsn` alanı **WAL kuralının uygulanma noktasıdır**: bir sayfa diske yazılmadan önce, `pd_lsn`'e kadar olan WAL'ın diske yazılmış olması zorunludur (aşağıda §7.3).

## 4.3 Satır başlığı — MVCC'nin fiziksel karşılığı

[src/include/access/htup_details.h:122](../src/include/access/htup_details.h#L122)

```c
typedef struct HeapTupleFields
{
    TransactionId t_xmin;       /* ekleyen transaction ID */
    TransactionId t_xmax;       /* silen veya kilitleyen transaction ID */

    union
    {
        CommandId   t_cid;      /* ekleyen/silen komut ID'si */
        TransactionId t_xvac;   /* eski usul VACUUM FULL */
    }           t_field3;
} HeapTupleFields;
```

[src/include/access/htup_details.h:153](../src/include/access/htup_details.h#L153)

```c
struct HeapTupleHeaderData
{
    union {
        HeapTupleFields  t_heap;
        DatumTupleFields t_datum;
    }           t_choice;

    ItemPointerData t_ctid;     /* bu satırın veya DAHA YENİ sürümünün TID'i */

    uint16      t_infomask2;    /* sütun sayısı + bayraklar */
    uint16      t_infomask;     /* bayraklar (aşağıda) */
    uint8       t_hoff;         /* başlık + null bitmap boyutu */

    /* ^ - 23 byte - ^ */

    uint8       t_bits[];       /* NULL bitmap */
    /* VERİ BURADAN SONRA GELİR */
};
```

**Her satır 23 byte'lık bir MVCC vergisi öder** (hizalamayla genellikle 24). `t_ctid` alanı normalde satırın kendi adresini gösterir; satır güncellenmişse **yeni sürümü** gösterir — güncelleme zinciri böyle kurulur.

Bunu SQL'den görebilirsiniz:

```sql
SELECT ctid, xmin, xmax, * FROM users LIMIT 5;
```

---

# 5. Buffer manager — disk ile RAM arasındaki kapı

Diskteki her sayfaya erişim **tek bir kapıdan** geçer: buffer manager. Executor asla doğrudan dosya okumaz.

## 5.1 Buffer descriptor

[src/include/storage/buf_internals.h:326](../src/include/storage/buf_internals.h#L326)

```c
typedef struct BufferDesc
{
    BufferTag   tag;            /* hangi sayfa: (relfilenode, fork, blocknum) */
    int         buf_id;         /* buffer dizisindeki indeks, hiç değişmez */
    pg_atomic_uint64 state;     /* bayraklar + refcount + usagecount, ATOMİK */
    ...
```

`state` alanında üç şey tek bir 64-bit atomik değerde paketlenmiştir: bayraklar (`BM_DIRTY`, `BM_VALID`, `BM_IO_IN_PROGRESS`…), pin sayacı ve kullanım sayacı. Tek bir CAS ile güncellenebilmesi için — kilit almadan.

## 5.2 Okuma yolu

[src/backend/storage/buffer/bufmgr.c:879](../src/backend/storage/buffer/bufmgr.c#L879) — `ReadBuffer`
→ [bufmgr.c:926](../src/backend/storage/buffer/bufmgr.c#L926) `ReadBufferExtended`
→ [bufmgr.c:1276](../src/backend/storage/buffer/bufmgr.c#L1276) `ReadBuffer_common`
→ [bufmgr.c:2197](../src/backend/storage/buffer/bufmgr.c#L2197) `BufferAlloc`

```
ReadBuffer(rel, blockNum)
   │
   ├─ Buffer hash tablosunda ara  (BufTableLookup)
   │    ├─ BULUNDU  → PinBuffer() → dön          ◄── "buffer hit"
   │    └─ YOK      ↓
   │
   ├─ GetVictimBuffer()                bufmgr.c:2548
   │    ├─ StrategyGetBuffer()          freelist.c:184   ◄── clock-sweep
   │    ├─ kurban dirty ise → FlushBuffer()  (WAL'ı önce flush et!)
   │    └─ eski tag'i hash'ten çıkar, yenisini ekle
   │
   └─ smgrreadv()                      smgr.c:721   ◄── gerçek disk okuması
        └─ mdreadv()                   md.c:858     ◄── pread()
```

## 5.3 Clock-sweep — hangi sayfa atılır?

[src/backend/storage/buffer/freelist.c:184](../src/backend/storage/buffer/freelist.c#L184) — `StrategyGetBuffer`

LRU değil. Bir saat ibresi buffer dizisi üzerinde döner:

```
her buffer'ı ziyaret et:
    pinlenmiş mi?        → atla
    usage_count > 0 ?    → usage_count-- , devam et
    usage_count == 0 ?   → KURBAN BU
```

Neden LRU değil? LRU her erişimde paylaşımlı bir listeyi güncellemeyi gerektirir — yüksek eşzamanlılıkta ölümcül kilit çekişmesi. Clock-sweep yaklaşık aynı sonucu, süreç-yerel atomik işlemlerle verir.

**`BufferAccessStrategy` (ring buffer):** `VACUUM`, `COPY`, büyük seq scan gibi işlemler küçük bir buffer halkasıyla sınırlandırılır. Yoksa 100 GB'lık bir tablo taraması tüm sıcak önbelleği süpürürdü.

## 5.4 Kirletme

[src/backend/storage/buffer/bufmgr.c:3170](../src/backend/storage/buffer/bufmgr.c#L3170) — `MarkBufferDirty`

Bir sayfa değiştirildiğinde sadece `BM_DIRTY` bayrağı set edilir. Diske yazma işini **checkpointer** veya **bgwriter** yapar — yazan backend değil. Bu, yazma gecikmesini transaction'dan çıkarır.

---

# 6. MVCC — aynı satırın birden fazla sürümü

## 6.1 Temel kural

```
UPDATE users SET name = 'Ali' WHERE id = 1;
```

Yaptığı **yerinde değişiklik değildir**:

```
ÖNCE:
  [ xmin=100, xmax=0,   name='Veli' ]   ← ctid (0,1)

SONRA (transaction 205 içinde):
  [ xmin=100, xmax=205, name='Veli' ]   ← ctid (0,1), t_ctid → (0,2)
  [ xmin=205, xmax=0,   name='Ali'  ]   ← ctid (0,2)   YENİ SATIR
```

Eski satır **silinmez**. Hâlâ eski bir snapshot'la çalışan transaction'lar onu görmeye devam eder. Ölü sürümü daha sonra `VACUUM` toplar.

**Bunun üç doğrudan sonucu:**

1. `UPDATE` ≈ `DELETE` + `INSERT` maliyetindedir
2. Okuyucular yazıcıları, yazıcılar okuyucuları **bloklamaz**
3. Tablolar şişer (bloat) — `VACUUM` zorunludur, opsiyonel değil

## 6.2 Snapshot

[src/include/utils/snapshot.h:138](../src/include/utils/snapshot.h#L138)

```c
typedef struct SnapshotData
{
    SnapshotType snapshot_type;

    /* Bir MVCC snapshot'ı XID >= xmax olan hiçbir etkiyi göremez.
       xmin'den küçük tüm XID'lerin etkisini görür, listede olanlar hariç. */
    TransactionId xmin;     /* xmin'den küçük tüm XID'ler bana görünür */
    TransactionId xmax;     /* xmax'tan büyük/eşit tüm XID'ler görünmez */

    TransactionId *xip;     /* aradaki DEVAM EDEN transaction'lar */
    uint32      xcnt;
    ...
```

Üç alanla tam bir zaman dilimi tanımlanır:

```
       görünür              belirsiz bölge           görünmez
  ──────────────────┼───────────────────────────┼──────────────────►  XID
                  xmin                        xmax

  belirsiz bölgede: xip[] listesindeyse → devam ediyor → GÖRÜNMEZ
                    listede yoksa       → bitmiş       → CLOG'a sor
```

[src/backend/storage/ipc/procarray.c:2114](../src/backend/storage/ipc/procarray.c#L2114) — `GetSnapshotData`

Bu fonksiyon `ProcArray`'i tarar (her canlı backend'in `PGPROC`'u) ve devam eden XID'leri toplar. `max_connections` çok yüksekse bu tarama pahalılaşır — snapshot alma sıklığı yüksek yüklerde darboğaz olabilir.

## 6.3 Görünürlük kararı

[src/backend/access/heap/heapam_visibility.c:939](../src/backend/access/heap/heapam_visibility.c#L939) — `HeapTupleSatisfiesMVCC`

Bu fonksiyon PostgreSQL'in en sık çağrılan fonksiyonlarından biridir. **Taranan her satır için** çalışır. Karar ağacı:

```c
if (!HeapTupleHeaderXminCommitted(tuple))          /* hint bit yok */
{
    if (HeapTupleHeaderXminInvalid(tuple))
        return false;                              /* ekleyen abort etti */

    else if (TransactionIdIsCurrentTransactionId(HeapTupleHeaderGetRawXmin(tuple)))
    {
        /* BEN ekledim */
        if (HeapTupleHeaderGetCmin(tuple) >= snapshot->curcid)
            return false;                          /* tarama başladıktan sonra eklenmiş */

        if (tuple->t_infomask & HEAP_XMAX_INVALID)
            return true;                           /* silinmemiş → görünür */
        ...
    }
    else if (XidInMVCCSnapshot(HeapTupleHeaderGetRawXmin(tuple), snapshot))
        return false;                              /* ekleyen hâlâ devam ediyor */

    else if (TransactionIdDidCommit(HeapTupleHeaderGetRawXmin(tuple)))
        SetHintBitsExt(tuple, buffer, HEAP_XMIN_COMMITTED, ...);   /* ◄── hint bit */
    else
        /* ekleyen abort etmiş */
```

Sonra aynı mantık `xmax` için tekrarlanır. Özetle:

| Durum | Sonuç |
|---|---|
| `xmin` commit etmemiş / abort | görünmez |
| `xmin` snapshot'ta devam ediyor | görünmez |
| `xmin` commit + `xmax` yok/geçersiz | **görünür** |
| `xmin` commit + `xmax` commit (snapshot'tan önce) | görünmez (silinmiş) |
| `xmin` commit + `xmax` devam ediyor | **görünür** (silme henüz kesinleşmedi) |

## 6.4 Hint bit — sessiz optimizasyon

`SetHintBitsExt` çağrısına dikkat edin. Bir transaction'ın durumu ilk kez CLOG'dan öğrenildiğinde, sonuç **satır başlığına yazılır** (`HEAP_XMIN_COMMITTED`).

Sonuç: aynı satır ikinci kez okunduğunda CLOG'a gitmeye gerek kalmaz.

**Yan etkisi şaşırtıcıdır:** salt-okunur bir `SELECT` sayfayı kirletebilir ve disk yazması tetikleyebilir. `pg_stat_statements`'ta "neden bu SELECT yazma yapıyor?" sorusunun cevabı genellikle budur.

## 6.5 CLOG — transaction durum defteri

[src/backend/access/transam/clog.c:743](../src/backend/access/transam/clog.c#L743) — `TransactionIdGetStatus`
[src/backend/access/transam/clog.c:192](../src/backend/access/transam/clog.c#L192) — `TransactionIdSetTreeStatus`

`pg_xact/` dizininde, transaction başına **2 bit**:

```
00 = IN_PROGRESS
01 = COMMITTED
10 = ABORTED
11 = SUB_COMMITTED
```

Bir sayfaya 32768 transaction sığar. SLRU (Simple LRU) önbelleğiyle bellekte tutulur.

---

# 7. Transaction ve WAL

## 7.1 Transaction yaşam döngüsü

[src/backend/access/transam/xact.c:2106](../src/backend/access/transam/xact.c#L2106) — `StartTransaction`
[src/backend/access/transam/xact.c:2270](../src/backend/access/transam/xact.c#L2270) — `CommitTransaction`
[src/backend/access/transam/xact.c:2855](../src/backend/access/transam/xact.c#L2855) — `AbortTransaction`

**XID tembel atanır.** [src/backend/access/transam/xact.c:456](../src/backend/access/transam/xact.c#L456) — `GetCurrentTransactionId` ilk kez veri değiştirildiğinde çağrılır. Salt-okunur transaction'lar XID tüketmez; bu, XID wraparound baskısını ciddi biçimde azaltır.

## 7.2 Commit — asıl kalıcılık anı

[src/backend/access/transam/xact.c:1345](../src/backend/access/transam/xact.c#L1345) — `RecordTransactionCommit`

Kritik sıra ([xact.c:1484](../src/backend/access/transam/xact.c#L1484) civarı):

```c
/* 1. Commit kaydını WAL'a yaz (henüz bellekte) */
XactLogCommitRecord(GetCurrentTransactionStopTimestamp(),
                    nchildren, children, nrels, rels, ...);

/* 2. Dayanıklılık kararı */
if ((wrote_xlog && markXidCommitted &&
     synchronous_commit > SYNCHRONOUS_COMMIT_OFF) || forceSyncCommit || ...)
{
    XLogFlush(XactLastRecEnd);        /* ◄── fsync BURADA. İstemci bunu bekler. */

    /* 3. CLOG'a "commit edildi" yaz */
    TransactionIdCommitTree(xid, nchildren, children);
}
```

**Sıra bozulamaz.** Önce WAL diske, sonra CLOG. Ters olsaydı: CLOG "commit" der, çökme olur, WAL'da kayıt yoktur → replay sonrası tutarsız durum.

`synchronous_commit = off` tam olarak `XLogFlush` çağrısını atlar. Kazanç: fsync gecikmesi transaction'dan çıkar. Risk: çökmede son ~`wal_writer_delay` kadar commit kaybolur — ama **veri bozulmaz**, çünkü WAL sırası korunur.

## 7.3 WAL yazma

[src/backend/access/transam/xloginsert.c:153](../src/backend/access/transam/xloginsert.c#L153) — `XLogBeginInsert`
[src/backend/access/transam/xloginsert.c:372](../src/backend/access/transam/xloginsert.c#L372) — `XLogRegisterData`
[src/backend/access/transam/xloginsert.c:482](../src/backend/access/transam/xloginsert.c#L482) — `XLogInsert`

Tipik kullanım kalıbı (`heap_insert` içinden):

```c
XLogBeginInsert();
XLogRegisterData((char *) &xlrec, SizeOfHeapInsert);
XLogRegisterBuffer(0, buffer, REGBUF_STANDARD);
recptr = XLogInsert(RM_HEAP_ID, XLOG_HEAP_INSERT);
PageSetLSN(page, recptr);           /* ◄── sayfaya LSN damgası */
```

[src/backend/access/transam/xlog.c:784](../src/backend/access/transam/xlog.c#L784) — `XLogInsertRecord` gerçek yerleştirmeyi yapar:

```c
START_CRIT_SECTION();               /* buradan sonra hata = PANIC */

if (likely(class == WALINSERT_NORMAL))
{
    WALInsertLockAcquire();

    /* Full-page write gerekiyor mu? Checkpoint'ten sonra ilk dokunuş mu? */
    doPageWrites = (Insert->fullPageWrites || Insert->runningBackups > 0);

    if (doPageWrites &&
        (!prevDoPageWrites ||
         (XLogRecPtrIsValid(fpw_lsn) && fpw_lsn <= RedoRecPtr)))
    {
        /* Çağıranın yedeklemediği bir sayfa yedeklenmeli. Baştan başla. */
        WALInsertLockRelease();
        END_CRIT_SECTION();
        return InvalidXLogRecPtr;
    }

    /* WAL'da yer ayır — xl_prev işaretçisini de kurar */
    ReserveXLogInsertLocation(rechdr->xl_tot_len, &StartPos, &EndPos,
                              &rechdr->xl_prev);
```

**`START_CRIT_SECTION()` ne demek?** Bu blok içinde `ERROR` fırlatmak yasaktır — çünkü WAL'a yarım kayıt yazılmış olabilir. Herhangi bir hata otomatik olarak `PANIC`'e yükseltilir ve sunucu yeniden başlar. Tutarsız bir WAL'la devam etmektense çökmek yeğdir.

**Full-page write:** checkpoint'ten sonra bir sayfaya ilk dokunuşta, sayfanın **tamamı** WAL'a yazılır. Neden? İşletim sistemi 8 KB'lık bir yazmayı yarıda kesebilir ("torn page"). Yarım yazılmış sayfaya delta uygulamak felakettir. Bu, checkpoint sonrası WAL hacminin neden sıçradığını açıklar.

## 7.4 Flush

[src/backend/access/transam/xlog.c:2800](../src/backend/access/transam/xlog.c#L2800) — `XLogFlush`
[src/backend/access/transam/xlog.c:2325](../src/backend/access/transam/xlog.c#L2325) — `XLogWrite`

**WAL kuralı (write-ahead logging):**

> Bir veri sayfası diske yazılmadan **önce**, o sayfayı değiştiren tüm WAL kayıtları diske yazılmış olmalıdır.

Uygulaması `FlushBuffer` içinde: sayfanın `pd_lsn`'ine bakılır, `XLogFlush(pd_lsn)` çağrılır, ancak ondan sonra `smgrwritev` yapılır. [src/backend/storage/buffer/bufmgr.c:4526](../src/backend/storage/buffer/bufmgr.c#L4526)

## 7.5 Resource manager'lar

[src/include/access/rmgrlist.h:28](../src/include/access/rmgrlist.h#L28)

WAL kayıtları tiplenmiştir; her tipin bir redo fonksiyonu vardır:

```c
PG_RMGR(RM_XLOG_ID,  "XLOG",        xlog_redo,  xlog_desc,  ...)
PG_RMGR(RM_XACT_ID,  "Transaction", xact_redo,  xact_desc,  ...)
PG_RMGR(RM_HEAP_ID,  "Heap",        heap_redo,  heap_desc,  ...)
PG_RMGR(RM_BTREE_ID, "Btree",       btree_redo, btree_desc, ...)
```

`pg_waldump` çıktısındaki isimler bunlardır.

## 7.6 Çökme sonrası kurtarma

[src/backend/access/transam/xlog.c:5845](../src/backend/access/transam/xlog.c#L5845) — `StartupXLOG`
[src/backend/access/transam/xlogrecovery.c:1617](../src/backend/access/transam/xlogrecovery.c#L1617) — `PerformWalRecovery`
[src/backend/access/transam/xlogrecovery.c:1888](../src/backend/access/transam/xlogrecovery.c#L1888) — `ApplyWalRecord`

```
1. pg_control oku → son checkpoint'in konumu
2. O checkpoint'ten itibaren WAL'ı sırayla oku
3. Her kayıt için:
     sayfayı buffer'a getir
     sayfanın pd_lsn >= kaydın LSN'i mi?  → ATLA (zaten uygulanmış)
     değilse → rmgr->rm_redo(record) çağır
4. WAL bitince: commit etmemiş transaction'lar zaten görünmez (CLOG'da yok)
5. Yeni checkpoint yaz, bağlantıları kabul et
```

**Adım 3'teki LSN karşılaştırması idempotency'yi sağlar.** Kurtarma yarıda kesilse bile baştan çalıştırılabilir. Abort eden transaction'lar için geri alma (undo) **yoktur** — MVCC sayesinde gerek de yoktur; o satırlar hiçbir snapshot'ta görünmez, `VACUUM` onları toplar.

---

# 8. Kilitler — üç ayrı seviye

| Seviye | Kod | Kullanım | Deadlock tespiti |
|---|---|---|---|
| **Spinlock** | [s_lock.c](../src/backend/storage/lmgr/s_lock.c) | Onlarca komut, busy-wait | Yok |
| **LWLock** | [lwlock.c:1150](../src/backend/storage/lmgr/lwlock.c#L1150) | Shared bellek yapıları, buffer içeriği | Yok |
| **Heavyweight** | [lock.c:806](../src/backend/storage/lmgr/lock.c#L806) | Tablo, satır, ilişki seviyesi | **Var** |

## 8.1 LWLock

[src/backend/storage/lmgr/lwlock.c:1150](../src/backend/storage/lmgr/lwlock.c#L1150) — `LWLockAcquire`
[src/backend/storage/lmgr/lwlock.c:1767](../src/backend/storage/lmgr/lwlock.c#L1767) — `LWLockRelease`

Shared/Exclusive modları, kuyruk var, ama **deadlock tespiti yok**. Bu yüzden LWLock'lar için katı bir edinme sırası kuralı vardır ([src/backend/storage/lmgr/README](../src/backend/storage/lmgr/README)). Sırayı ihlal eden kod kilitlenir ve fark etmezsiniz.

`pg_stat_activity.wait_event_type = 'LWLock'` gördüğünüzde buradasınız (`WALWriteLock`, `BufferContent`, `ProcArrayLock`…).

## 8.2 Heavyweight kilitler

[src/backend/storage/lmgr/lock.c:833](../src/backend/storage/lmgr/lock.c#L833) — `LockAcquireExtended`

Sekiz tablo seviyesi mod, `AccessShareLock`'tan `AccessExclusiveLock`'a. Çakışma matrisi `LockConflicts[]` dizisindedir.

**Pratik kural:** `SELECT` → `AccessShareLock`. `ALTER TABLE` → `AccessExclusiveLock`. Bunlar çakışır. Uzun süren bir `SELECT`, `ALTER TABLE`'ı bekletir; bekleyen `ALTER TABLE` de arkasındaki **tüm** yeni `SELECT`'leri bekletir (kilit kuyruğu FIFO'dur). Production'da migration'ların sistemi kilitlemesinin en yaygın sebebi budur.

Çözüm: `SET lock_timeout = '2s';` migration öncesi.

---

# 9. Yardımcı süreçler — arka planda ne oluyor

| Süreç | Giriş noktası | Görevi |
|---|---|---|
| **checkpointer** | [checkpointer.c:206](../src/backend/postmaster/checkpointer.c#L206) | Periyodik olarak tüm dirty buffer'ları diske yazar |
| **bgwriter** | [bgwriter.c:89](../src/backend/postmaster/bgwriter.c#L89) | Sürekli, azar azar dirty buffer yazar |
| **walwriter** | [walwriter.c:89](../src/backend/postmaster/walwriter.c#L89) | WAL buffer'larını diske akıtır |
| **autovacuum launcher** | [autovacuum.c:411](../src/backend/postmaster/autovacuum.c#L411) | Ne zaman vacuum gerektiğine karar verir, worker başlatır |
| **startup** | [startup.c:216](../src/backend/postmaster/startup.c#L216) | WAL replay (kurtarma / standby) |

## 9.1 Checkpoint

[src/backend/access/transam/xlog.c:7400](../src/backend/access/transam/xlog.c#L7400) — `CreateCheckPoint`

```
1. Bu andaki WAL konumunu kaydet (redo point)
2. TÜM dirty buffer'ları diske yaz
3. fsync
4. pg_control'e "checkpoint burada bitti" yaz
5. Artık gereksiz WAL segmentlerini geri dönüştür/sil
```

**Neden önemli:** kurtarma süresi = son checkpoint'ten beri biriken WAL miktarı. Sık checkpoint = hızlı kurtarma ama daha çok I/O ve daha çok full-page write. `checkpoint_timeout` ve `max_wal_size` bu dengeyi ayarlar.

`checkpoint_completion_target` (varsayılan 0.9): yazma işini checkpoint aralığına yayar, tek seferde I/O fırtınası çıkarmamak için.

## 9.2 Background writer

[src/backend/storage/buffer/bufmgr.c:3854](../src/backend/storage/buffer/bufmgr.c#L3854) — `BgBufferSync`

Clock-sweep ibresinin **önünde** gider ve yakında kurban olacak buffer'ları önceden temizler. Amaç: backend'lerin `GetVictimBuffer` içinde dirty bir sayfaya rastlayıp kendi kendine yazma yapmak zorunda kalmaması.

Etkisini `pg_stat_bgwriter`'da görürsünüz: `buffers_backend` yüksekse bgwriter yetişemiyor demektir.

---

# 10. VACUUM — MVCC'nin faturası

[src/backend/access/heap/vacuumlazy.c:624](../src/backend/access/heap/vacuumlazy.c#L624) — `heap_vacuum_rel`
[src/backend/access/heap/vacuumlazy.c:1279](../src/backend/access/heap/vacuumlazy.c#L1279) — `lazy_scan_heap`

Üç işi birden yapar:

**1. Ölü satırları toplar.** Hiçbir aktif snapshot'ın göremeyeceği satır sürümleri silinir, yerleri yeniden kullanılabilir hale gelir.

**2. Visibility map'i günceller.** Bir sayfadaki tüm satırlar herkese görünürse `vm` fork'unda bit set edilir. Bunun sonucu **index-only scan**: index'ten gelen bir TID için heap'e gitmeye gerek kalmaz.

**3. XID wraparound'u önler.** XID 32 bittir ve döner. Yeterince eski satırların `xmin`'i `FrozenTransactionId` ile değiştirilir ("freeze") — böylece o satır her zaman görünür sayılır.

**Neden anti-wraparound vacuum durdurulamaz:** XID alanı tükenirse veritabanı yazmaları reddetmek zorunda kalır. Bu yüzden `autovacuum_freeze_max_age`'a yaklaşıldığında vacuum, `autovacuum = off` olsa bile çalışır.

İzleme:

```sql
SELECT relname, n_dead_tup, last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;

-- wraparound'a ne kadar var?
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;
```

---

# 11. Genişletilebilirlik — PostgreSQL'i özel kılan şey

## 11.1 Her şey katalogda

Tip, fonksiyon, operatör, index metodu — hepsi tablolarda satır. Yeni bir tip eklemek katalog satırı eklemektir, motoru değiştirmek değil.

[src/backend/utils/cache/syscache.c:221](../src/backend/utils/cache/syscache.c#L221) — `SearchSysCache1` ile bu satırlar önbellekten okunur.

## 11.2 Table Access Method

[src/include/access/tableam.h:321](../src/include/access/tableam.h#L321) — `TableAmRoutine`
[src/backend/access/heap/heapam_handler.c:2659](../src/backend/access/heap/heapam_handler.c#L2659) — `heapam_methods`

Yaklaşık 40 fonksiyon işaretçisinden oluşan bir yapı. `heap` bunun sadece bir uygulamasıdır. Executor `table_scan_getnextslot()` çağırır ve altında ne olduğunu bilmez — sütunlu depolama (Citus, OrioleDB) bu arayüzden geçer.

## 11.3 Index Access Method

[src/include/access/amapi.h:233](../src/include/access/amapi.h#L233) — `IndexAmRoutine`
[src/backend/access/nbtree/nbtree.c:118](../src/backend/access/nbtree/nbtree.c#L118) — `bthandler`

B-tree, hash, GiST, GIN, SP-GiST, BRIN — hepsi aynı arayüzü uygular. `pg_am` tablosundaki bir satır `bthandler` gibi bir fonksiyona işaret eder; o fonksiyon `IndexAmRoutine` doldurup döner. Bir extension yeni bir index tipi ekleyebilir.

## 11.4 Hook'lar

Sabit noktalarda global fonksiyon işaretçileri. Bir extension `_PG_init()` içinde bunları zincirler:

| Hook | Yer | Kim kullanır |
|---|---|---|
| `planner_hook` | [planner.c](../src/backend/optimizer/plan/planner.c) | `pg_hint_plan` |
| `ExecutorStart_hook` / `ExecutorRun_hook` | [execMain.c](../src/backend/executor/execMain.c) | `pg_stat_statements`, `auto_explain` |
| `ProcessUtility_hook` | [utility.c:71](../src/backend/tcop/utility.c#L71) | denetim/audit extension'ları |
| `ClientAuthentication_hook` | [auth.c](../src/backend/libpq/auth.c) | özel kimlik doğrulama |
| `shmem_request_hook` | [ipci.c](../src/backend/storage/ipc/ipci.c) | shared memory isteyen extension'lar |

Örnekler için [contrib/](../contrib/) dizinine bakın — `pg_stat_statements` en öğretici olanı.

---

# 12. Bellek yönetimi — `free()` neredeyse hiç yok

[src/backend/utils/mmgr/mcxt.c:1390](../src/backend/utils/mmgr/mcxt.c#L1390) — `palloc`
[src/backend/utils/mmgr/mcxt.c:406](../src/backend/utils/mmgr/mcxt.c#L406) — `MemoryContextReset`

PostgreSQL kodu `malloc` kullanmaz; **memory context** ağacından `palloc` yapar:

```
TopMemoryContext                   (süreç ömrü)
 ├─ CacheMemoryContext             (relcache, syscache — kalıcı)
 ├─ MessageContext                 (bir istemci mesajı — parse ağacı, plan)
 │   └─ ...
 └─ PortalContext
     └─ es_query_cxt               (bir sorgu yürütmesi)
         └─ per-tuple context      (BİR SATIR — her satırda reset)
```

**Kural:** bellek tek tek serbest bırakılmaz. Kapsam bitince tüm context tek hamlede sıfırlanır (`MemoryContextReset`).

Bu yüzden:
- Bir sorgu `ERROR` verse bile bellek sızmaz — context yok edilir
- 100 milyon satırlık bir `SELECT` sabit bellekte çalışır — per-tuple context her satırda resetlenir
- `int4out` gibi fonksiyonlar gönül rahatlığıyla `palloc` yapar; temizliği kimin yaptığını düşünmezler

---

# 13. Sorgu boru hattı — özet

Ayrıntısı [SELECT-NASIL-CALISIR.md](SELECT-NASIL-CALISIR.md) dosyasında. Buradaki bağlam için kısa hali:

```
'Q' mesajı
   └─ exec_simple_query                 tcop/postgres.c:1030
      ├─ raw_parser         metin → SelectStmt      (sözdizimi, katalog yok)
      ├─ parse_analyze      SelectStmt → Query      (OID'ler, tipler, kilitler)
      ├─ QueryRewrite       Query → Query listesi   (view, rule, RLS)
      ├─ planner            Query → PlannedStmt     (maliyet tabanlı arama)
      ├─ ExecutorStart      Plan → PlanState ağacı
      └─ ExecutePlan        satır satır çek → DestReceiver → sokete yaz
```

**Dört ayrı ağaç vardır ve karıştırılmamalıdır:** `SelectStmt` (sözdizimi) → `Query` (anlam) → `Plan` (reçete) → `PlanState` (çalışma durumu).

DDL komutları planlayıcıyı atlar; [src/backend/tcop/utility.c:504](../src/backend/tcop/utility.c#L504) `ProcessUtility` üzerinden [src/backend/commands/](../src/backend/commands/) altındaki ilgili fonksiyona gider.

---

# 14. Uçtan uca: tek bir `UPDATE`'in tam yolculuğu

```sql
BEGIN;
UPDATE users SET name = 'Ali' WHERE id = 1;
COMMIT;
```

| # | Ne olur | Kod |
|---|---|---|
| 1 | `'Q'` mesajı okunur | [postgres.c:4364](../src/backend/tcop/postgres.c#L4364) |
| 2 | Transaction başlar, XID **henüz yok** | [xact.c:2106](../src/backend/access/transam/xact.c#L2106) |
| 3 | Parse → analyze → plan; `users` üzerinde `RowExclusiveLock` | [analyze.c](../src/backend/parser/analyze.c) |
| 4 | Snapshot alınır | [procarray.c:2114](../src/backend/storage/ipc/procarray.c#L2114) |
| 5 | Index scan → TID bulunur | [nbtree.c](../src/backend/access/nbtree/nbtree.c) |
| 6 | Sayfa buffer'a getirilir (yoksa diskten) | [bufmgr.c:879](../src/backend/storage/buffer/bufmgr.c#L879) |
| 7 | Satır görünür mü? | [heapam_visibility.c:939](../src/backend/access/heap/heapam_visibility.c#L939) |
| 8 | **XID şimdi atanır** (ilk yazma) | [xact.c:456](../src/backend/access/transam/xact.c#L456) |
| 9 | Eski satıra `xmax = XID`; yeni satır yazılır | [heapam.c:3267](../src/backend/access/heap/heapam.c#L3267) |
| 10 | WAL kaydı üretilir, sayfaya LSN damgası | [xloginsert.c:482](../src/backend/access/transam/xloginsert.c#L482) |
| 11 | Buffer `BM_DIRTY` işaretlenir — **diske yazılmaz** | [bufmgr.c:3170](../src/backend/storage/buffer/bufmgr.c#L3170) |
| 12 | Index girdisi eklenir (HOT değilse) | [nbtinsert.c](../src/backend/access/nbtree/nbtinsert.c) |
| 13 | `COMMIT`: commit kaydı WAL'a | [xact.c:1345](../src/backend/access/transam/xact.c#L1345) |
| 14 | **`XLogFlush` → fsync.** İstemci burayı bekler | [xlog.c:2800](../src/backend/access/transam/xlog.c#L2800) |
| 15 | CLOG'a "COMMITTED" yazılır | [clog.c:192](../src/backend/access/transam/clog.c#L192) |
| 16 | Kilitler bırakılır, `CommandComplete` gönderilir | [xact.c:2270](../src/backend/access/transam/xact.c#L2270) |
| 17 | *Daha sonra:* checkpointer sayfayı diske yazar | [xlog.c:7400](../src/backend/access/transam/xlog.c#L7400) |
| 18 | *Daha sonra:* VACUUM eski satırı toplar | [vacuumlazy.c:624](../src/backend/access/heap/vacuumlazy.c#L624) |

**Adım 11 ile 17 arasındaki boşluğa dikkat.** Veri sayfası commit anında diskte değildir. Kalıcılığı sağlayan tek şey adım 14'teki WAL flush'ıdır. Sunucu 16. adımdan sonra çökerse, kurtarma WAL'dan değişikliği yeniden uygular.

---

# 15. Derleme ve kendiniz çalıştırma

## 15.1 Build

```bash
# Meson (modern yol — meson.build:1)
meson setup build --prefix=$HOME/pgsql -Dcassert=true -Ddebug=true
ninja -C build
ninja -C build install

# veya autoconf (configure.ac)
./configure --prefix=$HOME/pgsql --enable-debug --enable-cassert
make -j$(nproc) && make install
```

`-Dcassert=true` geliştirme için önemlidir: iç tutarlılık `Assert`'lerini açar.

## 15.2 Çalıştırma

```bash
initdb -D ~/pgdata
pg_ctl -D ~/pgdata -l ~/pg.log start
psql postgres
```

## 15.3 İçeriyi gözlemleme

```sql
-- Süreçler
SELECT pid, backend_type, state, wait_event_type, wait_event, query
FROM pg_stat_activity;

-- Buffer pool içinde ne var? (pg_buffercache extension)
CREATE EXTENSION pg_buffercache;
SELECT c.relname, count(*) AS buffers
FROM pg_buffercache b JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY 1 ORDER BY 2 DESC LIMIT 10;

-- Satırların MVCC alanları
SELECT ctid, xmin, xmax, * FROM users LIMIT 5;

-- WAL üretim hızı
SELECT pg_current_wal_lsn();

-- Kilitler
SELECT * FROM pg_locks WHERE NOT granted;

-- Checkpoint / bgwriter davranışı
SELECT * FROM pg_stat_bgwriter;
```

```bash
# WAL kayıtlarını okunabilir hale getir
pg_waldump -p ~/pgdata/pg_wal 000000010000000000000001 | head -50

# Kontrol dosyası: son checkpoint, LSN'ler, wraparound durumu
pg_controldata ~/pgdata
```

## 15.4 gdb ile canlı takip

```bash
psql -Atc "SELECT pg_backend_pid()"       # → 12345
gdb -p 12345
```

```
(gdb) break exec_simple_query        # sorgu girişi
(gdb) break heap_update              # satır güncelleme
(gdb) break XLogInsertRecord         # WAL yazımı
(gdb) break RecordTransactionCommit  # commit
(gdb) break GetVictimBuffer          # buffer eviction
(gdb) continue
```

```
# postgresql.conf içinde ağaçları loglayın
debug_print_parse = on
debug_print_rewritten = on
debug_print_plan = on
debug_pretty_print = on
log_min_messages = debug1
```

---

# Akılda kalması gereken 5 şey

1. **Süreç başına bağlantı, paylaşımlı bellek ortak.** Postmaster kabul eder ve fork eder; veriye asla dokunmaz — böylece bir backend çökerse temiz kurtarma yapabilir.

2. **WAL kalıcılığı sağlar, veri dosyaları değil.** Commit = WAL'ın fsync'lenmesi. Veri sayfaları saatler sonra checkpointer tarafından yazılabilir. Bir sayfa diske yazılmadan önce onun WAL'ı diskte olmak zorundadır (`pd_lsn` kuralı).

3. **Yerinde güncelleme yok.** `UPDATE` yeni satır sürümü yazar, eskisine `xmax` damgalar. Okuyucular yazıcıları bloklamaz — bedeli bloat ve zorunlu `VACUUM`'dur.

4. **Görünürlük = snapshot + `xmin`/`xmax` + CLOG.** Taranan her satır için `HeapTupleSatisfiesMVCC` çalışır; sonuç hint bit olarak satıra yazılır — bu yüzden `SELECT` bile disk yazması tetikleyebilir.

5. **Memory context'ler `free()`'yi ortadan kaldırır.** Bellek kapsam bitince toplu sıfırlanır. Bu yüzden hata durumunda sızıntı olmaz ve devasa sorgular sabit bellekte çalışır.

---

# Buradan nereye

| Ne öğrenmek istiyorsanız | Nereye bakın |
|---|---|
| WAL + MVCC tasarım gerekçeleri | [src/backend/access/transam/README](../src/backend/access/transam/README) |
| Buffer değiştirme algoritması | [src/backend/storage/buffer/README](../src/backend/storage/buffer/README) |
| Kilit hiyerarşisi, deadlock | [src/backend/storage/lmgr/README](../src/backend/storage/lmgr/README) |
| Planlayıcı iç yapısı | [src/backend/optimizer/README](../src/backend/optimizer/README) |
| B-tree eşzamanlılığı (Lehman-Yao) | [src/backend/access/nbtree/README](../src/backend/access/nbtree/README) |
| Sorgu boru hattı, satır satır | [SELECT-NASIL-CALISIR.md](SELECT-NASIL-CALISIR.md) |
