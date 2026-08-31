# PostgreSQL transaction yönetimi — kodun içinden

> Bu depo: PostgreSQL **20devel** ([meson.build](../meson.build) `version: '20devel'`).
> Satır numaraları sürüme bağlıdır.
> Kardeş dosyalar: [SNAPSHOT-VE-GORUNURLUK.md](SNAPSHOT-VE-GORUNURLUK.md) · [KILIT-MEKANIZMALARI.md](KILIT-MEKANIZMALARI.md) · [UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md)
> Ana kaynak: [../src/backend/access/transam/README](../src/backend/access/transam/README) — bu dosyayı okumadan koda dalmayın.

---

## 30 saniyelik özet

Bir transaction'ın ömrü boyunca **iki kimliği** olabilir: her zaman bir `VirtualTransactionId` (vxid), ve *ancak yazma yaparsa* bir `TransactionId` (xid).

```
  BEGIN                          ilk INSERT/UPDATE/DELETE              COMMIT
    │                                       │                            │
    ▼                                       ▼                            ▼
┌──────────────────┐            ┌────────────────────────┐   ┌──────────────────────────┐
│ vxid = (procNo,  │            │ xid = GetNewTransaction│   │ 1. WAL: XLOG_XACT_COMMIT │
│        lxid)     │───────────►│       Id()             │──►│ 2. XLogFlush()           │
│ xid  = YOK       │  tembel    │ ProcArray'e yazılır    │   │ 3. CLOG: bit = 01        │
│ CLOG'da yeri yok │  tahsis    │ XactLockTableInsert    │   │ 4. ProcArray: xid temizle│
└──────────────────┘            └────────────────────────┘   │ 5. Kilitleri bırak       │
        ▲                                                    └──────────────────────────┘
        │
  salt-okunur transaction burada kalır: hiç xid almaz,
  CLOG'a yazmaz, ProcArrayLock'ı exclusive almadan biter
```

Kalıcı durum dört ayrı dizinde, hepsi aynı altyapı (`slru.c`) üzerinde:

```
   pg_xact/          pg_subtrans/       pg_multixact/        pg_commit_ts/
   (CLOG)            (parent xid)       offsets + members    (commit zamanı)
   2 bit/xid         4 byte/xid         8 byte + 5 byte      ~12 byte/xid
       │                  │                     │                  │
       └──────────────────┴─────────────────────┴──────────────────┘
                                    │
                          slru.c — banklı LRU sayfa cache'i
                       (+ pg_serial, pg_notify da aynı kodu kullanır)
```

**Altı maddelik model:**

1. **İki katmanlı durum makinesi.** `TransState` motorun iç durumu (START/INPROGRESS/COMMIT/ABORT/PREPARE), `TBlockState` kullanıcının gördüğü blok durumu (BEGIN/INPROGRESS/END/ABORT + subtransaction karşılıkları). `postgres.c` her komuttan önce/sonra `StartTransactionCommand`/`CommitTransactionCommand` çağırır; bu ikisi durum-duyarlıdır.
2. **xid tahsisi tembeldir.** Salt-okunur transaction ömrü boyunca xid almaz. Bu, xid tüketimini ve CLOG büyümesini okuma trafiğinden tamamen ayırır.
3. **Subtransaction = TransactionState yığını.** `SAVEPOINT` bir `PushTransaction` demektir. Alt xid alınırsa önce ebeveynlerinin xid'leri alınır — çocuk xid'i her zaman ebeveynden büyüktür.
4. **Commit sırası pazarlıksızdır:** WAL yaz → flush → CLOG'a işaretle → ProcArray'den çık → kilitleri bırak. Her adım bir öncekinin dayanağı.
5. **CLOG transaction başına 2 bit tutar.** Görünürlük kararının nihai kaynağı budur; hint bit'ler sadece bir cache'tir.
6. **SLRU dört farklı veriyi aynı kodla yönetir.** Ayrı LWLock bank'ları, ayrı GUC'lar, ortak sayfa cache'i ve ortak `pg_stat_slru` görünümü.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/access/transam/README](../src/backend/access/transam/README) | Tasarım belgesi: durum makinesi, numaralandırma, interlocking |
| [../src/backend/access/transam/xact.c](../src/backend/access/transam/xact.c) | Durum makinesi, commit/abort/prepare, savepoint (~200 KB) |
| [../src/backend/access/transam/varsup.c](../src/backend/access/transam/varsup.c) | `GetNewTransactionId`, wraparound eşikleri |
| [../src/backend/access/transam/transam.c](../src/backend/access/transam/transam.c) | `TransactionIdDidCommit` — CLOG'un üstündeki ince katman |
| [../src/backend/access/transam/clog.c](../src/backend/access/transam/clog.c) | pg_xact: 2 bit/transaction commit durumu |
| [../src/backend/access/transam/subtrans.c](../src/backend/access/transam/subtrans.c) | pg_subtrans: her xid'in ebeveyni |
| [../src/backend/access/transam/slru.c](../src/backend/access/transam/slru.c) | Ortak sayfa cache altyapısı |
| [../src/backend/access/transam/multixact.c](../src/backend/access/transam/multixact.c) | pg_multixact: çoklu satır kilitçisi (~90 KB) |
| [../src/backend/access/transam/twophase.c](../src/backend/access/transam/twophase.c) | `PREPARE TRANSACTION`, pg_twophase, kurtarma |
| [../src/backend/access/transam/commit_ts.c](../src/backend/access/transam/commit_ts.c) | pg_commit_ts: `track_commit_timestamp` |
| [../src/backend/storage/ipc/procarray.c](../src/backend/storage/ipc/procarray.c) | `ProcArrayEndTransaction`, grup temizliği |
| [../src/backend/utils/resowner/resowner.c](../src/backend/utils/resowner/resowner.c) | Kaynak sahipliği; abort'ta sızıntı temizliği |
| [../src/include/storage/proc.h](../src/include/storage/proc.h) | PGPROC: xid, xmin, subxid cache, vxid |
| [../src/include/access/clog.h](../src/include/access/clog.h) | Dört durum sabiti |
| [../src/include/storage/lock.h](../src/include/storage/lock.h) | `VirtualTransactionId` tanımı |

---

# 1. İki katmanlı durum makinesi

## 1.1. `TransState` — motorun iç durumu

