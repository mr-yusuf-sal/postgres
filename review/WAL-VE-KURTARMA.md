# WAL ve kurtarma: PostgreSQL çökmeden nasıl geri döner?

Kaynak ağacı: PostgreSQL **20devel** ([meson.build](../meson.build) `version: '20devel'`).
Satır numaraları sürüme bağlıdır — başka bir sürümde bakarsanız kayarlar.

Bu not tek bir soruyu takip ediyor: **elektrik kesildiğinde commit edilmiş veri
neden kaybolmuyor?** Cevap "WAL" değil; cevap WAL'ın nasıl yazıldığı, ne zaman
diske indiği, checkpoint'in onu nasıl kısalttığı ve redo'nun onu nasıl tekrar
oynattığı.

## 30 saniyelik özet

Temel kural tek cümle: **veri sayfası diske inmeden önce, o sayfayı anlatan WAL
kaydı diske inmiş olmalı.** ("Write-Ahead")

```
  BACKEND                          SHARED MEMORY                     DİSK
  ───────                          ─────────────                     ────

  heap_update()
    │
    ├─ sayfayı değiştir ──────────► shared buffer (dirty)
    │                                    │  PageSetLSN(page, LSN)
    │                                    │
    └─ XLogInsert() ─────────────► WAL buffer'ları ─────────────► pg_wal/0000...
             │                       (wal_buffers)                  │
             │  1) yer rezerve et                                   │
             │     (insertpos_lck spinlock)                         │
             │  2) kopyala                                          │
             │     (WAL insert lock, 8 tane)                        │
                                                                    │
  COMMIT ───► XLogFlush(LSN) ──────────────────────────► fsync ─────┘
                                                                    ▲
  bufmgr: dirty sayfayı yazmadan önce ── XLogFlush(page LSN) ────────┘
                                          ("WAL rule")
```

Çökme sonrası:

```
  pg_control ──► "son checkpoint burada, redo noktası şu LSN"
       │
       ▼
  StartupXLOG ──► InitWalRecovery ──► PerformWalRecovery
                                           │
                                           │  redo noktasından başla, kayıt kayıt oku
                                           ▼
                                    ┌─────────────────────┐
                                    │ CRC doğru mu?       │
                                    │ prev-link tutuyor?  │──── hayır ──► WAL'ın sonu, dur
                                    └──────────┬──────────┘
                                               │ evet
                                    GetRmgr(xl_rmid).rm_redo(record)
                                               │
                                    ┌──────────▼──────────────────┐
                                    │ FPI var mı? → sayfayı komple │
                                    │              geri yükle      │
                                    │ yoksa: page LSN >= kayıt LSN │
                                    │   ise ATLA (zaten uygulanmış)│
                                    │   değilse uygula             │
                                    └──────────────────────────────┘
```

Zihinsel model — beş madde:

- **LSN her şeyi bağlar.** LSN bir WAL bayt konumudur. Her veri sayfası, kendisini
  en son değiştiren kaydın LSN'ini taşır. Kurtarmada `sayfa LSN >= kayıt LSN` ise
  o kayıt zaten uygulanmıştır → atlanır. Bu tek karşılaştırma redo'yu **idempotent**
  yapar; aynı WAL'ı iki kez oynatmak zarar vermez.
- **Full-page write torn page'e karşı.** Checkpoint'ten sonra bir sayfaya ilk
  dokunan kayıt, sayfanın **tam kopyasını** içerir. Çünkü 8 kB'lik bir sayfa
  yazımının atomik olduğunu varsayamayız — yarısı yeni yarısı eski bir sayfaya
  incremental redo uygulamak çöp üretir.
- **Insert iki adımlıdır.** Önce spinlock altında **yer rezerve edilir** (sadece
  `CurrBytePos += size`), sonra lock bırakılıp **kopyalama paralel yapılır**. Bu
  yüzden WAL insert yüksek eşzamanlılıkta tıkanmaz.
- **Checkpoint bir "redo başlangıcı" tayin eder.** Checkpoint kirli buffer'ları
  diske indirir; böylece redo noktasından öncesindeki WAL artık gereksizdir ve
  segmentler silinebilir/geri dönüştürülebilir. Checkpoint uzun sürer çünkü işin
  büyük kısmı `CheckPointBuffers` — yani I/O.
- **Recovery ile normal çalışma aynı kodu paylaşır.** Standby'daki replay,
  crash recovery ile aynı `rm_redo` fonksiyonlarını çağırır. Fark: standby
  durmaz, ve checkpoint yerine **restartpoint** yapar.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/access/transam/README](../src/backend/access/transam/README) | Tasarım belgesi — WAL bölümü satır 399'dan başlar, önce bu okunur |
| [../src/include/access/xlogrecord.h](../src/include/access/xlogrecord.h) | WAL kayıt formatı: `XLogRecord`, blok başlıkları, FPI başlığı |
| [../src/include/access/xlog_internal.h](../src/include/access/xlog_internal.h) | WAL **sayfa** başlığı, segment boyu makroları, dosya adlandırma |
| [../src/backend/access/transam/xloginsert.c](../src/backend/access/transam/xloginsert.c) | Kayıt inşası: `XLogBeginInsert`, `XLogRegister*`, `XLogRecordAssemble` — FPI kararı burada |
| [../src/backend/access/transam/xlog.c](../src/backend/access/transam/xlog.c) | Motor: `XLogInsertRecord`, `XLogWrite`, `XLogFlush`, `CreateCheckPoint`, `StartupXLOG` |
| [../src/backend/access/transam/xlogreader.c](../src/backend/access/transam/xlogreader.c) | Kayıt okuma/çözme, CRC ve prev-link doğrulaması, `RestoreBlockImage` |
| [../src/backend/access/transam/xlogrecovery.c](../src/backend/access/transam/xlogrecovery.c) | Replay döngüsü: `PerformWalRecovery`, `ApplyWalRecord`, tutarlılık noktası |
| [../src/backend/access/transam/xlogutils.c](../src/backend/access/transam/xlogutils.c) | `XLogReadBufferForRedo` — redo rutinlerinin ortak giriş kapısı, LSN karşılaştırması |
| [../src/backend/access/transam/xlogarchive.c](../src/backend/access/transam/xlogarchive.c) | `.ready` / `.done` dosyaları, arşivciyi uyandırma |
| [../src/backend/access/transam/timeline.c](../src/backend/access/transam/timeline.c) | `.history` dosyaları, timeline soyağacı |
| [../src/include/access/rmgrlist.h](../src/include/access/rmgrlist.h) | Resource manager tablosu — `xl_rmid` → redo/desc fonksiyonları |
| [../src/backend/access/rmgrdesc/](../src/backend/access/rmgrdesc/) | Kayıt açıklama fonksiyonları (`pg_waldump` bunları kullanır) |
| [../src/include/catalog/pg_control.h](../src/include/catalog/pg_control.h) | `CheckPoint` ve `ControlFileData` yapıları |
| [../src/backend/postmaster/checkpointer.c](../src/backend/postmaster/checkpointer.c) | Checkpointer süreci, zamanlama ve throttling |
| [../src/backend/postmaster/walwriter.c](../src/backend/postmaster/walwriter.c) | WAL writer süreci, arka plan flush |
| [../src/backend/storage/buffer/bufmgr.c](../src/backend/storage/buffer/bufmgr.c) | WAL kuralının uygulandığı yer: `FlushBuffer` |

---

# 1. WAL kayıt formatı

Bir WAL kaydı sabit başlık + blok referansları + veri şeklinde dizilir. Yerleşim
başlık dosyasının en üstünde anlatılıyor:

