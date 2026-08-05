# PostgreSQL kilit mekanizmaları — kodun içinden

> Bu depo: PostgreSQL **20devel**.
> Kardeş dosyalar: [POSTGRES-NASIL-CALISIR.md](POSTGRES-NASIL-CALISIR.md) · [SELECT-NASIL-CALISIR.md](SELECT-NASIL-CALISIR.md)
> Ana kaynak: [src/backend/storage/lmgr/README](../src/backend/storage/lmgr/README)

---

## 30 saniyelik özet

PostgreSQL'de **dört** ayrı süreçler-arası kilit türü var. Aynı şey değiller, birbirinin yerine geçmezler.

```
                     ne kadar tutulur?          deadlock tespiti?   otomatik release?
┌────────────────┬──────────────────────────┬──────────────────┬──────────────────┐
│ Spinlock       │ birkaç makine komutu     │ yok              │ yok              │
│ LWLock         │ birkaç yüz komut         │ yok              │ elog() sırasında │
│ Heavyweight    │ transaction sonuna kadar │ VAR              │ VAR              │
│ SIReadLock     │ commit sonrasına kadar   │ n/a (bloklamaz)  │ VAR              │
└────────────────┴──────────────────────────┴──────────────────┴──────────────────┘
        │                │                        │                    │
   s_lock.c          lwlock.c                  lock.c              predicate.c
   (atomik TAS)   (atomik CAS + semafor)   (shared hash tablo)   (SSI, çakışma grafiği)
```

**Beş cümlelik model:**

1. **Spinlock** = donanım test-and-set, meşgul döngü. LWLock altyapısı için var.
2. **LWLock** = paylaşımlı bellekteki veri yapılarını korur (buffer pool, hash tablolar). Shared/exclusive.
3. **Heavyweight lock** = kullanıcının gördüğü kilit. `LOCK TABLE`, `SELECT FOR UPDATE`, DDL. 8 mod, çakışma matrisi, deadlock dedektörü.
4. **Tuple lock** = satır kilidi tuple header'ında saklanır (XMAX + infomask), kilit tablosunda değil. Milyonlarca satır kilitlenebilsin diye.
5. **SIReadLock** = SERIALIZABLE için predicate lock. Kimseyi bloklamaz, sadece çakışma grafiğine kenar ekler.

Bu dosya çoğunlukla **3'ü** anlatır, çünkü kod hacminin çoğu orada.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [src/backend/storage/lmgr/lock.c](../src/backend/storage/lmgr/lock.c) | Heavyweight lock manager çekirdeği (154 KB) |
| [src/backend/storage/lmgr/lmgr.c](../src/backend/storage/lmgr/lmgr.c) | `LockRelationOid` gibi kullanıcı-dostu sarmalayıcılar |
| [src/backend/storage/lmgr/proc.c](../src/backend/storage/lmgr/proc.c) | Bekleme kuyruğu, uyku, uyandırma (`ProcSleep`) |
| [src/backend/storage/lmgr/deadlock.c](../src/backend/storage/lmgr/deadlock.c) | Waits-for grafiği, döngü arama, topolojik sıralama |
| [src/backend/storage/lmgr/lwlock.c](../src/backend/storage/lmgr/lwlock.c) | Hafif kilitler |
| [src/backend/storage/lmgr/s_lock.c](../src/backend/storage/lmgr/s_lock.c) | Spinlock gecikme döngüsü |
| [src/backend/storage/lmgr/predicate.c](../src/backend/storage/lmgr/predicate.c) | SSI / predicate lock (168 KB) |
| [src/backend/storage/lmgr/condition_variable.c](../src/backend/storage/lmgr/condition_variable.c) | Koşul değişkenleri (kilit değil, bekleme aracı) |
| [src/include/storage/lockdefs.h](../src/include/storage/lockdefs.h) | 8 kilit modunun tanımı |
| [src/include/storage/locktag.h](../src/include/storage/locktag.h) | Kilitlenebilir nesne türleri |
| [src/backend/access/heap/README.tuplock](../src/backend/access/heap/README.tuplock) | Satır kilitleri + MultiXact |

---

# 1. Spinlock — en alt katman

[src/include/storage/spin.h](../src/include/storage/spin.h)

```c
void SpinLockInit(volatile slock_t *lock)
void SpinLockAcquire(volatile slock_t *lock)   /* meşgul döngü, ~1 dk sonra abort() */
void SpinLockRelease(volatile slock_t *lock)
```

Kuralları sert:

- **Birkaç düzine komuttan fazla tutulamaz.** Kernel çağrısı yasak, önemsiz olmayan fonksiyon çağrısı yasak.
- Deadlock tespiti **yok**. Hata durumunda otomatik bırakma **yok**.
- `CHECK_FOR_INTERRUPTS()` spinlock altında olamaz varsayılır.