[../src/backend/access/transam/xact.c:143-151](../src/backend/access/transam/xact.c#L143-L151)

```c
typedef enum TransState
{
	TRANS_DEFAULT,				/* idle */
	TRANS_START,				/* transaction starting */
	TRANS_INPROGRESS,			/* inside a valid transaction */
	TRANS_COMMIT,				/* commit in progress */
	TRANS_ABORT,				/* abort in progress */
	TRANS_PREPARE,				/* prepare in progress */
} TransState;
```

Altı değer. Bunlar kullanıcıya hiç görünmez.

## 1.2. `TBlockState` — kullanıcının gördüğü durum

[../src/backend/access/transam/xact.c:159-186](../src/backend/access/transam/xact.c#L159-L186)

```c
typedef enum TBlockState
{
	/* not-in-transaction-block states */
	TBLOCK_DEFAULT,				/* idle */
	TBLOCK_STARTED,				/* running single-query transaction */

	/* transaction block states */
	TBLOCK_BEGIN,				/* starting transaction block */
	TBLOCK_INPROGRESS,			/* live transaction */
	TBLOCK_IMPLICIT_INPROGRESS, /* live transaction after implicit BEGIN */
	TBLOCK_PARALLEL_INPROGRESS, /* live transaction inside parallel worker */
	TBLOCK_END,					/* COMMIT received */
	TBLOCK_ABORT,				/* failed xact, awaiting ROLLBACK */
	TBLOCK_ABORT_END,			/* failed xact, ROLLBACK received */
	TBLOCK_ABORT_PENDING,		/* live xact, ROLLBACK received */
	TBLOCK_PREPARE,				/* live xact, PREPARE received */
	...
```

20 değerin 9'u subtransaction'a ait (`TBLOCK_SUB*`). Ayrım neden gerekli? README'nin verdiği örnek nettir: kullanıcının yazdığı `ROLLBACK` ile sistemin ürettiği hata farklı sonraki-durumlar gerektirir.

[../src/backend/access/transam/README](../src/backend/access/transam/README) (satır 118-121):

```
* AbortCurrentTransaction leaves us in TBLOCK_ABORT,
* UserAbortTransactionBlock leaves us in TBLOCK_ABORT_END
```

`TBLOCK_ABORT` durumunda backend, `ROLLBACK` gelene kadar tüm komutları yok sayar — "current transaction is aborted, commands ignored until end of transaction block" mesajının kaynağı budur.

## 1.3. Durum yığını — `TransactionStateData`

[../src/backend/access/transam/xact.c:195-221](../src/backend/access/transam/xact.c#L195-L221)

```c
typedef struct TransactionStateData
{
	FullTransactionId fullTransactionId;	/* my FullTransactionId */
	SubTransactionId subTransactionId;	/* my subxact ID */
	char	   *name;			/* savepoint name, if any */
	int			savepointLevel; /* savepoint level */
	TransState	state;			/* low-level state */
	TBlockState blockState;		/* high-level state */
	int			nestingLevel;	/* transaction nesting depth */
	...
	TransactionId *childXids;	/* subcommitted child XIDs, in XID order */
	int			nChildXids;		/* # of subcommitted child XIDs */
	...
	struct TransactionStateData *parent;	/* back link to parent */
} TransactionStateData;
```

`parent` alanı yığını oluşturur. En üstteki eleman statik `TopTransactionStateData`, altındakiler `TopTransactionContext` içinde ayrılır.

## 1.4. Komut sınırları — `StartTransactionCommand`

[../src/backend/access/transam/xact.c:3112](../src/backend/access/transam/xact.c#L3112)

```c
StartTransactionCommand(void)
{
	TransactionState s = CurrentTransactionState;

	switch (s->blockState)
	{
		case TBLOCK_DEFAULT:
			StartTransaction();
			s->blockState = TBLOCK_STARTED;
			break;
		...
		case TBLOCK_INPROGRESS:
		case TBLOCK_IMPLICIT_INPROGRESS:
		case TBLOCK_SUBINPROGRESS:
			break;
```

Yani `BEGIN` içindeki her komut için gerçek bir transaction başlatılmaz; sadece durum kontrol edilir. Karşılığı [../src/backend/access/transam/xact.c:3210](../src/backend/access/transam/xact.c#L3210) `CommitTransactionCommand`'dır; asıl iş [../src/backend/access/transam/xact.c:3228](../src/backend/access/transam/xact.c#L3228) `CommitTransactionCommandInternal` içindedir.

## 1.5. `StartTransaction` — vxid burada doğar

[../src/backend/access/transam/xact.c:2203-2217](../src/backend/access/transam/xact.c#L2203-L2217)

```c
	vxid.procNumber = MyProcNumber;
	vxid.localTransactionId = GetNextLocalTransactionId();

	/*
	 * Lock the virtual transaction id before we announce it in the proc array
	 */
	VirtualXactLockTableInsert(vxid);
	...
	MyProc->vxid.lxid = vxid.localTransactionId;
```

Dikkat: burada **hiçbir paylaşımlı sayaç** artırılmıyor. `GetNextLocalTransactionId` backend-yerel bir sayaçtır — transaction başlatmak neredeyse bedavadır.

---

# 2. xid tahsisi tembeldir

## 2.1. İki API, bir harf farkı

[../src/backend/access/transam/xact.c:456](../src/backend/access/transam/xact.c#L456)

```c
TransactionId
GetCurrentTransactionId(void)
{
	TransactionState s = CurrentTransactionState;

	if (!FullTransactionIdIsValid(s->fullTransactionId))
		AssignTransactionId(s);
	return XidFromFullTransactionId(s->fullTransactionId);
}
```

[../src/backend/access/transam/xact.c:473](../src/backend/access/transam/xact.c#L473)

```c
TransactionId
GetCurrentTransactionIdIfAny(void)
{
	return XidFromFullTransactionId(CurrentTransactionState->fullTransactionId);
}
```

Fark tek satır: birincisi **xid tahsis eder**, ikincisi sadece **var olanı okur**. Kod içinde hangisinin çağrıldığı, o kod yolunun xid tüketip tüketmediğini belirler. `heap_insert`/`heap_update` birinciyi, `RecordTransactionCommit` ikinciyi çağırır.

## 2.2. `AssignTransactionId` — dört yan etki

[../src/backend/access/transam/xact.c:637](../src/backend/access/transam/xact.c#L637)

Fonksiyon sırasıyla:

**(a) Ebeveynleri özyinelemesiz doldurur** — çocuk xid'i her zaman ebeveynden büyük olmalı:

```c
	if (isSubXact && !FullTransactionIdIsValid(s->parent->fullTransactionId))
	{
		...
		parents = palloc_array(TransactionState, s->nestingLevel);
		while (p != NULL && !FullTransactionIdIsValid(p->fullTransactionId))
		{
			parents[parentOffset++] = p;
			p = p->parent;
		}
```

Neden döngü? Yorumun dediği gibi: "Mustn't recurse here, or we might get a stack overflow if we're at the bottom of a huge stack of subtransactions none of which have XIDs yet."

**(b) xid'i alır ve pg_subtrans'a yazar** — [../src/backend/access/transam/xact.c:708-714](../src/backend/access/transam/xact.c#L708-L714):

```c
	s->fullTransactionId = GetNewTransactionId(isSubXact);
	if (!isSubXact)
		XactTopFullTransactionId = s->fullTransactionId;

	if (isSubXact)
		SubTransSetParent(XidFromFullTransactionId(s->fullTransactionId),
						  XidFromFullTransactionId(s->parent->fullTransactionId));
```

Sıra kritik. Aynı fonksiyonun yorumu:

> NB: we must make the subtrans entry BEFORE the Xid appears anywhere in shared storage other than PGPROC; because if there's no room for it in PGPROC, the subtrans entry is needed to ensure that other backends see the Xid as "running".

**(c) xid üzerine kilit alır** — [../src/backend/access/transam/xact.c:731](../src/backend/access/transam/xact.c#L731):

```c
	XactLockTableInsert(XidFromFullTransactionId(s->fullTransactionId));
```

Bu, `XactLockTableWait` ile başkalarının bu transaction'ı beklemesini sağlayan mekanizmadır (bkz. [KILIT-MEKANIZMALARI.md](KILIT-MEKANIZMALARI.md)).

**(d) Standby için WAL kaydı** — her `PGPROC_MAX_CACHED_SUBXIDS` (64) alt xid'de bir `XLOG_XACT_ASSIGNMENT` yazılır; hot standby'ın `KnownAssignedXids` yapısı bunu izler.

## 2.3. `GetNewTransactionId` — xid nasıl yayınlanır

[../src/backend/access/transam/varsup.c:68](../src/backend/access/transam/varsup.c#L68)

Önce dosya alanı açılır, sonra sayaç ilerletilir:

[../src/backend/access/transam/varsup.c:199-209](../src/backend/access/transam/varsup.c#L199-L209)

```c
	ExtendCLOG(xid);
	ExtendCommitTs(xid);
	ExtendSUBTRANS(xid);

	/*
	 * Now advance the nextXid counter.  This must not happen until after we
	 * have successfully completed ExtendCLOG() --- if that routine fails, we
	 * want the next incoming transaction to try it again.  We cannot assign
	 * more XIDs until there is CLOG space for them.
	 */
	FullTransactionIdAdvance(&TransamVariables->nextXid);
```

Sonra xid ProcArray'e yazılır — **XidGenLock hâlâ tutulurken**:

[../src/backend/access/transam/varsup.c:252-255](../src/backend/access/transam/varsup.c#L252-L255)

```c
		/* LWLockRelease acts as barrier */
		MyProc->xid = xid;
		ProcGlobal->xids[MyProc->pgxactoff] = xid;
```

Alt transaction ise PGPROC'taki cache'e girer, yer yoksa overflow bayrağı kalkar:

[../src/backend/access/transam/varsup.c:259-272](../src/backend/access/transam/varsup.c#L259-L272)

```c
		int			nxids = MyProc->subxidStatus.count;
		...
		if (nxids < PGPROC_MAX_CACHED_SUBXIDS)
		{
			MyProc->subxids.xids[nxids] = xid;
			pg_write_barrier();
			MyProc->subxidStatus.count = substat->count = nxids + 1;
		}
		else
			MyProc->subxidStatus.overflowed = substat->overflowed = true;
```

Bu `overflowed` bayrağı [SNAPSHOT-VE-GORUNURLUK.md](SNAPSHOT-VE-GORUNURLUK.md)'te tekrar karşımıza çıkacak: overflow olduğunda görünürlük kontrolü pg_subtrans'a inmek zorunda kalır.

## 2.4. VirtualTransactionId

[../src/include/storage/lock.h:62-66](../src/include/storage/lock.h#L62-L66)

```c
typedef struct VirtualTransactionId
{
	ProcNumber	procNumber;		/* proc number of the PGPROC */
	LocalTransactionId localTransactionId;	/* lxid from PGPROC */
} VirtualTransactionId;
```

Aynı dosyanın 47-53. satırlarındaki yorum, neden diske yazılmadığını söyler: "These are guaranteed unique over the short term, but will be reused after a database restart or XID wraparound; hence they should never be stored on disk."

Salt-okunur bir transaction kilit alabilsin diye vxid vardır. `pg_locks.virtualxid` ve `virtualtransaction` kolonları buradan gelir.

---

# 3. Subtransaction

## 3.1. `SAVEPOINT` = `PushTransaction`

[../src/backend/access/transam/xact.c:4427](../src/backend/access/transam/xact.c#L4427) `DefineSavepoint`:

```c
		case TBLOCK_INPROGRESS:
		case TBLOCK_SUBINPROGRESS:
			/* Normal subtransaction start */
			PushTransaction();
			s = CurrentTransactionState;	/* changed by push */
			if (name)
				s->name = MemoryContextStrdup(TopTransactionContext, name);
			break;
```

[../src/backend/access/transam/xact.c:5475](../src/backend/access/transam/xact.c#L5475) `PushTransaction` yeni bir `TransactionStateData` ayırır ve **xid vermez**:

```c
	s->fullTransactionId = InvalidFullTransactionId;	/* until assigned */
	s->subTransactionId = currentSubTransactionId;
	s->parent = p;
	s->nestingLevel = p->nestingLevel + 1;
```

`SubTransactionId` her top-level transaction'ın başında sıfırlanan bir sayaçtır; üst transaction 1'dir. README'nin ifadesiyle: "subtransactions do not have their own VXIDs; they use the parent top transaction's VXID."

## 3.2. Sub-commit gerçek bir commit değildir

[../src/backend/access/transam/xact.c:1706](../src/backend/access/transam/xact.c#L1706) `AtSubCommit_childXids`, xid'i ebeveynin listesine taşır:

```c
	s->parent->childXids[s->parent->nChildXids] = XidFromFullTransactionId(s->fullTransactionId);

	if (s->nChildXids > 0)
		memcpy(&s->parent->childXids[s->parent->nChildXids + 1],
			   s->childXids,
			   s->nChildXids * sizeof(TransactionId));
```

Yorumun açıkladığı gibi dizi sıralı kalır: "We rely on the fact that the XID of a child always follows that of its parent."

Bu liste en sonunda `RecordTransactionCommit` içinde COMMIT kaydına ve CLOG'a gider. Yani **alt transaction'ın commit'i, üst transaction commit edene kadar CLOG'a `COMMITTED` olarak yazılmaz.**

## 3.3. Sub-abort ise hemen CLOG'a yazılır

[../src/backend/access/transam/xact.c:5386](../src/backend/access/transam/xact.c#L5386) `AbortSubTransaction` içinden `RecordTransactionAbort(true)` çağrılır; o da [../src/backend/access/transam/xact.c:1893](../src/backend/access/transam/xact.c#L1893):

```c
	TransactionIdAbortTree(xid, nchildren, children);
```

ve ardından PGPROC cache'inden düşürür — [../src/backend/access/transam/xact.c:1907](../src/backend/access/transam/xact.c#L1907):

```c
	if (isSubXact)
		XidCacheRemoveRunningXids(xid, nchildren, children, latestXid);
```

Asimetri kasıtlıdır: abort'un kaybı zararsızdır (çökmede zaten abort varsayılır), commit'in kaybı veri kaybıdır.

## 3.4. Subxid cache ve overflow

[../src/include/storage/proc.h:31-43](../src/include/storage/proc.h#L31-L43)

```c
/*
 * Each backend advertises up to PGPROC_MAX_CACHED_SUBXIDS TransactionIds
 * for non-aborted subtransactions of its current top transaction.  These
 * have to be treated as running XIDs by other backends.
 *
 * We also keep track of whether the cache overflowed (ie, the transaction has
 * generated at least one subtransaction that didn't fit in the cache).
 * If none of the caches have overflowed, we can assume that an XID that's not
 * listed anywhere in the PGPROC array is not a running transaction.  Else we
 * have to look at pg_subtrans.
 *
 * See src/test/isolation/specs/subxid-overflow.spec if you change this.
 */
#define PGPROC_MAX_CACHED_SUBXIDS 64	/* XXX guessed-at value */
```

64 sabittir, GUC değildir. **Pratik sonuç:** bir transaction içinde 64'ten fazla xid almış subtransaction varsa (PL/pgSQL `EXCEPTION` blokları dahil), o backend'in overflow bayrağı kalkar ve sistemdeki her görünürlük kontrolü pg_subtrans I/O'suna açılır. Bu, üretimde "yavaşlayan ama neden yavaşladığı görünmeyen" sınıfının klasik nedenidir.

## 3.5. pg_subtrans — sadece ebeveyn işaretçisi

[../src/backend/access/transam/subtrans.c:6-20](../src/backend/access/transam/subtrans.c#L6-L20)

```
 * The pg_subtrans manager is a pg_xact-like manager that stores the parent
 * transaction Id for each transaction.  It is a fundamental part of the
 * nested transactions implementation.  A main transaction has a parent
 * of InvalidTransactionId, and each subtransaction has its immediate parent.
 * ...
 * There are no XLOG interactions since we do not care about preserving
 * data across crashes.
```

[../src/backend/access/transam/subtrans.c:55](../src/backend/access/transam/subtrans.c#L55)

```c
#define SUBTRANS_XACTS_PER_PAGE (BLCKSZ / sizeof(TransactionId))
```

8 kB sayfa → 2048 xid/sayfa. CLOG'un 32768'ine göre 16 kat daha fazla sayfa demektir; overflow senaryosunda I/O'nun neden fark edildiğinin bir nedeni budur.

Ağacın tepesini bulan fonksiyon [../src/backend/access/transam/subtrans.c:170](../src/backend/access/transam/subtrans.c#L170):

```c
TransactionId
SubTransGetTopmostTransaction(TransactionId xid)
{
	TransactionId parentXid = xid,
				previousXid = xid;

	/* Can't ask about stuff that might not be around anymore */
	Assert(TransactionIdFollowsOrEquals(xid, TransactionXmin));

	while (TransactionIdIsValid(parentXid))
	{
		previousXid = parentXid;
		if (TransactionIdPrecedes(parentXid, TransactionXmin))
			break;
		parentXid = SubTransGetParent(parentXid);
```

Döngü `TransactionXmin`'de durur — yorumun dediği gibi, o noktadan eskisi zaten görünürlük açısından fark etmez.

---

# 4. Commit protokolü — sıra neden bu

## 4.1. Üst seviye

[../src/backend/access/transam/xact.c:2270](../src/backend/access/transam/xact.c#L2270) `CommitTransaction` sırayla:

```
  1. AfterTriggerFireDeferred / PreCommit_Portals   (kullanıcı kodu, döngüde)
  2. RecordTransactionCommit()                      xact.c:2407  ← DURABILITY BURADA
  3. ProcArrayEndTransaction(MyProc, latestXid)     xact.c:2431  ← "artık koşmuyorum"
  4. ResourceOwnerRelease(..._BEFORE_LOCKS)         xact.c:2456
  5. AtEOXact_Inval(true)                           katalog invalidasyonu
  6. ResourceOwnerRelease(..._LOCKS)                xact.c:2480  ← KİLİTLER BURADA BIRAKILIR
  7. smgrDoPendingDeletes / AtCommit_Notify         dosya silme, NOTIFY
```

Sıranın gerekçesi kodun içinde yazılıdır — [../src/backend/access/transam/xact.c:2426-2431](../src/backend/access/transam/xact.c#L2426-L2431):

```c
	/*
	 * Let others know about no transaction in progress by me. Note that this
	 * must be done _before_ releasing locks we hold and _after_
	 * RecordTransactionCommit.
	 */
	ProcArrayEndTransaction(MyProc, latestXid);
```

İki kısıt:

- **`RecordTransactionCommit`'ten sonra:** Aksi halde başka bir backend bizi "koşmuyor" görüp CLOG'a bakabilir ve henüz yazılmamış durumu "crash olmuş" sanabilir.
- **Kilitleri bırakmadan önce:** Bizi bekleyen bir backend uyandığında, bizim tamamen bitmiş görünmemiz gerekir. Yorumun devamı: "We want to release locks at the point where any backend waiting for us will see our transaction as being fully cleaned up."

## 4.2. `RecordTransactionCommit` — WAL, sonra CLOG

[../src/backend/access/transam/xact.c:1345](../src/backend/access/transam/xact.c#L1345)

xid yoksa hiçbir şey yapılmaz:

```c
	TransactionId xid = GetTopTransactionIdIfAny();
	bool		markXidCommitted = TransactionIdIsValid(xid);
```

Bu tek satır, salt-okunur transaction'ların commit maliyetini sıfıra indirir.

xid varsa **kritik bölüm** açılır ve checkpoint geciktirilir:

```c
		Assert((MyProc->delayChkptFlags & DELAY_CHKPT_IN_COMMIT) == 0);
		START_CRIT_SECTION();
		MyProc->delayChkptFlags |= DELAY_CHKPT_IN_COMMIT;
```

Yorum nedeni açıklar: "Without this, it is possible for the checkpoint to set REDO after the XLOG record but fail to flush the pg_xact update to disk, leading to loss of the transaction commit if the system crashes a little later."

Sonra senkron yolda önce flush, sonra CLOG — [../src/backend/access/transam/xact.c:1544-1550](../src/backend/access/transam/xact.c#L1544-L1550):

```c
		XLogFlush(XactLastRecEnd);

		/*
		 * Now we may update the CLOG, if we wrote a COMMIT record above
		 */
		if (markXidCommitted)
			TransactionIdCommitTree(xid, nchildren, children);
```

`synchronous_commit = off` yolunda flush atlanır ve CLOG'a LSN ile birlikte yazılır — [../src/backend/access/transam/xact.c:1573](../src/backend/access/transam/xact.c#L1573):

```c
		if (markXidCommitted)
			TransactionIdAsyncCommitTree(xid, nchildren, children, XactLastRecEnd);
```

Bu LSN, CLOG sayfası diske yazılmadan önce WAL'ın o noktaya kadar flush edilmesini garanti eder — "write xlog before data" kuralının CLOG için uygulanışıdır.

## 4.3. Abort neden farklı

[../src/backend/access/transam/xact.c:1796](../src/backend/access/transam/xact.c#L1796) `RecordTransactionAbort` yorumu:

```
	 * We do not flush XLOG to disk here, since the default assumption after a
	 * crash would be that we aborted, anyway.  For the same reason, we don't
	 * need to worry about interlocking against checkpoint start.
```

`TRANSACTION_STATUS_IN_PROGRESS` (0b00) bit deseni hem "koşuyor" hem "çökmüş" anlamına gelir; ayrım ProcArray'e bakılarak yapılır. Bu yüzden abort kaydının kaybı zararsızdır.

---

# 5. CLOG — iki bit/transaction

## 5.1. Dört durum

[../src/include/access/clog.h:27-30](../src/include/access/clog.h#L27-L30)

```c
#define TRANSACTION_STATUS_IN_PROGRESS		0x00
#define TRANSACTION_STATUS_COMMITTED		0x01
#define TRANSACTION_STATUS_ABORTED			0x02
#define TRANSACTION_STATUS_SUB_COMMITTED	0x03
```

[../src/backend/access/transam/clog.c:64-66](../src/backend/access/transam/clog.c#L64-L66)

```c
#define CLOG_BITS_PER_XACT	2
#define CLOG_XACTS_PER_BYTE 4
#define CLOG_XACTS_PER_PAGE (BLCKSZ * CLOG_XACTS_PER_BYTE)
```

8 kB sayfada 32768 transaction. 2 milyar xid ≈ 512 MB pg_xact.

## 5.2. `TransactionIdSetTreeStatus` — atomiklik hilesi

[../src/backend/access/transam/clog.c:192](../src/backend/access/transam/clog.c#L192)

Sorun: bir transaction ağacının xid'leri farklı CLOG sayfalarına düşebilir ve tek kilit altında güncellenemez. Çözüm üç adımlıdır — [../src/backend/access/transam/clog.c:161-168](../src/backend/access/transam/clog.c#L161-L168):

```
 * In the commit case, atomicity is limited by whether all the subxids are in
 * the same CLOG page as xid.  If they all are, then the lock will be grabbed
 * only once, and the status will be set to committed directly.  Otherwise
 * we must
 *	 1. set sub-committed all subxids that are not on the same page as the
 *		main xid
 *	 2. atomically set committed the main xid and the subxids on the same page
 *	 3. go over the first bunch again and set them committed
```

`SUB_COMMITTED` durumunun tek amacı budur: geçici bir "ebeveynime bak" işareti. Kod:

```c
		if (status == TRANSACTION_STATUS_COMMITTED)
			set_status_by_pages(nsubxids - nsubxids_on_first_page,
								subxids + nsubxids_on_first_page,
								TRANSACTION_STATUS_SUB_COMMITTED, lsn);
```

Böylece dışarıdan bakan biri için **üst transaction'ın commit'i hâlâ atomiktir**: ya ana xid `COMMITTED`, ya değil.

## 5.3. Bit oynama

[../src/backend/access/transam/clog.c:669](../src/backend/access/transam/clog.c#L669) `TransactionIdSetStatusBit`:

```c
	byteptr = XactCtl->shared->page_buffer[slotno] + byteno;
	curval = (*byteptr >> bshift) & CLOG_XACT_BITMASK;
	...
	byteval = *byteptr;
	byteval &= ~(((1 << CLOG_BITS_PER_XACT) - 1) << bshift);
	byteval |= (status << bshift);
	*byteptr = byteval;
```

Okuma tarafı [../src/backend/access/transam/clog.c:743](../src/backend/access/transam/clog.c#L743) `TransactionIdGetStatus`.

## 5.4. Üstündeki katman — `TransactionLogFetch`

[../src/backend/access/transam/transam.c:51](../src/backend/access/transam/transam.c#L51)

Tek elemanlı bir cache var:

```c
	if (TransactionIdEquals(transactionId, cachedFetchXid))
		return cachedFetchXidStatus;
```

ve cache sadece **değişmeyecek** durumlar için doldurulur:

```c
	if (xidstatus != TRANSACTION_STATUS_IN_PROGRESS &&
		xidstatus != TRANSACTION_STATUS_SUB_COMMITTED)
	{
		cachedFetchXid = transactionId;
		cachedFetchXidStatus = xidstatus;
		cachedCommitLSN = xidlsn;
	}
```

[../src/backend/access/transam/transam.c:126](../src/backend/access/transam/transam.c#L126) `TransactionIdDidCommit` `SUB_COMMITTED` görürse pg_subtrans'a inip ebeveyni özyinelemeli sorar:

```c
		parentXid = SubTransGetParent(transactionId);
		if (!TransactionIdIsValid(parentXid))
		{
			elog(WARNING, "no pg_subtrans entry for subcommitted XID %u",
				 transactionId);
			return false;
		}
		return TransactionIdDidCommit(parentXid);
```

---

# 6. SLRU altyapısı

## 6.1. Yedi kullanıcı, tek kod

| SLRU | Dizin | Tanım |
|---|---|---|
| CLOG | `pg_xact` | [../src/backend/access/transam/clog.c:813](../src/backend/access/transam/clog.c#L813) |
| CommitTs | `pg_commit_ts` | [../src/backend/access/transam/commit_ts.c:553](../src/backend/access/transam/commit_ts.c#L553) |
| MultiXactOffset | `pg_multixact/offsets` | [../src/backend/access/transam/multixact.c:1787](../src/backend/access/transam/multixact.c#L1787) |
| MultiXactMember | `pg_multixact/members` | [../src/backend/access/transam/multixact.c:1802](../src/backend/access/transam/multixact.c#L1802) |
| SubTrans | `pg_subtrans` | [../src/backend/access/transam/subtrans.c:248](../src/backend/access/transam/subtrans.c#L248) |
| Notify | `pg_notify` | [../src/backend/commands/async.c:812](../src/backend/commands/async.c#L812) |
| Serial (SSI) | `pg_serial` | [../src/backend/storage/lmgr/predicate.c:1220](../src/backend/storage/lmgr/predicate.c#L1220) |

Kayıt şöyle görünür — [../src/backend/access/transam/clog.c:811-825](../src/backend/access/transam/clog.c#L811-L825):

```c
	SimpleLruRequest(.desc = &XactSlruDesc,
					 .name = "transaction",
					 .Dir = "pg_xact",
					 .long_segment_names = false,

					 .nslots = CLOGShmemBuffers(),
					 .nlsns = CLOG_LSNS_PER_PAGE,

					 .sync_handler = SYNC_HANDLER_CLOG,
					 .PagePrecedes = CLOGPagePrecedes,
					 .errdetail_for_io_error = clog_errdetail_for_io_error,

					 .buffer_tranche_id = LWTRANCHE_XACT_BUFFER,
					 .bank_tranche_id = LWTRANCHE_XACT_SLRU,
		);
```

## 6.2. Bank'lı LRU

[../src/backend/access/transam/slru.c:16-24](../src/backend/access/transam/slru.c#L16-L24)

```
 * We use a simple least-recently-used scheme to manage a pool of shared
 * page buffers, split in banks by the lowest bits of the page number, and
 * the management algorithm only processes the bank to which the desired
 * page belongs, so a linear search is sufficient; there's no need for a
 * hashtable or anything fancy.  The algorithm is straight LRU except that
 * we will never swap out the latest page (since we know it's going to be
 * hit again eventually).
```

Sayfa durumları — [../src/include/access/slru.h:36-39](../src/include/access/slru.h#L36-L39):

```c
	SLRU_PAGE_EMPTY,			/* buffer is not in use */
	SLRU_PAGE_READ_IN_PROGRESS, /* page is being read in */
	SLRU_PAGE_VALID,			/* page is valid and not being written */
	SLRU_PAGE_WRITE_IN_PROGRESS,	/* page is being written out */
```

## 6.3. `SimpleLruReadPage`

[../src/backend/access/transam/slru.c:550](../src/backend/access/transam/slru.c#L550)

```c
int
SimpleLruReadPage(SlruDesc *ctl, int64 pageno, bool write_ok,
				  const void *opaque_data)
{
	SlruShared	shared = ctl->shared;
	LWLock	   *banklock = SimpleLruGetBankLock(ctl, pageno);

	Assert(LWLockHeldByMeInMode(banklock, LW_EXCLUSIVE));

	/* Outer loop handles restart if we must wait for someone else's I/O */
	for (;;)
	{
		...
		slotno = SlruSelectLRUPage(ctl, pageno);

		/* Did we find the page in memory? */
		if (shared->page_status[slotno] != SLRU_PAGE_EMPTY &&
			shared->page_number[slotno] == pageno)
		{
```

Salt-okuma yolu için ayrı bir giriş noktası var: [../src/backend/access/transam/slru.c:654](../src/backend/access/transam/slru.c#L654) `SimpleLruReadPage_ReadOnly` — önce shared bank lock ile dener, ıskalarsa exclusive'e yükselir. `TransactionIdGetStatus` bunu kullanır.

## 6.4. Segment düzeni

[../src/include/pg_config_manual.h:30](../src/include/pg_config_manual.h#L30)

```c
#define SLRU_PAGES_PER_SEGMENT	32
```

32 sayfa × 8 kB = 256 kB'lık dosyalar. `pg_xact/0000`, `pg_xact/0001` … isimleri buradan gelir.

---

# 7. İki fazlı commit

## 7.1. `PrepareTransaction`

[../src/backend/access/transam/xact.c:2559](../src/backend/access/transam/xact.c#L2559)

`CommitTransaction`'a çok benzer ama üç kritik farkı vardır. İlki, durum dosyasının toplanması:

```c
	StartPrepare(gxact);

	AtPrepare_Notify();
	AtPrepare_Locks();
	AtPrepare_PredicateLocks();
	AtPrepare_PgStat();
	AtPrepare_MultiXact();
	AtPrepare_RelationMap();
	...
	EndPrepare(gxact);
```

İkincisi, kilitlerin **bırakılmayıp devredilmesi**:

```c
	/*
	 * Transfer our locks to a dummy PGPROC.  This has to be done before
	 * ProcArrayClearTransaction().  Otherwise, a GetLockConflicts() would
	 * conclude "xact already committed or aborted" for our locks.
	 */
	PostPrepare_Locks(fxid);
	...
	ProcArrayClearTransaction(MyProc);
```

Üçüncüsü: `PREPARE`'den sonra xid **hâlâ koşuyor sayılır**. Sahte bir PGPROC girdisi tutulur. [../src/backend/access/transam/twophase.c:23-26](../src/backend/access/transam/twophase.c#L23-L26):

```
 *		A global transaction (gxact) also has dummy PGPROC; this is what keeps
 *		the XID considered running by TransactionIdIsInProgress.  It is also
 *		convenient as a PGPROC to hook the gxact's locks to.
```

**Pratik sonuç:** unutulmuş bir prepared transaction, `xmin` horizon'unu sonsuza kadar geriye çiviler ve VACUUM'u durdurur. Bu, "prepared transaction unutuldu" arızasının klasik belirtisidir.

İki `PREPARE` yasağı vardır — geçici nesne kullanmak ve snapshot export etmiş olmak ([../src/backend/access/transam/xact.c:2664-2670](../src/backend/access/transam/xact.c#L2664-L2670)):

```c
	if (XactHasExportedSnapshots())
		ereport(ERROR,
				(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
				 errmsg("cannot PREPARE a transaction that has exported snapshots")));
```

## 7.2. Önce WAL, gerekirse dosya

[../src/backend/access/transam/twophase.c:1151](../src/backend/access/transam/twophase.c#L1151) `EndPrepare` durum verisini WAL'a yazar ve flush eder:

```c
	gxact->prepare_end_lsn = XLogInsert(RM_XACT_ID, XLOG_XACT_PREPARE);
	...
	XLogFlush(gxact->prepare_end_lsn);

	/* If we crash now, we have prepared: WAL replay will fix things */
```

`pg_twophase/` dosyası ancak checkpoint sırasında yazılır — [../src/backend/access/transam/twophase.c:1830](../src/backend/access/transam/twophase.c#L1830) `CheckPointTwoPhase`. Dosya adı [../src/backend/access/transam/twophase.c:956](../src/backend/access/transam/twophase.c#L956):

```c
static inline int
TwoPhaseFilePath(char *path, FullTransactionId fxid)
{
	return snprintf(path, MAXPGPATH, TWOPHASE_DIR "/%08X%08X",
					EpochFromFullTransactionId(fxid),
					XidFromFullTransactionId(fxid));
}
```

Yani `pg_twophase/0000000100000ABC` gibi 16 haneli adlar. Dosya biçimi [../src/backend/access/transam/twophase.c:963-976](../src/backend/access/transam/twophase.c#L963-L976)'da tanımlı: başlık, alt xid listesi, silinecek dosyalar, invalidation mesajları, rmgr kayıtları, CRC-32C.

## 7.3. Kurtarma

[../src/backend/access/transam/twophase.c:2091](../src/backend/access/transam/twophase.c#L2091) `RecoverPreparedTransactions` — recovery sonunda pg_twophase taranır, her gxact için sahte PGPROC yeniden kurulur ve kilitler geri alınır. `max_prepared_transactions = 0` (varsayılan) ise 2PC tamamen kapalıdır — [../src/backend/access/transam/twophase.c:378](../src/backend/access/transam/twophase.c#L378).

---

# 8. MultiXact

## 8.1. Ne zaman oluşur

Bir satırı **birden fazla** transaction aynı anda kilitlemek istediğinde. `xmax` alanı tek bir xid tutabilir; birden fazla kilitçi gerekince oraya bir MultiXactId yazılır ve `HEAP_XMAX_IS_MULTI` bayrağı kalkar.

[../src/backend/access/transam/multixact.c:6-17](../src/backend/access/transam/multixact.c#L6-L17)

```
 * The pg_multixact manager is a pg_xact-like manager that stores an array of
 * MultiXactMember for each MultiXactId.  It is a fundamental part of the
 * shared-row-lock implementation.  Each MultiXactMember is comprised of a
 * TransactionId and a set of flag bits.  The name is a bit historical:
 * originally, a MultiXactId consisted of more than one TransactionId (except
 * in rare corner cases), hence "multi".  Nowadays, however, it's perfectly
 * legitimate to have MultiXactIds that only include a single Xid.
```

Tetikleyen tipik SQL: `SELECT ... FOR SHARE` / `FOR KEY SHARE` ile aynı satıra birden fazla oturumun dokunması — yabancı anahtar kontrollerinin klasik deseni.

[../src/backend/access/transam/multixact.c:358](../src/backend/access/transam/multixact.c#L358)

```c
MultiXactId
MultiXactIdCreate(TransactionId xid1, MultiXactStatus status1,
				  TransactionId xid2, MultiXactStatus status2)
{
	MultiXactId newMulti;
	MultiXactMember members[2];
	...
	newMulti = MultiXactIdCreateFromMembers(2, members);
```

Var olan bir multi'ye üye eklemek **yeni bir multi yaratır** — [../src/backend/access/transam/multixact.c:411](../src/backend/access/transam/multixact.c#L411) `MultiXactIdExpand`:

```
 * Note that we do NOT actually modify the membership of a pre-existing
 * MultiXactId; instead we create a new one.  This is necessary to avoid
 * a race condition against code trying to wait for one MultiXactId to finish
```

Bu, yoğun FK trafiğinde multi üretiminin neden hızla arttığının nedenidir.

## 8.2. İki SLRU: offsets + members

Değişken uzunluklu diziyi sabit boyutlu sayfalara sığdırmak için iki alan kullanılır:

[../src/include/access/multixact_internal.h:32](../src/include/access/multixact_internal.h#L32)

```c
#define MULTIXACT_OFFSETS_PER_PAGE (BLCKSZ / sizeof(MultiXactOffset))
```

[../src/include/access/multixact_internal.h:53-77](../src/include/access/multixact_internal.h#L53-L77)

```
 * The situation for members is a bit more complex: we store one byte of
 * additional flag bits for each TransactionId.  To do this without getting
 * into alignment issues, we store four bytes of flags, and then the
 * corresponding 4 Xids.  Each such 5-word (20-byte) set we call a "group", and
 * are stored as a whole in pages.  Thus, with 8kB BLCKSZ, we keep 409 groups
 * per page.
```

```
   pg_multixact/offsets            pg_multixact/members
   ┌───────────────────┐           ┌──────────────────────────────┐
   │ multi 100 → 4200  │──────────►│ ... [xid,flags][xid,flags]...│
   │ multi 101 → 4203  │──────┐    │      4200  4201  4202        │
   │ multi 102 → 4205  │      └───►│      4203  4204              │
   └───────────────────┘           └──────────────────────────────┘
      8 byte / multi                 5 byte / member (grup halinde)
```

## 8.3. Member alanı — 20devel'de ne değişti

Bu ağaçta `MultiXactOffset` 64 bit'tir — [../src/include/c.h:807](../src/include/c.h#L807):

```c
typedef uint64 MultiXactOffset;
```

Sonuç: eski sürümlerin ünlü "multixact members limit exceeded" wraparound'u burada pratikte ortadan kalkmıştır. Kod artık sadece bir sağduyu kontrolü yapar — [../src/backend/access/transam/multixact.c:1106-1116](../src/backend/access/transam/multixact.c#L1106-L1116):

```c
	/*
	 * Offsets are 64-bit integers and will never wrap around.  Firstly, it
	 * would take an unrealistic amount of time and resources to consume 2^64
	 * offsets. ...
	 */
	if (nextOffset + nmembers < nextOffset)
		ereport(ERROR,
				(errcode(ERRCODE_PROGRAM_LIMIT_EXCEEDED),
				 errmsg("MultiXact members would wrap around")));
```

MultiXactId'nin kendisi hâlâ 32 bit'tir ve wraparound koruması sürer — [../src/backend/access/transam/multixact.c:1009-1018](../src/backend/access/transam/multixact.c#L1009-L1018) `multiVacLimit` / `multiWarnLimit` / `multiStopLimit` üçlüsü `GetNewTransactionId`'dekiyle aynı desendir.

## 8.4. Altı kilit modu

[../src/include/access/multixact.h:38-46](../src/include/access/multixact.h#L38-L46)

```c
	MultiXactStatusForKeyShare = 0x00,
	MultiXactStatusForShare = 0x01,
	MultiXactStatusForNoKeyUpdate = 0x02,
	MultiXactStatusForUpdate = 0x03,

	MultiXactStatusNoKeyUpdate = 0x04,

	MultiXactStatusUpdate = 0x05,
```

İlk dördü "sadece kilit", son ikisi gerçek güncelleme. `HEAP_XMAX_IS_MULTI` görüldüğünde görünürlük kodunun `HeapTupleGetUpdateXid` ile gerçek güncelleyiciyi çıkarması bu ayrımdan gelir.

---

# 9. Abort yolu ve ResourceOwner

## 9.1. İki fazlı abort

README'nin (satır 122-129) tarifi:

```
* AbortTransaction executes as soon as we realize the transaction has
  failed.  It should release all shared resources (locks etc) so that we do
  not delay other backends unnecessarily.
* CleanupTransaction executes when we finally see a user COMMIT
  or ROLLBACK command; it cleans things up and gets us out of the transaction
  completely.  In particular, we mustn't destroy TopTransactionContext until
  this point.
```

[../src/backend/access/transam/xact.c:2855](../src/backend/access/transam/xact.c#L2855) `AbortTransaction` ilk iş olarak **LWLock'ları** bırakır:

```c
	/*
	 * Release any LW locks we might be holding as quickly as possible.
	 * (Regular locks, however, must be held till we finish aborting.)
	 * Releasing LW locks is critical since we might try to grab them again
	 * while cleaning up!
	 */
	LWLockReleaseAll();
```

Ardından `UnlockBuffers()`, `XLogResetInsertion()`, `pgaio_error_cleanup()`, `ConditionVariableCancelSleep()` — hepsi "hata anında yarım kalmış olabilecek" durumları düzeltir. Sıra commit'inkiyle aynıdır: ABORT kaydı → `ProcArrayEndTransaction` → kilitler ([../src/backend/access/transam/xact.c:3004](../src/backend/access/transam/xact.c#L3004), [../src/backend/access/transam/xact.c:3028](../src/backend/access/transam/xact.c#L3028)).

## 9.2. ResourceOwner — üç fazlı bırakma

[../src/include/utils/resowner.h:52-57](../src/include/utils/resowner.h#L52-L57)

```c
typedef enum
{
	RESOURCE_RELEASE_BEFORE_LOCKS = 1,
	RESOURCE_RELEASE_LOCKS,
	RESOURCE_RELEASE_AFTER_LOCKS,
} ResourceReleasePhase;
```

BEFORE_LOCKS içinde sıra da önemlidir; öncelikler başlıkta sabittir:

[../src/include/utils/resowner.h:62-66](../src/include/utils/resowner.h#L62-L66)

```c
#define RELEASE_PRIO_BUFFER_IOS			    100
#define RELEASE_PRIO_BUFFER_PINS		    200
#define RELEASE_PRIO_RELCACHE_REFS			300
#define RELEASE_PRIO_DSMS					400
#define RELEASE_PRIO_JIT_CONTEXTS			500
```

[../src/backend/utils/resowner/resowner.c:685](../src/backend/utils/resowner/resowner.c#L685) `ResourceOwnerReleaseInternal` önce çocuklara iner, sonra kaynakları önceliğe göre sıralar:

```c
	/* Recurse to handle descendants */
	for (child = owner->firstchild; child != NULL; child = child->nextchild)
		ResourceOwnerReleaseInternal(child, phase, isCommit, isTopLevel);
```

Yapı, veri saklamak için 32 elemanlı bir dizi + hash tablo kullanır — [../src/backend/utils/resowner/resowner.c:11-16](../src/backend/utils/resowner/resowner.c#L11-L16):

```
 * The implementation consists of a small fixed-size array and a hash table.
 * New entries are inserted to the fixed-size array, and when the array
 * fills up, all the entries are moved to the hash table.
```

Her transaction ve her subtransaction kendi ResourceOwner'ına sahiptir (`s->curTransactionOwner`); `AssignTransactionId` bile xid kilidini doğru sahibe bağlamak için geçici olarak `CurrentResourceOwner`'ı değiştirir.

**`isCommit = true` ile çağrıldığında** bırakılan her kaynak için WARNING üretilir — "buffer refcount leak" gibi mesajların kaynağı burasıdır. Abort'ta (`false`) sessizce temizlenir.

---

# İzleme ve hata ayıklama

## Kim xid almış, kim almamış

```sql
SELECT pid, state, backend_xid, backend_xmin,
       now() - xact_start  AS xact_age,
       now() - state_change AS idle_age,
       left(query, 60)     AS query
FROM pg_stat_activity
WHERE backend_type = 'client backend'
ORDER BY xact_start NULLS LAST;
```

`backend_xid IS NULL` → salt-okunur, xid tahsis edilmemiş. Görünüm tanımı: [../src/backend/catalog/system_views.sql:939](../src/backend/catalog/system_views.sql#L939).

## Kendi xid'ini görmek

```sql
SELECT pg_current_xact_id_if_assigned();   -- NULL ise henüz yazma yapmadın
SELECT pg_current_xact_id();               -- ÇAĞIRINCA xid TAHSİS EDER
```

İkincisi yan etkilidir; salt-okunur bir oturumda çağırmayın. Tanımlar: [../src/include/catalog/pg_proc.dat:10821](../src/include/catalog/pg_proc.dat#L10821) ve [../src/include/catalog/pg_proc.dat:10824](../src/include/catalog/pg_proc.dat#L10824).

Bir xid'in akıbeti:

```sql
SELECT pg_xact_status('12345'::xid8);      -- in progress / committed / aborted
```

## Uzun süren transaction avı

```sql
SELECT pid, now() - xact_start AS age, state, backend_xid, backend_xmin, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND now() - xact_start > interval '5 minutes'
ORDER BY xact_start;
```

Otomatik önlem GUC'ları:

| GUC | Ne yapar |
|---|---|
| `idle_in_transaction_session_timeout` | Boşta bekleyen açık transaction'ı öldürür |
| `transaction_timeout` | Toplam transaction süresini sınırlar ([../src/backend/access/transam/xact.c:2258](../src/backend/access/transam/xact.c#L2258) `enable_timeout_after`) |
| `statement_timeout` | Tek komut süresi |

## Unutulmuş prepared transaction

```sql
SELECT gid, prepared, owner, database,
       now() - prepared AS age
FROM pg_prepared_xacts
ORDER BY prepared;
```

Görünüm: [../src/backend/catalog/system_views.sql:459](../src/backend/catalog/system_views.sql#L459). Boşsa ve `max_prepared_transactions = 0` ise 2PC kapalıdır. Temizlik: `ROLLBACK PREPARED '<gid>';`

## Subxid overflow'u yakalamak

Doğrudan gösteren bir görünüm yok; dolaylı işaretler:

```sql
-- pg_subtrans'a düşen okumalar artıyor mu?
SELECT name, blks_zeroed, blks_hit, blks_read, blks_written, blks_exists
FROM pg_stat_slru
ORDER BY blks_read DESC;
```

`pg_stat_slru` görünümü: [../src/backend/catalog/system_views.sql:993](../src/backend/catalog/system_views.sql#L993). `name = 'subtransaction'` satırında `blks_read` hızla artıyorsa, bir yerde 64'ten fazla xid'li subtransaction üreten bir döngü vardır (çoğunlukla PL/pgSQL `BEGIN ... EXCEPTION` içeren bir `LOOP`).

`SubtransSLRU` / `SubtransBuffer` wait event'leri de aynı işarettir:

```sql
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
GROUP BY 1, 2 ORDER BY 3 DESC;
```

## SLRU boyut GUC'ları

| GUC | Dosya |
|---|---|
| `transaction_buffers` | [../src/backend/utils/misc/guc_parameters.dat:3241](../src/backend/utils/misc/guc_parameters.dat#L3241) |
| `subtransaction_buffers` | [../src/backend/utils/misc/guc_parameters.dat:2915](../src/backend/utils/misc/guc_parameters.dat#L2915) |
| `multixact_offset_buffers` | [../src/backend/utils/misc/guc_parameters.dat:2283](../src/backend/utils/misc/guc_parameters.dat#L2283) |
| `multixact_member_buffers` | [../src/backend/utils/misc/guc_parameters.dat:2273](../src/backend/utils/misc/guc_parameters.dat#L2273) |
| `commit_timestamp_buffers` | [../src/backend/utils/misc/guc_parameters.dat:501](../src/backend/utils/misc/guc_parameters.dat#L501) |
| `max_prepared_transactions` | [../src/backend/utils/misc/guc_parameters.dat:2117](../src/backend/utils/misc/guc_parameters.dat#L2117) |

Hepsi `PGC_POSTMASTER` — restart gerektirir. `0` verilirse `shared_buffers`'a göre otomatik ayarlanır ([../src/backend/access/transam/clog.c:776](../src/backend/access/transam/clog.c#L776) `CLOGShmemBuffers`).

## Durum makinesini canlı izlemek

```sql
SET client_min_messages = DEBUG5;
BEGIN;
SAVEPOINT s1;
INSERT INTO t VALUES (1);
ROLLBACK TO s1;
COMMIT;
```

Her adımda [../src/backend/access/transam/xact.c:5707](../src/backend/access/transam/xact.c#L5707) `ShowTransactionState` şu biçimde satır basar:

```
StartSubTransaction(2) name: s1; blockState: SUBBEGIN; state: INPROGRESS, xid/subid/cid: 0/2/0
```

Metinler [../src/backend/access/transam/xact.c:5766](../src/backend/access/transam/xact.c#L5766) `BlockStateAsString` ve [../src/backend/access/transam/xact.c:5819](../src/backend/access/transam/xact.c#L5819) `TransStateAsString`'ten gelir. Durum makinesini öğrenmenin en hızlı yolu budur.

## MultiXact incelemek

```sql
-- xmax bir multi mi?
SELECT ctid, xmin, xmax FROM t WHERE ...;

-- üyeleri
SELECT * FROM pg_get_multixact_members('12345'::xid);
```

Fonksiyon: [../src/include/catalog/pg_proc.dat:6656](../src/include/catalog/pg_proc.dat#L6656). `mode` kolonu yukarıdaki altı `MultiXactStatus` değerinden birini verir.

---

# Tek sayfalık özet

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  KOMUT DÖNGÜSÜ                          postgres.c her komut için çağırır    ║
║    StartTransactionCommand  →  komut  →  CommitTransactionCommand            ║
║         xact.c:3112                           xact.c:3210                    ║
╚══════════════════════════════════════════════════════════════════════════════╝
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
  StartTransaction            AssignTransactionId          CommitTransaction
  xact.c:2106                 xact.c:637                   xact.c:2270
  ┌────────────────┐          ┌──────────────────┐         ┌────────────────────┐
  │ vxid ata       │          │ ebeveynlere xid  │         │ 1 WAL COMMIT       │
  │ VirtualXact-   │          │ GetNewTxnId()    │         │ 2 XLogFlush        │
  │  LockTableIns. │          │ SubTransSetParent│         │ 3 CLOG = 01        │
  │ PGPROC.vxid.lxid│         │ XactLockTableIns.│         │ 4 ProcArray temizle│
  │ xid YOK        │          │ PGPROC.xid yaz   │         │ 5 kilitleri bırak  │
  └────────────────┘          └──────────────────┘         └────────────────────┘
                                       │                          │
                                       │                    RecordTransactionCommit
                                       │                    xact.c:1345
                                       ▼
                          ┌──────────────────────────┐
                          │  SUBTRANSACTION YIĞINI   │
                          │  PushTransaction 5475    │
                          │  StartSubTransaction 5121│
                          │  sub-commit  → childXids │
                          │  sub-abort   → CLOG'a 10 │
                          │  cache: 64 subxid/PGPROC │
                          │  dolarsa → overflowed    │
                          └──────────────────────────┘

╔══════════════════════════════════════════════════════════════════════════════╗
║  KALICI DURUM — hepsi slru.c üzerinde                                        ║
║                                                                              ║
║   pg_xact       2 bit/xid    00 running  01 committed                        ║
║   (CLOG)        32768/sayfa  10 aborted  11 sub-committed                    ║
║   clog.c:192    TransactionIdSetTreeStatus — çok sayfalı ağaç için 3 adım    ║
║                                                                              ║
║   pg_subtrans   4 byte/xid   ebeveyn işaretçisi; crash'te korunmaz           ║
║   subtrans.c:170  SubTransGetTopmostTransaction — TransactionXmin'de durur   ║
║                                                                              ║
║   pg_multixact  offsets (8B/multi) + members (5B/üye, 4'lü gruplar)          ║
║   multixact.c:358  MultiXactIdCreate — üye eklemek YENİ multi yaratır        ║
║   20devel: MultiXactOffset = uint64 → member wraparound pratikte yok         ║
║                                                                              ║
║   pg_commit_ts  track_commit_timestamp açıksa commit zamanı + origin         ║
║   pg_twophase   PREPARE durum dosyası; önce WAL, checkpoint'te dosya         ║
╚══════════════════════════════════════════════════════════════════════════════╝

  SIRA KURALLARI (bozulursa ne olur)
  ──────────────────────────────────
  WAL flush  →  CLOG          CLOG önce yazılsa: çökmede commit "olmuş" görünür
  CLOG       →  ProcArray     ProcArray önce temizlense: okuyan "crash" sanar
  ProcArray  →  kilitler      kilit önce bırakılsa: bekleyen yarım durum görür
  parent xid →  child xid     tersi olsa: xid sırası varsayımları çöker
  pg_subtrans→  paylaşımlı    tersi olsa: overflow'da xid "koşmuyor" görünür

  MALİYET TABLOSU
  ───────────────
  Salt-okunur transaction    : vxid + snapshot. xid yok, CLOG yok, WAL yok.
  Yazan transaction          : + xid, + CLOG 2 bit, + COMMIT kaydı + flush
  64'ten az subtransaction   : + PGPROC cache girdisi (bedava sayılır)
  64'ten fazla subtransaction: + overflow → SİSTEM GENELİNDE pg_subtrans I/O
  Prepared transaction       : + pg_twophase, + sonsuza kadar koşuyor sayılır
  MultiXact                  : + iki SLRU sayfası, + her genişletmede yeni multi
```