[../src/include/access/xlogrecord.h:20-40](../src/include/access/xlogrecord.h#L20-L40)

```c
/*
 * The overall layout of an XLOG record is:
 *		Fixed-size header (XLogRecord struct)
 *		XLogRecordBlockHeader struct
 *		XLogRecordBlockHeader struct
 *		...
 *		XLogRecordDataHeader[Short|Long] struct
 *		block data
 *		block data
 *		...
 *		main data
 *
 * There can be zero or more XLogRecordBlockHeaders, and 0 or more bytes of
 * rmgr-specific data not associated with a block.  XLogRecord structs
 * always start on MAXALIGN boundaries in the WAL files, but the rest of
 * the fields are not aligned.
 */
```

## 1.1 Sabit başlık

[../src/include/access/xlogrecord.h:41-55](../src/include/access/xlogrecord.h#L41-L55)

```c
typedef struct XLogRecord
{
	uint32		xl_tot_len;		/* total len of entire record */
	TransactionId xl_xid;		/* xact id */
	XLogRecPtr	xl_prev;		/* ptr to previous record in log */
	uint8		xl_info;		/* flag bits, see below */
	RmgrId		xl_rmid;		/* resource manager for this record */
	/* 2 bytes of padding here, initialize to zero */
	pg_crc32c	xl_crc;			/* CRC for this record */

	/* XLogRecordBlockHeaders and XLogRecordDataHeader follow, no padding */

} XLogRecord;
```

Dört alan işin tamamını taşıyor:

- `xl_rmid` — hangi **resource manager**? Kurtarmada dispatch tablosunun indeksi.
- `xl_info` — üst 4 bit rmgr'ye ait (INSERT mi UPDATE mi), alt 4 bit WAL altyapısının.
- `xl_prev` — bir önceki kaydın LSN'i. Zincir geriye doğru bağlı. Bu alan aynı
  zamanda **çöp kayıt tespit sensörü** (bkz. bölüm 8.2).
- `xl_crc` — kaydın bütününün CRC32C'si.

Kayıt boyutunun üst sınırı, okuyucunun tek parçada palloc edebilmesi kısıtından
geliyor:

[../src/include/access/xlogrecord.h:65-74](../src/include/access/xlogrecord.h#L65-L74)

```c
/*
 * XLogReader needs to allocate all the data of a WAL record in a single
 * chunk.  This means that a single XLogRecord cannot exceed MaxAllocSize
 * in length if we ignore any allocation overhead of the XLogReader.
 * ...
 */
#define XLogRecordMaxSize	(1020 * 1024 * 1024)
```

## 1.2 Blok referansları

Bir kaydın dokunduğu her sayfa için bir `XLogRecordBlockHeader` var:

[../src/include/access/xlogrecord.h:103-115](../src/include/access/xlogrecord.h#L103-L115)

```c
typedef struct XLogRecordBlockHeader
{
	uint8		id;				/* block reference ID */
	uint8		fork_flags;		/* fork within the relation, and flags */
	uint16		data_length;	/* number of payload bytes (not including page
								 * image) */

	/* If BKPBLOCK_HAS_IMAGE, an XLogRecordBlockImageHeader struct follows */
	/* If BKPBLOCK_SAME_REL is not set, a RelFileLocator follows */
	/* BlockNumber follows */
} XLogRecordBlockHeader;
```

`fork_flags` alanının alt 4 biti fork numarası (main/fsm/vm/init), üst 4 biti bayrak:

[../src/include/access/xlogrecord.h:192-202](../src/include/access/xlogrecord.h#L192-L202)

```c
#define BKPBLOCK_FORK_MASK	0x0F
#define BKPBLOCK_FLAG_MASK	0xF0
#define BKPBLOCK_HAS_IMAGE	0x10	/* block data is an XLogRecordBlockImage */
#define BKPBLOCK_HAS_DATA	0x20
#define BKPBLOCK_WILL_INIT	0x40	/* redo will re-init the page */
#define BKPBLOCK_SAME_REL	0x80	/* RelFileLocator omitted, same as
									 * previous */
```

`BKPBLOCK_SAME_REL` küçük ama önemli bir tasarruf: aynı tabloya ait ardışık blok
referanslarında `RelFileLocator` (16+ bayt) tekrar yazılmaz.

`BKPBLOCK_WILL_INIT` bir sözleşme: "redo bu sayfayı sıfırdan kuracak, diskteki
halini okumana gerek yok." Redo rutini bu sözü tutmazsa PANIC yer
([../src/backend/access/transam/xlogutils.c:390-393](../src/backend/access/transam/xlogutils.c#L390-L393)).

## 1.3 Ana veri başlığı

Kaydın bloklarla ilişkisiz "main data" kısmı 256 bayttan küçükse tek baytlık
uzunluk alanı kullanılır:

[../src/include/access/xlogrecord.h:213-227](../src/include/access/xlogrecord.h#L213-L227)

```c
typedef struct XLogRecordDataHeaderShort
{
	uint8		id;				/* XLR_BLOCK_ID_DATA_SHORT */
	uint8		data_length;	/* number of payload bytes */
}			XLogRecordDataHeaderShort;

#define SizeOfXLogRecordDataHeaderShort (sizeof(uint8) * 2)

typedef struct XLogRecordDataHeaderLong
{
	uint8		id;				/* XLR_BLOCK_ID_DATA_LONG */
	/* followed by uint32 data_length, unaligned */
}			XLogRecordDataHeaderLong;
```

WAL formatının her yerinde aynı takıntı görülüyor: **bayt saymak**. WAL hacmi
doğrudan disk I/O ve replikasyon bant genişliği demek.

---

# 2. Full-page write: torn page problemi

## 2.1 Neden var

README bunu tek paragrafta anlatıyor:

[../src/backend/access/transam/README:424-435](../src/backend/access/transam/README#L424-L435)

```
Usually, log entries contain just enough information to redo a single
incremental update on a page (or small group of pages).  This will work only
if the filesystem and hardware implement data page writes as atomic actions,
so that a page is never left in a corrupt partly-written state.  Since that's
often an untenable assumption in practice, we log additional information to
allow complete reconstruction of modified pages.  The first WAL record
affecting a given page after a checkpoint is made to contain a copy of the
entire page, and we implement replay by restoring that page copy instead of
redoing the update.  (This is more reliable than the data storage itself would
be because we can check the validity of the WAL record's CRC.)  We can detect
the "first change after checkpoint" by noting whether the page's old LSN
precedes the end of WAL as of the last checkpoint (the RedoRecPtr).
```

Somut senaryo: PostgreSQL sayfası 8 kB, disk sektörü 512 bayt veya 4 kB. İşletim
sistemi 8 kB'yi tek atomik işlem olarak yazacağını **garanti etmez**. Elektrik
tam ortada kesilirse sayfanın ilk yarısı yeni, ikinci yarısı eski olur — "torn
page". Böyle bir sayfaya "offset 5'teki tuple'ın xmax'ini 500 yap" tarzı
incremental bir redo uygulamak anlamsızdır; sayfanın kendisi tutarsızdır.

Çözüm: checkpoint'ten sonra sayfaya **ilk** dokunan kayıt sayfanın tamamını
taşısın. Redo o kaydı gördüğünde incremental değişikliği uygulamaz — sayfayı
komple WAL'daki kopyayla değiştirir. Bundan sonraki kayıtlar sağlam bir zemine
inşa eder.

## 2.2 Karar nerede veriliyor

`XLogRecordAssemble` her kayıtlı buffer için tek tek karar veriyor:

[../src/backend/access/transam/xloginsert.c:678-700](../src/backend/access/transam/xloginsert.c#L678-L700)

```c
		/* Determine if this block needs to be backed up */
		if (regbuf->flags & REGBUF_FORCE_IMAGE)
			needs_backup = true;
		else if (regbuf->flags & REGBUF_NO_IMAGE)
			needs_backup = false;
		else if (!doPageWrites)
			needs_backup = false;
		else
		{
			/*
			 * We assume page LSN is first data on *every* page that can be
			 * passed to XLogInsert, whether it has the standard page layout
			 * or not.
			 */
			XLogRecPtr	page_lsn = PageGetLSN(regbuf->page);

			needs_backup = (page_lsn <= RedoRecPtr);
			if (!needs_backup)
			{
				if (!XLogRecPtrIsValid(*fpw_lsn) || page_lsn < *fpw_lsn)
					*fpw_lsn = page_lsn;
			}
		}
```

Kalbi tek satır: `needs_backup = (page_lsn <= RedoRecPtr)`. Sayfanın LSN'i son
checkpoint'in redo noktasından eskiyse, bu sayfa checkpoint'ten beri hiç
değişmemiştir → biz ilk dokunanız → tam kopya alalım.

FPI alındığında blokla ilişkili incremental veri **yazılmaz**, çünkü gereksiz:

[../src/backend/access/transam/xloginsert.c:702-708](../src/backend/access/transam/xloginsert.c#L702-L708)

```c
		/* Determine if the buffer data needs to included */
		if (regbuf->rdata_len == 0)
			needs_data = false;
		else if ((regbuf->flags & REGBUF_KEEP_DATA) != 0)
			needs_data = true;
		else
			needs_data = !needs_backup;
```

## 2.3 Sayfadaki "delik" ve sıkıştırma

8 kB'lik sayfanın ortası genelde boştur (line pointer'lar başta, tuple'lar sonda).
WAL bu deliği atar:

[../src/include/access/xlogrecord.h:141-154](../src/include/access/xlogrecord.h#L141-L154)

```c
typedef struct XLogRecordBlockImageHeader
{
	uint16		length;			/* number of page image bytes */
	uint16		hole_offset;	/* number of bytes before "hole" */
	uint8		bimg_info;		/* flag bits, see below */

	/*
	 * If BKPIMAGE_HAS_HOLE and BKPIMAGE_COMPRESSED(), an
	 * XLogRecordBlockCompressHeader struct follows.
	 */
} XLogRecordBlockImageHeader;
```

`wal_compression` açıksa delik atıldıktan sonra kalan kısım ayrıca sıkıştırılır:

[../src/include/access/xlogrecord.h:156-167](../src/include/access/xlogrecord.h#L156-L167)

```c
/* Information stored in bimg_info */
#define BKPIMAGE_HAS_HOLE		0x01	/* page image has "hole" */
#define BKPIMAGE_APPLY			0x02	/* page image should be restored
										 * during replay */
/* compression methods supported */
#define BKPIMAGE_COMPRESS_PGLZ	0x04
#define BKPIMAGE_COMPRESS_LZ4	0x08
#define BKPIMAGE_COMPRESS_ZSTD	0x10
```

`BKPIMAGE_APPLY` ayrı bir bayrak, çünkü FPI her zaman geri yüklenmez:
`wal_consistency_checking` açıkken kayıtlara **doğrulama amaçlı** FPI eklenir; o
görüntü redo sırasında uygulanmaz, sadece sonuçla karşılaştırılır
([../src/backend/access/transam/xlogrecovery.c:1978-1979](../src/backend/access/transam/xlogrecovery.c#L1978-L1979)).

## 2.4 Bunun bedeli

Checkpoint sıklığı doğrudan WAL hacmini belirler. Checkpoint'ten hemen sonraki
kısa pencerede neredeyse her yazma bir FPI üretir — 8 kB WAL, tek satırlık bir
UPDATE için. Bu yüzden `checkpoint_timeout` düşürmek "daha güvenli" değil, çoğu
zaman sadece **daha çok WAL** demektir. `pg_stat_wal.wal_fpi` bu maliyeti ölçer.

---

# 3. `XLogInsertRecord`: kaydın WAL'a giriş yarışı

Bu, PostgreSQL'in en yoğun trafikli kod yollarından biri: her yazan backend
buradan geçer. Tasarım, fonksiyonun içindeki yorumda anlatılıyor:

[../src/backend/access/transam/xlog.c:824-855](../src/backend/access/transam/xlog.c#L824-L855)

```c
	/*----------
	 *
	 * We have now done all the preparatory work we can without holding a
	 * lock or modifying shared state. From here on, inserting the new WAL
	 * record to the shared WAL buffer cache is a two-step process:
	 *
	 * 1. Reserve the right amount of space from the WAL. The current head of
	 *	  reserved space is kept in Insert->CurrBytePos, and is protected by
	 *	  insertpos_lck.
	 *
	 * 2. Copy the record to the reserved WAL space. This involves finding the
	 *	  correct WAL buffer containing the reserved space, and copying the
	 *	  record in place. This can be done concurrently in multiple processes.
	 *
	 * To keep track of which insertions are still in-progress, each concurrent
	 * inserter acquires an insertion lock. In addition to just indicating that
	 * an insertion is in progress, the lock tells others how far the inserter
	 * has progressed. There is a small fixed number of insertion locks,
	 * determined by NUM_XLOGINSERT_LOCKS. When an inserter crosses a page
	 * boundary, it updates the value stored in the lock to the how far it has
	 * inserted, to allow the previous buffer to be flushed.
	 *
	 * Holding onto an insertion lock also protects RedoRecPtr and
	 * fullPageWrites from changing until the insertion is finished.
	 *----------
	 */
```

## 3.1 Insert lock'lar

Sekiz tane var, sabit:

[../src/backend/access/transam/xlog.c:157](../src/backend/access/transam/xlog.c#L157)

```c
#define NUM_XLOGINSERT_LOCKS  8
```

Backend hangi lock'u alacağını seçerken **affinity** gözetiyor — cache line
zıplamasını azaltmak için:

[../src/backend/access/transam/xlog.c:1427-1449](../src/backend/access/transam/xlog.c#L1427-L1449)

```c
	static int	lockToTry = -1;

	if (lockToTry == -1)
		lockToTry = MyProcNumber % NUM_XLOGINSERT_LOCKS;
	MyLockNo = lockToTry;

	/*
	 * The insertingAt value is initially set to 0, as we don't know our
	 * insert location yet.
	 */
	immed = LWLockAcquire(&WALInsertLocks[MyLockNo].l.lock, LW_EXCLUSIVE);
	if (!immed)
	{
		/*
		 * If we couldn't get the lock immediately, try another lock next
		 * time.  On a system with more insertion locks than concurrent
		 * inserters, this causes all the inserters to eventually migrate to a
		 * lock that no-one else is using. ...
		 */
		lockToTry = (lockToTry + 1) % NUM_XLOGINSERT_LOCKS;
	}
```

Yani "aynı lock'u dene, alamazsan bir sonrakine geç" — basit ama etkili bir
kendi kendine dengeleme.

## 3.2 RedoRecPtr yeniden kontrolü ve baştan başlama

Kayıt zaten `XLogRecordAssemble` tarafından, lock **olmadan** okunan bir
`RedoRecPtr` değerine göre kuruldu. Lock alındıktan sonra bu değer doğrulanır:

[../src/backend/access/transam/xlog.c:878-896](../src/backend/access/transam/xlog.c#L878-L896)

```c
		if (RedoRecPtr != Insert->RedoRecPtr)
		{
			Assert(RedoRecPtr < Insert->RedoRecPtr);
			RedoRecPtr = Insert->RedoRecPtr;
		}
		doPageWrites = (Insert->fullPageWrites || Insert->runningBackups > 0);

		if (doPageWrites &&
			(!prevDoPageWrites ||
			 (XLogRecPtrIsValid(fpw_lsn) && fpw_lsn <= RedoRecPtr)))
		{
			/*
			 * Oops, some buffer now needs to be backed up that the caller
			 * didn't back up.  Start over.
			 */
			WALInsertLockRelease();
			END_CRIT_SECTION();
			return InvalidXLogRecPtr;
		}
```

Araya bir checkpoint girdiyse, FPI almadığımız bir sayfanın artık FPI'ye ihtiyacı
olabilir. O zaman `InvalidXLogRecPtr` döner ve çağıran **kaydı sıfırdan kurar**:

[../src/backend/access/transam/xloginsert.c:512-535](../src/backend/access/transam/xloginsert.c#L512-L535)

```c
	do
	{
		...
		/*
		 * Get values needed to decide whether to do full-page writes. Since
		 * we don't yet have an insertion lock, these could change under us,
		 * but XLogInsertRecord will recheck them once it has a lock.
		 */
		GetFullPageWriteInfo(&RedoRecPtr, &doPageWrites);

		rdt = XLogRecordAssemble(rmid, info, RedoRecPtr, doPageWrites,
								 &fpw_lsn, &num_fpi, &fpi_bytes,
								 &topxid_included);

		EndPos = XLogInsertRecord(rdt, fpw_lsn, curinsert_flags, num_fpi,
								  fpi_bytes, topxid_included);
	} while (!XLogRecPtrIsValid(EndPos));
```

Optimistic concurrency: yaygın durumda kilitsiz hızlı git, nadir çakışmada baştan
başla.

## 3.3 Rezervasyon: spinlock altındaki üç satır

[../src/backend/access/transam/xlog.c:1162-1184](../src/backend/access/transam/xlog.c#L1162-L1184)

```c
	/*
	 * The duration the spinlock needs to be held is minimized by minimizing
	 * the calculations that have to be done while holding the lock. The
	 * current tip of reserved WAL is kept in CurrBytePos, as a byte position
	 * that only counts "usable" bytes in WAL, that is, it excludes all WAL
	 * page headers. The mapping between "usable" byte positions and physical
	 * positions (XLogRecPtrs) can be done outside the locked region, and
	 * because the usable byte position doesn't include any headers, reserving
	 * X bytes from WAL is almost as simple as "CurrBytePos += X".
	 */
	SpinLockAcquire(&Insert->insertpos_lck);

	startbytepos = Insert->CurrBytePos;
	endbytepos = startbytepos + size;
	prevbytepos = Insert->PrevBytePos;
	Insert->CurrBytePos = endbytepos;
	Insert->PrevBytePos = startbytepos;

	SpinLockRelease(&Insert->insertpos_lck);

	*StartPos = XLogBytePosToRecPtr(startbytepos);
	*EndPos = XLogBytePosToEndRecPtr(endbytepos);
	*PrevPtr = XLogBytePosToRecPtr(prevbytepos);
```

"Usable byte position" numarası şu işe yarıyor: WAL sayfa başlıkları
hesaplamayı doğrusal olmaktan çıkarır (her 8 kB'de 24-40 bayt başlık var).
Başlıksız bir sanal eksende toplama yapılıp dönüşüm **spinlock dışına** taşınıyor.
Spinlock altında kalan iş: iki okuma, iki toplama, iki yazma.

Not: `xl_prev` de burada dolduruluyor (`prevbytepos`). Yani kaydın CRC'si ancak
rezervasyondan **sonra** hesaplanabilir:

[../src/backend/access/transam/xlog.c:944-961](../src/backend/access/transam/xlog.c#L944-L961)

```c
	if (inserted)
	{
		/*
		 * Now that xl_prev has been filled in, calculate CRC of the record
		 * header.
		 */
		rdata_crc = rechdr->xl_crc;
		COMP_CRC32C(rdata_crc, rechdr, offsetof(XLogRecord, xl_crc));
		FIN_CRC32C(rdata_crc);
		rechdr->xl_crc = rdata_crc;

		/*
		 * All the record data, including the header, is now ready to be
		 * inserted. Copy the record in the space reserved.
		 */
		CopyXLogRecordToWAL(rechdr->xl_tot_len,
							class == WALINSERT_SPECIAL_SWITCH, rdata,
							StartPos, EndPos, insertTLI);
	}
```

## 3.4 Kopyalama ve sayfa sınırları

`CopyXLogRecordToWAL` kaydı buffer'lara yazarken sayfa sınırlarını yönetir. Bir
kayıt sayfaya sığmazsa devam eder ve yeni sayfanın başlığına iz bırakır:

[../src/backend/access/transam/xlog.c:1296-1321](../src/backend/access/transam/xlog.c#L1296-L1321)

```c
		while (rdata_len > freespace)
		{
			/*
			 * Write what fits on this page, and continue on the next page.
			 */
			Assert(CurrPos % XLOG_BLCKSZ >= SizeOfXLogShortPHD || freespace == 0);
			memcpy(currpos, rdata_data, freespace);
			rdata_data += freespace;
			rdata_len -= freespace;
			written += freespace;
			CurrPos += freespace;

			/*
			 * Get pointer to beginning of next page, and set the xlp_rem_len
			 * in the page header. Set XLP_FIRST_IS_CONTRECORD.
			 * ...
			 */
			currpos = GetXLogBuffer(CurrPos, tli);
			pagehdr = (XLogPageHeader) currpos;
			pagehdr->xlp_rem_len = write_len - written;
			pagehdr->xlp_info |= XLP_FIRST_IS_CONTRECORD;
```

`XLP_FIRST_IS_CONTRECORD` + `xlp_rem_len` ikilisi, okuyucunun bir sayfanın
ortasından başlayıp "bu sayfanın başındaki bayt yığını önceki kaydın devamı"
diyebilmesini sağlar.

## 3.5 Üç insert sınıfı

Normal kayıt tek insert lock alır. İki özel durum **hepsini** alır:

[../src/backend/access/transam/xlog.c:802-809](../src/backend/access/transam/xlog.c#L802-L809)

```c
	/* Does this record type require special handling? */
	if (unlikely(rechdr->xl_rmid == RM_XLOG_ID))
	{
		if (info == XLOG_SWITCH)
			class = WALINSERT_SPECIAL_SWITCH;
		else if (info == XLOG_CHECKPOINT_REDO)
			class = WALINSERT_SPECIAL_CHECKPOINT;
	}
```

- `XLOG_SWITCH` (`pg_switch_wal()`): segmentin geri kalanını rezerve edeceği için
  kimsenin araya girmemesi gerekir ([xlog.c:908-924](../src/backend/access/transam/xlog.c#L908-L924)).
- `XLOG_CHECKPOINT_REDO`: bu kaydın **başlangıç LSN'i** yeni redo noktası olacak;
  paylaşılan `RedoRecPtr` atomik olarak güncellenmeli
  ([xlog.c:936-941](../src/backend/access/transam/xlog.c#L936-L941)).

---

# 4. WAL buffer'ları, `XLogWrite`, `XLogFlush`

## 4.1 `wal_buffers`

Varsayılan `-1`, yani otomatik:

[../src/backend/access/transam/xlog.c:5011-5033](../src/backend/access/transam/xlog.c#L5011-L5033)

```c
/*
 * Auto-tune the number of XLOG buffers.
 *
 * The preferred setting for wal_buffers is about 3% of shared_buffers, with
 * a maximum of one XLOG segment (there is little reason to think that more
 * is helpful, at least so long as we force an fsync when switching log files)
 * and a minimum of 8 blocks (which was the default value prior to PostgreSQL
 * 9.1, when auto-tuning was added).
 * ...
 */
static int
XLOGChooseNumBuffers(void)
{
	int			xbuffers;

	xbuffers = NBuffers / 32;
	if (xbuffers > (wal_segment_size / XLOG_BLCKSZ))
		xbuffers = (wal_segment_size / XLOG_BLCKSZ);
	if (xbuffers < 8)
		xbuffers = 8;
	return xbuffers;
}
```

Üst sınır bir WAL segmenti (varsayılan 16 MB), çünkü segment değişiminde zaten
fsync yapılıyor — daha fazla buffer tutmanın anlamı yok.

Buffer'lar dolduğunda insert yapan backend'in kendisi yazmak zorunda kalır — bu
istenmeyen durumdur ve sayılır:

[../src/backend/access/transam/xlog.c:2092-2101](../src/backend/access/transam/xlog.c#L2092-L2101)

```c
				else
				{
					/* Have to write it ourselves */
					TRACE_POSTGRESQL_WAL_BUFFER_WRITE_DIRTY_START();
					WriteRqst.Write = OldPageRqstPtr;
					WriteRqst.Flush = InvalidXLogRecPtr;
					XLogWrite(WriteRqst, tli, false);
					LWLockRelease(WALWriteLock);
					pgWalUsage.wal_buffers_full++;
					TRACE_POSTGRESQL_WAL_BUFFER_WRITE_DIRTY_DONE();
```

`pg_stat_wal.wal_buffers_full` sürekli artıyorsa `wal_buffers` küçüktür.

## 4.2 `XLogWrite`: buffer'lardan dosyaya

[../src/backend/access/transam/xlog.c:2325](../src/backend/access/transam/xlog.c#L2325)

Döngü, ardışık buffer sayfalarını **tek `pwrite`** çağrısında toplamaya çalışır:

[../src/backend/access/transam/xlog.c:2442-2456](../src/backend/access/transam/xlog.c#L2442-L2456)

```c
			from = XLogCtl->pages + startidx * (Size) XLOG_BLCKSZ;
			nbytes = npages * (Size) XLOG_BLCKSZ;
			nleft = nbytes;
			do
			{
				errno = 0;

				/*
				 * Measure I/O timing to write WAL data, for pg_stat_io.
				 */
				start = pgstat_prepare_io_time(track_wal_io_timing);

				pgstat_report_wait_start(WAIT_EVENT_WAL_WRITE);
				written = pg_pwrite(openLogFile, from, nleft, startoffset);
				pgstat_report_wait_end();
```

Segment dolduğunda dört iş birden olur — fsync, arşiv bildirimi, zaman damgası,
checkpoint gerekip gerekmediği kontrolü:

[../src/backend/access/transam/xlog.c:2498-2525](../src/backend/access/transam/xlog.c#L2498-L2525)

```c
			if (finishing_seg)
			{
				issue_xlog_fsync(openLogFile, openLogSegNo, tli);

				/* signal that we need to wakeup walsenders later */
				WalSndWakeupRequest();

				LogwrtResult.Flush = LogwrtResult.Write;	/* end of page */

				if (XLogArchivingActive())
					XLogArchiveNotifySeg(openLogSegNo, tli);

				XLogCtl->lastSegSwitchTime = (pg_time_t) time(NULL);
				XLogCtl->lastSegSwitchLSN = LogwrtResult.Flush;

				/*
				 * Request a checkpoint if we've consumed too much xlog since
				 * the last one. ...
				 */
				if (IsUnderPostmaster && XLogCheckpointNeeded(openLogSegNo))
				{
					(void) GetRedoRecPtr();
					if (XLogCheckpointNeeded(openLogSegNo))
						RequestCheckpoint(CHECKPOINT_CAUSE_XLOG);
				}
			}
```

`max_wal_size` tetiklemesi buradan geçiyor — yani WAL yazımının kendi içinden.

## 4.3 `XLogFlush` ve group commit

[../src/backend/access/transam/xlog.c:2793-2800](../src/backend/access/transam/xlog.c#L2793-L2800)

```c
/*
 * Ensure that all XLOG data through the given position is flushed to disk.
 *
 * NOTE: this differs from XLogWrite mainly in that the WALWriteLock is not
 * already held, and we try to avoid acquiring it if possible.
 */
void
XLogFlush(XLogRecPtr record)
```

İlk iki kontrol, işin çoğunu hiç yapmadan bitirir:

[../src/backend/access/transam/xlog.c:2813-2821](../src/backend/access/transam/xlog.c#L2813-L2821)

```c
	if (!XLogInsertAllowed())
	{
		UpdateMinRecoveryPoint(record, false);
		return;
	}

	/* Quick exit if already known flushed */
	if (record <= LogwrtResult.Flush)
		return;
```

Recovery sırasında WAL yazılmaz — bunun yerine `minRecoveryPoint` güncellenir.
Bu, standby'da "diske indirdiğim veri değişikliğinden geri gidemem" garantisidir.

Asıl group commit hilesi `LWLockAcquireOrWait`:

[../src/backend/access/transam/xlog.c:2867-2890](../src/backend/access/transam/xlog.c#L2867-L2890)

```c
		/*
		 * Try to get the write lock. If we can't get it immediately, wait
		 * until it's released, and recheck if we still need to do the flush
		 * or if the backend that held the lock did it for us already. This
		 * helps to maintain a good rate of group committing when the system
		 * is bottlenecked by the speed of fsyncing.
		 */
		if (!LWLockAcquireOrWait(WALWriteLock, LW_EXCLUSIVE))
		{
			/*
			 * The lock is now free, but we didn't acquire it yet. Before we
			 * do, loop back to check if someone else flushed the record for
			 * us already.
			 */
			continue;
		}

		/* Got the lock; recheck whether request is satisfied */
		RefreshXLogWriteResult(LogwrtResult);
		if (record <= LogwrtResult.Flush)
		{
			LWLockRelease(WALWriteLock);
			break;
		}
```

Yani: 100 backend aynı anda commit ediyorsa, biri fsync yapar, diğer 99'u
uyandıklarında kendi LSN'lerinin çoktan flush edildiğini görüp hiç fsync yapmadan
çıkar. Tek fsync, 100 commit.

`commit_delay` bu davranışı bilerek abartır — fsync'ten **önce** uyuyup daha çok
yolcu toplar:

[../src/backend/access/transam/xlog.c:2892-2906](../src/backend/access/transam/xlog.c#L2892-L2906)

```c
		/*
		 * Sleep before flush! By adding a delay here, we may give further
		 * backends the opportunity to join the backlog of group commit
		 * followers; this can significantly improve transaction throughput,
		 * at the risk of increasing transaction latency.
		 *
		 * We do not sleep if enableFsync is not turned on, nor if there are
		 * fewer than CommitSiblings other backends with active transactions.
		 */
		if (CommitDelay > 0 && enableFsync &&
			MinimumActiveBackends(CommitSiblings))
		{
			pgstat_report_wait_start(WAIT_EVENT_COMMIT_DELAY);
			pg_usleep(CommitDelay);
			pgstat_report_wait_end();
```

## 4.4 WAL kuralının uygulandığı tek nokta

Bütün "write-ahead" garantisi aslında `FlushBuffer` içindeki tek `if`:

[../src/backend/storage/buffer/bufmgr.c:4565-4585](../src/backend/storage/buffer/bufmgr.c#L4565-L4585)

```c
	recptr = BufferGetLSN(buf);

	/*
	 * Force XLOG flush up to buffer's LSN.  This implements the basic WAL
	 * rule that log updates must hit disk before any of the data-file changes
	 * they describe do.
	 *
	 * However, this rule does not apply to unlogged relations, which will be
	 * lost after a crash anyway. ...
	 */
	if (pg_atomic_read_u64(&buf->state) & BM_PERMANENT)
		XLogFlush(recptr);
```

Kirli sayfa diske inmeden hemen önce, o sayfanın LSN'ine kadar WAL flush edilir.
Bu kadar. Geri kalan her şey bu kuralı ucuza mal etmek için yapılmış optimizasyon.

## 4.5 `synchronous_commit` seviyeleri

Beş seviye var:

[../src/include/access/xact.h:69-81](../src/include/access/xact.h#L69-L81)

```c
typedef enum
{
	SYNCHRONOUS_COMMIT_OFF,		/* asynchronous commit */
	SYNCHRONOUS_COMMIT_LOCAL_FLUSH, /* wait for local flush only */
	SYNCHRONOUS_COMMIT_REMOTE_WRITE,	/* wait for local flush and remote
										 * write */
	SYNCHRONOUS_COMMIT_REMOTE_FLUSH,	/* wait for local and remote flush */
	SYNCHRONOUS_COMMIT_REMOTE_APPLY,	/* wait for local and remote flush and
										 * remote apply */
}			SyncCommitLevel;

/* Define the default setting for synchronous_commit */
#define SYNCHRONOUS_COMMIT_ON	SYNCHRONOUS_COMMIT_REMOTE_FLUSH
```

Karar `RecordTransactionCommit` içinde:

[../src/backend/access/transam/xact.c:1540-1573](../src/backend/access/transam/xact.c#L1540-L1573)

```c
	if ((wrote_xlog && markXidCommitted &&
		 synchronous_commit > SYNCHRONOUS_COMMIT_OFF) ||
		forceSyncCommit || nrels > 0)
	{
		XLogFlush(XactLastRecEnd);

		/*
		 * Now we may update the CLOG, if we wrote a COMMIT record above
		 */
		if (markXidCommitted)
			TransactionIdCommitTree(xid, nchildren, children);
	}
	else
	{
		/*
		 * Asynchronous commit case:
		 *
		 * This enables possible committed transaction loss in the case of a
		 * postmaster crash because WAL buffers are left unwritten. ...
		 */
		XLogSetAsyncXactLSN(XactLastRecEnd);

		/*
		 * We must not immediately update the CLOG, since we didn't flush the
		 * XLOG. Instead, we store the LSN up to which the XLOG must be
		 * flushed before the CLOG may be updated.
		 */
		if (markXidCommitted)
			TransactionIdAsyncCommitTree(xid, nchildren, children, XactLastRecEnd);
	}
```

Asenkron dalın inceliği: CLOG **hemen güncellenmez**. Güncellenseydi, çökme
sonrası "commit edilmiş" görünen ama WAL'ı diskte olmayan bir transaction
kalabilirdi. Bunun yerine CLOG girdisine "şu LSN flush edilmeden beni yazma"
notu iliştirilir.

Dikkat: `nrels > 0` (silinecek dosya var) veya `forceSyncCommit` durumunda
`synchronous_commit=off` olsa bile flush yapılır — dosya silinip commit kaydı
kaybolursa geri dönüşü yok.

## 4.6 WAL writer

Asenkron commit'lerin flush'ını arka planda yapan süreç:

[../src/backend/postmaster/walwriter.c:237-257](../src/backend/postmaster/walwriter.c#L237-L257)

```c
		/*
		 * Do what we're here for; then, if XLogBackgroundFlush() found useful
		 * work to do, reset hibernation counter.
		 */
		if (XLogBackgroundFlush())
			left_till_hibernate = LOOPS_UNTIL_HIBERNATE;
		else if (left_till_hibernate > 0)
			left_till_hibernate--;

		/* report pending statistics to the cumulative stats system */
		pgstat_report_wal(false);

		/*
		 * Sleep until we are signaled or WalWriterDelay has elapsed.  If we
		 * haven't done anything useful for quite some time, lengthen the
		 * sleep time so as to reduce the server's idle power consumption.
		 */
		if (left_till_hibernate > 0)
			cur_timeout = WalWriterDelay;	/* in ms */
		else
			cur_timeout = WalWriterDelay * HIBERNATE_FACTOR;
```

`XLogBackgroundFlush` önce tam sayfa sınırına yuvarlar, sonra asenkron commit
LSN'ine bakar:

[../src/backend/access/transam/xlog.c:3027-3038](../src/backend/access/transam/xlog.c#L3027-L3038)

```c
	/* back off to last completed page boundary */
	WriteRqst.Write -= WriteRqst.Write % XLOG_BLCKSZ;

	/* if we have already flushed that far, consider async commit records */
	RefreshXLogWriteResult(LogwrtResult);
	if (WriteRqst.Write <= LogwrtResult.Flush)
	{
		SpinLockAcquire(&XLogCtl->info_lck);
		WriteRqst.Write = XLogCtl->asyncXactLSN;
		SpinLockRelease(&XLogCtl->info_lck);
		flexible = false;		/* ensure it all gets written */
	}
```

Pratik sonuç: `synchronous_commit=off` ile veri kaybı penceresi kabaca
`wal_writer_delay` (varsayılan 200 ms) mertebesindedir — sonsuz değil.

---

# 5. Segment dosyaları, `pg_wal` düzeni, arşivleme

## 5.1 Sayfa başlığı

WAL dosyaları 8 kB'lik sayfalardan oluşur ve her sayfanın başlığı vardır:

[../src/include/access/xlog_internal.h:35-51](../src/include/access/xlog_internal.h#L35-L51)

```c
#define XLOG_PAGE_MAGIC 0xD121	/* can be used as WAL version indicator */

typedef struct XLogPageHeaderData
{
	uint16		xlp_magic;		/* magic value for correctness checks */
	uint16		xlp_info;		/* flag bits, see below */
	TimeLineID	xlp_tli;		/* TimeLineID of first record on page */
	XLogRecPtr	xlp_pageaddr;	/* XLOG address of this page */

	/*
	 * When there is not enough space on current page for whole record, we
	 * continue on the next page.  xlp_rem_len is the number of bytes
	 * remaining from a previous page; it tracks xl_tot_len in the initial
	 * header.  Note that the continuation data isn't necessarily aligned.
	 */
	uint32		xlp_rem_len;	/* total len of remaining data for record */
} XLogPageHeaderData;
```

Her segmentin **ilk** sayfası uzun başlık taşır — dosyanın hangi cluster'a ait
olduğunu doğrulamak için:

[../src/include/access/xlog_internal.h:62-68](../src/include/access/xlog_internal.h#L62-L68)

```c
typedef struct XLogLongPageHeaderData
{
	XLogPageHeaderData std;		/* standard header fields */
	uint64		xlp_sysid;		/* system identifier from pg_control */
	uint32		xlp_seg_size;	/* just as a cross-check */
	uint32		xlp_xlog_blcksz;	/* just as a cross-check */
} XLogLongPageHeaderData;
```

`xlp_sysid`, yanlış cluster'ın WAL dosyalarını yanlışlıkla replay etmeyi engeller.

## 5.2 Segment boyu ve dosya adı

[../src/include/access/xlog_internal.h:86-97](../src/include/access/xlog_internal.h#L86-L97)

```c
/* wal_segment_size can range from 1MB to 1GB */
#define WalSegMinSize 1024 * 1024
#define WalSegMaxSize 1024 * 1024 * 1024
/* default number of min and max wal segments */
#define DEFAULT_MIN_WAL_SEGS 5
#define DEFAULT_MAX_WAL_SEGS 64

/* check that the given size is a valid wal_segment_size */
#define IsPowerOf2(x) (x > 0 && ((x) & ((x)-1)) == 0)
#define IsValidWalSegSize(size) \
	 (IsPowerOf2(size) && \
	 ((size) >= WalSegMinSize && (size) <= WalSegMaxSize))
```

`wal_segment_size` **initdb zamanında** sabitlenir (`initdb --wal-segsize`), sonra
değiştirilemez ([../src/bin/initdb/initdb.c:3383](../src/bin/initdb/initdb.c#L3383)).
Varsayılan 16 MB.

Dosya adı 24 hex karakter — 8 timeline + 8 log + 8 segment:

[../src/include/access/xlog_internal.h:164-170](../src/include/access/xlog_internal.h#L164-L170)

```c
static inline void
XLogFileName(char *fname, TimeLineID tli, XLogSegNo logSegNo, int wal_segsz_bytes)
{
	snprintf(fname, MAXFNAMELEN, "%08X%08X%08X", tli,
			 (uint32) (logSegNo / XLogSegmentsPerXLogId(wal_segsz_bytes)),
			 (uint32) (logSegNo % XLogSegmentsPerXLogId(wal_segsz_bytes)));
}
```

Böylece dosyalar **alfabetik sıralandığında kronolojik** olur — `ls` çıktısı
doğru sırada gelir. Bu tesadüf değil, tasarım.

## 5.3 `pg_wal` düzeni

```
$PGDATA/
├── global/pg_control              ← kurtarmanın başlangıç noktası
└── pg_wal/
    ├── 000000010000000000000001   ← segment dosyaları (24 hex, varsayılan 16 MB)
    ├── 000000010000000000000002
    ├── 00000002.history           ← timeline geçmişi (TLI 1 dışında)
    ├── archive_status/
    │   ├── 000000010000000000000001.ready   ← arşivlenmeyi bekliyor
    │   └── 000000010000000000000002.done    ← arşivlendi
    └── summaries/                 ← WAL summarizer çıktısı (artımlı yedek için)
```

Dizin adları ve doğrulama:
[../src/include/access/xlog_internal.h:148-149](../src/include/access/xlog_internal.h#L148-L149),
[../src/backend/access/transam/xlog.c:4155-4185](../src/backend/access/transam/xlog.c#L4155-L4185).

## 5.4 Yeni dosya sıfırla dolduruluyor

[../src/backend/access/transam/xlog.c:3300-3313](../src/backend/access/transam/xlog.c#L3300-L3313)

`wal_init_zero` açıkken (varsayılan) yeni segment `pg_pwrite_zeros` ile baştan
sona sıfırlanır. Amaç dosya sistemine alanı **önceden ayırttırmak**: WAL yazarken
"disk dolu" hatası almak PANIC demektir.

## 5.5 Geri dönüştürme (recycle)

Eski segmentler silinmez, **yeniden adlandırılır**:

[../src/backend/access/transam/xlog.c:4071-4089](../src/backend/access/transam/xlog.c#L4071-L4089)

```c
	/*
	 * Before deleting the file, see if it can be recycled as a future log
	 * segment. Only recycle normal files, because we don't want to recycle
	 * symbolic links pointing to a separate archive directory.
	 */
	if (wal_recycle &&
		*endlogSegNo <= recycleSegNo &&
		XLogCtl->InstallXLogFileSegmentActive &&	/* callee rechecks this */
		get_dirent_type(path, segment_de, false, DEBUG2) == PGFILETYPE_REG &&
		InstallXLogFileSegment(endlogSegNo, path,
							   true, recycleSegNo, insertTLI))
	{
		ereport(DEBUG2,
				(errmsg_internal("recycled write-ahead log file \"%s\"",
								 segname)));
		CheckpointStats.ckpt_segs_recycled++;
```

`min_wal_size` bu havuzun alt sınırıdır: o kadar segment her zaman hazır bekletilir,
böylece yoğunluk arttığında yeni dosya oluşturma maliyeti ödenmez.

## 5.6 Arşivleme

Arşivleme bir dosya adı üzerinden mesajlaşmaya dayanıyor:

[../src/backend/access/transam/xlogarchive.c:435-451](../src/backend/access/transam/xlogarchive.c#L435-L451)

```c
/*
 * XLogArchiveNotify
 *
 * Create an archive notification file
 *
 * The name of the notification file is the message that will be picked up
 * by the archiver, e.g. we write 0000000100000001000000C6.ready
 * and the archiver then knows to archive XLOGDIR/0000000100000001000000C6,
 * then when complete, rename it to 0000000100000001000000C6.done
 */
void
XLogArchiveNotify(const char *xlog)
{
	char		archiveStatusPath[MAXPGPATH];
	FILE	   *fd;

	/* insert an otherwise empty file called <XLOG>.ready */
	StatusFilePath(archiveStatusPath, xlog, ".ready");
```

Timeline history dosyaları öncelikli — promosyon yarışlarını önlemek için:

[../src/backend/access/transam/xlogarchive.c:470-486](../src/backend/access/transam/xlogarchive.c#L470-L486)

```c
	/*
	 * Timeline history files are given the highest archival priority to lower
	 * the chance that a promoted standby will choose a timeline that is
	 * already in use. ...
	 */
	if (IsTLHistoryFileName(xlog))
		PgArchForceDirScan();

	/* Notify archiver that it's got something to do */
	if (IsUnderPostmaster)
		PgArchWakeup();
```

Kritik nokta: `.ready` dosyası varken ilgili segment **silinemez**. Arşiv komutu
sürekli başarısız oluyorsa `pg_wal` şişer ve disk dolar — arşivlemenin klasik
operasyonel tuzağı budur.

`wal_level` en az `replica` olmalı:

[../src/include/access/xlog.h:112](../src/include/access/xlog.h#L112)

```c
#define XLogIsNeeded() (wal_level >= WAL_LEVEL_REPLICA)
```

---

# 6. Checkpoint

## 6.1 Ne işe yarar

Checkpoint iki şeyi aynı anda yapar:

1. O ana kadarki tüm kirli buffer'ları diske indirir.
2. WAL'da bir **redo noktası** işaretler ve bunu `pg_control`'e yazar.

Sonuç: bu noktadan öncesindeki WAL artık gereksizdir (crash recovery redo
noktasından başlar). Yani checkpoint hem **kurtarma süresini** hem de **pg_wal
boyutunu** sınırlar.

## 6.2 Ne zaman tetiklenir

Üç yol:

**(a) Zaman** — `checkpoint_timeout` (varsayılan 5 dk):

[../src/backend/postmaster/checkpointer.c:399-413](../src/backend/postmaster/checkpointer.c#L399-L413)

```c
		/*
		 * Force a checkpoint if too much time has elapsed since the last one.
		 * Note that we count a timed checkpoint in stats only when this
		 * occurs without an external request, but we set the CAUSE_TIME flag
		 * bit even if there is also an external request.
		 */
		now = (pg_time_t) time(NULL);
		elapsed_secs = now - last_checkpoint_time;
		if (elapsed_secs >= CheckPointTimeout)
		{
			if (!do_checkpoint)
				chkpt_or_rstpt_timed = true;
			do_checkpoint = true;
			flags |= CHECKPOINT_CAUSE_TIME;
		}
```

**(b) WAL hacmi** — `max_wal_size`. Hedef segment sayısı şöyle hesaplanır:

[../src/backend/access/transam/xlog.c:2196-2217](../src/backend/access/transam/xlog.c#L2196-L2217)

```c
	/*-------
	 * Calculate the distance at which to trigger a checkpoint, to avoid
	 * exceeding max_wal_size_mb. This is based on two assumptions:
	 *
	 * a) we keep WAL for only one checkpoint cycle ...
	 * b) during checkpoint, we consume checkpoint_completion_target *
	 *	  number of segments consumed between checkpoints.
	 *-------
	 */
	target = (double) ConvertToXSegs(max_wal_size_mb, wal_segment_size) /
		(1.0 + CheckPointCompletionTarget);

	/* round down */
	CheckPointSegments = (int) target;

	if (CheckPointSegments < 1)
		CheckPointSegments = 1;
```

Kontrol her segment dolduğunda yapılır:

[../src/backend/access/transam/xlog.c:2300-2310](../src/backend/access/transam/xlog.c#L2300-L2310)

```c
bool
XLogCheckpointNeeded(XLogSegNo new_segno)
{
	XLogSegNo	old_segno;

	XLByteToSeg(RedoRecPtr, old_segno, wal_segment_size);

	if (new_segno >= old_segno + (uint64) (CheckPointSegments - 1))
		return true;
	return false;
}
```

**(c) Açık istek** — `CHECKPOINT` komutu, shutdown, `pg_basebackup` vb.

## 6.3 Boşta atlama

Hiç WAL yazılmamışsa checkpoint yazmak boşa yer harcamaktır:

[../src/backend/access/transam/xlog.c:7482-7497](../src/backend/access/transam/xlog.c#L7482-L7497)

```c
	/*
	 * If this isn't a shutdown or forced checkpoint, and if there has been no
	 * WAL activity requiring a checkpoint, skip it.  The idea here is to
	 * avoid inserting duplicate checkpoints when the system is idle.
	 */
	if ((flags & (CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_END_OF_RECOVERY |
				  CHECKPOINT_FORCE)) == 0)
	{
		if (last_important_lsn == ControlFile->checkPoint)
		{
			END_CRIT_SECTION();
			ereport(DEBUG1,
					(errmsg_internal("checkpoint skipped because system is idle")));
			return false;
		}
	}
```

## 6.4 Redo noktası nasıl seçilir

Burası checkpoint'in en ince yeri. **Shutdown checkpoint**'te eşzamanlı yazan
kimse olmadığı için redo noktası doğrudan mevcut insert konumudur:

[../src/backend/access/transam/xlog.c:7529-7561](../src/backend/access/transam/xlog.c#L7529-L7561)

```c
	if (shutdown)
	{
		XLogRecPtr	curInsert = XLogBytePosToRecPtr(Insert->CurrBytePos);

		/*
		 * Compute new REDO record ptr = location of next XLOG record.
		 *
		 * Since this is a shutdown checkpoint, there can't be any concurrent
		 * WAL insertion.
		 */
		...
		checkPoint.redo = curInsert;

		/*
		 * Here we update the shared RedoRecPtr for future XLogInsert calls;
		 * this must be done while holding all the insertion locks.
		 *
		 * Note: if we fail to complete the checkpoint, RedoRecPtr will be
		 * left pointing past where it really needs to point.  This is okay;
		 * the only consequence is that XLogInsert might back up whole buffers
		 * that it didn't really need to. ...
		 */
		RedoRecPtr = XLogCtl->Insert.RedoRecPtr = checkPoint.redo;
	}
```

**Online checkpoint**'te ise özel bir kayıt yazılır ve o kaydın başlangıç LSN'i
redo noktası olur:

[../src/backend/access/transam/xlog.c:7570-7602](../src/backend/access/transam/xlog.c#L7570-L7602)

```c
	/*
	 * If this is an online checkpoint, we have not yet determined the redo
	 * point. We do so now by inserting the special XLOG_CHECKPOINT_REDO
	 * record; the LSN at which it starts becomes the new redo pointer. We
	 * don't do this for a shutdown checkpoint, because in that case no WAL
	 * can be written between the redo point and the insertion of the
	 * checkpoint record itself, so the checkpoint record itself serves to
	 * mark the redo point.
	 */
	if (!shutdown)
	{
		xl_checkpoint_redo redo_rec;
		...
		XLogBeginInsert();
		XLogRegisterData(&redo_rec, sizeof(xl_checkpoint_redo));
		(void) XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT_REDO);

		/*
		 * XLogInsertRecord will have updated XLogCtl->Insert.RedoRecPtr in
		 * shared memory and RedoRecPtr in backend-local memory, but we need
		 * to copy that into the record that will be inserted when the
		 * checkpoint is complete.
		 */
		checkPoint.redo = RedoRecPtr;
	}
```

Bu ayrım kritik: online checkpoint'te **redo noktası ile checkpoint kaydı
arasında dakikalarca WAL yazılabilir**. Redo noktası "checkpoint'in başladığı an",
checkpoint kaydı ise "bittiği an"dır. Kurtarma redo noktasından başlar, kayıttan
değil.

## 6.5 Asıl iş: `CheckPointGuts`

[../src/backend/access/transam/xlog.c:8055-8082](../src/backend/access/transam/xlog.c#L8055-L8082)

```c
	TRACE_POSTGRESQL_BUFFER_CHECKPOINT_START(flags);
	CheckpointStats.ckpt_write_t = GetCurrentTimestamp();
	CheckPointCLOG();
	CheckPointCommitTs();
	CheckPointSUBTRANS();
	CheckPointMultiXact();
	CheckPointPredicate();
	CheckPointBuffers(flags);

	/* Perform all queued up fsyncs */
	TRACE_POSTGRESQL_BUFFER_CHECKPOINT_SYNC_START();
	CheckpointStats.ckpt_sync_t = GetCurrentTimestamp();
	ProcessSyncRequests();
	CheckpointStats.ckpt_sync_end_t = GetCurrentTimestamp();
	TRACE_POSTGRESQL_BUFFER_CHECKPOINT_DONE();
	...
	CheckPointReplicationSlots(flags & CHECKPOINT_IS_SHUTDOWN);
	CheckPointSnapBuild();
	CheckPointLogicalRewriteHeap();
	CheckPointTwoPhase(checkPointRedo);
```

`CheckPointBuffers` shared_buffers'ın tamamını tarayıp kirli olanları yazar.
`shared_buffers = 32GB` ise bu on binlerce sayfa demek.

## 6.6 Neden uzun sürer — ve kasıtlı yavaşlatma

Checkpoint I/O tsunamisi yaratmasın diye **bilerek** yavaşlatılır:

[../src/backend/postmaster/checkpointer.c:782-811](../src/backend/postmaster/checkpointer.c#L782-L811)

```c
/*
 * CheckpointWriteDelay -- control rate of checkpoint
 *
 * This function is called after each page write performed by BufferSync().
 * It is responsible for throttling BufferSync()'s write rate to hit
 * checkpoint_completion_target.
 * ...
 */
void
CheckpointWriteDelay(int flags, double progress)
{
	static int	absorb_counter = WRITES_PER_ABSORB;

	/* Do nothing if checkpoint is being executed by non-checkpointer process */
	if (!AmCheckpointerProcess())
		return;

	/*
	 * Perform the usual duties and take a nap, unless we're behind schedule,
	 * in which case we just try to catch up as quickly as possible.
	 */
	if (!(flags & CHECKPOINT_FAST) &&
		...
		IsCheckpointOnSchedule(progress))
```

`checkpoint_completion_target` (varsayılan 0.9) demek ki: checkpoint, bir sonraki
checkpoint'e kadarki sürenin %90'ına yayılsın. 5 dakikalık `checkpoint_timeout`
ile checkpoint'in ~4.5 dakika sürmesi **normaldir** — yavaş değil, kasıtlı.

Checkpoint bittiğinde `pg_control` güncellenir ve eski segmentler temizlenir:

[../src/backend/access/transam/xlog.c:7788-7805](../src/backend/access/transam/xlog.c#L7788-L7805)

```c
	LWLockAcquire(ControlFileLock, LW_EXCLUSIVE);
	if (shutdown)
		ControlFile->state = DB_SHUTDOWNED;
	ControlFile->checkPoint = ProcLastRecPtr;
	ControlFile->checkPointCopy = checkPoint;
	/* crash recovery should always recover to the end of WAL */
	ControlFile->minRecoveryPoint = InvalidXLogRecPtr;
	ControlFile->minRecoveryPointTLI = 0;
	...
	UpdateControlFile();
	LWLockRelease(ControlFileLock);
```

[../src/backend/access/transam/xlog.c:7846-7872](../src/backend/access/transam/xlog.c#L7846-L7872)

```c
	/*
	 * Delete old log files, those no longer needed for last checkpoint to
	 * prevent the disk holding the xlog from growing full.
	 */
	XLByteToSeg(RedoRecPtr, _logSegNo, wal_segment_size);
	KeepLogSeg(recptr, &_logSegNo);
	...
	_logSegNo--;
	RemoveOldXlogFiles(_logSegNo, RedoRecPtr, recptr,
					   checkPoint.ThisTimeLineID);

	/*
	 * Make more log segments if needed.  (Do this after recycling old log
	 * segments, since that may supply some of the needed files.)
	 */
	if (!shutdown)
		PreallocXlogFiles(recptr, checkPoint.ThisTimeLineID);
```

`KeepLogSeg` replication slot'ları ve `wal_keep_size`'ı hesaba katar — slot geride
kalmışsa segmentler silinmez. Terk edilmiş bir replication slot'un `pg_wal`'ı
şişirmesinin sebebi budur.

---

# 7. Kurtarma

## 7.1 `pg_control`: başlangıç noktası

Kurtarma tek bir dosyadan başlar: `global/pg_control`.

[../src/include/catalog/pg_control.h:112-145](../src/include/catalog/pg_control.h#L112-L145)

```c
typedef struct ControlFileData
{
	/*
	 * Unique system identifier --- to ensure we match up xlog files with the
	 * installation that produced them.
	 */
	uint64		system_identifier;
	...
	uint32		pg_control_version; /* PG_CONTROL_VERSION */
	uint32		catalog_version_no; /* see catversion.h */

	/*
	 * System status data
	 */
	DBState		state;			/* see enum above */
	pg_time_t	time;			/* time stamp of last pg_control update */
	XLogRecPtr	checkPoint;		/* last check point record ptr */

	CheckPoint	checkPointCopy; /* copy of last check point record */
```

Son checkpoint kaydının **tam kopyası** burada tutulur — WAL okunabilir hale
gelmeden önce nextXid, oldestXid, timeline gibi değerlere ihtiyaç var.

`minRecoveryPoint` ise arşiv kurtarmasının güvenlik kilidi:

[../src/include/catalog/pg_control.h:147-177](../src/include/catalog/pg_control.h#L147-L177)

```c
	/*
	 * These two values determine the minimum point we must recover up to
	 * before starting up:
	 *
	 * minRecoveryPoint is updated to the latest replayed LSN whenever we
	 * flush a data change during archive recovery. That guards against
	 * starting archive recovery, aborting it, and restarting with an earlier
	 * stop location. If we've already flushed data changes from WAL record X
	 * to disk, we mustn't start up until we reach X again. Zero when not
	 * doing archive recovery.
	 * ...
	 */
	XLogRecPtr	minRecoveryPoint;
	TimeLineID	minRecoveryPointTLI;
```

## 7.2 `StartupXLOG`

[../src/backend/access/transam/xlog.c:5845](../src/backend/access/transam/xlog.c#L5845)

İlk iş cluster'ın nasıl kapandığını anlamak:

[../src/backend/access/transam/xlog.c:5881-5932](../src/backend/access/transam/xlog.c#L5881-L5932)

```c
	switch (ControlFile->state)
	{
		case DB_SHUTDOWNED:

			/*
			 * This is the expected case, so don't be chatty in standalone
			 * mode
			 */
			ereport(IsPostmasterEnvironment ? LOG : NOTICE,
					(errmsg("database system was shut down at %s",
							str_time(ControlFile->time,
									 timebuf, sizeof(timebuf)))));
			break;
		...
		case DB_IN_PRODUCTION:
			ereport(LOG,
					(errmsg("database system was interrupted; last known up at %s",
							str_time(ControlFile->time,
									 timebuf, sizeof(timebuf)))));
			break;
```

`DB_IN_PRODUCTION` görüyorsanız: çökme olmuş, redo gerekiyor. Log'daki bu satır
crash recovery'nin ilk işaretidir.

Sonra checkpoint bulunur ve paylaşılan durum ondan doldurulur:

[../src/backend/access/transam/xlog.c:5982-6003](../src/backend/access/transam/xlog.c#L5982-L6003)

```c
	/*
	 * Prepare for WAL recovery if needed.
	 *
	 * InitWalRecovery analyzes the control file and the backup label file, if
	 * any.  It updates the in-memory ControlFile buffer according to the
	 * starting checkpoint, and sets InRecovery and ArchiveRecoveryRequested.
	 * It also applies the tablespace map file, if any.
	 */
	InitWalRecovery(ControlFile, &wasShutdown,
					&haveBackupLabel, &haveTblspcMap);
	checkPoint = ControlFile->checkPointCopy;

	/* initialize shared memory variables from the checkpoint record */
	TransamVariables->nextXid = checkPoint.nextXid;
	TransamVariables->nextOid = checkPoint.nextOid;
	TransamVariables->oidCount = 0;
	MultiXactSetNextMXact(checkPoint.nextMulti, checkPoint.nextMultiOffset);
	AdvanceOldestClogXid(checkPoint.oldestXid);
	SetTransactionIdLimit(checkPoint.oldestXid, checkPoint.oldestXidDB);
	SetMultiXactIdLimit(checkPoint.oldestMulti, checkPoint.oldestMultiDB);
	SetCommitTsLimit(checkPoint.oldestCommitTsXid,
					 checkPoint.newestCommitTsXid);
```

`backup_label` dosyası varsa başlangıç noktası `pg_control` yerine oradan gelir —
base backup'tan kurtarmanın çalışma şekli budur.

Sonra replay:

[../src/backend/access/transam/xlog.c:6273-6289](../src/backend/access/transam/xlog.c#L6273-L6289)

```c
		/*
		 * We're all set for replaying the WAL now. Do it.
		 */
		PerformWalRecovery();
		performedWalRecovery = true;
	}
	else
		performedWalRecovery = false;

	/*
	 * Finish WAL recovery.
	 */
	endOfRecoveryInfo = FinishWalRecovery();
	EndOfLog = endOfRecoveryInfo->endOfLog;
	EndOfLogTLI = endOfRecoveryInfo->endOfLogTLI;
```

## 7.3 `PerformWalRecovery`: redo döngüsü

[../src/backend/access/transam/xlogrecovery.c:1617](../src/backend/access/transam/xlogrecovery.c#L1617)

Başlangıç noktası seçimi, online checkpoint'in iki-noktalı yapısını yansıtıyor:

[../src/backend/access/transam/xlogrecovery.c:1663-1691](../src/backend/access/transam/xlogrecovery.c#L1663-L1691)

```c
	/*
	 * Find the first record that logically follows the checkpoint --- it
	 * might physically precede it, though.
	 */
	if (RedoStartLSN < CheckPointLoc)
	{
		/* back up to find the record */
		replayTLI = RedoStartTLI;
		XLogPrefetcherBeginRead(xlogprefetcher, RedoStartLSN);
		record = ReadRecord(xlogprefetcher, PANIC, false, replayTLI);

		/*
		 * If a checkpoint record's redo pointer points back to an earlier
		 * LSN, the record at that LSN should be an XLOG_CHECKPOINT_REDO
		 * record.
		 */
		if (record->xl_rmid != RM_XLOG_ID ||
			(record->xl_info & ~XLR_INFO_MASK) != XLOG_CHECKPOINT_REDO)
			ereport(FATAL,
					errmsg("unexpected record type found at redo point %X/%08X",
						   LSN_FORMAT_ARGS(xlogreader->ReadRecPtr)));
	}
```

"Mantıksal olarak checkpoint'ten sonra ama fiziksel olarak öncesinde" — online
checkpoint'te redo noktası, checkpoint kaydından **geride** olduğu için.

Ana döngü:

[../src/backend/access/transam/xlogrecovery.c:1715-1811](../src/backend/access/transam/xlogrecovery.c#L1715-L1811)

```c
		do
		{
			...
			/* Handle interrupt signals of startup process */
			ProcessStartupProcInterrupts();
			...
			/*
			 * Have we reached our recovery target?
			 */
			if (recoveryStopsBefore(xlogreader))
			{
				reachedRecoveryTarget = true;
				break;
			}
			...
			/*
			 * Apply the record
			 */
			ApplyWalRecord(xlogreader, record, &replayTLI);
			...
			/* Else, try to fetch the next WAL record */
			record = ReadRecord(xlogprefetcher, LOG, false, replayTLI);
		} while (record != NULL);
```

Döngü `record == NULL` olduğunda biter — yani WAL'ın sonuna gelindiğinde ya da
bozuk kayda rastlandığında (aynı şey; bkz. 7.5).

## 7.4 Dispatch: `rm_redo`

[../src/backend/access/transam/xlogrecovery.c:1963-1971](../src/backend/access/transam/xlogrecovery.c#L1963-L1971)

```c
	/*
	 * Some XLOG record types that are related to recovery are processed
	 * directly here, rather than in xlog_redo()
	 */
	if (record->xl_rmid == RM_XLOG_ID)
		xlogrecovery_redo(xlogreader, *replayTLI);

	/* Now apply the WAL record itself */
	GetRmgr(record->xl_rmid).rm_redo(xlogreader);
```

Tek satırlık tablo indeksleme. `xl_rmid` → rmgr → `rm_redo`. Her AM kendi redo
kodunu kendi getirir; WAL altyapısı içeriği hiç bilmez.

Örnek: heap rmgr'nin ikinci seviye dispatch'i

[../src/backend/access/heap/heapam_xlog.c:1276-1296](../src/backend/access/heap/heapam_xlog.c#L1276-L1296)

```c
	switch (info & XLOG_HEAP_OPMASK)
	{
		case XLOG_HEAP_INSERT:
			heap_xlog_insert(record);
			break;
		case XLOG_HEAP_DELETE:
			heap_xlog_delete(record);
			break;
		case XLOG_HEAP_UPDATE:
			heap_xlog_update(record, false);
			break;
		...
		case XLOG_HEAP_HOT_UPDATE:
			heap_xlog_update(record, true);
			break;
```

## 7.5 İdempotency: `XLogReadBufferForRedo`

Her redo rutini sayfayı bu fonksiyondan alır ve dönüş değerine göre davranır:

[../src/backend/access/transam/xlogutils.c:277-284](../src/backend/access/transam/xlogutils.c#L277-L284)

```c
 * Returns one of the following:
 *
 *	BLK_NEEDS_REDO	- changes from the WAL record need to be applied
 *	BLK_DONE		- block doesn't need replaying
 *	BLK_RESTORED	- block was restored from a full-page image included in
 *					  the record
 *	BLK_NOTFOUND	- block was not found (because it was truncated away by
 *					  an operation later in the WAL stream)
```

İki dallı mantık. Önce FPI yolu:

[../src/backend/access/transam/xlogutils.c:395-430](../src/backend/access/transam/xlogutils.c#L395-L430)

```c
	/* If it has a full-page image and it should be restored, do it. */
	if (XLogRecBlockImageApply(record, block_id))
	{
		Assert(XLogRecHasBlockImage(record, block_id));
		*buf = XLogReadBufferExtended(rlocator, forknum, blkno,
									  get_cleanup_lock ? RBM_ZERO_AND_CLEANUP_LOCK : RBM_ZERO_AND_LOCK,
									  prefetch_buffer);
		page = BufferGetPage(*buf);
		if (!RestoreBlockImage(record, block_id, page))
			ereport(ERROR, ...);

		/*
		 * The page may be uninitialized. If so, we can't set the LSN because
		 * that would corrupt the page.
		 */
		if (!PageIsNew(page))
		{
			PageSetLSN(page, lsn);
		}

		MarkBufferDirty(*buf);
		...
		return BLK_RESTORED;
	}
```

Dikkat: `RBM_ZERO_AND_LOCK` — diskteki sayfa **hiç okunmaz**. Zaten üzerine
yazılacak. Bu, FPI'yi hem güvenli hem de hızlı yapar.

FPI yoksa LSN karşılaştırması:

[../src/backend/access/transam/xlogutils.c:432-451](../src/backend/access/transam/xlogutils.c#L432-L451)

```c
	else
	{
		*buf = XLogReadBufferExtended(rlocator, forknum, blkno, mode, prefetch_buffer);
		if (BufferIsValid(*buf))
		{
			if (mode != RBM_ZERO_AND_LOCK && mode != RBM_ZERO_AND_CLEANUP_LOCK)
			{
				if (get_cleanup_lock)
					LockBufferForCleanup(*buf);
				else
					LockBuffer(*buf, BUFFER_LOCK_EXCLUSIVE);
			}
			if (lsn <= PageGetLSN(BufferGetPage(*buf)))
				return BLK_DONE;
			else
				return BLK_NEEDS_REDO;
		}
		else
			return BLK_NOTFOUND;
	}
```

`lsn <= PageGetLSN(...)` → `BLK_DONE`. **Bütün idempotency bu satırda.**

Neden gerekli? Çünkü checkpoint kirli sayfaları diske indirir ama redo noktasından
sonraki WAL kayıtlarının bir kısmının etkisi zaten diske inmiş olabilir. Redo
noktasından itibaren her şeyi körü körüne yeniden uygulamak yanlış olurdu; LSN
karşılaştırması hangi kaydın gereksiz olduğunu söyler.

Bir redo rutini bu sözleşmeyi böyle kullanır:

[../src/backend/access/heap/heapam_xlog.c:443-453](../src/backend/access/heap/heapam_xlog.c#L443-L453)

```c
	if (XLogRecGetInfo(record) & XLOG_HEAP_INIT_PAGE)
	{
		buffer = XLogInitBufferForRedo(record, HEAP_INSERT_BLKREF_HEAP);
		page = BufferGetPage(buffer);
		PageInit(page, BufferGetPageSize(buffer), 0);
		action = BLK_NEEDS_REDO;
	}
	else
		action = XLogReadBufferForRedo(record, HEAP_INSERT_BLKREF_HEAP,
									   &buffer);
	if (action == BLK_NEEDS_REDO)
	{
```

ve iş bitince mutlaka LSN damgalar:

[../src/backend/access/heap/heapam_xlog.c:484-497](../src/backend/access/heap/heapam_xlog.c#L484-L497)

```c
		if (PageAddItem(page, htup, newlen, xlrec->offnum, true, true) == InvalidOffsetNumber)
			elog(PANIC, "failed to add tuple");
		...
		PageSetLSN(page, lsn);
```

## 7.6 WAL'ın sonu nasıl anlaşılır

Crash recovery'de WAL dosyasının sonu genellikle **çöp** içerir (geri
dönüştürülmüş segmentin eski içeriği). İki savunma var.

CRC:

[../src/backend/access/transam/xlogreader.c:1233-1253](../src/backend/access/transam/xlogreader.c#L1233-L1253)

```c
static bool
ValidXLogRecord(XLogReaderState *state, XLogRecord *record, XLogRecPtr recptr)
{
	pg_crc32c	crc;

	Assert(record->xl_tot_len >= SizeOfXLogRecord);

	/* Calculate the CRC */
	INIT_CRC32C(crc);
	COMP_CRC32C(crc, ((char *) record) + SizeOfXLogRecord, record->xl_tot_len - SizeOfXLogRecord);
	/* include the record header last */
	COMP_CRC32C(crc, (char *) record, offsetof(XLogRecord, xl_crc));
	FIN_CRC32C(crc);

	if (!EQ_CRC32C(record->xl_crc, crc))
	{
		report_invalid_record(state,
							  "incorrect resource manager data checksum in record at %X/%08X",
							  LSN_FORMAT_ARGS(recptr));
		return false;
	}
```

ve `xl_prev` zinciri:

[../src/backend/access/transam/xlogreader.c:1202-1217](../src/backend/access/transam/xlogreader.c#L1202-L1217)

```c
	else
	{
		/*
		 * Record's prev-link should exactly match our previous location. This
		 * check guards against torn WAL pages where a stale but valid-looking
		 * WAL record starts on a sector boundary.
		 */
		if (record->xl_prev != PrevRecPtr)
		{
			report_invalid_record(state,
								  "record with incorrect prev-link %X/%08X at %X/%08X",
								  LSN_FORMAT_ARGS(record->xl_prev),
								  LSN_FORMAT_ARGS(RecPtr));
			return false;
		}
	}
```

`xl_prev`'in asıl varlık sebebi bu: geri dönüştürülmüş segmentte eski bir kayıt
CRC'siyle birlikte sağlam durabilir. Ama prev-link'i bizim geldiğimiz yeri
göstermez — yakalanır.

## 7.7 Tutarlılık noktası

Standby'da sorgular ancak "tutarlı" noktadan sonra kabul edilir:

[../src/backend/access/transam/xlogrecovery.c:2203-2231](../src/backend/access/transam/xlogrecovery.c#L2203-L2231)

```c
	/*
	 * Have we passed our safe starting point? Note that minRecoveryPoint is
	 * known to be incorrectly set if recovering from a backup, until the
	 * XLOG_BACKUP_END arrives to advise us of the correct minRecoveryPoint.
	 * All we know prior to that is that we're not consistent yet.
	 */
	if (!reachedConsistency && !backupEndRequired &&
		minRecoveryPoint <= lastReplayedEndRecPtr)
	{
		/*
		 * Check to see if the XLOG sequence contained any unresolved
		 * references to uninitialized pages.
		 */
		XLogCheckInvalidPages();
		...
		reachedConsistency = true;
		SendPostmasterSignal(PMSIGNAL_RECOVERY_CONSISTENT);
		ereport(LOG,
				errmsg("consistent recovery state reached at %X/%08X",
					   LSN_FORMAT_ARGS(lastReplayedEndRecPtr)));
	}
```

Log'daki `consistent recovery state reached` satırı, hot standby'ın bağlantı kabul
etmeye başladığı andır.

## 7.8 Kurtarma sonu

[../src/backend/access/transam/xlog.c:6790-6823](../src/backend/access/transam/xlog.c#L6790-L6823)

```c
	 * Perform a checkpoint to update all our recovery activity to disk.
	 *
	 * Note that we write a shutdown checkpoint rather than an on-line one.
	 * This is not particularly critical, but since we may be assigning a new
	 * TLI, using a shutdown checkpoint allows us to have the rule that TLI
	 * only changes in shutdown checkpoints, which allows some extra error
	 * checking in xlog_redo.
	 *
	 * In promotion, only create a lightweight end-of-recovery record instead
	 * of a full checkpoint. ...
	 */
	if (ArchiveRecoveryRequested && IsUnderPostmaster &&
		PromoteIsTriggered())
	{
		promoted = true;
		...
		CreateEndOfRecoveryRecord();
	}
	else
	{
		RequestCheckpoint(CHECKPOINT_END_OF_RECOVERY |
						  CHECKPOINT_FAST |
						  CHECKPOINT_WAIT);
	}
```

Promosyonda tam checkpoint beklenmez — `XLOG_END_OF_RECOVERY` kaydı yazılıp
hizmete hemen başlanır, checkpoint sonraya bırakılır.

---

# 8. Resource manager tablosu ve `rmgrdesc`

## 8.1 Tablo

Tek dosya, tek makro, sıra numaraları WAL'a gömülü:

[../src/include/access/rmgrlist.h:18-27](../src/include/access/rmgrlist.h#L18-L27)

```c
/*
 * List of resource manager entries.  Note that order of entries defines the
 * numerical values of each rmgr's ID, which is stored in WAL records.  New
 * entries should be added at the end, to avoid changing IDs of existing
 * entries.
 *
 * Changes to this list possibly need an XLOG_PAGE_MAGIC bump.
 */

/* symbol name, textual name, redo, desc, identify, startup, cleanup, mask, decode */
```

[../src/include/access/rmgrlist.h:28-50](../src/include/access/rmgrlist.h#L28-L50)

```c
PG_RMGR(RM_XLOG_ID, "XLOG", xlog_redo, xlog_desc, xlog_identify, NULL, NULL, NULL, xlog_decode)
PG_RMGR(RM_XACT_ID, "Transaction", xact_redo, xact_desc, xact_identify, NULL, NULL, NULL, xact_decode)
PG_RMGR(RM_SMGR_ID, "Storage", smgr_redo, smgr_desc, smgr_identify, NULL, NULL, NULL, NULL)
PG_RMGR(RM_CLOG_ID, "CLOG", clog_redo, clog_desc, clog_identify, NULL, NULL, NULL, NULL)
PG_RMGR(RM_DBASE_ID, "Database", dbase_redo, dbase_desc, dbase_identify, NULL, NULL, NULL, NULL)
PG_RMGR(RM_TBLSPC_ID, "Tablespace", tblspc_redo, tblspc_desc, tblspc_identify, NULL, NULL, NULL, NULL)
PG_RMGR(RM_MULTIXACT_ID, "MultiXact", multixact_redo, multixact_desc, multixact_identify, NULL, NULL, NULL, NULL)
PG_RMGR(RM_RELMAP_ID, "RelMap", relmap_redo, relmap_desc, relmap_identify, NULL, NULL, NULL, NULL)
PG_RMGR(RM_STANDBY_ID, "Standby", standby_redo, standby_desc, standby_identify, NULL, NULL, NULL, standby_decode)
PG_RMGR(RM_HEAP2_ID, "Heap2", heap2_redo, heap2_desc, heap2_identify, NULL, NULL, heap_mask, heap2_decode)
PG_RMGR(RM_HEAP_ID, "Heap", heap_redo, heap_desc, heap_identify, NULL, NULL, heap_mask, heap_decode)
PG_RMGR(RM_BTREE_ID, "Btree", btree_redo, btree_desc, btree_identify, btree_xlog_startup, btree_xlog_cleanup, btree_mask, NULL)
PG_RMGR(RM_HASH_ID, "Hash", hash_redo, hash_desc, hash_identify, NULL, NULL, hash_mask, NULL)
PG_RMGR(RM_GIN_ID, "Gin", gin_redo, gin_desc, gin_identify, gin_xlog_startup, gin_xlog_cleanup, gin_mask, NULL)
PG_RMGR(RM_GIST_ID, "Gist", gist_redo, gist_desc, gist_identify, gist_xlog_startup, gist_xlog_cleanup, gist_mask, NULL)
PG_RMGR(RM_SEQ_ID, "Sequence", seq_redo, seq_desc, seq_identify, NULL, NULL, seq_mask, NULL)
PG_RMGR(RM_SPGIST_ID, "SPGist", spg_redo, spg_desc, spg_identify, spg_xlog_startup, spg_xlog_cleanup, spg_mask, NULL)
PG_RMGR(RM_BRIN_ID, "BRIN", brin_redo, brin_desc, brin_identify, NULL, NULL, brin_mask, NULL)
PG_RMGR(RM_COMMIT_TS_ID, "CommitTs", commit_ts_redo, commit_ts_desc, commit_ts_identify, NULL, NULL, NULL, NULL)
PG_RMGR(RM_REPLORIGIN_ID, "ReplicationOrigin", replorigin_redo, replorigin_desc, replorigin_identify, NULL, NULL, NULL, NULL)
PG_RMGR(RM_GENERIC_ID, "Generic", generic_redo, generic_desc, generic_identify, NULL, NULL, generic_mask, NULL)
PG_RMGR(RM_LOGICALMSG_ID, "LogicalMessage", logicalmsg_redo, logicalmsg_desc, logicalmsg_identify, NULL, NULL, NULL, logicalmsg_decode)
PG_RMGR(RM_XLOG2_ID, "XLOG2", xlog2_redo, xlog2_desc, xlog2_identify, NULL, NULL, NULL, xlog2_decode)
```

Dokuz sütun, dokuz farklı iş:

| Sütun | Ne yapar |
|---|---|
| `redo` | Kurtarmada kaydı uygular |
| `desc` | Kaydı insan okunur metne çevirir (`pg_waldump`) |
| `identify` | `xl_info`'daki op kodunu isme çevirir ("INSERT", "UPDATE") |
| `startup` / `cleanup` | Redo başında/sonunda AM'ye özel kurulum-temizlik |
| `mask` | `wal_consistency_checking` için sayfadaki nondeterministik alanları maskeler |
| `decode` | Logical decoding için kaydı yorumlar |

`mask` sütunu ilginç: FPI ile sayfa karşılaştırması yapılırken hint bit'ler,
boş alan gibi replay'de farklı olabilecek alanlar maskelenir, yoksa her
karşılaştırma yanlış alarm verirdi.

`XLOG2` (ID 22) tablonun sonunda — yeni girdilerin sona eklenmesi kuralının
sonucu. WAL kayıtlarında ID sayısal olarak saklandığı için sıra değiştirilemez.

## 8.2 `rmgrdesc/`

[../src/backend/access/rmgrdesc/README:3-15](../src/backend/access/rmgrdesc/README#L3-L15)

```
WAL resource manager description functions
==========================================

For debugging purposes, there is a "description function", or rmgrdesc
function, for each WAL resource manager. The rmgrdesc function parses the WAL
record and prints the contents of the WAL record in a somewhat human-readable
format.

The rmgrdesc functions for all resource managers are gathered in this
directory, because they are also used in the stand-alone pg_waldump program.
They could potentially be used by out-of-tree debugging tools too, although
neither the description functions nor the output format should be considered
part of a stable API
```

Ayrı dizinde durmalarının sebebi: `pg_waldump` bir frontend programı, backend
kodunu linkleyemez. Bu 22 dosya ikisinin ortak alanı.

Örnek — heap kayıtları:

[../src/backend/access/rmgrdesc/heapdesc.c:191-217](../src/backend/access/rmgrdesc/heapdesc.c#L191-L217)

```c
	if (info == XLOG_HEAP_INSERT)
	{
		xl_heap_insert *xlrec = (xl_heap_insert *) rec;

		appendStringInfo(buf, "off: %u, flags: 0x%02X",
						 xlrec->offnum,
						 xlrec->flags);
	}
	else if (info == XLOG_HEAP_DELETE)
	{
		xl_heap_delete *xlrec = (xl_heap_delete *) rec;

		appendStringInfo(buf, "xmax: %u, off: %u, ",
						 xlrec->xmax, xlrec->offnum);
		infobits_desc(buf, xlrec->infobits_set, "infobits");
		appendStringInfo(buf, ", flags: 0x%02X", xlrec->flags);
	}
```

Format JSON'a benzer ama JSON değil:

[../src/backend/access/rmgrdesc/README:26-39](../src/backend/access/rmgrdesc/README#L26-L39)

```
Record descriptions are similar to JSON style key/value objects.  However,
there is no explicit "string" type/string escaping.  Top-level { } brackets
should be omitted.  For example:

snapshotConflictHorizon: 0, flags: 0x03
...
ndeleted: 0, nupdated: 1, deleted: [], updated: [{ off: 45, nptids: 1, ptids: [0] }]
```

## 8.3 XLOG rmgr'nin kendi kayıt tipleri

[../src/include/catalog/pg_control.h:72-86](../src/include/catalog/pg_control.h#L72-L86)

```c
#define XLOG_CHECKPOINT_SHUTDOWN		0x00
#define XLOG_CHECKPOINT_ONLINE			0x10
#define XLOG_NOOP						0x20
#define XLOG_NEXTOID					0x30
#define XLOG_SWITCH						0x40
#define XLOG_BACKUP_END					0x50
...
#define XLOG_END_OF_RECOVERY			0x90
#define XLOG_FPI_FOR_HINT				0xA0
#define XLOG_FPI						0xB0
...
#define XLOG_CHECKPOINT_REDO			0xE0
```

`XLOG_FPI_FOR_HINT` özel bir hikâye: hint bit yazımı normalde WAL'sızdır, ama
checksum açıkken torn page riski doğar, o yüzden sırf FPI almak için kayıt yazılır.

[../src/backend/access/transam/README:637-646](../src/backend/access/transam/README#L637-L646)

```
If the buffer is clean and checksums are in use then MarkBufferDirtyHint()
inserts an XLOG_FPI_FOR_HINT record to ensure that we take a full page image
that includes the hint. We do this to avoid a partial page write, when we
write the dirtied page. WAL is not written during recovery, so we simply skip
dirtying blocks because of hints when in recovery.

If you do decide to optimise away a WAL record, then any calls to
MarkBufferDirty() must be replaced by MarkBufferDirtyHint(),
otherwise you will expose the risk of partial page writes.
```

`data_checksums` açmanın WAL hacmini artırmasının sebebi budur.

---

# 9. Restart point, timeline, `pg_control`

## 9.1 Restart point

Standby WAL yazamaz, dolayısıyla checkpoint kaydı da yazamaz. Bunun yerine
**replay ettiği** bir checkpoint kaydını "restart point" olarak kullanır.

Startup süreci uygun kayıtları shared memory'ye kopyalar:

[../src/backend/access/transam/xlog.c:8085-8121](../src/backend/access/transam/xlog.c#L8085-L8121)

```c
/*
 * Save a checkpoint for recovery restart if appropriate
 *
 * This function is called each time a checkpoint record is read from XLOG.
 * It must determine whether the checkpoint represents a safe restartpoint or
 * not.  If so, the checkpoint record is stashed in shared memory so that
 * CreateRestartPoint can consult it.  (Note that the latter function is
 * executed by the checkpointer, while this one will be executed by the
 * startup process.)
 */
static void
RecoveryRestartPoint(const CheckPoint *checkPoint, XLogReaderState *record)
{
	/*
	 * Also refrain from creating a restartpoint if we have seen any
	 * references to non-existent pages. ...
	 */
	if (XLogHaveInvalidPages())
	{
		elog(DEBUG2,
			 "could not record restart point at %X/%08X because there are unresolved references to invalid pages",
			 LSN_FORMAT_ARGS(checkPoint->redo));
		return;
	}
	...
	XLogCtl->lastCheckPointRecPtr = record->ReadRecPtr;
	XLogCtl->lastCheckPointEndPtr = record->EndRecPtr;
	XLogCtl->lastCheckPoint = *checkPoint;
```

Checkpointer sonra bunu alıp gerçek işi yapar:

[../src/backend/access/transam/xlog.c:8124-8136](../src/backend/access/transam/xlog.c#L8124-L8136)

```c
/*
 * Establish a restartpoint if possible.
 *
 * This is similar to CreateCheckPoint, but is used during WAL recovery
 * to establish a point from which recovery can roll forward without
 * replaying the entire recovery log.
 *
 * Returns true if a new restartpoint was established. We can only establish
 * a restartpoint if we have replayed a safe checkpoint record since last
 * restartpoint.
 */
bool
CreateRestartPoint(int flags)
```

Aynı `CheckPointGuts` çağrılır ([xlog.c:8236](../src/backend/access/transam/xlog.c#L8236)),
ama WAL'a kayıt yazılmaz — sadece `pg_control` güncellenir:

[../src/backend/access/transam/xlog.c:8256-8266](../src/backend/access/transam/xlog.c#L8256-L8266)

```c
	LWLockAcquire(ControlFileLock, LW_EXCLUSIVE);
	if (ControlFile->checkPointCopy.redo < lastCheckPoint.redo)
	{
		/*
		 * Update the checkpoint information.  We do this even if the cluster
		 * does not show DB_IN_ARCHIVE_RECOVERY to match with the set of WAL
		 * segments recycled below.
		 */
		ControlFile->checkPoint = lastCheckPointRecPtr;
		ControlFile->checkPointCopy = lastCheckPoint;
```

Checkpointer ikisi arasında `RecoveryInProgress()` ile seçim yapar
([../src/backend/postmaster/checkpointer.c:497-499](../src/backend/postmaster/checkpointer.c#L497-L499)).

## 9.2 Timeline

Timeline, "aynı LSN'in iki farklı geleceği" problemini çözer. Bir standby promote
edildiğinde yeni bir TLI alır; eski primary'nin aynı LSN'de yazdığı WAL ile
karışmaması için.

[../src/backend/access/transam/timeline.c:6-22](../src/backend/access/transam/timeline.c#L6-L22)

```c
 * A timeline history file lists the timeline changes of the timeline, in
 * a simple text format. They are archived along with the WAL segments.
 *
 * The files are named like "<tli>.history". For example, if the database
 * starts up and switches to timeline 5, the timeline history file would be
 * called "00000005.history".
 *
 * Each line in the file represents a timeline switch:
 *
 * <parentTLI> <switchpoint> <reason>
 *
 *	parentTLI	ID of the parent timeline
 *	switchpoint XLogRecPtr of the WAL location where the switch happened
 *	reason		human-readable explanation of why the timeline was changed
```

TLI, WAL dosya adının ilk 8 hex karakteri olduğu için farklı timeline'ların
segmentleri **aynı dizinde çakışmadan** durabilir.

Replay sırasında timeline değişimi `ApplyWalRecord` içinde yakalanır:

[../src/backend/access/transam/xlogrecovery.c:1912-1944](../src/backend/access/transam/xlogrecovery.c#L1912-L1944)

```c
	if (record->xl_rmid == RM_XLOG_ID)
	{
		TimeLineID	newReplayTLI = *replayTLI;
		TimeLineID	prevReplayTLI = *replayTLI;
		uint8		info = record->xl_info & ~XLR_INFO_MASK;

		if (info == XLOG_CHECKPOINT_SHUTDOWN)
		{
			CheckPoint	checkPoint;

			memcpy(&checkPoint, XLogRecGetData(xlogreader), sizeof(CheckPoint));
			newReplayTLI = checkPoint.ThisTimeLineID;
			prevReplayTLI = checkPoint.PrevTimeLineID;
		}
		else if (info == XLOG_END_OF_RECOVERY)
		{
			xl_end_of_recovery xlrec;
			...
		}

		if (newReplayTLI != *replayTLI)
		{
			/* Check that it's OK to switch to this TLI */
			checkTimeLineSwitch(xlogreader->EndRecPtr,
								newReplayTLI, prevReplayTLI, *replayTLI);

			/* Following WAL records should be run with new TLI */
			*replayTLI = newReplayTLI;
			switchedTLI = true;
		}
	}
```

Sadece iki kayıt tipi TLI değiştirebilir: shutdown checkpoint ve end-of-recovery.
Bu, bölüm 7.8'deki "TLI sadece shutdown checkpoint'te değişir" kuralının karşılığı.

## 9.3 `pg_control`'ün checkpoint kopyası

[../src/include/catalog/pg_control.h:35-53](../src/include/catalog/pg_control.h#L35-L53)

```c
typedef struct CheckPoint
{
	XLogRecPtr	redo;			/* next RecPtr available when we began to
								 * create CheckPoint (i.e. REDO start point) */
	TimeLineID	ThisTimeLineID; /* current TLI */
	TimeLineID	PrevTimeLineID; /* previous TLI, if this record begins a new
								 * timeline (equals ThisTimeLineID otherwise) */
	bool		fullPageWrites; /* current full_page_writes */
	int			wal_level;		/* current wal_level */
	bool		logicalDecodingEnabled; /* current logical decoding status */
	FullTransactionId nextXid;	/* next free transaction ID */
	Oid			nextOid;		/* next free OID */
	MultiXactId nextMulti;		/* next free MultiXactId */
	MultiXactOffset nextMultiOffset;	/* next free MultiXact offset */
	TransactionId oldestXid;	/* cluster-wide minimum datfrozenxid */
	Oid			oldestXidDB;	/* database with minimum datfrozenxid */
	MultiXactId oldestMulti;	/* cluster-wide minimum datminmxid */
	Oid			oldestMultiDB;	/* database with minimum datminmxid */
	pg_time_t	time;			/* time stamp of checkpoint */
```

Bu yapı hem WAL kaydının gövdesi hem de `pg_control` içinde saklanan kopya.
Dolayısıyla **`pg_control` bozulursa cluster açılmaz** — checkpoint'i nerede
arayacağını bilemez. Dosya sürümü de sıkı kontrol edilir:

[../src/include/catalog/pg_control.h:25](../src/include/catalog/pg_control.h#L25)

```c
#define PG_CONTROL_VERSION	1902
```

---

# İzleme ve hata ayıklama

## `pg_waldump` — WAL'ın içine bakmak

[../src/bin/pg_waldump/pg_waldump.c:900-923](../src/bin/pg_waldump/pg_waldump.c#L900-L923)

```
  -b, --bkp-details      output detailed information about backup blocks
  -B, --block=N          with --relation, only show records that modify block N
  -e, --end=RECPTR       stop reading at WAL location RECPTR
  -f, --follow           keep retrying after reaching end of WAL
  -n, --limit=N          number of records to display
  -r, --rmgr=RMGR        only show records generated by resource manager RMGR;
  -R, --relation=T/D/R   only show records that modify blocks in relation T/D/R
  -s, --start=RECPTR     start reading at WAL location RECPTR
  -w, --fullpage         only show records with a full page write
  -x, --xid=XID          only show records with transaction ID XID
  -z, --stats[=record]   show statistics instead of records
  --save-fullpage=DIR    save full page images to DIR
```

Sık kullanılan üçlü:

```bash
# WAL'ı ne dolduruyor? rmgr bazında dağılım
pg_waldump --stats $PGDATA/pg_wal/000000010000000000000042

# Kayıt tipi kırılımı — hangi işlem pahalı?
pg_waldump --stats=record $PGDATA/pg_wal/000000010000000000000042

# Sadece full-page write üreten kayıtlar — checkpoint çok mu sık?
pg_waldump --fullpage -n 50 $PGDATA/pg_wal/000000010000000000000042

# Belirli bir tabloyu kim değiştirdi? (T/D/R = tablespace/db/relfilenode)
pg_waldump -R 1663/16384/16400 -n 100 $PGDATA/pg_wal/0000000100000000*
```

## `pg_controldata` — kurtarma durumu

[../src/bin/pg_controldata/pg_controldata.c:251-267](../src/bin/pg_controldata/pg_controldata.c#L251-L267)

```c
	printf(_("Database cluster state:               %s\n"), ...);
	printf(_("Latest checkpoint location:           %X/%08X\n"), ...);
	printf(_("Latest checkpoint's REDO location:    %X/%08X\n"), ...);
	printf(_("Latest checkpoint's REDO WAL file:    %s\n"), ...);
	printf(_("Latest checkpoint's TimeLineID:       %u\n"), ...);
	printf(_("Latest checkpoint's PrevTimeLineID:   %u\n"), ...);
	printf(_("Latest checkpoint's full_page_writes: %s\n"), ...);
```

Bakılacak satırlar:

- **Database cluster state** — `in production` görüyorsanız cluster çalışıyor ya
  da temiz kapanmamış. `shut down` temiz kapanış.
- **Latest checkpoint location** vs **REDO location** — aradaki fark, çökme
  durumunda replay edilecek WAL miktarının alt sınırıdır.
- **Minimum recovery ending location** — standby'da sıfır değilse bu noktanın
  altına inilemez.

## SQL ile izleme

```sql
-- Şu an neredeyiz? (insert / write / flush üç ayrı nokta)
SELECT pg_current_wal_insert_lsn() AS insert,
       pg_current_wal_lsn()        AS write,
       pg_current_wal_flush_lsn()  AS flush;

-- Son checkpoint'ten bu yana ne kadar WAL üretildi?
SELECT pg_size_pretty(
         pg_wal_lsn_diff(pg_current_wal_lsn(), checkpoint_lsn)) AS since_checkpoint
FROM pg_control_checkpoint();

-- Bu LSN hangi dosyada?
SELECT pg_walfile_name(pg_current_wal_lsn());

-- pg_wal ne kadar şişmiş?
SELECT count(*) AS segments,
       pg_size_pretty(sum(size)) AS total
FROM pg_ls_waldir();

-- Standby'da: ne kadar geride?
SELECT pg_is_in_recovery(),
       pg_last_wal_receive_lsn(),
       pg_last_wal_replay_lsn(),
       pg_wal_lsn_diff(pg_last_wal_receive_lsn(),
                       pg_last_wal_replay_lsn()) AS replay_lag_bytes;
```

## `pg_stat_wal` — WAL üretim profili

[../src/backend/catalog/system_views.sql:1296-1304](../src/backend/catalog/system_views.sql#L1296-L1304)

```sql
CREATE VIEW pg_stat_wal AS
    SELECT
        w.wal_records,
        w.wal_fpi,
        w.wal_bytes,
        w.wal_fpi_bytes,
        w.wal_buffers_full,
        w.stats_reset
    FROM pg_stat_get_wal() w;
```

İki oran hemen bir şey söyler:

```sql
SELECT wal_records, wal_fpi,
       round(100.0 * wal_fpi / nullif(wal_records, 0), 1)   AS fpi_pct,
       pg_size_pretty(wal_bytes)                            AS total,
       pg_size_pretty(wal_fpi_bytes)                        AS fpi_bytes,
       round(100.0 * wal_fpi_bytes / nullif(wal_bytes, 0), 1) AS fpi_byte_pct,
       wal_buffers_full
FROM pg_stat_wal;
```

- **`fpi_byte_pct` yüksek (>%50)** → checkpoint çok sık. `max_wal_size` ve
  `checkpoint_timeout` artırılabilir.
- **`wal_buffers_full` sürekli artıyor** → `wal_buffers` küçük (bölüm 4.1).

## `pg_stat_checkpointer` — checkpoint sağlığı

[../src/backend/catalog/system_views.sql:1247-1259](../src/backend/catalog/system_views.sql#L1247-L1259)

```sql
CREATE VIEW pg_stat_checkpointer AS
    SELECT
        pg_stat_get_checkpointer_num_timed() AS num_timed,
        pg_stat_get_checkpointer_num_requested() AS num_requested,
        pg_stat_get_checkpointer_num_performed() AS num_done,
        pg_stat_get_checkpointer_restartpoints_timed() AS restartpoints_timed,
        ...
        pg_stat_get_checkpointer_write_time() AS write_time,
        pg_stat_get_checkpointer_sync_time() AS sync_time,
        pg_stat_get_checkpointer_buffers_written() AS buffers_written,
```

`num_requested` (max_wal_size tetiklemesi) `num_timed`'dan büyükse checkpoint'ler
zaman aşımıyla değil WAL hacmiyle tetikleniyor demektir — `max_wal_size`
yetersiz. Sunucu zaten uyarıyor:

[../src/backend/postmaster/checkpointer.c:471-479](../src/backend/postmaster/checkpointer.c#L471-L479)

```c
			if (!do_restartpoint &&
				(flags & CHECKPOINT_CAUSE_XLOG) &&
				elapsed_secs < CheckPointWarning)
				ereport(LOG,
						(errmsg_plural("checkpoints are occurring too frequently (%d second apart)",
									   ...
						 errhint("Consider increasing the configuration parameter \"%s\".", "max_wal_size")));
```

## İlgili GUC'lar

| GUC | Varsayılan | Etki |
|---|---|---|
| `wal_level` | `replica` | `minimal` WAL'ı küçültür ama replikasyon/PITR'ı kapatır ([xlog.h:74-78](../src/include/access/xlog.h#L74-L78)) |
| `synchronous_commit` | `on` (= `remote_flush`) | `off` → commit'te fsync beklenmez, ~`wal_writer_delay` kadar kayıp riski |
| `wal_buffers` | `-1` (auto) | `shared_buffers/32`, üst sınır bir segment ([xlog.c:5022](../src/backend/access/transam/xlog.c#L5022)) |
| `wal_writer_delay` | `200ms` | Arka plan flush periyodu |
| `wal_writer_flush_after` | `1MB` | Bu kadar birikince periyodu beklemeden flush |
| `commit_delay` | `0` | fsync öncesi bekleme, group commit'i büyütür ([xlog.c:2901](../src/backend/access/transam/xlog.c#L2901)) |
| `commit_siblings` | `5` | `commit_delay`'in devreye girmesi için gereken aktif backend sayısı |
| `full_page_writes` | `on` | Kapatmak WAL'ı küçültür ama **torn page riskini açar** |
| `wal_compression` | `off` | FPI'leri sıkıştırır (`pglz`/`lz4`/`zstd`) — CPU karşılığı WAL hacmi |
| `wal_log_hints` | `off` | Hint bit yazımlarını da FPI'ye tabi tutar (`pg_rewind` için gerekli) |
| `checkpoint_timeout` | `5min` | Checkpoint arası azami süre |
| `checkpoint_completion_target` | `0.9` | Checkpoint I/O'sunun yayıldığı oran |
| `max_wal_size` | `1GB` | Checkpoint'ler arası hedef WAL hacmi |
| `min_wal_size` | `80MB` | Geri dönüştürme için tutulacak asgari segment havuzu |
| `wal_recycle` | `on` | Eski segmentleri sil yerine yeniden adlandır |
| `wal_init_zero` | `on` | Yeni segmenti sıfırla doldur (alan ön-tahsisi) |
| `wal_consistency_checking` | `''` | Her kayda doğrulama FPI'si ekler — **sadece hata ayıklama**, çok pahalı |
| `archive_mode` | `off` | `on`/`always`; `wal_level >= replica` gerektirir |
| `recovery_prefetch` | `try` | Redo sırasında sayfa önyüklemesi |

## Log'da aranacak satırlar

```
database system was interrupted; last known up at ...   ← crash recovery başlıyor
redo starts at 0/1A2B3C4D                               ← replay başladı
consistent recovery state reached at 0/1B000000         ← hot standby açılabilir
redo done at 0/1C4D5E6F system usage: ...               ← replay bitti
checkpoint starting: time                               ← log_checkpoints=on ile
checkpoint complete: wrote 1234 buffers (7.5%); ...
checkpoints are occurring too frequently (23 seconds apart)   ← max_wal_size küçük
```

`log_checkpoints` varsayılan olarak açıktır ve `checkpoint complete` satırındaki
`write=... sync=... total=...` üçlüsü checkpoint'in nerede takıldığını söyler:
`sync` yüksekse dosya sistemi fsync'te boğuluyor demektir.

---

# Tek sayfalık özet

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                      YAZMA YOLU (normal çalışma)                             ║
╚══════════════════════════════════════════════════════════════════════════════╝

  heap_update() / btree insert / ...
        │
        │ START_CRIT_SECTION()
        ├──► sayfayı shared buffer'da değiştir
        ├──► MarkBufferDirty()
        │
        ├──► XLogBeginInsert()                     xloginsert.c:153
        │    XLogRegisterBuffer(0, buf, ...)       xloginsert.c:246
        │    XLogRegisterData(&xlrec, ...)         xloginsert.c:372
        │    XLogInsert(RM_HEAP_ID, XLOG_HEAP_UPDATE)
        │         │
        │         ├─► XLogRecordAssemble()         xloginsert.c:620
        │         │     her blok için:  page_lsn <= RedoRecPtr ?
        │         │        EVET → FPI ekle (8 kB, delik atılmış, belki sıkıştırılmış)
        │         │        HAYIR → sadece incremental veri
        │         │
        │         └─► XLogInsertRecord()           xlog.c:784
        │               ├─ WALInsertLockAcquire()       (8 lock'tan biri)
        │               ├─ RedoRecPtr değişmiş mi? → evet ise BAŞTAN BAŞLA
        │               ├─ ReserveXLogInsertLocation()  ← spinlock, 3 satır
        │               ├─ CRC hesapla (xl_prev artık belli)
        │               ├─ CopyXLogRecordToWAL()        ← paralel, lock'suz
        │               └─ WALInsertLockRelease()
        │
        ├──► PageSetLSN(page, EndPos)      ◄── sayfa artık WAL'a bağlı
        └──► END_CRIT_SECTION()

  COMMIT:
        RecordTransactionCommit()          xact.c
              │
              ├─ synchronous_commit > off ?
              │     EVET → XLogFlush(LSN)  ──► WALWriteLock ──► pwrite + fsync
              │            └─ group commit: 100 backend, 1 fsync
              │     HAYIR → XLogSetAsyncXactLSN(LSN)
              │            └─ walwriter ~200 ms içinde flush eder
              └─ CLOG güncelle

  Kirli sayfa diske inerken (bufmgr.c:4585):
        FlushBuffer() ──► XLogFlush(page LSN) ──► SONRA sayfayı yaz
                          └── "WAL RULE": log önce, veri sonra


╔══════════════════════════════════════════════════════════════════════════════╗
║                        CHECKPOINT (checkpointer)                             ║
╚══════════════════════════════════════════════════════════════════════════════╝

  tetik: checkpoint_timeout | max_wal_size | CHECKPOINT | shutdown
        │
  CreateCheckPoint()                       xlog.c:7400
        ├─ boşta mı? → atla
        ├─ REDO NOKTASI SEÇ:
        │     shutdown  → mevcut insert konumu
        │     online    → XLOG_CHECKPOINT_REDO kaydı yaz, LSN'i al
        ├─ CheckPointGuts()                xlog.c:8049   ◄── ZAMANIN %95'İ
        │     CLOG, CommitTs, SUBTRANS, MultiXact, Predicate,
        │     CheckPointBuffers  ← tüm kirli buffer'lar (throttled)
        │     ProcessSyncRequests ← biriken fsync'ler
        ├─ XLOG_CHECKPOINT_ONLINE / _SHUTDOWN kaydı yaz + XLogFlush
        ├─ pg_control güncelle (checkPoint, checkPointCopy)
        └─ KeepLogSeg + RemoveOldXlogFiles  ← eski segmentleri sil/recycle
              (replication slot geride ise SİLİNMEZ)

  standby'da aynı iş → CreateRestartPoint()  xlog.c:8136
        (WAL kaydı YOK, sadece pg_control)


╔══════════════════════════════════════════════════════════════════════════════╗
║                          KURTARMA (startup süreci)                           ║
╚══════════════════════════════════════════════════════════════════════════════╝

  global/pg_control ──► state? checkPoint? checkPointCopy.redo?
        │
  StartupXLOG()                            xlog.c:5845
        ├─ InitWalRecovery()  ← backup_label varsa ondan başla
        ├─ nextXid/nextOid/oldestXid = checkpoint'ten
        │
        ├─ PerformWalRecovery()            xlogrecovery.c:1617
        │     redo noktasından başla, ReadRecord() döngüsü:
        │        ├─ ValidXLogRecordHeader()  ← xl_prev zinciri tutuyor mu?
        │        ├─ ValidXLogRecord()        ← CRC32C doğru mu?
        │        │      ikisinden biri HAYIR → WAL'ın sonu, DUR
        │        │
        │        └─ ApplyWalRecord()         xlogrecovery.c:1888
        │              GetRmgr(xl_rmid).rm_redo(record)
        │                    │
        │                    └─ XLogReadBufferForRedo()   xlogutils.c:361
        │                          FPI var + APPLY?
        │                             → sayfayı sıfırla, FPI'yi yaz  BLK_RESTORED
        │                          yoksa lsn <= PageGetLSN(page)?
        │                             → BLK_DONE     (zaten uygulanmış, ATLA)
        │                             → BLK_NEEDS_REDO (uygula, PageSetLSN)
        │
        ├─ FinishWalRecovery()
        ├─ minRecoveryPoint aşıldı → "consistent recovery state reached"
        └─ promosyon? → XLOG_END_OF_RECOVERY (+ yeni TLI)
           değilse   → CHECKPOINT_END_OF_RECOVERY


  ┌─────────────────────────────────────────────────────────────────────────┐
  │  ÜÇ NUMARA, HEPSİ BU:                                                   │
  │                                                                         │
  │  1. LSN     — sayfa "hangi WAL kaydına kadar güncelim" der.             │
  │               Redo bunu okur, gereksiz işi atlar. İdempotency.          │
  │                                                                         │
  │  2. FPI     — checkpoint'ten sonra sayfaya ilk dokunan kayıt sayfanın    │
  │               tamamını taşır. Torn page'e karşı tek savunma.            │
  │                                                                         │
  │  3. xl_prev — kayıtlar geriye bağlı zincir. Geri dönüştürülmüş          │
  │               segmentteki eski-ama-CRC'si-geçerli çöp kaydı yakalar.    │
  └─────────────────────────────────────────────────────────────────────────┘
```