Bekleme stratejisi [src/backend/storage/lmgr/s_lock.c:57-60](../src/backend/storage/lmgr/s_lock.c#L57-L60):

```c
#define MIN_SPINS_PER_DELAY 10
#define MAX_SPINS_PER_DELAY 1000
#define NUM_DELAYS          1000
#define MIN_DELAY_USEC      1000L
```

Önce sıkı döngü (tek CPU mu çok CPU mu diye adapte olur), sonra rastgele artan `pg_usleep()` — 1 ms'den başlar, ~1 saniyeye kadar çıkar, sıfırlanır. 1000 gecikmeden sonra **"stuck spinlock"** hatası verip abort eder. Bu ~2 dakika demek; yani gerçekten bir hata durumu.

Rastgele artışın sebebi: sabit 1 ms uyku kullanılırsa, kilidi tutan süreç scheduler tarafından düşük önceliğe atılmışsa hiç çalışamaz → açlık.

---

# 2. LWLock — paylaşımlı bellek yapılarının kilidi

[src/backend/storage/lmgr/lwlock.c](../src/backend/storage/lmgr/lwlock.c)

Shared buffer pool girişleri, hash tabloları, WAL insert slotları — hepsi LWLock ile korunur.

## Durum tek bir atomik `uint32`'te

[src/backend/storage/lmgr/lwlock.c:96-108](../src/backend/storage/lmgr/lwlock.c#L96-L108)

```c
#define LW_FLAG_HAS_WAITERS       ((uint32) 1 << 31)
#define LW_FLAG_WAKE_IN_PROGRESS  ((uint32) 1 << 30)
#define LW_FLAG_LOCKED            ((uint32) 1 << 29)

#define LW_VAL_EXCLUSIVE          (MAX_BACKENDS + 1)
#define LW_VAL_SHARED             1

#define LW_SHARED_MASK            MAX_BACKENDS
#define LW_LOCK_MASK              (MAX_BACKENDS | LW_VAL_EXCLUSIVE)
```

Tek `state` değişkeni:

```
 bit 31 30 29 │ 28 ......................... 0
      W  P  L │ exclusive sentinel + shared sayaç
      │  │  └── bekleme kuyruğu kilidi
      │  └───── uyandırma sürüyor
      └──────── bekleyen var
```

- **Shared kilit** = sayacı 1 artır.
- **Exclusive kilit** = `LW_VAL_EXCLUSIVE` sentinel değerini yerleştir. `MAX_BACKENDS` bir 2'nin kuvveti eksi bir olduğu için sentinel asla shared sayaç aralığıyla çakışmaz.

Eskiden shared/exclusive için ayrı sayaçlar vardı ve bunları bir spinlock korurdu. Çok sık shared alınan kilitlerde spinlock'ta dönmek darboğaz oldu — bu yüzden tek atomik değişkene geçildi. Çakışma yoksa shared alım **wait-free**.

## Dört fazlı alım

[src/backend/storage/lmgr/lwlock.c:64-73](../src/backend/storage/lmgr/lwlock.c#L64-L73) — koddaki yorumun kendisi:

```
Faz 1: Atomik CAS ile dene, olduysa bitti
Faz 2: Kendini kilidin bekleme kuyruğuna ekle
Faz 3: Tekrar dene; olduysa kendini kuyruktan çıkar
Faz 4: Uyandırılana kadar uyu, Faz 1'e dön
```

Faz 3 neden var: yarış durumu. "Beklemem gerek" diye karar verdikten sonra, kuyruğa girene kadar kilit sahibi işini bitirmiş olabilir. O zaman OS'ta sonsuza kadar uyursun. Faz 2'den sonra kuyrukta olduğun için kimse "çok hızlı" bırakamaz.

Ana fonksiyonlar:

| Fonksiyon | Satır |
|---|---|
| `LWLockAttemptLock` | [lwlock.c:764](../src/backend/storage/lmgr/lwlock.c#L764) |
| `LWLockQueueSelf` | [lwlock.c:1018](../src/backend/storage/lmgr/lwlock.c#L1018) |
| `LWLockDequeueSelf` | [lwlock.c:1061](../src/backend/storage/lmgr/lwlock.c#L1061) |
| `LWLockAcquire` | [lwlock.c:1150](../src/backend/storage/lmgr/lwlock.c#L1150) |
| `LWLockConditionalAcquire` | [lwlock.c:1321](../src/backend/storage/lmgr/lwlock.c#L1321) |
| `LWLockWaitForVar` | [lwlock.c:1566](../src/backend/storage/lmgr/lwlock.c#L1566) |
| `LWLockRelease` | [lwlock.c:1767](../src/backend/storage/lmgr/lwlock.c#L1767) |
| `LWLockReleaseAll` | [lwlock.c:1866](../src/backend/storage/lmgr/lwlock.c#L1866) |

## Kritik davranış farkları

- Bekleyen süreç **SysV semaforunda** bloklanır, CPU yakmaz.
- Sıra **geliş sırasına göre** verilir. Timeout **yok**.
- LWLock (veya spinlock) tutulurken **query cancel ve die() ertelenir**. Yani `Ctrl+C` LWLock beklerken çalışmaz. Bu yüzden bekleme süresi birkaç saniyeyi geçebilecek yerlerde LWLock kullanmak kötü fikir.
- `elog()` sırasında otomatik bırakılır — bu yüzden LWLock tutarken hata fırlatmak güvenli.

---

# 3. Heavyweight lock — kullanıcının gördüğü kilit

## 3.1. Sekiz mod

[src/include/storage/lockdefs.h:35-46](../src/include/storage/lockdefs.h#L35-L46)

```c
#define NoLock                   0   /* kilit alma demek, mod değil */
#define AccessShareLock          1   /* SELECT */
#define RowShareLock             2   /* SELECT FOR UPDATE/FOR SHARE */
#define RowExclusiveLock         3   /* INSERT, UPDATE, DELETE */
#define ShareUpdateExclusiveLock 4   /* VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY */
#define ShareLock                5   /* CREATE INDEX (CONCURRENTLY olmadan) */
#define ShareRowExclusiveLock    6   /* EXCLUSIVE gibi ama ROW SHARE'e izin verir */
#define ExclusiveLock            7   /* ROW SHARE / SELECT...FOR UPDATE'i bloklar */
#define AccessExclusiveLock      8   /* ALTER TABLE, DROP TABLE, VACUUM FULL, LOCK TABLE */
```

## 3.2. Çakışma matrisi

[src/backend/storage/lmgr/lock.c:68-107](../src/backend/storage/lmgr/lock.c#L68-L107) — kodda bitmask dizisi olarak duruyor:

```c
static const LOCKMASK LockConflicts[] = {
    0,
    /* AccessShareLock */
    LOCKBIT_ON(AccessExclusiveLock),
    /* RowShareLock */
    LOCKBIT_ON(ExclusiveLock) | LOCKBIT_ON(AccessExclusiveLock),
    ...
```

Okunabilir hali — **X = çakışır**:

```
tutulan →      ACCESS  ROW    ROW    SHARE    SHARE  SHARE ROW  EXCL   ACCESS
istenen ↓      SHARE   SHARE  EXCL   UPD EXCL        EXCL              EXCL
─────────────────────────────────────────────────────────────────────────────
ACCESS SHARE                                                             X
ROW SHARE                                                          X     X
ROW EXCL                                       X        X          X     X
SHARE UPD EXCL                        X        X        X          X     X
SHARE                  .      X       X        X        X          X     X
SHARE ROW EXCL                X       X        X        X          X     X
EXCLUSIVE              X      X       X        X        X          X     X
ACCESS EXCL      X     X      X       X        X        X          X     X
```

Pratik okumalar:

- `SELECT` (ACCESS SHARE) sadece `ACCESS EXCLUSIVE` ile çakışır → `DROP TABLE`/`ALTER TABLE` dışında hiçbir şey SELECT'i bloklamaz.
- `INSERT/UPDATE/DELETE` (ROW EXCLUSIVE) birbiriyle **çakışmaz** → tablo seviyesinde paralel DML serbest. Satır seviyesindeki çakışmalar ayrı mekanizma (bkz. Bölüm 5).
- `SHARE UPDATE EXCLUSIVE` **kendisiyle** çakışır → aynı anda iki `VACUUM` aynı tabloda çalışamaz.
- Çakışma kontrolü tek satır: [lock.c:624](../src/backend/storage/lmgr/lock.c#L624)
  ```c
  if (lockMethodTable->conflictTab[mode1] & LOCKBIT_ON(mode2))
  ```

**Bir süreç kendisiyle asla çakışmaz.** Zaten exclusive tutuyorken read lock alabilir.

## 3.3. Neyi kilitleyebilirsin — LOCKTAG

[src/include/storage/locktag.h:36-51](../src/include/storage/locktag.h#L36-L51)

```c
typedef enum LockTagType
{
    LOCKTAG_RELATION,            /* tüm tablo */
    LOCKTAG_RELATION_EXTEND,     /* tabloyu büyütme hakkı */
    LOCKTAG_DATABASE_FROZEN_IDS, /* pg_database.datfrozenxid */
    LOCKTAG_PAGE,                /* bir sayfa */
    LOCKTAG_TUPLE,               /* bir fiziksel satır */
    LOCKTAG_TRANSACTION,         /* xact bitene kadar bekleme */
    LOCKTAG_VIRTUALTRANSACTION,  /* sanal xact */
    LOCKTAG_SPECULATIVE_TOKEN,   /* speculative insert (ON CONFLICT) */
    LOCKTAG_OBJECT,              /* tablo olmayan DB nesnesi */
    LOCKTAG_USERLOCK,            /* eski contrib/userlock için ayrılmış */
    LOCKTAG_ADVISORY,            /* advisory lock */
    LOCKTAG_APPLY_TRANSACTION,   /* logical replication subscriber */
} LockTagType;
```

`LOCKTAG` **tam 16 byte, padding yok** — [locktag.h:64-72](../src/include/storage/locktag.h#L64-L72). Padding olsaydı hash hesabı rastgele olurdu. Üç adet 32-bit + bir 16-bit + iki 8-bit alan:

```c
typedef struct LOCKTAG
{
    uint32  locktag_field1;      /* tablo için: DB OID (shared ise 0) */
    uint32  locktag_field2;      /* tablo için: REL OID */
    uint32  locktag_field3;
    uint16  locktag_field4;
    uint8   locktag_type;
    uint8   locktag_lockmethodid;
} LOCKTAG;
```

## 3.4. Üç veri yapısı

```
      ┌─────────────────────────────────────────────────────────────┐
      │  PAYLAŞIMLI BELLEK                                          │
      │                                                             │
      │   LOCK (nesne başına 1)          PROCLOCK (nesne×backend)   │
      │   ┌──────────────────┐           ┌────────────────────┐     │
      │   │ tag  (16 byte)   │◄──────────┤ tag.myLock         │     │
      │   │ grantMask        │           │ tag.myProc  ───────┼──►  │
      │   │ waitMask         │           │ holdMask           │ PGPROC
      │   │ procLocks   ─────┼──────────►│ releaseMask        │     │
      │   │ waitProcs        │           │ lockLink, procLink │     │
      │   │ nRequested       │           └────────────────────┘     │
      │   │ requested[8]     │                                      │
      │   │ nGranted         │                                      │
      │   │ granted[8]       │                                      │
      │   └──────────────────┘                                      │
      └─────────────────────────────────────────────────────────────┘
                    ▲
                    │  her backend kendi özel kopyasını tutar
      ┌─────────────┴───────────────────────────────────────────────┐
      │  BACKEND-YEREL (paylaşımlı değil)                           │
      │   LOCALLOCK                                                 │
      │   ┌────────────────────────────────────┐                    │
      │   │ tag = LOCKTAG + mode               │                    │
      │   │ nLocks      ← kaç kez alındı       │                    │
      │   │ lockOwners[]← hangi ResourceOwner  │                    │
      │   │ hashcode    ← partition hesabı için│                    │
      │   └────────────────────────────────────┘                    │
      └─────────────────────────────────────────────────────────────┘
```

**LOCK** alanları ([README](../src/backend/storage/lmgr/README) satır ~90):

- `grantMask` — hangi modların **verilmiş** olduğunun bitmask'i. Bit i = 1 ⟺ `granted[i] > 0`.
- `waitMask` — hangi modların **beklendiği**. Bit i = 1 ⟺ `requested[i] > granted[i]`.
- `procLocks` — bu nesnenin tüm PROCLOCK'ları (verilmiş **ve** bekleyen).
- `waitProcs` — uyuyan backend'lerin PGPROC kuyruğu.
- `requested[]` / `granted[]` — mod başına sayaçlar. Tümü sıfıra düşünce LOCK objesi silinir.

Değişmez: `0 <= nGranted <= nRequested` ve her i için `0 <= granted[i] <= requested[i]`.

**LOCALLOCK neden var:** paylaşımlı yapılar nesne+mod+backend başına **tek** bir izin kaydı tutar. Ama bir transaction aynı kilidi 50 kez alıp bırakabilir, hem session hem transaction seviyesinde tutabilir. Bu sayaçlar LOCALLOCK'ta tutulur ki paylaşımlı belleğe dokunmak gerekmesin.

## 3.5. Partition'lama — 8.2 öncesindeki darboğaz

PostgreSQL 8.2'den önce tüm lock manager yapıları **tek bir LWLock** (LockMgrLock) tarafından korunuyordu. Beklenebileceği gibi darboğaz oldu.

[src/include/storage/lwlock.h:86-91](../src/include/storage/lwlock.h#L86-L91)

```c
#define LOG2_NUM_LOCK_PARTITIONS  4
#define NUM_LOCK_PARTITIONS  (1 << LOG2_NUM_LOCK_PARTITIONS)   /* = 16 */
```

- Her kilit, `LOCKTAG` hash'inin **düşük bitlerine** göre 16 partition'dan birine düşer.
- Her partition'ın kendi LWLock'u var. Normal alım/bırakma sadece **tek** partition'ı kilitler.
- PROCLOCK'ların hash'i, ilişkili LOCK ile **aynı düşük bitlere** sahip olmak zorunda → özel hash fonksiyonu `proclock_hash`.
- PGPROC'un PROCLOCK listesi de partition başına bölünmüş: [src/include/storage/proc.h:327](../src/include/storage/proc.h#L327) `dlist_head myProcLocks[NUM_LOCK_PARTITIONS];`

**Kural:** birden fazla partition kilitlemek gerekiyorsa **partition numarası sırasına göre** kilitle. Yoksa LWLock deadlock'u olur. Deadlock dedektörü basitlik adına 16'sını da sırayla kilitler.

## 3.6. Fast path — asıl performans hilesi

Problem: kilitler partition'lansa bile, **belirli bir tablo** hep aynı partition'a düşer. Aynı tabloya vuran binlerce kısa sorgu → o partition lock'u darboğaz. Bu etki 2 çekirdekli sunucuda bile ölçülebiliyor.

Çözüm (9.2'den beri): **"zayıf" kilitleri hiç paylaşımlı tabloya yazma.**

Uygunluk şartları:

1. DEFAULT lockmethod
2. Shared olmayan bir **tablo** kilidi
3. **Zayıf** mod: `AccessShareLock`, `RowShareLock`, `RowExclusiveLock`
4. Çakışan bir kilit olamayacağı hızlıca doğrulanabilmeli

Slotlar PGPROC içinde — [src/include/storage/proc.h:101-104](../src/include/storage/proc.h#L101-L104):

```c
#define FP_LOCK_GROUPS_PER_BACKEND_MAX  1024
#define FP_LOCK_SLOTS_PER_GROUP         16   /* don't change */
```

**Nasıl çakışma kontrolü yapılıyor** — 1024 tamsayı sayaçlık bir dizi:

```
FastPathStrongRelationLocks->count[1024]
        │
        └── "güçlü" kilit sayısı (ShareLock, ShareRowExclusiveLock,
            ExclusiveLock, AccessExclusiveLock) — 1024 yönlü partition

    count[i] != 0  →  o partition'da fast path KULLANILAMAZ
```

Akış [src/backend/storage/lmgr/lock.c:978-1015](../src/backend/storage/lmgr/lock.c#L978-L1015):

```c
if (EligibleForRelationFastPath(locktag, lockmode))
{
    LWLockAcquire(&MyProc->fpInfoLock, LW_EXCLUSIVE);
    if (FastPathStrongRelationLocks->count[fasthashcode] != 0)
        acquired = false;                    /* güçlü kilit var, normal yola git */
    else
        acquired = FastPathGrantRelationLock(locktag->locktag_field2, lockmode);
    LWLockRelease(&MyProc->fpInfoLock);
    if (acquired)
    {
        locallock->lock = NULL;              /* bayat pointer'lar temizlenmeli */
        locallock->proclock = NULL;
        GrantLockLocal(locallock, owner);
        return LOCKACQUIRE_OK;               /* paylaşımlı tabloya hiç dokunmadık */
    }
}
```

**Güçlü kilit isteyen taraf ne yapar** [lock.c:1032-1040](../src/backend/storage/lmgr/lock.c#L1032-L1040):

```c
if (ConflictsWithRelationFastPath(locktag, lockmode))
{
    BeginStrongLockAcquire(locallock, fasthashcode);       /* sayacı artır */
    if (!FastPathTransferRelationLocks(lockMethodTable, locktag, hashcode))
        ...                                                /* her backend'in dizisini tara,
                                                              eşleşenleri ana tabloya taşı */
}
```

**Bellek senkronizasyonu nasıl garanti ediliyor:** LWLock alımı bir *memory sequence point*. Zayıf kilit alan backend, `FastPathStrongRelationLocks`'a bakmadan **önce** kendi `fpInfoLock`'unu alır. Güçlü kilit alan backend ise, transfer için **her** backend'in fast-path LWLock'unu sırayla alır. Yani sıfır görürsek: ya gerçekten sıfırdır, ya da güçlü kilitçi henüz bizim tuttuğumuz LWLock'u almamıştır — aldığında bizim kilidimizi görecektir.

**VXID kilitleri** ayrı: `FastPathStrongRelationLocks` tablosunu hiç kullanmazlar. Bir VXID üzerindeki ilk kilit her zaman sahibinin aldığı ExclusiveLock'tur, dolayısıyla **her zaman** kontrolsüz fast path'ten alınabilir. Sonraki kilitçiler share bekleyicilerdir. VXID kilitlerinin lock manager'ı kullanmasının tek sebebi zaten **deadlock tespiti**.

Deadlock dedektörü fast-path yapılarına bakmaz — deadlock'a karışabilecek her kilit zaten önceden ana tabloya taşınmış olmak zorunda.

## 3.7. `LockAcquireExtended` — tam akış

[src/backend/storage/lmgr/lock.c:833](../src/backend/storage/lmgr/lock.c#L833)

```
LockAcquire(locktag, lockmode, sessionLock, dontWait)
   │
   ├─[1] Recovery kontrolü: standby'da RowExclusiveLock'tan güçlüsü yasak
   │
   ├─[2] LOCALLOCK ara/oluştur (backend-yerel hash)
   │        nLocks > 0 mı? ──► evet: GrantLockLocal, DÖN (LOCKACQUIRE_ALREADY_HELD)
   │                                 paylaşımlı belleğe hiç dokunulmadı
   │
   ├─[3] AccessExclusiveLock + tablo + standby aktif?
   │        ──► WAL kaydı hazırla (LogAccessExclusiveLockPrepare)
   │
   ├─[4] FAST PATH denemesi (zayıf tablo kilidi ise)
   │        ──► başarılı: DÖN (LOCKACQUIRE_OK)
   │
   ├─[5] GÜÇLÜ kilit ise: sayacı artır + tüm backend'lerin
   │        fast-path kilitlerini ana tabloya TAŞI
   │
   ├─[6] partitionLock al (LWLock)
   │     SetupLockInTable()  → LOCK + PROCLOCK oluştur/bul     lock.c:1291
   │
   ├─[7] LockCheckConflicts()                                   lock.c:1544
   │        conflictTab[lockmode] & lock->grantMask  ==  0 ?
   │        ├─ çakışma yok ──► GrantLock, partitionLock bırak, DÖN
   │        └─ çakışma var  ──► [8]
   │
   ├─[8] dontWait ise: temizle ve DÖN (LOCKACQUIRE_NOT_AVAIL)
   │
   └─[9] JoinWaitQueue() + ProcSleep()                proc.c:1209 / proc.c:1378
            uyandırılana ya da deadlock hatası alana kadar bekle
```

Dönüş değerleri [lock.c:791-794](../src/backend/storage/lmgr/lock.c#L791-L794):

```
LOCKACQUIRE_NOT_AVAIL      alınamadı (dontWait=true idi)
LOCKACQUIRE_OK             alındı
LOCKACQUIRE_ALREADY_HELD   zaten tutuluyordu, sayaç arttı
LOCKACQUIRE_ALREADY_CLEAR  zaten tutuluyordu ve lockCleared set
```

## 3.8. Bekleme kuyruğu kuralları

[README](../src/backend/storage/lmgr/README) "Deadlock Detection Algorithm" bölümünden:

**Alırken (`LockAcquire` / `ProcSleep`):**

1. Var olan **veya bekleyen** hiçbir istekle çakışmıyorsa, ya da süreç zaten aynı modu tutuyorsa → **anında ver**.
2. Aksi halde kuyruğa gir. Normalde **sona**. İstisna: süreç bu nesnede zaten bir kilit tutuyorsa ve bu kilit bekleyenlerden birinin isteğiyle çakışıyorsa → o bekleyenin **hemen önüne** ekle.
   - Bu kontrol olmasaydı deadlock dedektörü zaten kuyruğu yeniden sıralayacaktı, ama `ProcSleep`'te yapmak `deadlock_timeout` kadar beklemekten ucuz.
   - Özel durum: eklenme noktasından önceki hiçbir şeyle çakışma yoksa, beklemeden **ver**.

**Bırakırken (`ProcLockWakeup`)** — [proc.c:1840](../src/backend/storage/lmgr/proc.c#L1840):

Kuyruğu tarar, bir bekleyeni uyandırır eğer:
- (a) isteği verilmiş kilitlerle çakışmıyorsa, **ve**
- (b) isteği kendisinden önceki uyandırılamayan bekleyenlerin istekleriyle çakışmıyorsa.

Kural (b) çakışan isteklerin **geliş sırasına** göre verilmesini sağlar. Deadlock'tan kaçınmak için bazen sonraki bir bekleyenin öne geçmesi gerekir — ama bunu tanımak `ProcLockWakeup`'ın işi değil, deadlock dedektörü kuyruğu yeniden sıralar.

---

# 4. Deadlock tespiti

## 4.1. İyimser bekleme

Ana tasarım kararı: **deadlock olmadığında hiçbir maliyet ödeme.**

[src/backend/storage/lmgr/proc.c:1409-1440](../src/backend/storage/lmgr/proc.c#L1409-L1440)

```c
if (LockTimeout > 0)
{
    EnableTimeoutParams timeouts[2];
    timeouts[0].id       = DEADLOCK_TIMEOUT;
    timeouts[0].delay_ms = DeadlockTimeout;      /* varsayılan 1000 ms */
    timeouts[1].id       = LOCK_TIMEOUT;
    timeouts[1].delay_ms = LockTimeout;
    enable_timeouts(timeouts, 2);
}
else
    enable_timeout_after(DEADLOCK_TIMEOUT, DeadlockTimeout);
```

Süreç kilidi alamazsa **hiçbir deadlock kontrolü yapmadan** uyur, sadece bir zamanlayıcı kurar. `deadlock_timeout` (varsayılan **1s**) dolmadan uyandırılırsa maliyet sıfır.

Zamanlayıcı çalışması için alınan **aynı zaman damgası** `MyProc->waitStart`'a yazılır — ekstra `gettimeofday()` çağrısı yapmamak için. Bu yüzden `pg_locks.waitstart` çok kısa bir süre için NULL görünebilir.

## 4.2. `CheckDeadLock` — 16 partition'ın hepsini kilitle

[src/backend/storage/lmgr/proc.c:1887](../src/backend/storage/lmgr/proc.c#L1887)

```c
/* Partition numarası SIRASIYLA — yoksa LWLock deadlock'u */
for (i = 0; i < NUM_LOCK_PARTITIONS; i++)
    LWLockAcquire(LockHashPartitionLockByIndex(i), LW_EXCLUSIVE);

/* Bu arada uyandırılmış olabilir miyiz? */
if (dlist_node_is_detached(&MyProc->waitLink))
{
    result = DS_NO_DEADLOCK;
    goto check_done;
}

result = DeadLockCheck(MyProc);

if (result == DS_HARD_DEADLOCK)
    RemoveFromWaitQueue(MyProc, LockTagHashCode(&(MyProc->waitLock->tag)));

check_done:
    /* TERS sırada bırak: (1) çok kilit gerekeni erken uyandırma,
                          (2) LWLockRelease içinde O(N^2) davranıştan kaçın */
    for (i = NUM_LOCK_PARTITIONS; --i >= 0;)
        LWLockRelease(LockHashPartitionLockByIndex(i));
```

## 4.3. Waits-for grafiği: sert ve yumuşak kenarlar

Süreçler düğüm, "bekliyor" ilişkisi yönlü kenar. **Döngü varsa deadlock var.**

```
   SERT kenar (hard edge)              YUMUŞAK kenar (soft edge)
   ────────────────────────            ─────────────────────────
   A bir kilit bekliyor,               A, B'nin arkasında kuyrukta
   B çakışan kilidi TUTUYOR            ve istekleri çakışıyor
                                       (ProcLockWakeup A'yı B'den
   A ──────► B                          önce asla uyandırmaz)

   Çözümü: birini abort et             A ┈┈┈┈► B
                                       Çözümü: KUYRUĞU YENİDEN SIRALA
```

Üç olası sonuç:

1. Tüm yollar çalışan bir süreçte biter → deadlock yok.
2. **Başlangıç noktasına** dönülür → deadlock. Başlangıç noktasının isteği iptal edilir, hata verilir. Döngüyü kırmak için **tek** bir isteği iptal etmek yeterli.
3. Başka bir düğüme dönülür → deadlock var ama bizim değil. Yok sayılır — bizi öldürmek o deadlock'u çözmez.

## 4.4. Algoritmanın zor kısmı: yeniden sıralama

`FindLockCycle()` [deadlock.c:446](../src/backend/storage/lmgr/deadlock.c#L446) döngü bulunca, geri sarılırken döngüdeki **yumuşak kenarların listesini** de oluşturur.

- Liste **boş** → sert deadlock, çözüm yok, abort.
- Liste **dolu** → listedeki herhangi bir kenarı ters çevirmek döngüyü kırar. Ama başka yerde yeni döngü yaratabilir → her olasılığı denemek gerekebilir.

Bunun için `FindLockCycle` **varsayımsal** konfigürasyonlar üzerinde çalışabilmeli. Çözüm: bir **lookaside tablosu** — yeniden sıralamayı düşündüğümüz her bekleme kuyruğu için önerilen yeni sıra. `FindLockCycle` bu tabloda kayıt olan kilitler için gerçek sıra yerine önerilen sıraya inanır.

Önerilen sıra mevcut girişlerin **topolojik sıralaması** ile üretilir ([deadlock.c:862](../src/backend/storage/lmgr/deadlock.c#L862) `TopoSort`). Ters çevirmeyi düşündüğümüz her yumuşak kenar, sıralamanın uyması gereken bir kısmi-sıra kısıtı yaratır.

| Fonksiyon | Satır |
|---|---|
| `InitDeadLockChecking` | [deadlock.c:143](../src/backend/storage/lmgr/deadlock.c#L143) |
| `DeadLockCheck` | [deadlock.c:220](../src/backend/storage/lmgr/deadlock.c#L220) |
| `DeadLockCheckRecurse` | [deadlock.c:312](../src/backend/storage/lmgr/deadlock.c#L312) |
| `FindLockCycle` | [deadlock.c:446](../src/backend/storage/lmgr/deadlock.c#L446) |
| `FindLockCycleRecurse` | [deadlock.c:457](../src/backend/storage/lmgr/deadlock.c#L457) |
| `ExpandConstraints` | [deadlock.c:790](../src/backend/storage/lmgr/deadlock.c#L790) |
| `TopoSort` | [deadlock.c:862](../src/backend/storage/lmgr/deadlock.c#L862) |
| `DeadLockReport` | [deadlock.c:1075](../src/backend/storage/lmgr/deadlock.c#L1075) |

## 4.5. Autovacuum özel durumu

[proc.c:1546-1560](../src/backend/storage/lmgr/proc.c#L1546-L1560): deadlock yok ama autovacuum kaynaklı bir işi bekliyorsak, ona sinyal gönderip **iptal ettiririz**. `deadlock_state == DS_BLOCKED_BY_AUTOVACUUM`. Yani `ALTER TABLE`'ınız autovacuum'u bekliyorsa, autovacuum öldürülür.

---

# 5. Satır (tuple) kilitleri — iki katmanlı

[src/backend/access/heap/README.tuplock](../src/backend/access/heap/README.tuplock)

**Problem:** bir transaction milyonlarca satır kilitlemek isteyebilir. Bunları paylaşımlı bellekte tutmak imkânsız.

**Çözüm — 1. katman: kilit bilgisi tuple header'ında.**

Satır kilitlendiğinde XMAX'e kilitleyenin XID'i yazılır, infomask bitleri bunu "silinmiş" durumundan ayırır. Sınırsız sayıda satır kilitlenebilir çünkü ekstra bellek gerekmez.

**Çözüm — 2. katman: sıra hakemi için standart lock manager.**

`XactLockTableWait` tüm bekleyenleri **aynı anda** uyandırır → kimin alacağı yarış olur → açlık riski. Özellikle share-lock'lar: sürekli gelen share-lockçı akışı bir exclusive-lockçıyı sonsuza kadar bekletebilir.

Gerçek protokol:

```
LockTuple()              ← lock manager, kim sırada belirlensin diye
XactLockTableWait()      ← XMAX'teki transaction bitene kadar bekle
tuple'ı kendine kilitli işaretle
UnlockTuple()
```

Backend başına **en fazla bir** tuple kilidi tutulur veya beklenir → kilit tablosu taşmaz. Aktif çakışma yoksa ekstra maliyet yok.

**İstisna (deadlock önleme):** zaten bir kilit tutup daha güçlüsünü almaya çalışanlar, tuple birden fazla oturum tarafından eşzamanlı kilitleniyorsa `LockTuple()` çağrısını **atlar**. Atlamasalar deadlock riski olurdu.

## 5.1. Dört satır kilidi seviyesi

```
                  UPDATE       NO KEY UPDATE    SHARE        KEY SHARE
UPDATE           çakışır         çakışır       çakışır       çakışır
NO KEY UPDATE    çakışır         çakışır       çakışır
SHARE            çakışır         çakışır
KEY SHARE        çakışır
```

| SQL | Anlamı |
|---|---|
| `SELECT FOR UPDATE` | Her türlü değişikliği engeller. `DELETE` ve **key alanı değiştiren** `UPDATE` bunu örtük alır. |
| `SELECT FOR NO KEY UPDATE` | Silmeyi ve key değişikliğini engeller. Key değiştirmeyen `UPDATE` bunu örtük alır. |
| `SELECT FOR SHARE` | Paylaşımlı; her türlü değişikliği engeller. |
| `SELECT FOR KEY SHARE` | Paylaşımlı; sadece silme + key değişikliğini engeller. **RI kontrolleri için var** — foreign key kontrolü sırasında satırın kaybolmamasını garanti eder ama key'i değiştirmeyen UPDATE'leri bloklamaz. |

`FOR KEY SHARE`'in varlığı doğrudan foreign key performansı meselesidir: eskiden FK kontrolü satırı share-lock'lardı ve o satıra yapılan her UPDATE bloklanırdı.

## 5.2. MultiXact — birden fazla kilitçi

Tuple header'da tek bir XID'lik yer var. İkinci kilitçi gelince ilk kilitçinin XID'i bir **MultiXactId** ile değiştirilir. MultiXact = XID dizisi + her biri için bayraklar (kilit gücü, saf kilitçi mi güncelleyici mi).

Önemli noktalar:

- Eskiden MultiXact "birden fazla share kilitçi" demekti. Artık **update/delete XID'i de içerebilir**.
- Her kilit onu alan **subtransaction'a** atfedilir. Abort eden subxact kilitlerini bırakmış sayılır; eşzamanlı transaction'lar ana transaction'ı beklemeden ilerler.
- MultiXact'ler **crash'e dayanmalı** — sonraki okuyucu içindeki update'in commit mi abort mu olduğunu bilmek zorunda. Bu yüzden `pg_multixact` dizini var.
- Temizlik: `VACUUM` freeze sırasında eski MultiXact'leri kaldırır. Alt sınır `pg_class.relminmxid` → `pg_database.datminmxid` → checkpoint kaydı. `CHECKPOINT` o değerden eski `pg_multixact/` segmentlerini siler.

---

# 6. Predicate lock (SIReadLock) — SERIALIZABLE için

[src/backend/storage/lmgr/README-SSI](../src/backend/storage/lmgr/README-SSI) · [src/backend/storage/lmgr/predicate.c](../src/backend/storage/lmgr/predicate.c)

PostgreSQL 9.1'den beri `SERIALIZABLE` gerçekten serileştirilebilir. Yöntem **SSI** (Serializable Snapshot Isolation), Cahill/Röhm/Fekete 2008 makalesine dayanıyor.

**Klasik yöntem S2PL** (Strict Two-Phase Locking) okunmuş veriye yazmayı, yazılmış veriye her türlü erişimi **bloklar**. Yüksek çekişmede throughput'u yerle bir eder.

**SSI farkı:** SIREAD kilitleri **hiçbir şeyi bloklamaz**. Bunlar kilitten çok **bayrak**. Sadece transaction'lar arası okuma-yazma bağımlılık grafiğini kurmaya yararlar. Tehlikeli desen (bir transaction'a hem gelen hem giden rw-conflict) tespit edilince biri `40001` ile rollback edilir.

Sonuçları:

- SIREAD kilitleri bloklamadığı için **intent locking çalışmaz** (üst nesneyi alt nesneden önce kilitleme).
- Kilitler transaction commit ettikten **sonra da** tutulmaya devam eder — eşzamanlı transaction'lar bitene kadar.
- Granülerlik tuple seviyesinden başlar; kaynak tükenmesini önlemek için **kabalaştırılır** (tuple → page → relation). Kaba granülerlik **yanlış pozitif** çakışma üretebilir; miktarı plan seçimine göre değişir.
- Tuple üzerinde SIREAD varken aynı transaction sayfa veya tabloyu da kilitliyorsa, ince kilitler otomatik bırakılır.
- Yazma kilitleri RAM'de değil **heap tuple'ında** saklanır — iki RAM kilidi arasında eşleştirme yapılmaz.

Partition sayısı [lwlock.h:90-91](../src/include/storage/lwlock.h#L90-L91): `NUM_PREDICATELOCK_PARTITIONS = 16`.

Ana giriş noktaları:

| Fonksiyon | Satır |
|---|---|
| `PredicateLockRelation` | [predicate.c:2505](../src/backend/storage/lmgr/predicate.c#L2505) |
| `PredicateLockPage` | [predicate.c:2528](../src/backend/storage/lmgr/predicate.c#L2528) |
| `PredicateLockPageSplit` | [predicate.c:3073](../src/backend/storage/lmgr/predicate.c#L3073) |
| `CheckForSerializableConflictOut` | [predicate.c:3952](../src/backend/storage/lmgr/predicate.c#L3952) |
| `CheckForSerializableConflictIn` | [predicate.c:4265](../src/backend/storage/lmgr/predicate.c#L4265) |
| `PreCommit_CheckForSerializationFailure` | [predicate.c:4632](../src/backend/storage/lmgr/predicate.c#L4632) |

---

# 7. Buffer kilitleri — ayrı bir dünya

Shared buffer'lar heavyweight lock kullanmaz. İki ayrı mekanizma var:

**Pin** — "bu buffer benim tarafımdan kullanılıyor, evict etme." Referans sayacı. İçeriğe erişim hakkı vermez.

**Content lock** — sayfanın içeriğini okuma/yazma hakkı. LWLock üzerine kurulu.

[src/include/storage/bufmgr.h:205-223](../src/include/storage/bufmgr.h#L205-L223)

```c
typedef enum BufferLockMode
{
    BUFFER_LOCK_UNLOCK,
    BUFFER_LOCK_SHARE,            /* exclusive ile çakışır */
    BUFFER_LOCK_SHARE_EXCLUSIVE,  /* kendisiyle ve exclusive ile çakışır */
    BUFFER_LOCK_EXCLUSIVE,        /* her şeyle çakışır */
} BufferLockMode;
```

```c
extern bool ConditionalLockBuffer(Buffer buffer);
extern void LockBufferForCleanup(Buffer buffer);        /* pin sayısı 1 olana kadar bekle */
extern bool ConditionalLockBufferForCleanup(Buffer buffer);
```

`LockBufferForCleanup` özel: exclusive content lock **artı** başka hiç kimsenin pin tutmaması. `VACUUM` satırları fiziksel olarak kaldırırken bunu ister — kimse o sayfaya bakıyor olmamalı.

Hot Standby'da buffer pin deadlock'u mümkün: startup process bir cleanup lock beklerken, bir sorgu pin tutuyor olabilir. `ProcSleep` başında kontrol edilir — [proc.c:1401](../src/backend/storage/lmgr/proc.c#L1401) `CheckRecoveryConflictDeadlock()`.

---

# 8. lmgr.c — üst seviye API

[src/backend/storage/lmgr/lmgr.c](../src/backend/storage/lmgr/lmgr.c) ham `LockAcquire`'i sarar:

| Fonksiyon | Satır | Ne için |
|---|---|---|
| `LockRelationOid` | [lmgr.c:107](../src/backend/storage/lmgr/lmgr.c#L107) | Tablo kilidi (+ sinval mesajlarını yut) |
| `LockRelation` | [lmgr.c:246](../src/backend/storage/lmgr/lmgr.c#L246) | Relation pointer ile |
| `LockRelationIdForSession` | [lmgr.c:391](../src/backend/storage/lmgr/lmgr.c#L391) | Transaction'dan uzun yaşayan kilit |
| `LockRelationForExtension` | [lmgr.c:424](../src/backend/storage/lmgr/lmgr.c#L424) | Tabloyu büyütme hakkı |
| `LockPage` | [lmgr.c:507](../src/backend/storage/lmgr/lmgr.c#L507) | Sayfa kilidi |
| `LockTuple` | [lmgr.c:562](../src/backend/storage/lmgr/lmgr.c#L562) | Satır kilidi hakemliği |
| `XactLockTableInsert` | [lmgr.c:622](../src/backend/storage/lmgr/lmgr.c#L622) | Her xact kendi XID'ini kilitler |
| `XactLockTableWait` | [lmgr.c:663](../src/backend/storage/lmgr/lmgr.c#L663) | Bir xact bitene kadar bekle |
| `SpeculativeInsertionWait` | [lmgr.c:828](../src/backend/storage/lmgr/lmgr.c#L828) | `ON CONFLICT` için |
| `WaitForLockers` | [lmgr.c:989](../src/backend/storage/lmgr/lmgr.c#L989) | `CREATE INDEX CONCURRENTLY` |
| `LockDatabaseObject` | [lmgr.c:1008](../src/backend/storage/lmgr/lmgr.c#L1008) | Tablo olmayan nesneler |
| `LockApplyTransactionForSession` | [lmgr.c:1209](../src/backend/storage/lmgr/lmgr.c#L1209) | Logical replication |

**`XactLockTableWait` numarası:** her transaction başlarken kendi XID'i üzerine ExclusiveLock alır ve **sonuna kadar tutar**. "X transaction'ı bitene kadar bekle" demek, o XID üzerinde ShareLock istemek demek. Transaction bitince kilit bırakılır, bekleyen uyanır. Bedavaya deadlock tespiti de gelir.

---

# 9. Ayarlar

[src/backend/utils/misc/postgresql.conf.sample:876-885](../src/backend/utils/misc/postgresql.conf.sample#L876-L885)

```
#deadlock_timeout = 1s
#max_locks_per_transaction = 128        # min 10 (yeniden başlatma gerekir)
#max_pred_locks_per_transaction = 64    # min 10 (yeniden başlatma gerekir)
#max_pred_locks_per_relation = -2       # negatif = (max_pred_locks_per_transaction
                                        #            / -max_pred_locks_per_relation) - 1
#max_pred_locks_per_page = 2            # min 0
```

Ek olarak:

- `lock_timeout` — 0 = kapalı. [proc.c:64](../src/backend/storage/lmgr/proc.c#L64) `int LockTimeout = 0;`
- `log_lock_waits` — varsayılan **on**; `deadlock_timeout`'tan uzun beklemeleri loglar. [postgresql.conf.sample:670](../src/backend/utils/misc/postgresql.conf.sample#L670)
- `log_lock_failures` — [lock.c:57](../src/backend/storage/lmgr/lock.c#L57) `bool log_lock_failures = false;`

Kilit tablosu boyutu [lock.c:59-60](../src/backend/storage/lmgr/lock.c#L59-L60):

```c
#define NLOCKENTS() \
    mul_size(max_locks_per_xact, add_size(MaxBackends, max_prepared_xacts))
```

Yani `max_locks_per_transaction` **sert bir limit değil**, paylaşımlı bellek boyutunun çarpanı. Bir transaction fazlasını alabilir, yeter ki toplam havuzu aşmasın. Aşarsa: `out of shared memory / You might need to increase "max_locks_per_transaction"`.

---

# 10. İzleme

```sql
-- Şu an tutulan/beklenen tüm heavyweight kilitler
SELECT locktype, relation::regclass, mode, granted, waitstart, pid
FROM pg_locks
WHERE NOT granted;

-- Kim kimi blokluyor
SELECT pid, pg_blocking_pids(pid), wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

`pg_locks` sütun eşlemesi:

| Sütun | Kaynak |
|---|---|
| `locktype` | `LockTagTypeNames[locktag_type]` |
| `mode` | `lock_mode_names[mode]` — [lock.c:112-122](../src/backend/storage/lmgr/lock.c#L112-L122) |
| `granted` | PROCLOCK `holdMask`'te bit var mı |
| `waitstart` | `MyProc->waitStart` — [proc.c:1454](../src/backend/storage/lmgr/proc.c#L1454) |
| `fastpath` | Fast path dizisinden mi geldi |

**Dikkat:** `pg_locks` fast-path kilitlerini de gösterir ama bunlar için `granted` her zaman true'dur (fast path'ten alınan kilit tanımı gereği çakışmasızdır).

---

# 11. Sık karşılaşılan durumlar

| Belirti | Sebep | Bakılacak yer |
|---|---|---|
| `ERROR: deadlock detected` | Waits-for grafiğinde döngü, yumuşak kenar yok | `DeadLockReport` [deadlock.c:1075](../src/backend/storage/lmgr/deadlock.c#L1075) — log'da tam döngüyü basar |
| `ALTER TABLE` sonsuza kadar asılı | ACCESS EXCLUSIVE tüm SELECT'lerle çakışır; uzun süren bir SELECT engelliyor | `pg_locks` + `pg_blocking_pids` |
| `ALTER TABLE` **arkasında** her şey birikiyor | Bekleyen ACCESS EXCLUSIVE `waitMask`'e girdi; yeni SELECT'ler de kuyruğa takılıyor (README kural 1: "var olan **veya bekleyen**") | `lock_timeout` kullan |
| İki `VACUUM` aynı tabloda | SHARE UPDATE EXCLUSIVE kendisiyle çakışıyor | tasarım gereği |
| `out of shared memory` + lock ipucu | `NLOCKENTS()` havuzu doldu | `max_locks_per_transaction` artır (restart) |
| Çok sayıda partition'da sorgu → lock çekişmesi | Fast-path slotları doldu (`FP_LOCK_SLOTS_PER_GROUP = 16` × grup) | `pgstat` fastpath-exceeded sayacı [lock.c:1019](../src/backend/storage/lmgr/lock.c#L1019) |
| `could not serialize access` (40001) | SSI çakışma grafiğinde tehlikeli desen | Uygulama retry yapmalı |
| "stuck spinlock" panic | Spinlock ~2 dk alınamadı — gerçek bir bug | [s_lock.c](../src/backend/storage/lmgr/s_lock.c) |

---

# 12. Tek sayfalık özet

```
KULLANICI SEVİYESİ                          MOTOR SEVİYESİ
──────────────────                          ──────────────
SELECT                → AccessShareLock  ──┐
INSERT/UPDATE/DELETE  → RowExclusiveLock ──┤ FAST PATH (zayıf, çakışmasız)
SELECT FOR UPDATE     → RowShareLock     ──┘   → PGPROC içi dizi, paylaşımlı tabloya dokunma
                                               → FastPathStrongRelationLocks[1024] == 0 ise
VACUUM/ANALYZE        → ShareUpdateExcl  ──┐
CREATE INDEX          → ShareLock        ──┤ ANA TABLO (güçlü / çakışabilir)
ALTER/DROP/VACUUM FULL→ AccessExclusive  ──┘   → 16 partition, her biri LWLock'lu
                                               → LOCK + PROCLOCK hash tabloları
                                               → çakışma varsa kuyruk + uyku
                                               → 1s sonra deadlock kontrolü

SATIR SEVİYESİ                              MVCC SEVİYESİ
──────────────                              ─────────────
FOR KEY SHARE   ─┐                          tuple header: XMAX + infomask
FOR SHARE        ├─► tuple header'a yaz     birden fazla kilitçi → MultiXactId
FOR NO KEY UPDATE│   sıra için LockTuple()  pg_multixact/ dizini
FOR UPDATE      ─┘

SERIALIZABLE                                İÇ KİLİTLER
────────────                                ───────────
SIReadLock → bloklamaz, grafik kenarı       LWLock  → paylaşımlı yapılar, deadlock tespiti yok
tuple→page→relation kabalaşma               Spinlock→ birkaç komut, LWLock altyapısı
40001 ile rollback                          Buffer pin + content lock → sayfa erişimi
```
