# Buffer manager ve asenkron IO (AIO) — PostgreSQL 20devel

Kaynak ağacı: PostgreSQL **20devel** ([meson.build](../meson.build) `version: '20devel'`).
Satır numaraları sürüme bağlıdır; bu notta verilen tüm numaralar bu ağaçtan doğrulanmıştır.

Bu ağacın en büyük farkı: **AIO altyapısı artık var**. `ReadBuffer()` tek blok
okumak için artık doğrudan `pread()` çağırmıyor; bir `PgAioHandle` alıp IO'yu
kuyruğa veriyor ve sonra bekliyor. Üstüne `read_stream.c` geldi: bloklar
önceden, toplu ve paralel okunuyor. Not boyunca "klasik yol" ile "yeni yol"
ayrımını koruyoruz.

---

## 30 saniyelik özet

```
 ┌──────────────────────────────────────────────────────────────────────┐
 │  KULLANICI KODU     seqscan / vacuum / analyze / index scan          │
 └───────────────┬──────────────────────────────┬───────────────────────┘
                 │ read_stream_next_buffer()    │ ReadBuffer()
                 ▼                              ▼
 ┌───────────────────────────────┐  ┌───────────────────────────────────┐
 │  read_stream.c                │  │  ReadBuffer_common()              │
 │  • callback blok üretir       │  │  tek blok, "hemen bekleyeceğim"   │
 │  • ardışık blokları birleştir │  └──────────────┬────────────────────┘
 │  • önden okuma penceresi      │                 │
 └───────────────┬───────────────┘                 │
                 └────────────┬────────────────────┘
                              ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │  bufmgr.c :  StartReadBuffers() → AsyncReadBuffers() → WaitReadBuffers│
 │     PinBufferForBlock → BufferAlloc → (buf mapping hash / clock sweep)│
 │     BufferDesc.state:  refcount | usagecount | flags | content lock   │
 └──────────────────────────────┬───────────────────────────────────────┘
                                ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │  AIO :  pgaio_io_acquire → smgrstartreadv → pgaio_io_start_readv      │
 │         io_method = sync | worker | io_uring                          │
 │         tamamlanınca callback: BufferDesc'i BM_VALID yapar            │
 └──────────────────────────────┬───────────────────────────────────────┘
                                ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │  smgr / md.c :  blok numarası → (1 GB segment dosyası, offset)        │
 │                 base/16384/24576   .1  .2  ...     _fsm  _vm  _init   │
 └──────────────────────────────────────────────────────────────────────┘
```

Zihinsel model — altı madde:

- **Buffer pool sabit boyutlu bir dizidir.** `shared_buffers / 8kB` tane
  `BufferDesc` (metadata) + aynı sayıda 8 kB'lık sayfa. Kim hangi bloğu tutuyor
  bilgisi ayrı bir **hash table**'da (buffer mapping table), 128 partition'a
  bölünmüş.
- **Pin ≠ kilit.** Pin "bu buffer'ı benden habersiz başkasına verme" demektir
  (refcount). Content lock "bu sayfanın içeriğini okuyor/yazıyorum" demektir.
  `BM_IO_IN_PROGRESS` ise üçüncü ve bağımsız bir şeydir: "bu buffer'a diskten
  okuma/diske yazma sürüyor".
- **Kurban seçimi clock-sweep'tir, LRU değil.** Her buffer'ın 0..5 arası bir
  `usage_count`'u var. Saat ibresi dolaşır, sıfır olmayanı azaltır, sıfır ve
  pinsiz olanı kurban alır.
- **Büyük taramalar cache'i süpürmez**, çünkü kendilerine küçük bir **ring
  buffer** (BufferAccessStrategy) tahsis edilir: seqscan 256 kB, VACUUM 2 MB,
  bulk write 16 MB.
- **Okuma artık toplu ve asenkron.** `StartReadBuffers()` `io_combine_limit`
  bloğa kadar ardışık bloğu tek `preadv`'e paketler; `read_stream.c` bunu
  `effective_io_concurrency` kadar paralel IO ile besler.
- **Geçici tablolar tamamen ayrı yaşar.** `localbuf.c`, `temp_buffers` kadar
  process-local buffer tutar: kilit yok, spinlock yok, checkpoint yok.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/storage/buffer/README](../src/backend/storage/buffer/README) | Buffer erişim kuralları, clock-sweep ve ring buffer tasarımı (önce bu okunur) |
| [../src/backend/storage/aio/README.md](../src/backend/storage/aio/README.md) | AIO tasarım belgesi: handle yaşam döngüsü, callback, target, deadlock riskleri |
| [../src/include/storage/buf_internals.h](../src/include/storage/buf_internals.h) | `BufferDesc`, `BufferTag`, `BM_*` bayrakları, state bit düzeni |
| [../src/backend/storage/buffer/buf_init.c](../src/backend/storage/buffer/buf_init.c) | Shared memory yerleşimi: descriptor dizisi, blok dizisi, CV dizisi |
| [../src/backend/storage/buffer/buf_table.c](../src/backend/storage/buffer/buf_table.c) | Buffer mapping hash table (`BufTableLookup/Insert/Delete`) |
| [../src/backend/storage/buffer/bufmgr.c](../src/backend/storage/buffer/bufmgr.c) | İşin çoğu: pin, okuma, yazma, checkpoint taraması, AIO callback'leri |
| [../src/backend/storage/buffer/freelist.c](../src/backend/storage/buffer/freelist.c) | Clock-sweep (`StrategyGetBuffer`) ve ring buffer stratejileri |
| [../src/backend/storage/buffer/localbuf.c](../src/backend/storage/buffer/localbuf.c) | Geçici tablolar için process-local buffer yöneticisi |
| [../src/backend/storage/aio/read_stream.c](../src/backend/storage/aio/read_stream.c) | Önden okuma / IO birleştirme katmanı |
| [../src/backend/storage/aio/aio.c](../src/backend/storage/aio/aio.c) | `PgAioHandle` yaşam döngüsü, state machine, batch mode |
| [../src/backend/storage/aio/aio_io.c](../src/backend/storage/aio/aio_io.c) | `pgaio_io_start_readv/writev`, senkron yürütme |
| [../src/backend/storage/aio/method_sync.c](../src/backend/storage/aio/method_sync.c) | `io_method=sync` |
| [../src/backend/storage/aio/method_worker.c](../src/backend/storage/aio/method_worker.c) | `io_method=worker` (varsayılan) |
| [../src/backend/storage/aio/method_io_uring.c](../src/backend/storage/aio/method_io_uring.c) | `io_method=io_uring` (Linux 5.1+) |
| [../src/backend/storage/smgr/smgr.c](../src/backend/storage/smgr/smgr.c) | Storage manager arayüzü: `smgrreadv`, `smgrstartreadv` |
| [../src/backend/storage/smgr/md.c](../src/backend/storage/smgr/md.c) | "magnetic disk": blok → segment dosyası + offset |

---

## 1. Buffer pool düzeni

### 1.1 Shared memory'de ne var

Buffer pool tek bir yapı değil, **paralel dizilerdir**. `BufferManagerShmemRequest`
dört blok ister:

[../src/backend/storage/buffer/buf_init.c:78-108](../src/backend/storage/buffer/buf_init.c#L78-L108)

```c
	ShmemRequestStruct(.name = "Buffer Descriptors",
					   .size = NBuffers * sizeof(BufferDescPadded),
	/* Align descriptors to a cacheline boundary. */
					   .alignment = PG_CACHE_LINE_SIZE,
					   .ptr = (void **) &BufferDescriptors,
		);

	ShmemRequestStruct(.name = "Buffer Blocks",
					   .size = NBuffers * (Size) BLCKSZ,
	/* Align buffer pool on IO page size boundary. */
					   .alignment = PG_IO_ALIGN_SIZE,
					   .ptr = (void **) &BufferBlocks,
		);
```

Yani `Buffer` numarası (1'den başlar) hem descriptor dizisine hem blok dizisine
index'tir. `shared_buffers = 128MB` ise `NBuffers = 16384`.

Blok dizisinin `PG_IO_ALIGN_SIZE` hizalanması rastgele değil: **direct IO**
kullanılabilsin diye. Descriptor'ların cache line hizalanması ise false sharing
içindir.

### 1.2 BufferTag — bir bloğun kimliği

[../src/include/storage/buf_internals.h:161-168](../src/include/storage/buf_internals.h#L161-L168)

```c
typedef struct buftag
{
	Oid			spcOid;			/* tablespace oid */
	Oid			dbOid;			/* database oid */
	RelFileNumber relNumber;	/* relation file number */
	ForkNumber	forkNum;		/* fork number */
	BlockNumber blockNum;		/* blknum relative to begin of reln */
} BufferTag;
```

Tag'in kritik özelliği: **pg_class'a bakmadan bloğun diskteki yerini belirlemeye
yeter**. Buffer'ı diske yazan backend, ilgili tablonun kendi transaction'ında
görünür olmadığı bir durumda bile yazabilmelidir.

### 1.3 BufferDesc ve 64 bitlik `state`

[../src/include/storage/buf_internals.h:326-359](../src/include/storage/buf_internals.h#L326-L359)

```c
typedef struct BufferDesc
{
	BufferTag	tag;
	int			buf_id;
	pg_atomic_uint64 state;
	int			wait_backend_pgprocno;
	PgAioWaitRef io_wref;		/* set iff AIO is in progress */
	proclist_head lock_waiters;
} BufferDesc;
```

`state` tek bir 64 bitlik atomik değişkendir ve içinde **dört ayrı şey** taşır:

[../src/include/storage/buf_internals.h:49-52](../src/include/storage/buf_internals.h#L49-L52)

```c
#define BUF_REFCOUNT_BITS 18
#define BUF_USAGECOUNT_BITS 4
#define BUF_FLAG_BITS 12
#define BUF_LOCK_BITS (18+2)
```

```
 63                                                                    0
 ┌──────────────────┬────────────┬────────────┬──────────┬────────────┐
 │  content lock    │   flags    │ usagecount │      refcount          │
 │   20 bit         │   12 bit   │   4 bit    │        18 bit          │
 └──────────────────┴────────────┴────────────┴──────────┴────────────┘
   ^ 18 bit shared sayacı + 1 bit share-exclusive + 1 bit exclusive
```

Bu ağaçta dikkat çeken şey: **content lock artık ayrı bir LWLock değil**, aynı
`state` kelimesinin içinde. Bekleyenler `lock_waiters` proclist'inde tutulur.
Bu sayede pin alma + usage count artırma + kilit alma tek bir CAS döngüsünde
yapılabiliyor.

Bayraklar:

[../src/include/storage/buf_internals.h:106-125](../src/include/storage/buf_internals.h#L106-L125)

```c
/* buffer header is locked */
#define BM_LOCKED					BUF_DEFINE_FLAG( 0)
/* data needs writing */
#define BM_DIRTY					BUF_DEFINE_FLAG( 1)
/* data is valid */
#define BM_VALID					BUF_DEFINE_FLAG( 2)
/* tag is assigned */
#define BM_TAG_VALID				BUF_DEFINE_FLAG( 3)
/* read or write in progress */
#define BM_IO_IN_PROGRESS			BUF_DEFINE_FLAG( 4)
/* previous I/O failed */
#define BM_IO_ERROR					BUF_DEFINE_FLAG( 5)
```

Üç bayrağın ayrımı önemlidir:

| Bayrak | Anlamı |
|---|---|
| `BM_TAG_VALID` | Hash table'da bu buffer için bir kayıt var — yani buffer "sahiplenilmiş" |
| `BM_VALID` | Sayfanın içeriği gerçekten diskten okundu ve geçerli |
| `BM_DIRTY` | İçerik değişti, diske yazılması gerek |

`BM_TAG_VALID` var ama `BM_VALID` yok = "bu bloğu okumaya başladık, henüz
bitmedi". Bu ara durum AIO ile çok daha uzun sürdüğü için artık daha kritik.

### 1.4 Buffer mapping hash table ve partition'lar

[../src/backend/storage/buffer/buf_table.c:62-71](../src/backend/storage/buffer/buf_table.c#L62-L71)

```c
	size = NBuffers + NUM_BUFFER_PARTITIONS;

	ShmemRequestHash(.name = "Shared Buffer Lookup Table",
					 .nelems = size,
					 .ptr = &SharedBufHash,
					 .hash_info.keysize = sizeof(BufferTag),
					 .hash_info.entrysize = sizeof(BufferLookupEnt),
					 .hash_info.num_partitions = NUM_BUFFER_PARTITIONS,
```

`NUM_BUFFER_PARTITIONS` = 128 ([../src/include/storage/lwlock.h:83](../src/include/storage/lwlock.h#L83)).
Bir tag'in hangi partition'a düştüğü hash'inin düşük bitlerinden belirlenir:

[../src/include/storage/buf_internals.h:247-258](../src/include/storage/buf_internals.h#L247-L258)

```c
static inline uint32
BufTableHashPartition(uint32 hashcode)
{
	return hashcode % NUM_BUFFER_PARTITIONS;
}

static inline LWLock *
BufMappingPartitionLock(uint32 hashcode)
{
	return &MainLWLockArray[BUFFER_MAPPING_LWLOCK_OFFSET +
							BufTableHashPartition(hashcode)].lock;
}
```

Neden `NBuffers + NUM_BUFFER_PARTITIONS` boyut? Çünkü `BufferAlloc()` eskisini
silmeden yenisini eklemeye çalışır; en kötü durumda 128 partition'da aynı anda
bu olur.

---

## 2. Pin, content lock, IO_IN_PROGRESS — üç ayrı mekanizma

Buffer README'nin ilk cümlesi konuyu özetler:

[../src/backend/storage/buffer/README:6-10](../src/backend/storage/buffer/README#L6-L10)

```
There are two separate access control mechanisms for shared disk buffers:
reference counts (a/k/a pin counts) and buffer content locks.
```

### 2.1 Pin

Pin = shared refcount + backend-local refcount. Local sayaç sayesinde aynı
backend bir buffer'ı ikinci kez pin'lerken atomik işlem yapmaz:

[../src/backend/storage/buffer/bufmgr.c:245-254](../src/backend/storage/buffer/bufmgr.c#L245-L254)

```c
 * PrivateRefCountArray) and an overflow hash table (PrivateRefCountHash) to
 * keep track of backend local pins.
 *
 * Until no more than REFCOUNT_ARRAY_ENTRIES buffers are pinned at once, all
```

`REFCOUNT_ARRAY_ENTRIES` = 8 ([../src/backend/storage/buffer/bufmgr.c:145](../src/backend/storage/buffer/bufmgr.c#L145)).

`PinBuffer()` içindeki CAS döngüsü hem refcount'u hem usage count'u aynı anda
günceller — ve **strateji varsa usage count'u 1'in üstüne çıkarmaz**:

[../src/backend/storage/buffer/bufmgr.c:3332-3349](../src/backend/storage/buffer/bufmgr.c#L3332-L3349)

```c
			/* increase refcount */
			buf_state += BUF_REFCOUNT_ONE;

			if (strategy == NULL)
			{
				/* Default case: increase usagecount unless already max. */
				if (BUF_STATE_GET_USAGECOUNT(buf_state) < BM_MAX_USAGE_COUNT)
					buf_state += BUF_USAGECOUNT_ONE;
			}
			else
			{
				/*
				 * Ring buffers shouldn't evict others from pool.  Thus we
				 * don't make usagecount more than 1.
				 */
				if (BUF_STATE_GET_USAGECOUNT(buf_state) == 0)
					buf_state += BUF_USAGECOUNT_ONE;
			}
```

Bu `else` dalı, bölüm 4'teki "VACUUM neden cache'i süpürmüyor" sorusunun
yarısıdır.

### 2.2 Content lock

Üç mod var — ve bu ağaçta **üçüncü mod (share-exclusive) yeni**:

[../src/include/storage/bufmgr.h:205-222](../src/include/storage/bufmgr.h#L205-L222)

```c
typedef enum BufferLockMode
{
	BUFFER_LOCK_UNLOCK,
	...
	BUFFER_LOCK_SHARE,
	...
	BUFFER_LOCK_SHARE_EXCLUSIVE,
	...
	BUFFER_LOCK_EXCLUSIVE,
```

README bunu şöyle anlatıyor:

[../src/backend/storage/buffer/README:28-36](../src/backend/storage/buffer/README#L28-L36)

```
Buffer content locks: there are three kinds of buffer lock, shared,
share-exclusive and exclusive:
a) multiple backends can hold shared locks on the same buffer
b) one backend can hold a share-exclusive lock on a buffer while multiple
   backends can hold a share lock
c) an exclusive lock prevents anyone else from holding a shared,
   share-exclusive or exclusive lock.
```

Share-exclusive'in varlık nedeni yazma yoludur:

[../src/backend/storage/buffer/README:109-111](../src/backend/storage/buffer/README#L109-L111)

```
6. To write out a buffer, a share-exclusive lock needs to be held. This
prevents the buffer from being modified while written out, which could corrupt
checksums and cause issues on the OS or device level when direct-IO is used.
```

Yani: okuyucular sayfayı okumaya devam edebilir, ama hint bit yazanlar ve
`FlushBuffer` birbirini dışlar. Direct IO çağında "yazılırken değişen sayfa"
artık gerçek bir bozulma sebebi.

### 2.3 BM_IO_IN_PROGRESS — kilit değil, ama kilit gibi

[../src/backend/storage/buffer/README:162-169](../src/backend/storage/buffer/README#L162-L169)

```
* The BM_IO_IN_PROGRESS flag acts as a kind of lock, used to wait for I/O on a
buffer to complete (and in releases before 14, it was accompanied by a
per-buffer LWLock).  The process starting a read or write sets the flag. When
the I/O is completed, be it by the process that initiated the I/O or by
another process, the flag is removed and the Buffer's condition variable is
signalled.  Processes that need to wait for the I/O to complete can wait for
asynchronous I/O by using BufferDesc->io_wref and for BM_IO_IN_PROGRESS to be
unset by sleeping on the buffer's condition variable.
```

Kritik nokta: **IO'yu başlatan backend ile bitiren backend farklı olabilir.**
`io_method=worker` ile IO'yu bir IO worker process'i tamamlar ve `BM_VALID`'i o
set eder. Bu yüzden bekleme iki katmanlıdır — `WaitIO()`:

[../src/backend/storage/buffer/bufmgr.c:7217-7230](../src/backend/storage/buffer/bufmgr.c#L7217-L7230)

```c
		iow = buf->io_wref;
		UnlockBufHdr(buf);

		/* no IO in progress, we don't need to wait */
		if (!(buf_state & BM_IO_IN_PROGRESS))
			break;

		/*
		 * The buffer has asynchronous IO in progress, wait for it to
		 * complete.
		 */
		if (pgaio_wref_valid(&iow))
		{
			pgaio_wref_wait(&iow);
```

`io_wref` geçerliyse AIO alt sistemi üzerinden beklenir; değilse buffer'ın
condition variable'ında uyunur.

`StartBufferIO()` üç sonuçtan birini döner ve bu değerler yeni okuma yolunun
belkemiğidir:

| Dönüş | Anlamı |
|---|---|
| `BUFFER_IO_READY_FOR_IO` | `BM_IO_IN_PROGRESS`'i biz set ettik, IO bizim |
| `BUFFER_IO_ALREADY_DONE` | Başkası bizden önce okudu, iş yok |
| `BUFFER_IO_IN_PROGRESS` | Başkası okuyor; `*io_wref` ile ona katılabiliriz |

[../src/backend/storage/buffer/bufmgr.c:7303-7317](../src/backend/storage/buffer/bufmgr.c#L7303-L7317)

```c
		/* Join the existing IO */
		if (io_wref != NULL && pgaio_wref_valid(&buf->io_wref))
		{
			*io_wref = buf->io_wref;
			UnlockBufHdr(buf);

			return BUFFER_IO_IN_PROGRESS;
		}
```

"Başkasının IO'suna katılmak" (`foreign_io`) bu sürümde eklenen ve aynı tabloyu
paralel tarayan backend'lerde performansı belirleyen davranıştır.

---

## 3. Klasik okuma yolu: ReadBuffer

```
ReadBuffer(rel, blk)
  └─ ReadBufferExtended(rel, MAIN_FORKNUM, blk, RBM_NORMAL, NULL)
       └─ ReadBuffer_common()
            ├─ P_NEW mi?  → ExtendBufferedRel()
            ├─ RBM_ZERO_*?→ PinBufferForBlock() + ZeroAndLockBuffer()
            └─ normal:      StartReadBuffer(... READ_BUFFERS_SYNCHRONOUSLY)
                             └─ StartReadBuffersImpl()
                                  ├─ PinBufferForBlock() → BufferAlloc()
                                  └─ AsyncReadBuffers()
                            WaitReadBuffers()
```

[../src/backend/storage/buffer/bufmgr.c:878-882](../src/backend/storage/buffer/bufmgr.c#L878-L882)

```c
Buffer
ReadBuffer(Relation reln, BlockNumber blockNum)
{
	return ReadBufferExtended(reln, MAIN_FORKNUM, blockNum, RBM_NORMAL, NULL);
}
```

`ReadBuffer_common` sonunda yeni API'yi çağırır ve kendini "hemen bekleyeceğim"
diye işaretler:

[../src/backend/storage/buffer/bufmgr.c:1348-1365](../src/backend/storage/buffer/bufmgr.c#L1348-L1365)

```c
	/*
	 * Signal that we are going to immediately wait. If we're immediately
	 * waiting, there is no benefit in actually executing the IO
	 * asynchronously, it would just add dispatch overhead.
	 */
	flags = READ_BUFFERS_SYNCHRONOUSLY;
	...
	if (StartReadBuffer(&operation,
						&buffer,
						blockNum,
						flags))
		WaitReadBuffers(&operation);
```

Yani `ReadBuffer()` hâlâ senkron davranır — ama artık AIO makinesinin üzerinden
geçerek. Asenkronluk `read_stream.c` üzerinden gelir.

`PinBufferForBlock()` shared/local ayrımının yapıldığı yerdir:

[../src/backend/storage/buffer/bufmgr.c:1248-1253](../src/backend/storage/buffer/bufmgr.c#L1248-L1253)

```c
	if (persistence == RELPERSISTENCE_TEMP)
		bufHdr = LocalBufferAlloc(smgr, forkNum, blockNum, foundPtr);
	else
		bufHdr = BufferAlloc(smgr, persistence, forkNum, blockNum,
							 strategy, foundPtr, io_context);
```

### 3.1 BufferAlloc — hash lookup, kurban, çakışma

`BufferAlloc` üç aşamalıdır. Önce **shared lock ile arama**:

[../src/backend/storage/buffer/bufmgr.c:2222-2240](../src/backend/storage/buffer/bufmgr.c#L2222-L2240)

```c
	/* see if the block is in the buffer pool already */
	LWLockAcquire(newPartitionLock, LW_SHARED);
	existing_buf_id = BufTableLookup(&newTag, newHash);
	if (existing_buf_id >= 0)
	{
		...
		valid = PinBuffer(buf, strategy, false);

		/* Can release the mapping lock as soon as we've pinned it */
		LWLockRelease(newPartitionLock);
```

Sıralama önemli: **önce pin, sonra partition lock'u bırak.** Aksi halde buffer
elimizden alınabilir.

Bulunamazsa kurban alınır (partition lock **bırakılmış** haldeyken — kurban
bulmak diske yazma gerektirebilir):

[../src/backend/storage/buffer/bufmgr.c:2263-2269](../src/backend/storage/buffer/bufmgr.c#L2263-L2269)

```c
	/*
	 * Acquire a victim buffer. Somebody else might try to do the same, we
	 * don't hold any conflicting locks. If so we'll have to undo our work
	 * later.
	 */
	victim_buffer = GetVictimBuffer(strategy, io_context);
```

Sonra exclusive lock ile hash'e eklenmeye çalışılır; başkası araya girdiyse
kendi kurbanımızı bırakıp onunkini kullanırız:

[../src/backend/storage/buffer/bufmgr.c:2276-2295](../src/backend/storage/buffer/bufmgr.c#L2276-L2295)

```c
	LWLockAcquire(newPartitionLock, LW_EXCLUSIVE);
	existing_buf_id = BufTableInsert(&newTag, newHash, victim_buf_hdr->buf_id);
	if (existing_buf_id >= 0)
	{
		...
		UnpinBuffer(victim_buf_hdr);
```

Son olarak tag yazılır ve `BM_TAG_VALID` + usage count 1 set edilir; `BM_VALID`
**henüz set edilmez** — sayfa daha okunmadı:

[../src/backend/storage/buffer/bufmgr.c:2336-2342](../src/backend/storage/buffer/bufmgr.c#L2336-L2342)

```c
	set_bits |= BM_TAG_VALID | BUF_USAGECOUNT_ONE;
	if (relpersistence == RELPERSISTENCE_PERMANENT || forkNum == INIT_FORKNUM)
		set_bits |= BM_PERMANENT;

	UnlockBufHdrExt(victim_buf_hdr, victim_buf_state,
					set_bits, 0, 0);
```

`BM_PERMANENT`'in koşulu okunmaya değer: unlogged tablonun **init fork'u**
kalıcı sayılır, çünkü crash sonrası tablo o fork'tan yeniden kurulur.

---

## 4. Kurban seçimi: clock-sweep

`freelist.c` adı yanıltıcıdır — bu ağaçta gerçek bir free list yoktur, sadece
saat ibresi vardır:

[../src/backend/storage/buffer/freelist.c:37-49](../src/backend/storage/buffer/freelist.c#L37-L49)

```c
	/*
	 * clock-sweep hand: index of next buffer to consider grabbing. Note that
	 * this isn't a concrete buffer - we only ever increase the value. So, to
	 * get an actual buffer, it needs to be used modulo NBuffers.
	 */
	pg_atomic_uint32 nextVictimBuffer;

	/*
	 * Statistics.  These counters should be wide enough that they can't
	 * overflow during a single bgwriter cycle.
	 */
	uint32		completePasses; /* Complete cycles of the clock-sweep */
	pg_atomic_uint32 numBufferAllocs;	/* Buffers allocated since last reset */
```

İbre `ClockSweepTick()` ([../src/backend/storage/buffer/freelist.c:110](../src/backend/storage/buffer/freelist.c#L110))
ile atomik olarak ilerletilir; spinlock sadece sarma anında alınır.

Ana döngü — README'deki 4 adımın kodu:

[../src/backend/storage/buffer/freelist.c:239-305](../src/backend/storage/buffer/freelist.c#L239-L305)

```c
	/* Use the "clock sweep" algorithm to find a free buffer */
	trycounter = NBuffers;
	for (;;)
	{
		...
		buf = GetBufferDescriptor(ClockSweepTick());
		...
			if (BUF_STATE_GET_REFCOUNT(local_buf_state) != 0)
			{
				if (--trycounter == 0)
					elog(ERROR, "no unpinned buffers available");
				break;
			}
			...
			if (BUF_STATE_GET_USAGECOUNT(local_buf_state) != 0)
			{
				local_buf_state -= BUF_USAGECOUNT_ONE;
				...
			}
			else
			{
				/* pin the buffer if the CAS succeeds */
				local_buf_state += BUF_REFCOUNT_ONE;
```

`BM_MAX_USAGE_COUNT` = 5 ([../src/include/storage/buf_internals.h:144](../src/include/storage/buf_internals.h#L144)).
Neden 5? README'deki yorum açık: değer NBuffers'a yaklaşırsa LRU'ya benzer ama
ibrenin bir buffer bulması `BM_MAX_USAGE_COUNT+1` tam tur alabilir.

Kurban kirliyse ne olur? `GetVictimBuffer()`:

[../src/backend/storage/buffer/bufmgr.c:2584-2611](../src/backend/storage/buffer/bufmgr.c#L2584-L2611)

```c
	if (buf_state & BM_DIRTY)
	{
		...
		if (!BufferLockConditional(buf, buf_hdr, BUFFER_LOCK_SHARE_EXCLUSIVE))
		{
			/*
			 * Someone else has locked the buffer, so give it up and loop back
			 * to get another one.
			 */
			UnpinBuffer(buf_hdr);
			goto again;
		}
```

Koşullu kilit şart: yoksa iki backend'in birbirinin kurbanını beklemesiyle
deadlock oluşabilir (yorumda btree page split örneği veriliyor).

---

## 5. BufferAccessStrategy: büyük taramalar neden cache'i süpürmez

Strateji nesnesi küçük bir **buffer halkasıdır**. Boyutlar:

[../src/backend/storage/buffer/freelist.c:459](../src/backend/storage/buffer/freelist.c#L459)

```c
				ring_size_kb = 256;
```

[../src/backend/storage/buffer/freelist.c:487-492](../src/backend/storage/buffer/freelist.c#L487-L492)

```c
		case BAS_BULKWRITE:
			ring_size_kb = 16 * 1024;
			break;
		case BAS_VACUUM:
			ring_size_kb = 2048;
			break;
```

| Tür | Halka | Kim kullanır |
|---|---|---|
| `BAS_NORMAL` | yok (NULL) | Normal sorgular |
| `BAS_BULKREAD` | 256 kB + AIO payı | Büyük seqscan |
| `BAS_BULKWRITE` | 16 MB | `COPY IN`, `CREATE TABLE AS` |
| `BAS_VACUUM` | 2 MB (`vacuum_buffer_usage_limit`) | VACUUM |

**AIO bu ağaçta halkayı büyüttü** — çünkü okunmakta olan buffer yeniden
kullanılamaz:

[../src/backend/storage/buffer/freelist.c:471-481](../src/backend/storage/buffer/freelist.c#L471-L481)

```c
				/*
				 * We would like the ring to additionally have space for the
				 * configured degree of IO concurrency. While being read in,
				 * buffers can obviously not yet be reused.
				 *
				 * Each IO can be up to io_combine_limit blocks large, and we
				 * want to start up to effective_io_concurrency IOs.
				 */
				ring_size_kb += (BLCKSZ / 1024) *
					io_combine_limit * effective_io_concurrency;
```

Halkadan buffer alma kuralı iki koşullu:

[../src/backend/storage/buffer/freelist.c:655-666](../src/backend/storage/buffer/freelist.c#L655-L666)

```c
		/*
		 * If the buffer is pinned we cannot use it under any circumstances.
		 *
		 * If usage_count is 0 or 1 then the buffer is fair game (we expect 1,
		 * since our own previous usage of the ring element would have left it
		 * there, but it might've been decremented by clock-sweep since then).
		 * A higher usage_count indicates someone else has touched the buffer,
		 * so we shouldn't re-use it.
		 */
		if (BUF_STATE_GET_REFCOUNT(local_buf_state) != 0
			|| BUF_STATE_GET_USAGECOUNT(local_buf_state) > 1)
			break;
```

Yani: **taramanın kendi getirdiği sayfa halkada döner; başkası dokunduysa
halkadan çıkar ve normal cache'e karışır.** Cache'in süpürülmemesinin ikinci
yarısı budur (birincisi bölüm 2.1'deki usage count sınırıydı).

Bir de bulkread'e özel kaçış var: halkadaki sayfa kirlendiyse ve yazmak için
WAL flush gerekiyorsa, sayfa halkadan atılır:

[../src/backend/storage/buffer/freelist.c:752-771](../src/backend/storage/buffer/freelist.c#L752-L771)

```c
bool
StrategyRejectBuffer(BufferAccessStrategy strategy, BufferDesc *buf, bool from_ring)
{
	/* We only do this in bulkread mode */
	if (strategy->btype != BAS_BULKREAD)
		return false;
	...
	/*
	 * Remove the dirty buffer from the ring; necessary to prevent infinite
	 * loop if all ring members are dirty.
	 */
	strategy->buffers[strategy->current] = InvalidBuffer;
```

README bunun sonucunu açıkça yazar: her sayfayı değiştiren bir bulk UPDATE'te
ring stratejisi pratikte normal stratejiye dejenere olur.

---

## 6. Kirletme ve yazma

### 6.1 MarkBufferDirty

[../src/backend/storage/buffer/bufmgr.c:3187-3203](../src/backend/storage/buffer/bufmgr.c#L3187-L3203)

```c
	Assert(BufferIsPinned(buffer));
	Assert(BufferIsLockedByMeInMode(buffer, BUFFER_LOCK_EXCLUSIVE));

	/*
	 * NB: We have to wait for the buffer header spinlock to be not held, as
	 * TerminateBufferIO() relies on the spinlock.
	 */
	old_buf_state = pg_atomic_read_u64(&bufHdr->state);
	for (;;)
	{
		if (old_buf_state & BM_LOCKED)
			old_buf_state = WaitBufHdrUnlocked(bufHdr);

		buf_state = old_buf_state;
		...
		buf_state |= BM_DIRTY;
```

Sadece bayrak set eder. Yazma sonra, başkası tarafından yapılır.

### 6.2 FlushBuffer ve WAL kuralı

[../src/backend/storage/buffer/bufmgr.c:4565-4586](../src/backend/storage/buffer/bufmgr.c#L4565-L4586)

```c
	recptr = BufferGetLSN(buf);

	/*
	 * Force XLOG flush up to buffer's LSN.  This implements the basic WAL
	 * rule that log updates must hit disk before any of the data-file changes
	 * they describe do.
	 */
	...
	if (pg_atomic_read_u64(&buf->state) & BM_PERMANENT)
		XLogFlush(recptr);
```

Bu satır **WAL'in tüm varlık nedenidir**: sayfa diske gitmeden önce onu
açıklayan WAL kaydı diske gitmiş olmalı. Unlogged ilişkilerde atlanır, çünkü
sahte (fake) LSN'ler WAL insert noktasını geçebilir.

Sonrası checksum + yazma:

[../src/backend/storage/buffer/bufmgr.c:4595-4604](../src/backend/storage/buffer/bufmgr.c#L4595-L4604)

```c
	/* Update page checksum if desired. */
	PageSetChecksum((Page) bufBlock, buf->tag.blockNum);

	io_start = pgstat_prepare_io_time(track_io_timing);

	smgrwrite(reln,
			  BufTagGetForkNum(&buf->tag),
			  buf->tag.blockNum,
			  bufBlock,
			  false);
```

Yazma yolu bu ağaçta **hâlâ senkrondur**; AIO şimdilik okuma tarafında
kullanılıyor (`PGAIO_HCB_*_READV` callback'leri, `PGAIO_OP_WRITEV` altyapısı
mevcut ama buffer yazımı için henüz devrede değil).

### 6.3 Kim yazar?

Üç aday var:

| Yazan | Fonksiyon | Ne zaman |
|---|---|---|
| Checkpointer | `BufferSync()` [bufmgr.c:3575](../src/backend/storage/buffer/bufmgr.c#L3575) | Checkpoint'te tüm kirli sayfalar |
| Background writer | `BgBufferSync()` [bufmgr.c:3854](../src/backend/storage/buffer/bufmgr.c#L3854) | Sürekli, ibrenin önünü temizler |
| Backend'in kendisi | `GetVictimBuffer()` içinden `FlushBuffer()` | Kurban kirliyse — **istenmeyen durum** |

`BufferSync` işe iki geçişle başlar: önce yazılacakları işaretler, sonra
yazar:

[../src/backend/storage/buffer/bufmgr.c:3600-3605](../src/backend/storage/buffer/bufmgr.c#L3600-L3605)

```c
	 * Loop over all buffers, and mark the ones that need to be written with
	 * BM_CHECKPOINT_NEEDED.  Count them as we go (num_to_scan), so that we
	 * can estimate how much work needs to be done.
	 *
	 * This allows us to write only those pages that were dirty when the
	 * checkpoint began, and not those that get dirtied while it proceeds.
```

Bu, checkpoint'in "kovalamaca"ya girmesini engeller: checkpoint başladıktan
sonra kirlenen sayfalar bir sonraki checkpoint'in işidir.

bgwriter ise ibreyi **ilerletmeden** onun önünü tarar
([../src/backend/storage/buffer/README:253-258](../src/backend/storage/buffer/README#L253-L258)) —
yani "birazdan kurban olacak sayfaları" önceden temizler. Amaç: backend'lerin
`GetVictimBuffer()` içinde diske yazmak zorunda kalmaması.

---

## 7. YENİ: toplu okuma — StartReadBuffers / WaitReadBuffers

API sözleşmesi:

[../src/backend/storage/buffer/bufmgr.c:1608-1612](../src/backend/storage/buffer/bufmgr.c#L1608-L1612)

```c
 * If false is returned, no I/O is necessary and the buffers covered by
 * *nblocks on exit are valid and ready to be accessed.  If true is returned,
 * an I/O has been started, and WaitReadBuffers() must be called with the same
 * operation object before the buffers covered by *nblocks on exit can be
 * accessed.
```

`StartReadBuffersImpl()` sırayla:

1. Her blok için `PinBufferForBlock()` — cache hit mi miss mi öğrenilir.
2. İlk blok hit ise: `*nblocks = 1`, `false` dön (IO yok).
3. Ortadaki bir blok hit ise: IO **bölünür**, kalan buffer'lar *forward* edilir.
4. Segment sınırı kontrol edilir:

[../src/backend/storage/buffer/bufmgr.c:1503-1512](../src/backend/storage/buffer/bufmgr.c#L1503-L1512)

```c
			if (i == 0 && actual_nblocks > 1)
			{
				maxcombine = smgrmaxcombine(operation->smgr,
											operation->forknum,
											blockNum);
				if (unlikely(maxcombine < actual_nblocks))
				{
					elog(DEBUG2, "limiting nblocks at %u from %u to %u",
						 blockNum, actual_nblocks, maxcombine);
					actual_nblocks = maxcombine;
				}
			}
```

5. Ve `io_method` ayrımı:

[../src/backend/storage/buffer/bufmgr.c:1527-1557](../src/backend/storage/buffer/bufmgr.c#L1527-L1557)

```c
	/*
	 * When using AIO, start the IO in the background. If not, issue prefetch
	 * requests if desired by the caller.
	 *
	 * The reason we have a dedicated path for IOMETHOD_SYNC here is to
	 * de-risk the introduction of AIO somewhat. It's a large architectural
	 * change, with lots of chances for unanticipated performance effects.
	 */
	if (io_method != IOMETHOD_SYNC)
	{
		...
		did_start_io = AsyncReadBuffers(operation, nblocks);
```

**"Forwarded buffer"** kavramı bu API'nin en ince yeridir: bölünen bir IO'nun
geri kalan pinli buffer'ları çağrana geri verilir ve bir sonraki çağrıda aynı
diziyle devam edilir; unpin/repin maliyeti ödenmez.

### 7.1 AsyncReadBuffers — IO'nun kurulduğu yer

Önce handle alınır (`BM_IO_IN_PROGRESS` set edilmeden **önce**, çünkü handle
alma bloklayabilir):

[../src/backend/storage/buffer/bufmgr.c:2025-2030](../src/backend/storage/buffer/bufmgr.c#L2025-L2030)

```c
	ioh = pgaio_io_acquire_nb(CurrentResourceOwner, &operation->io_return);
	if (unlikely(!ioh))
	{
		pgaio_submit_staged();
		ioh = pgaio_io_acquire(CurrentResourceOwner, &operation->io_return);
	}
```

Sonra ilk blok için `StartBufferIO(..., wait=true, &io_wref)`:

[../src/backend/storage/buffer/bufmgr.c:2058-2060](../src/backend/storage/buffer/bufmgr.c#L2058-L2060)

```c
	status = StartBufferIO(buffers[nblocks_done], true, true,
						   &operation->io_wref);
	if (status != BUFFER_IO_READY_FOR_IO)
```

Başkası okuyorsa `foreign_io = true` işaretlenir ve o IO beklenir. Yorum bunun
neden performans için şart olduğunu açıklıyor: index scan'de aynı heap bloğu
pencere içinde birden çok kez istenebilir, ve aynı tabloyu tarayan backend'ler
birbirini beklemek yerine aynı IO'ya bağlanır.

Sonra komşu bloklar taranıp aynı IO'ya eklenir — bu sefer beklemeden:

[../src/backend/storage/buffer/bufmgr.c:2114-2126](../src/backend/storage/buffer/bufmgr.c#L2114-L2126)

```c
	for (int i = nblocks_done + 1; i < operation->nblocks; i++)
	{
		/* Must be consecutive block numbers. */
		Assert(BufferGetBlockNumber(buffers[i - 1]) ==
			   BufferGetBlockNumber(buffers[i]) - 1);

		status = StartBufferIO(buffers[i], true, false, NULL);
		if (status != BUFFER_IO_READY_FOR_IO)
			break;
		...
		io_pages[io_buffers_len++] = BufferGetBlock(buffers[i]);
	}
```

Ve handle donatılıp smgr'a devredilir:

[../src/backend/storage/buffer/bufmgr.c:2129-2155](../src/backend/storage/buffer/bufmgr.c#L2129-L2155)

```c
	/* get a reference to wait for in WaitReadBuffers() */
	pgaio_io_get_wref(ioh, &operation->io_wref);

	/* provide the list of buffers to the completion callbacks */
	pgaio_io_set_handle_data_32(ioh, (uint32 *) io_buffers, io_buffers_len);

	pgaio_io_register_callbacks(ioh,
								persistence == RELPERSISTENCE_TEMP ?
								PGAIO_HCB_LOCAL_BUFFER_READV :
								PGAIO_HCB_SHARED_BUFFER_READV,
								flags);
	...
	smgrstartreadv(ioh, operation->smgr, forknum,
				   blocknum,
				   io_pages, io_buffers_len);
```

### 7.2 WaitReadBuffers — kısmi okumalar

`WaitReadBuffers()` bir döngüdür, çünkü tek IO işi bitirmeyebilir:

[../src/backend/storage/buffer/bufmgr.c:1792-1798](../src/backend/storage/buffer/bufmgr.c#L1792-L1798)

```c
	/*
	 * To handle partial reads, and IOMETHOD_SYNC, we re-issue IO until we're
	 * done. We may need multiple retries, not just because we could get
	 * multiple partial reads, but also because some of the remaining
	 * to-be-read buffers may have been read in by other backends, limiting
	 * the IO size.
	 */
	while (true)
```

AIO README'nin uyardığı nokta burada uygulanıyor: kısmi okumayı AIO alt sistemi
şeffaf biçimde tekrarlamaz, üst katman tekrar denemek zorundadır.

---

## 8. YENİ: read stream

`read_stream.c` bir **callback + dairesel kuyruk** makinesidir:

[../src/backend/storage/aio/read_stream.c:15-27](../src/backend/storage/aio/read_stream.c#L15-L27)

```c
 * A user-provided callback generates a stream of block numbers that is used
 * to form reads of up to io_combine_limit, by attempting to merge them with a
 * pending read.  When that isn't possible, the existing pending read is sent
 * to StartReadBuffers() so that a new one can begin to form.
 *
 * The algorithm for controlling the look-ahead distance is based on recent
 * cache / miss history, as well as whether we need to wait for I/O completion
 * after a miss.
```

Dosyanın başındaki ASCII şema (satır 39-57) iki paralel kuyruğu gösterir:
pinlenmiş buffer'lar ve devam eden IO'lar.

### 8.1 İki ayrı mesafe

Bu sürümde tek "distance" yerine **iki** değişken var:

[../src/backend/storage/aio/read_stream.c:105-115](../src/backend/storage/aio/read_stream.c#L105-L115)

```c
	/*
	 * The limits for read-ahead and combining are handled separately to allow
	 * for IO combining even in cases where the I/O subsystem can keep up at a
	 * low read-ahead distance, as doing larger IOs is more efficient.
	 *
	 * Set to 0 when the end of the stream is reached.
	 */
	int16		combine_distance;
	int16		readahead_distance;
```

- `combine_distance` — tek IO'ya kaç blok paketleyeyim (CPU verimliliği).
- `readahead_distance` — kaç blok önden okuyayım (IO paralelliği).

İkisi ayrı ayrı ikiye katlanır. `readahead_distance` **sadece gerçekten
beklendiğinde** büyür:

[../src/backend/storage/aio/read_stream.c:1229-1238](../src/backend/storage/aio/read_stream.c#L1229-L1238)

```c
		if (stream->readahead_distance > 0 && needed_wait)
		{
			/* wider temporary value, due to overflow risk */
			int32		readahead_distance;

			readahead_distance = stream->readahead_distance * 2;
			readahead_distance = Min(readahead_distance, stream->max_pinned_buffers);
			stream->readahead_distance = readahead_distance;
		}
```

`combine_distance` ise beklenmese bile IO gerektiği anda büyür — çünkü veri OS
page cache'inde olsa dahi büyük `preadv` daha az syscall demektir.

### 8.2 Pencere büyüklüğü nereden gelir

[../src/backend/storage/aio/read_stream.c:805-808](../src/backend/storage/aio/read_stream.c#L805-L808)

```c
	else if (flags & READ_STREAM_MAINTENANCE)
		max_ios = get_tablespace_maintenance_io_concurrency(tablespace_id);
	else
		max_ios = get_tablespace_io_concurrency(tablespace_id);
```

[../src/backend/storage/aio/read_stream.c:834-836](../src/backend/storage/aio/read_stream.c#L834-L836)

```c
	max_pinned_buffers = (max_ios + 1) * io_combine_limit;
	max_pinned_buffers = Min(max_pinned_buffers,
							 PG_INT16_MAX - queue_overflow - 1);
```

Yani: **pencere = (effective_io_concurrency + 1) × io_combine_limit blok**,
sonra strateji halkası ve backend pin limiti ile kırpılır.

### 8.3 Blok birleştirme

[../src/backend/storage/aio/read_stream.c:700-705](../src/backend/storage/aio/read_stream.c#L700-L705)

```c
		/* Can we merge it with the pending read? */
		if (stream->pending_read_nblocks > 0 &&
			stream->pending_read_blocknum + stream->pending_read_nblocks == blocknum)
		{
			stream->pending_read_nblocks++;
			continue;
		}
```

Callback ardışık blok verirse pending read büyür; vermezse mevcut read
başlatılır ve yenisi kurulur.

### 8.4 Fast path

Her şey cache'te ise kuyruk yönetimi tamamen atlanır:

[../src/backend/storage/aio/read_stream.c:1037-1043](../src/backend/storage/aio/read_stream.c#L1037-L1043)

```c
	/*
	 * A fast path for all-cached scans.  This is the same as the usual
	 * algorithm, but it is specialized for no I/O and no per-buffer data, so
	 * we can skip the queue management code, stay in the same buffer slot and
	 * use singular StartReadBuffer().
	 */
	if (likely(stream->fast_path))
```

Bu, tamamen bellekte olan tabloların taranmasında read stream'in ek maliyet
getirmemesini sağlar.

### 8.5 Kimler kullanıyor

| Yer | Bayraklar | Dosya |
|---|---|---|
| Sequential / TID range scan | `READ_STREAM_SEQUENTIAL \| USE_BATCHING` | [heapam.c:1297](../src/backend/access/heap/heapam.c#L1297) |
| Bitmap heap scan | `READ_STREAM_DEFAULT \| USE_BATCHING` | [heapam.c:1308](../src/backend/access/heap/heapam.c#L1308) |
| VACUUM 1. geçiş | `READ_STREAM_MAINTENANCE` (batching **yok**) | [vacuumlazy.c:1313](../src/backend/access/heap/vacuumlazy.c#L1313) |
| VACUUM 2. geçiş | `READ_STREAM_MAINTENANCE \| USE_BATCHING` | [vacuumlazy.c:2671](../src/backend/access/heap/vacuumlazy.c#L2671) |
| ANALYZE örnekleme | `READ_STREAM_MAINTENANCE \| USE_BATCHING` | [analyze.c:1304](../src/backend/commands/analyze.c#L1304) |
| btree / brin / gin / gist / hash / spgist vacuum | çeşitli | ilgili AM dosyaları |

VACUUM'un 1. geçişinde batching'in kapalı olması tesadüf değil:

[../src/backend/access/heap/vacuumlazy.c:1310-1312](../src/backend/access/heap/vacuumlazy.c#L1310-L1312)

```c
	 * This could be made safe for READ_STREAM_USE_BATCHING, but only with
	 * explicit work in heap_vac_scan_next_block.
```

Batch mode kuralı: callback, gönderilmemiş IO varken bloklayamaz — yoksa
tespit edilemeyen deadlock oluşur
([../src/include/storage/read_stream.h:44-64](../src/include/storage/read_stream.h#L44-L64)).

`READ_STREAM_SEQUENTIAL` ise `posix_fadvise` tavsiyesini **kapatır**: çekirdek
zaten ardışık erişimi kendisi tespit eder ve açık tavsiye daha kötü çalışır.

---

## 9. YENİ: AIO alt sistemi

### 9.1 Neden var

[../src/backend/storage/aio/README.md:7-10](../src/backend/storage/aio/README.md#L7-L10)

```
Until the introduction of asynchronous IO postgres relied on the operating
system to hide the cost of synchronous IO from postgres. While this worked
surprisingly well in a lot of workloads, it does not do as good a job on
prefetching and controlled writeback as we would like.
```

Ve direct IO ile birleşince asıl kazanç: veriyi çekirdek page cache'inden
buffer pool'a CPU ile kopyalamak yerine DMA ile doğrudan taşımak.

### 9.2 PgAioHandle yaşam döngüsü

[../src/include/storage/aio_internal.h:45-92](../src/include/storage/aio_internal.h#L45-L92)

```c
typedef enum PgAioHandleState
{
	/* not in use */
	PGAIO_HS_IDLE = 0,
	PGAIO_HS_HANDED_OUT,
	PGAIO_HS_DEFINED,
	PGAIO_HS_STAGED,
	PGAIO_HS_SUBMITTED,
	PGAIO_HS_COMPLETED_IO,
	PGAIO_HS_COMPLETED_SHARED,
	PGAIO_HS_COMPLETED_LOCAL,
} PgAioHandleState;
```

```
 IDLE ──pgaio_io_acquire()──► HANDED_OUT ──pgaio_io_start_readv()──► DEFINED
                                   │                                   │
                          pgaio_io_release()                    stage callback
                                   │                                   ▼
                                   ▼                                STAGED
                                 IDLE                                  │
                                                              submit (io_method)
                                                                       ▼
                                                                  SUBMITTED
                                                                       │
                                                            çekirdek/worker bitirdi
                                                                       ▼
                                                                 COMPLETED_IO
                                                                       │
                                                     complete_shared cb (herhangi backend)
                                                                       ▼
                                                              COMPLETED_SHARED
                                                                       │
                                                   complete_local cb (sahip backend)
                                                                       ▼
                                                              COMPLETED_LOCAL → IDLE
```

Kritik kural: **aynı anda sadece bir handle "handed out" olabilir.**

[../src/backend/storage/aio/aio.c:198-199](../src/backend/storage/aio/aio.c#L198-L199)

```c
	if (pgaio_my_backend->handed_out_io)
		elog(ERROR, "API violation: Only one IO can be handed out");
```

Sebep README'de: handle sayısı sınırlı ve `pgaio_io_acquire()` başarısız
olamaz; iki handle tutmaya izin verilse backend kendi kendini kilitleyebilirdi.

Handle tamamlandığında hemen yeniden kullanılabildiği için **beklemek handle
üzerinden değil, "wait reference" üzerinden yapılır** — wref handle'ın
generation'ını da taşır
([../src/backend/storage/aio/README.md:367-375](../src/backend/storage/aio/README.md#L367-L375)).

### 9.3 Katmanlı callback'ler

AIO'nun en zarif kısmı: tek IO'ya birden çok katmanın callback'i bağlanabilir.

[../src/backend/storage/aio/README.md:307-311](../src/backend/storage/aio/README.md#L307-L311)

```
Commonly several layers need to react to completion of an IO. E.g. for a read
md.c needs to check if the IO outright failed or was shorter than needed,
bufmgr.c needs to verify the page looks valid and bufmgr.c needs to update the
BufferDesc to update the buffer's state.
```

Bir shared buffer okuması için iki callback kaydedilir:

- `PGAIO_HCB_SHARED_BUFFER_READV` — bufmgr.c kaydeder ([bufmgr.c:2135](../src/backend/storage/buffer/bufmgr.c#L2135))
- `PGAIO_HCB_MD_READV` — md.c kaydeder ([md.c:1038](../src/backend/storage/smgr/md.c#L1038))

Callback'ler shared memory'de fonksiyon işaretçisi olarak değil, **ID olarak**
tutulur ([../src/backend/storage/aio/aio_callback.c:43-47](../src/backend/storage/aio/aio_callback.c#L43-L47)) —
çünkü `EXEC_BACKEND` + ASLR altında adresler process'ler arasında aynı değil.

`stage` callback'i AIO'ya kendi pin'ini aldırır:

[../src/backend/storage/buffer/bufmgr.c:8390-8398](../src/backend/storage/buffer/bufmgr.c#L8390-L8398)

```c
		/*
		 * Reflect that the buffer is now owned by the AIO subsystem.
		 *
		 * For local buffers: This can't be done just via LocalRefCount, as
		 * one might initially think, as this backend could error out while
		 * AIO is still in progress, releasing all the pins by the backend
		 * itself.
		 *
		 * This pin is released again in TerminateBufferIO().
		 */
```

Bu olmasaydı: IO uçarken backend hata verip pin'lerini bıraksa, buffer başkasına
kurban gidebilir ve çekirdek hâlâ o belleğe yazıyor olurdu.

### 9.4 pgaio_io_start_readv — en alt kat

[../src/backend/storage/aio/aio_io.c:77-88](../src/backend/storage/aio/aio_io.c#L77-L88)

```c
void
pgaio_io_start_readv(PgAioHandle *ioh,
					 int fd, int iovcnt, uint64 offset)
{
	pgaio_io_before_start(ioh);

	ioh->op_data.read.fd = fd;
	ioh->op_data.read.offset = offset;
	ioh->op_data.read.iov_length = iovcnt;

	pgaio_io_stage(ioh, PGAIO_OP_READV);
}
```

Çağrı zinciri tam olarak şudur:

```
AsyncReadBuffers()            bufmgr.c   — handle alır, buffer listesini bağlar
  └─ smgrstartreadv()         smgr.c:753 — AM'e dağıtır
       └─ mdstartreadv()      md.c:996   — blok → segment + offset, iovec kurar
            └─ FileStartReadV() fd.c:2205 — vfd → gerçek fd
                 └─ pgaio_io_start_readv()  aio_io.c:78 — handle'ı doldurur
                      └─ pgaio_io_stage()   aio.c:424   — submit eder
```

`pgaio_io_stage()` içinde IO ya kuyruğa alınır ya da hemen senkron yürütülür:

[../src/backend/storage/aio/aio.c:461-478](../src/backend/storage/aio/aio.c#L461-L478)

```c
	if (!needs_synchronous)
	{
		pgaio_my_backend->staged_ios[pgaio_my_backend->num_staged_ios++] = ioh;
		...
		/*
		 * Unless code explicitly opted into batching IOs, submit the IO
		 * immediately.
		 */
		if (!pgaio_my_backend->in_batchmode)
			pgaio_submit_staged();
	}
	else
	{
		pgaio_io_prepare_submit(ioh);
		pgaio_io_perform_synchronously(ioh);
	}
```

### 9.5 Üç yöntem

**`io_method=sync`** — AIO API'sini kullanır ama IO'yu hemen yapar:

[../src/backend/storage/aio/method_sync.c:28-38](../src/backend/storage/aio/method_sync.c#L28-L38)

```c
const IoMethodOps pgaio_sync_ops = {
	.needs_synchronous_execution = pgaio_sync_needs_synchronous_execution,
	.submit = pgaio_sync_submit,
};

static bool
pgaio_sync_needs_synchronous_execution(PgAioHandle *ioh)
{
	return true;
}
```

Regression avlamak için var. `StartReadBuffersImpl()` bu mod için ayrı bir yol
tutuyor ki AIO'dan önceki davranış birebir korunsun.

**`io_method=worker`** — varsayılan, her platformda çalışır:

[../src/backend/storage/aio/method_worker.c:6-12](../src/backend/storage/aio/method_worker.c#L6-L12)

```
 * IO workers consume IOs from a shared memory submission queue, run
 * traditional synchronous system calls, and perform the shared completion
 * handling immediately.  Client code submits most requests by pushing IOs
 * into the submission queue, and waits (if necessary) using condition
 * variables.  Some IOs cannot be performed in another process due to lack of
 * infrastructure for reopening the file, and must processed synchronously by
 * the client code when submitted.
```

Hangi IO worker'a verilemez?

[../src/backend/storage/aio/method_worker.c:472-479](../src/backend/storage/aio/method_worker.c#L472-L479)

```c
static bool
pgaio_worker_needs_synchronous_execution(PgAioHandle *ioh)
{
	return
		!IsUnderPostmaster
		|| ioh->flags & PGAIO_HF_REFERENCES_LOCAL
		|| !pgaio_io_can_reopen(ioh);
}
```

`PGAIO_HF_REFERENCES_LOCAL` = geçici tablo buffer'ı; başka process onu göremez.
Bu yüzden geçici tablolarda worker AIO devre dışı kalır ve IO senkron olur.

Kuyruk doluysa ya da lock alınamazsa geri kalan IO'lar senkron yapılır
([method_worker.c:526-534](../src/backend/storage/aio/method_worker.c#L526-L534)) — yani
worker modu asla tıkanmaz, sadece hızını kaybeder.

İlgili GUC'lar: `io_min_workers` = 2, `io_max_workers` = 8,
`io_worker_idle_timeout` = 60 s, `io_worker_launch_interval` = 100 ms
([method_worker.c:131-134](../src/backend/storage/aio/method_worker.c#L131-L134)).

**`io_method=io_uring`** — Linux 5.1+, backend başına bir ring:

[../src/backend/storage/aio/method_io_uring.c:6-11](../src/backend/storage/aio/method_io_uring.c#L6-L11)

```
 * For now we create one io_uring instance for each backend. These io_uring
 * instances have to be created in postmaster, during startup, to allow other
 * backends to process IO completions, if the issuing backend is currently
 * busy doing other things. Other backends may not use another backend's
 * io_uring instance to submit IO, that'd require additional locking that
 * would likely be harmful for performance.
```

Ring'lerin postmaster'da yaratılmasının sebebi AIO README'nin "deadlock
tehlikesi" bölümüdür: bir backend başkasının başlattığı IO'nun tamamlanmasını
işleyebilmelidir.

[../src/backend/storage/aio/method_io_uring.c:63-80](../src/backend/storage/aio/method_io_uring.c#L63-L80)

```c
const IoMethodOps pgaio_uring_ops = {
	.wait_on_fd_before_close = true,
	...
	.submit = pgaio_uring_submit,
	.wait_one = pgaio_uring_wait_one,
	.check_one = pgaio_uring_check_one,
};
```

Karşılaştırma:

| | `sync` | `worker` | `io_uring` |
|---|---|---|---|
| Gerçek asenkron | hayır | evet (dolaylı) | evet |
| Platform | hepsi | hepsi | Linux 5.1+ |
| Context switch | yok | yüksek | düşük |
| Geçici tablo AIO | — | hayır (senkron) | evet |
| Ne zaman | hata ayıklama | varsayılan | düşük gecikme isteyen yükler |

---

## 10. smgr ve md.c: blok numarasından dosyaya

`smgr.c` ince bir dispatch katmanıdır:

[../src/backend/storage/smgr/smgr.c:752-762](../src/backend/storage/smgr/smgr.c#L752-L762)

```c
void
smgrstartreadv(PgAioHandle *ioh,
			   SMgrRelation reln, ForkNumber forknum, BlockNumber blocknum,
			   void **buffers, BlockNumber nblocks)
{
	HOLD_INTERRUPTS();
	smgrsw[reln->smgr_which].smgr_startreadv(ioh,
											 reln, forknum, blocknum, buffers,
											 nblocks);
	RESUME_INTERRUPTS();
}
```

`md.c` gerçek dosya düzenini bilir. Bir tablo diskte şöyle görünür:

```
base/16384/24576         main fork, blok 0 .. RELSEG_SIZE-1        (1 GB)
base/16384/24576.1       main fork, sonraki 1 GB
base/16384/24576.2       ...
base/16384/24576_fsm     free space map
base/16384/24576_vm      visibility map
base/16384/24576_init    unlogged tablolar için "boş hali"
```

Fork listesi:

[../src/include/common/relpath.h:56-63](../src/include/common/relpath.h#L56-L63)

```c
typedef enum ForkNumber
{
	InvalidForkNumber = -1,
	MAIN_FORKNUM = 0,
	FSM_FORKNUM,
	VISIBILITYMAP_FORKNUM,
	INIT_FORKNUM,
```

Blok → offset çevirisi:

[../src/backend/storage/smgr/md.c:1007-1017](../src/backend/storage/smgr/md.c#L1007-L1017)

```c
	v = _mdfd_getseg(reln, forknum, blocknum, false,
					 EXTENSION_FAIL | EXTENSION_CREATE_RECOVERY);

	seekpos = (pgoff_t) BLCKSZ * (blocknum % ((BlockNumber) RELSEG_SIZE));

	Assert(seekpos < (pgoff_t) BLCKSZ * RELSEG_SIZE);

	nblocks_this_segment =
		Min(nblocks,
			RELSEG_SIZE - (blocknum % ((BlockNumber) RELSEG_SIZE)));
```

`RELSEG_SIZE` derleme zamanı sabitidir (varsayılan 1 GB / 8 kB = 131072 blok),
[../meson.build:539](../meson.build#L539) ile ayarlanır.

**IO birleştirme neden segment sınırında kesilir** sorusunun cevabı:

[../src/backend/storage/smgr/md.c:843-852](../src/backend/storage/smgr/md.c#L843-L852)

```c
uint32
mdmaxcombine(SMgrRelation reln, ForkNumber forknum,
			 BlockNumber blocknum)
{
	BlockNumber segoff;

	segoff = blocknum % ((BlockNumber) RELSEG_SIZE);

	return RELSEG_SIZE - segoff;
}
```

Tek `preadv` tek dosya tanıtıcısı üzerinde çalışır; segment sınırını geçen bir
istek `StartReadBuffersImpl()` tarafından önceden kırpılır.

Asenkron sürüm `mdstartreadv()` ise iovec'i **handle'ın içine** kurar
(shared memory'de olmalı, çünkü IO'yu başka process yapabilir):

[../src/backend/storage/smgr/md.c:1021-1038](../src/backend/storage/smgr/md.c#L1021-L1038)

```c
	iovcnt = pgaio_io_get_iovec(ioh, &iov);
	...
	iovcnt = buffers_to_iovec(iov, buffers, nblocks_this_segment);
	...
	if (!(io_direct_flags & IO_DIRECT_DATA))
		pgaio_io_set_flag(ioh, PGAIO_HF_BUFFERED);

	pgaio_io_set_target_smgr(ioh,
							 reln,
							 forknum,
							 blocknum,
							 nblocks,
							 false);
	pgaio_io_register_callbacks(ioh, PGAIO_HCB_MD_READV, 0);
```

**Target** kavramı burada belirir: handle'a "bu IO neyi hedefliyor" bilgisi
(RelFileLocator + fork + blok) yazılır. Worker modunda dosyayı yeniden açmak ve
hata mesajı üretmek için gerekir.

---

## 11. Local buffer'lar (geçici tablolar)

`localbuf.c` aynı kavramları kilitsiz olarak tekrar eder:

[../src/backend/storage/buffer/localbuf.c:3-5](../src/backend/storage/buffer/localbuf.c#L3-L5)

```
 *	  local buffer manager. Fast buffer manager for temporary tables,
 *	  which never need to be WAL-logged or checkpointed, etc.
```

Farklar:

| | Shared buffer | Local buffer |
|---|---|---|
| Bellek | shared memory | `calloc`, process-local |
| Boyut | `shared_buffers` | `temp_buffers` |
| Mapping | partitionlı shmem hash | basit `HTAB` (`LocalBufHash`) |
| Pin | atomik `state` refcount | `LocalRefCount[]` int32 dizisi |
| Content lock | var | **yok** (`LockBuffer` no-op döner) |
| `BM_IO_IN_PROGRESS` | var | kullanılmaz |
| Kurban | `StrategyGetBuffer` | `GetLocalVictimBuffer`, aynı clock-sweep |
| WAL / checkpoint | var | yok |
| AIO | tam destek | `io_method=worker` ile senkrona düşer |

Tahsis:

[../src/backend/storage/buffer/localbuf.c:772-775](../src/backend/storage/buffer/localbuf.c#L772-L775)

```c
	LocalBufferDescriptors = (BufferDesc *) calloc(nbufs, sizeof(BufferDesc));
	LocalBufferBlockPointers = (Block *) calloc(nbufs, sizeof(Block));
	LocalRefCount = (int32 *) calloc(nbufs, sizeof(int32));
```

Paralel worker'lar geçici tabloya erişemez, ve bu kontrol tam burada yapılır:

[../src/backend/storage/buffer/localbuf.c:766-769](../src/backend/storage/buffer/localbuf.c#L766-L769)

```c
	if (IsParallelWorker())
		ereport(ERROR,
				(errcode(ERRCODE_INVALID_TRANSACTION_STATE),
				 errmsg("cannot access temporary tables during a parallel operation")));
```

Local clock-sweep'te AIO yüzünden eklenmiş ilginç bir dal var:

[../src/backend/storage/buffer/localbuf.c:257-263](../src/backend/storage/buffer/localbuf.c#L257-L263)

```c
			else if (BUF_STATE_GET_REFCOUNT(buf_state) > 0)
			{
				/*
				 * This can be reached if the backend initiated AIO for this
				 * buffer and then errored out.
				 */
			}
```

`LocalRefCount` sıfır ama `state` refcount'u sıfır değil = AIO alt sisteminin
pin'i hâlâ duruyor.

---

# İzleme ve hata ayıklama

## pg_buffercache — buffer pool'un içini görmek

```sql
CREATE EXTENSION pg_buffercache;
```

Sütunlar ([contrib/pg_buffercache/pg_buffercache--1.2.sql:13-17](../contrib/pg_buffercache/pg_buffercache--1.2.sql#L13-L17)):
`bufferid, relfilenode, reltablespace, reldatabase, relforknumber,
relblocknumber, isdirty, usagecount, pinning_backends`.

Hangi tablo cache'i yiyor:

```sql
SELECT c.relname,
       count(*) AS buffers,
       pg_size_pretty(count(*) * 8192::bigint) AS size,
       count(*) FILTER (WHERE b.isdirty) AS dirty
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname
ORDER BY buffers DESC
LIMIT 15;
```

Clock-sweep'in sağlığı — usage count dağılımı:

```sql
SELECT * FROM pg_buffercache_usage_counts();
```

`usage_count = 0` çok fazlaysa cache basınç altındadır; hepsi 5 ise ibre her
turda 6 tur atmak zorunda kalır (yine kötü).

Fork bazlı bakış (`relforknumber`: 0 main, 1 fsm, 2 vm, 3 init):

```sql
SELECT relforknumber, count(*) FROM pg_buffercache GROUP BY 1 ORDER BY 1;
```

Özet:

```sql
SELECT * FROM pg_buffercache_summary();
-- buffers_used | buffers_unused | buffers_dirty | buffers_pinned | usagecount_avg
```

Bu ağaçta 1.7'ye kadar sürüm var: `pg_buffercache_evict()`,
`pg_buffercache_evict_relation()`, `pg_buffercache_evict_all()`,
`pg_buffercache_mark_dirty*()` ve NUMA/OS-page görünümleri
([contrib/pg_buffercache/pg_buffercache--1.6--1.7.sql](../contrib/pg_buffercache/pg_buffercache--1.6--1.7.sql)).
Bunlar test/deney içindir, üretimde kullanılmaz.

## pg_stat_io — IO'nun bağlam bazlı dökümü

[../src/backend/catalog/system_views.sql:1261-1283](../src/backend/catalog/system_views.sql#L1261-L1283)

Boyutlar: `backend_type` × `object` × `context`.

- `object`: `relation` | `temp relation` ([pgstat.h:281-282](../src/include/pgstat.h#L281-L282))
- `context`: `normal` | `vacuum` | `bulkread` | `bulkwrite` ([pgstat.h:288-296](../src/include/pgstat.h#L288-L296))

Anlamlı sütun çiftleri:

| Sütun | Ne söyler |
|---|---|
| `hits` / `reads` | Cache isabet oranı |
| `evictions` | Ring dışından buffer çalındı — halka yetmiyor |
| `reuses` | Ring içinden yeniden kullanıldı — strateji çalışıyor |
| `writebacks` | Çekirdeğe writeback tetiklendi |
| `read_bytes / reads` | **Ortalama IO boyutu** — IO birleştirme çalışıyor mu |

Ring stratejilerinin işe yarayıp yaramadığı:

```sql
SELECT backend_type, object, context, reads, hits, evictions, reuses
FROM pg_stat_io
WHERE context <> 'normal' AND (reads > 0 OR hits > 0);
```

IO birleştirmesinin gerçekten çalıştığını görmek (blok cinsinden ortalama):

```sql
SELECT backend_type, context,
       reads, read_bytes / NULLIF(reads,0) / 8192 AS avg_blocks_per_read
FROM pg_stat_io
WHERE reads > 0;
```

Bu sayı 1'e yapışıksa ya `io_combine_limit = 1`'dir ya erişim ardışık değildir
ya da `effective_io_concurrency = 0`'dır.

Backend'in kendi yazması (kötü işaret):

```sql
SELECT backend_type, writes, write_time
FROM pg_stat_io
WHERE context = 'normal' AND object = 'relation' AND writes > 0
ORDER BY writes DESC;
```

`client backend` satırında yüksek `writes` = bgwriter/checkpointer yetişemiyor.

## pg_aios — uçuştaki IO'lar

Bu sürümde gelen görünüm:

[../src/backend/catalog/system_views.sql:1567-1570](../src/backend/catalog/system_views.sql#L1567-L1570)

```sql
CREATE VIEW pg_aios AS
    SELECT * FROM pg_get_aios();
```

Sütunlar ([pg_proc.dat:12659-12660](../src/include/catalog/pg_proc.dat#L12659-L12660)):
`pid, io_id, io_generation, state, operation, off, length, target,
handle_data_len, raw_result, result, target_desc, f_sync, f_localmem,
f_buffered`.

```sql
SELECT pid, state, operation, length / 8192 AS blocks, target_desc, f_sync
FROM pg_aios
ORDER BY pid;
```

`state` doğrudan bölüm 9.2'deki state machine'dir. `f_sync = true` çok
görünüyorsa AIO gerçekte asenkron çalışmıyor demektir.

## Ayarlar

| GUC | Varsayılan | Kapsam | Etkisi |
|---|---|---|---|
| `shared_buffers` | 128 MB | postmaster | Pool boyutu = NBuffers |
| `io_method` | `worker` | postmaster | `sync` / `worker` / `io_uring` |
| `io_combine_limit` | 128 kB (16 blok) | session | Tek IO'ya kaç blok |
| `io_max_combine_limit` | 128 kB | postmaster | Yukarıdakinin tavanı |
| `effective_io_concurrency` | 16 | session | Kaç IO paralel (read stream `max_ios`) |
| `maintenance_io_concurrency` | 16 | session | VACUUM/ANALYZE için aynısı |
| `io_max_concurrency` | -1 (otomatik) | postmaster | Process başına AIO handle sayısı |
| `io_min_workers` / `io_max_workers` | 2 / 8 | sighup | IO worker havuzu |
| `temp_buffers` | — | session | Local buffer sayısı |
| `bgwriter_lru_maxpages`, `bgwriter_delay` | — | sighup | bgwriter agresifliği |

Kaynaklar:
[../src/include/storage/bufmgr.h:170-197](../src/include/storage/bufmgr.h#L170-L197),
[../src/backend/utils/misc/guc_parameters.dat:1371-1441](../src/backend/utils/misc/guc_parameters.dat#L1371-L1441),
[../src/include/storage/aio.h:32-42](../src/include/storage/aio.h#L32-L42).

Dikkat: `io_combine_limit` GUC'u `io_combine_limit_guc`'a yazar; efektif değer
`io_max_combine_limit` ile min alınarak `io_combine_limit`'e konur
([../src/backend/storage/buffer/bufmgr.c:210-217](../src/backend/storage/buffer/bufmgr.c#L210-L217)).

Ve `effective_io_concurrency = 0` **AIO'yu kapatır**:
`max_pinned_buffers = (0 + 1) * io_combine_limit` olur, tüm IO senkron
yürütülür. Read stream'in `needed_wait` mantığı bunu telafi etmeye çalışır
([read_stream.c:1200-1207](../src/backend/storage/aio/read_stream.c#L1200-L1207)).

## Denemeler

Read stream'i devre dışı bırakmadan davranışı görmek:

```sql
SET io_combine_limit = 1;          -- birleştirme kapalı
SET effective_io_concurrency = 0;  -- önden okuma kapalı
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM buyuk_tablo;
-- sonra:
RESET io_combine_limit; RESET effective_io_concurrency;
```

`EXPLAIN (BUFFERS)` çıktısındaki `shared read` / `shared hit` sayaçları
`AsyncReadBuffers()` içindeki `pgBufferUsage.shared_blks_read` ile aynı yerden
gelir ([bufmgr.c:2161-2164](../src/backend/storage/buffer/bufmgr.c#L2161-L2164)).

`DEBUG2` seviyesinde segment sınırı kırpmaları görünür
([bufmgr.c:1510](../src/backend/storage/buffer/bufmgr.c#L1510)):

```
LOG:  limiting nblocks at 131070 from 16 to 2
```

---

# Tek sayfalık özet

```
                    ═══ OKUMA YOLU ═══

  seqscan                                    ReadBuffer(rel, blk)
     │                                              │
     │ read_stream_next_buffer()                    │
     ▼                                              │
 ┌───────────────────────────────────┐              │
 │ read_stream.c                     │              │
 │  callback → 10,42,43,44,60        │              │
 │  ardışık olanları birleştir       │              │
 │  readahead_distance ×2 (bekleyince)              │
 │  combine_distance   ×2 (IO olunca)               │
 └────────────────┬──────────────────┘              │
                  └──────────┬───────────────────────┘
                             ▼
 ┌────────────────────────────────────────────────────────────────┐
 │ StartReadBuffers()                                             │
 │   for her blok:  PinBufferForBlock()                           │
 │      ├─ temp?  → LocalBufferAlloc()   [localbuf.c]             │
 │      └─ shared → BufferAlloc()                                 │
 │            1. hash lookup (partition lock SHARED)              │
 │               hit → PinBuffer(), foundPtr=true  ◄── EN SIK YOL │
 │            2. miss → GetVictimBuffer()                         │
 │                   StrategyGetBuffer():                         │
 │                     ring var mı? → GetBufferFromRing()         │
 │                     yoksa → clock sweep:                       │
 │                        usage>0 → azalt, ilerle                 │
 │                        usage=0 & pinsiz → AL                   │
 │                   kirliyse → FlushBuffer() (XLogFlush önce!)   │
 │            3. BufTableInsert (partition lock EXCLUSIVE)        │
 │               tag yaz, BM_TAG_VALID set — BM_VALID HENÜZ YOK   │
 │   sonra: AsyncReadBuffers()                                    │
 └────────────────────────────┬───────────────────────────────────┘
                              ▼
 ┌────────────────────────────────────────────────────────────────┐
 │ AsyncReadBuffers()                                             │
 │   ioh = pgaio_io_acquire()                                     │
 │   StartBufferIO(ilk blok, wait=true, &io_wref)                 │
 │      READY_FOR_IO   → BM_IO_IN_PROGRESS bizim                  │
 │      ALREADY_DONE   → başkası okudu, hit say                   │
 │      IN_PROGRESS    → foreign_io: onun IO'suna katıl           │
 │   komşu bloklar için StartBufferIO(wait=false) → io_pages[]    │
 │   pgaio_io_register_callbacks(SHARED_BUFFER_READV)             │
 │   smgrstartreadv(ioh, ...)                                     │
 └────────────────────────────┬───────────────────────────────────┘
                              ▼
 ┌────────────────────────────────────────────────────────────────┐
 │ md.c : mdstartreadv()                                          │
 │   blok → segment dosyası + offset  (RELSEG_SIZE = 1 GB)        │
 │   iovec'i HANDLE İÇİNE kur (shared memory!)                    │
 │   target = {relfilelocator, fork, blok}                        │
 │   callback PGAIO_HCB_MD_READV                                  │
 │        └─ FileStartReadV() → pgaio_io_start_readv()            │
 │              └─ pgaio_io_stage(PGAIO_OP_READV)                 │
 └────────────────────────────┬───────────────────────────────────┘
                              ▼
        ┌─────────────┬───────────────┬──────────────┐
        │   sync      │    worker     │   io_uring   │
        │ hemen       │ kuyruk + IO   │ SQE + ring   │
        │ pg_preadv() │ worker proc   │ io_uring_    │
        │             │               │  submit()    │
        └──────┬──────┴───────┬───────┴──────┬───────┘
               └──────────────┼──────────────┘
                              ▼
 ┌────────────────────────────────────────────────────────────────┐
 │ TAMAMLANMA (herhangi bir backend/worker çalıştırabilir)        │
 │   md_readv_complete()      : kısa okuma / hata var mı?         │
 │   buffer_readv_complete()  : checksum, sayfa doğrulama         │
 │        └─ TerminateBufferIO(BM_VALID set, IO_IN_PROGRESS sil,  │
 │                             AIO pin'ini bırak)                 │
 │        └─ ConditionVariableBroadcast(buffer CV)                │
 └────────────────────────────┬───────────────────────────────────┘
                              ▼
                    WaitReadBuffers()
                      pgaio_wref_wait()
                      kısmi okuma varsa → AsyncReadBuffers() tekrar
                      hata varsa → pgaio_result_report(ERROR)


                    ═══ YAZMA YOLU ═══

  MarkBufferDirty()  ── BM_DIRTY set (exclusive lock şart)
         │
         ├── checkpointer : BufferSync()   — BM_CHECKPOINT_NEEDED işaretle,
         │                                   tablespace'e göre sırala, yaz
         ├── bgwriter     : BgBufferSync() — ibrenin önünü temizle
         └── backend      : GetVictimBuffer() içinde — İSTENMEYEN
                              │
                              ▼
                        FlushBuffer()
                          share-exclusive content lock
                          XLogFlush(page LSN)   ◄── WAL kuralı
                          PageSetChecksum()
                          smgrwrite()  (hâlâ senkron)
                          ScheduleBufferTagForWriteback()


                    ═══ STATE KELİMESİ ═══

  63                                                            0
  ┌──────────────┬──────────┬────────────┬──────────────────────┐
  │ content lock │  flags   │ usagecount │       refcount       │
  │    20 bit    │  12 bit  │   4 bit    │        18 bit        │
  └──────────────┴──────────┴────────────┴──────────────────────┘
     20 bit         BM_DIRTY   0..5          pin sayısı
     shared/SE/X    BM_VALID   clock sweep
                    BM_TAG_VALID
                    BM_IO_IN_PROGRESS  ◄── pin ve kilitten BAĞIMSIZ
```
