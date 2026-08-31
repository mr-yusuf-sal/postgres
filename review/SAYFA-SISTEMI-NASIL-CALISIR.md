# PostgreSQL Sayfa (Page) Sistemi Nasıl Çalışır

> PostgreSQL **20devel** ağacı üzerinden yazıldı. Satır numaraları bu sürüme aittir.

## 30 saniyelik özet

PostgreSQL diskteki her şeyi **8 kB'lik sabit boyutlu bloklar** hâlinde tutar.
Bir blok belleğe alınıp biçimlendirildiğinde ona **page** denir. Page bir
*slotted page*'tir: başlıktan sonra **line pointer dizisi** öne doğru, **tuple
gövdeleri** sona doğru büyür; ortada kalan boşluk dolduğunda sayfa doludur.

```
0                                                                    8192
+----------------+--------------------------------+---------------------+
| PageHeaderData | linp1 linp2 linp3 ... linpN     |                     |
| (24 bayt)      | (her biri 4 bayt)               |                     |
+----------------+--------------------------------+                     |
                                          ^ pd_lower                     |
                                                                         |
                 v pd_upper                                              |
+------------------------+-------------------------+--------------------+
|      BOŞ ALAN          | tupleN ... tuple2 tuple1 |  "special space"   |
+------------------------+-------------------------+--------------------+
                                                    ^ pd_special
```

Zihinsel model, beş madde:

1. **Blok = I/O birimi.** Diskten hep 8 kB okunur, hep 8 kB yazılır. Tek bir
   satırı okumak için bile sayfanın tamamı buffer pool'a gelir.
2. **Line pointer bir dolaylama katmanıdır.** `ctid = (blok, offset)`
   satırın bayt adresini değil, **kaçıncı line pointer** olduğunu söyler.
   Tuple sayfa içinde yer değiştirebilir; index'lerdeki `ctid`'ler bozulmaz.
3. **İki uçtan büyüme.** Line pointer'lar `pd_lower`'ı ileri iter, tuple
   gövdeleri `pd_upper`'ı geri çeker. `pd_lower > pd_upper` olamaz.
4. **Special space AM'e özeldir.** Heap sayfasında boyutu sıfırdır; B-tree
   sayfasında kardeş blok numaralarını tutan 16 baytlık yapı oturur.
5. **Her sayfa kendini doğrular.** `pd_lsn` WAL kuralını, `pd_checksum` disk
   bozulmasını, `pd_lower/upper/special` üçlüsü tutarlılığı kontrol eder.

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/include/storage/bufpage.h](../src/include/storage/bufpage.h) | `PageHeaderData`, flag bitleri, tüm erişim inline fonksiyonları |
| [../src/include/storage/itemid.h](../src/include/storage/itemid.h) | `ItemIdData` (line pointer) ve dört durumu |
| [../src/include/storage/itemptr.h](../src/include/storage/itemptr.h) | `ItemPointerData` = `ctid` = (blok no, offset no) |
| [../src/backend/storage/page/bufpage.c](../src/backend/storage/page/bufpage.c) | `PageInit`, `PageAddItemExtended`, defragmentasyon, boş alan hesapları |
| [../src/backend/storage/page/checksum.c](../src/backend/storage/page/checksum.c) | Checksum implementasyon seçimi (AVX2 / fallback) |
| [../src/include/storage/checksum_impl.h](../src/include/storage/checksum_impl.h) | FNV-1a türevi blok checksum algoritması |
| [../src/backend/storage/smgr/md.c](../src/backend/storage/smgr/md.c) | Blok numarası → segment dosyası + dosya içi offset eşlemesi |
| [../src/backend/storage/freespace/freespace.c](../src/backend/storage/freespace/freespace.c) | FSM: hangi sayfada ne kadar yer var |
| [../src/backend/access/heap/visibilitymap.c](../src/backend/access/heap/visibilitymap.c) | VM: sayfa başına 2 bit görünürlük özeti |

## 1. Neden sayfa? Blok, segment, fork

Bir tablo diskte tek bir dosya değildir. `md.c` üç seviyeli bir eşleme yapar:

```
tablo (relation)
  └─ fork      : main (0) / fsm (1) / vm (2) / init (3)
       └─ segment dosyası : 16384, 16384.1, 16384.2 ...   (her biri 1 GB)
            └─ blok       : dosya içinde 8 kB'lik dilim
                 └─ page  : biçimlendirilmiş blok
```

Segmentleme kararının gerekçesi kaynakta açıkça yazılı — işletim sistemi dosya
boyutu sınırı: [../src/backend/storage/smgr/md.c:45-59](../src/backend/storage/smgr/md.c#L45-L59)

```c
 * we break relations up into "segment" files that are each shorter than
 * the OS file size limit.  The segment size is set by the RELSEG_SIZE
 * configuration constant in pg_config.h.
```

Blok numarasından dosya içi offset'e geçiş tek satırlık modüler aritmetik:
[../src/backend/storage/smgr/md.c:518](../src/backend/storage/smgr/md.c#L518)

```c
	seekpos = (pgoff_t) BLCKSZ * (blocknum % ((BlockNumber) RELSEG_SIZE));
```

Segment numarası da `blocknum / RELSEG_SIZE`. İki sabit derleme zamanında
belirlenir; varsayılanları [../meson.build:509-539](../meson.build#L509-L539) ile
[../meson_options.txt:5-18](../meson_options.txt#L5-L18) içinde:

| Sabit | Varsayılan | Anlamı |
|---|---|---|
| `BLCKSZ` | 8192 bayt | Bir bloğun boyutu |
| `RELSEG_SIZE` | 131072 blok (1 GB) | Bir segment dosyasındaki blok sayısı |
| `XLOG_BLCKSZ` | 8192 bayt | WAL bloğu (ayrı sabit) |

Bunları değiştirmek `initdb` gerektirir; çalışan bir küme üzerinde ayarlanamaz.

## 2. Page header — 24 bayt

[../src/include/storage/bufpage.h:184-197](../src/include/storage/bufpage.h#L184-L197)

```c
typedef struct PageHeaderData
{
	PageXLogRecPtr pd_lsn;		/* LSN: next byte after last byte of xlog
								 * record for last change to this page */
	uint16		pd_checksum;	/* checksum */
	uint16		pd_flags;		/* flag bits, see below */
	LocationIndex pd_lower;		/* offset to start of free space */
	LocationIndex pd_upper;		/* offset to end of free space */
	LocationIndex pd_special;	/* offset to start of special space */
	uint16		pd_pagesize_version;
	TransactionId pd_prune_xid; /* oldest prunable XID, or zero if none */
	ItemIdData	pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* line pointer array */
} PageHeaderData;
```

Toplam: 8 + 2 + 2 + 2 + 2 + 2 + 2 + 4 = **24 bayt**. `pd_linp` esnek dizi
olduğu için başlık boyutuna dahil değil:
[../src/include/storage/bufpage.h:242](../src/include/storage/bufpage.h#L242)

```c
#define SizeOfPageHeaderData (offsetof(PageHeaderData, pd_linp))
```

Alan alan ne işe yarıyor:

**`pd_lsn` — WAL kuralının bekçisi.** Buffer manager kirli bir sayfayı diske
yazmadan önce WAL'ın en az bu LSN'e kadar flush edilmiş olmasını şart koşar.
Kaynaktaki ifade: [../src/include/storage/bufpage.h:152-154](../src/include/storage/bufpage.h#L152-L154)

```c
 * The LSN is used by the buffer manager to enforce the basic rule of WAL:
 * "thou shalt write xlog before data".  A dirty buffer cannot be dumped
 * to disk until xlog has been flushed at least as far as the page's LSN.
```

İlginç bir ayrıntı: LSN 64 bitlik tek değer olarak saklanır ama **little-endian
makinelerde iki 32-bit yarısı takas edilmiş hâlde** durur — tarihsel olarak
`PageXLogRecPtr` iki `uint32` alanlı bir struct'tı ve disk formatı korunuyor:
[../src/include/storage/bufpage.h:122-134](../src/include/storage/bufpage.h#L122-L134)

```c
static inline XLogRecPtr
PageXLogRecPtrGet(const volatile PageXLogRecPtr *val)
{
	PageXLogRecPtr tmp = {val->lsn};

	return (tmp.lsn << 32) | (tmp.lsn >> 32);
}
```

**`pd_flags` — üç bit.** [../src/include/storage/bufpage.h:213-218](../src/include/storage/bufpage.h#L213-L218)

| Bit | Anlamı | Kesin mi? |
|---|---|---|
| `PD_HAS_FREE_LINES` | `pd_lower` öncesinde `LP_UNUSED` line pointer var | Hint — WAL'lanmaz, yanlış olabilir |
| `PD_PAGE_FULL` | Bir UPDATE yeni sürüm için yer bulamadı, prune gerekebilir | Hint |
| `PD_ALL_VISIBLE` | Sayfadaki tüm tuple'lar herkese görünür | VM ile birlikte tutulur |

İlk ikisi için kaynak açıkça uyarıyor: *"This should be considered a hint rather
than the truth, since changes to it are not WAL-logged."* Kod bu yüzden hint'e
güvenip sonra doğruluyor — `PageAddItemExtended` boş slot bulamazsa hint'i
temizliyor ([../src/backend/storage/page/bufpage.c:286-290](../src/backend/storage/page/bufpage.c#L286-L290)).

**`pd_pagesize_version` — tek alanda iki bilgi.** Sayfa boyutu 256'nın katı
olmak zorunda olduğu için alt 8 bit boşta kalıyor ve versiyon numarası oraya
sıkıştırılmış: [../src/include/storage/bufpage.h:301-315](../src/include/storage/bufpage.h#L301-L315)

```c
static inline Size
PageGetPageSize(const PageData *page)
{
	return (Size) (((const PageHeaderData *) page)->pd_pagesize_version & (uint16) 0xFF00);
}
```

Güncel layout versiyonu 8.3'ten beri **4**
([../src/include/storage/bufpage.h:232](../src/include/storage/bufpage.h#L232)).

**`pd_prune_xid`** — sayfadaki en eski budanabilir XID. Sadece bir ipucu; heap
sayfalarında kullanılır, index sayfalarında kullanılmaz
([../src/include/storage/bufpage.h:167-168](../src/include/storage/bufpage.h#L167-L168)). Ayrıntısı
[UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md) ve
[VACUUM-NASIL-CALISIR.md](VACUUM-NASIL-CALISIR.md) içinde.

## 3. Line pointer — 4 bayt, 32 bit, üç alan

[../src/include/storage/itemid.h:25-30](../src/include/storage/itemid.h#L25-L30)

```c
typedef struct ItemIdData
{
	unsigned	lp_off:15,		/* offset to tuple (from start of page) */
				lp_flags:2,		/* state of line pointer, see below */
				lp_len:15;		/* byte length of tuple */
} ItemIdData;
```

`lp_off` ve `lp_len`'in 15 bit olması **maksimum sayfa boyutunu 32 kB'a
kilitler** — `BLCKSZ`'yi 32768'in üstüne çıkaramamanızın sebebi budur
([../src/include/storage/bufpage.h:180-181](../src/include/storage/bufpage.h#L180-L181)).

Dört durum ([../src/include/storage/itemid.h:38-41](../src/include/storage/itemid.h#L38-L41)):

| Durum | `lp_len` | Anlamı |
|---|---|---|
| `LP_UNUSED` | 0 | Boş slot, hemen yeniden kullanılabilir |
| `LP_NORMAL` | >0 | Gerçek bir tuple'a işaret ediyor |
| `LP_REDIRECT` | 0 | HOT yönlendirmesi — `lp_off` başka bir **offset numarası** tutar |
| `LP_DEAD` | 0 veya >0 | Ölü; gövdesi hâlâ duruyor olabilir |

`LP_REDIRECT` özel: `lp_off` bayt offset'i değil, sayfa içindeki başka bir line
pointer'ın numarası. Bu yüzden ayrı bir makro var:
[../src/include/storage/itemid.h:78-79](../src/include/storage/itemid.h#L78-L79)

```c
#define ItemIdGetRedirect(itemId) \
   ((itemId)->lp_off)
```

Genel kural da başlıkta yazılı: *"By convention, lp_len == 0 in every line
pointer that does not have storage, independently of its lp_flags state."*
([../src/include/storage/itemid.h:21-23](../src/include/storage/itemid.h#L21-L23))

**Dolaylama neden var?** `ctid` bayt offset'i değil slot numarası tuttuğu için
tuple gövdeleri sayfa içinde serbestçe kaydırılabilir. Kaynağın kendi ifadesi:
[../src/include/storage/bufpage.h:63-69](../src/include/storage/bufpage.h#L63-L69)

```c
 * tuple1..N are added "backwards" on the page.  Since an ItemPointer
 * offset is used to access an ItemId entry rather than an actual
 * byte-offset position, tuples can be physically shuffled on a page
 * whenever the need arises.
```

`ctid`'nin kendisi 6 bayt: 4 bayt blok + 2 bayt offset
([../src/include/storage/itemptr.h:36-40](../src/include/storage/itemptr.h#L36-L40)).
Offset numaraları **1'den başlar**, bu yüzden `PageGetItemId` bir çıkarma yapar:
[../src/include/storage/bufpage.h:268-272](../src/include/storage/bufpage.h#L268-L272)

```c
static inline ItemId
PageGetItemId(Page page, OffsetNumber offsetNumber)
{
	return &((PageHeader) page)->pd_linp[offsetNumber - 1];
}
```

Sayfadaki kaç line pointer olduğu ayrıca saklanmaz, `pd_lower`'dan hesaplanır:
[../src/include/storage/bufpage.h:396-405](../src/include/storage/bufpage.h#L396-L405)

```c
	if (pageheader->pd_lower <= SizeOfPageHeaderData)
		return 0;
	else
		return (pageheader->pd_lower - SizeOfPageHeaderData) / sizeof(ItemIdData);
```

## 4. Sayfanın doğuşu — `PageInit`

[../src/backend/storage/page/bufpage.c:41-60](../src/backend/storage/page/bufpage.c#L41-L60)

```c
void
PageInit(Page page, Size pageSize, Size specialSize)
{
	PageHeader	p = (PageHeader) page;

	specialSize = MAXALIGN(specialSize);

	Assert(pageSize == BLCKSZ);
	Assert(pageSize > specialSize + SizeOfPageHeaderData);

	/* Make sure all fields of page are zero, as well as unused space */
	MemSet(p, 0, pageSize);

	p->pd_flags = 0;
	p->pd_lower = SizeOfPageHeaderData;
	p->pd_upper = pageSize - specialSize;
	p->pd_special = pageSize - specialSize;
	PageSetPageSizeAndVersion(page, pageSize, PG_PAGE_LAYOUT_VERSION);
	/* p->pd_prune_xid = InvalidTransactionId;		done by above MemSet */
}
```

Dikkat çeken üç nokta:

- **Checksum burada hesaplanmaz.** Fonksiyonun başındaki yorum açık: *"we don't
  calculate an initial checksum here; that's not done until it's time to write."*
- **Sayfa baştan sona sıfırlanır.** Boş alanda eski veri kalmasın diye — bu
  sayede sayfanın diske yazılan hâli deterministiktir.
- **Sıfırlanmış sayfa geçerlidir.** `PageIsNew` sadece `pd_upper == 0`'a bakar
  ([../src/include/storage/bufpage.h:258-262](../src/include/storage/bufpage.h#L258-L262)); `PageInit`
  çağrılmamış tümü-sıfır bir blok "yeni" sayılır ve hata vermez.

## 5. Tuple eklemek — `PageAddItemExtended`

Tek giriş noktası. Heap de, B-tree de, GIN de buradan geçer.
[../src/backend/storage/page/bufpage.c:202-365](../src/backend/storage/page/bufpage.c#L202-L365)

Akış beş adım:

**1) Paranoyak başlık kontrolü** — bozuk pointer'la devam etmektense PANIC:
[../src/backend/storage/page/bufpage.c:220-227](../src/backend/storage/page/bufpage.c#L220-L227)

```c
	if (phdr->pd_lower < SizeOfPageHeaderData ||
		phdr->pd_lower > phdr->pd_upper ||
		phdr->pd_upper > phdr->pd_special ||
		phdr->pd_special > BLCKSZ)
		ereport(PANIC,
				(errcode(ERRCODE_DATA_CORRUPTED),
				 errmsg("corrupted page pointers: lower = %u, upper = %u, special = %u",
						phdr->pd_lower, phdr->pd_upper, phdr->pd_special)));
```

**2) Slot seçimi.** Çağıran offset vermediyse önce `PD_HAS_FREE_LINES` hint'ine
bakılır; hint doğruysa dizinin **başından** taranıp ilk `LP_UNUSED` slot alınır.
Öne öncelik vermenin gerekçesi kaynakta yazılı — böylece kullanılmayan slot'lar
dizinin sonunda bitişik bir grup hâlinde toplanır ve
`PageTruncateLinePointerArray` onları kesip atabilir:
[../src/backend/storage/page/bufpage.c:262-285](../src/backend/storage/page/bufpage.c#L262-L285)

```c
			/*
			 * Always use earlier items first.  PageTruncateLinePointerArray
			 * can only truncate unused items when they appear as a contiguous
			 * group at the end of the line pointer array.
			 */
			for (offsetNumber = FirstOffsetNumber;
				 offsetNumber < limit;	/* limit is maxoff+1 */
				 offsetNumber++)
```

**3) Sığıyor mu?** Yeni `pd_lower` ve `pd_upper` hesaplanıp karşılaştırılır.
Aritmetik bilinçli olarak **işaretli** yapılıyor, taşma olmasın diye:
[../src/backend/storage/page/bufpage.c:319-329](../src/backend/storage/page/bufpage.c#L319-L329)

```c
	if (offsetNumber == limit || needshuffle)
		lower = phdr->pd_lower + sizeof(ItemIdData);
	else
		lower = phdr->pd_lower;

	alignedSize = MAXALIGN(size);

	upper = (int) phdr->pd_upper - (int) alignedSize;

	if (lower > upper)
		return InvalidOffsetNumber;
```

**4) Yazma.** Line pointer kurulur, gövde `pd_upper`'ın yeni değerine kopyalanır:
[../src/backend/storage/page/bufpage.c:340-358](../src/backend/storage/page/bufpage.c#L340-L358)

```c
	/* set the line pointer */
	ItemIdSetNormal(itemId, upper, size);
	...
	/* copy the item's data onto the page */
	memcpy((char *) page + upper, item, size);
```

**5) Başlık güncellenir.** İki alan, iki satır
([../src/backend/storage/page/bufpage.c:360-362](../src/backend/storage/page/bufpage.c#L360-L362)):

```c
	/* adjust page header */
	phdr->pd_lower = (LocationIndex) lower;
	phdr->pd_upper = (LocationIndex) upper;
```

Fonksiyonun başındaki büyük harfli uyarı önemli:
[../src/backend/storage/page/bufpage.c:200](../src/backend/storage/page/bufpage.c#L200)

```c
 *	!!! EREPORT(ERROR) IS DISALLOWED HERE !!!
```

Sebebi: bu fonksiyon WAL kritik bölümünde (`critical section`) çağrılır, orada
`ERROR` atmak PANIC'e dönüşür. Bu yüzden reddetme durumları `WARNING` +
`InvalidOffsetNumber` döndürerek bildirilir — sadece **zaten bozuk** bir sayfa
PANIC üretir.

## 6. Boş alan — dört farklı cevap

Sayfada ne kadar yer var sorusunun tek cevabı yok; kimin sorduğuna bağlı.

| Fonksiyon | Ne döndürür | Kim kullanır |
|---|---|---|
| `PageGetExactFreeSpace` | `pd_upper - pd_lower`, ham | FSM güncellemesi |
| `PageGetFreeSpace` | Ham eksi bir line pointer (4 bayt) | Index sayfaları |
| `PageGetFreeSpaceForMultipleTuples` | Ham eksi *n* line pointer | Toplu ekleme yapan index'ler |
| `PageGetHeapFreeSpace` | Yukarıdakine ek olarak `MaxHeapTuplesPerPage` sınırı | Heap sayfaları |

([../src/backend/storage/page/bufpage.c:915-1050](../src/backend/storage/page/bufpage.c#L915-L1050))

Fark neden var? Heap sayfasında line pointer sayısının sert bir üst sınırı var
ve `LP_DEAD` / `LP_REDIRECT` slot'ları yüzünden **bayt olarak yer varken slot
olarak yer bitmiş** olabilir. `PageGetHeapFreeSpace` bu durumda 0 döndürür:
[../src/backend/storage/page/bufpage.c:1010-1028](../src/backend/storage/page/bufpage.c#L1010-L1028)

```c
		nline = PageGetMaxOffsetNumber(page);
		if (nline >= MaxHeapTuplesPerPage)
		{
			if (PageHasFreeLinePointers(page))
			{
				/*
				 * Since this is just a hint, we must confirm that there is
				 * indeed a free line pointer
				 */
```

Yine hint'e güvenilmeyip doğrulanıyor. Tersi durumda ise ilginç bir teslimiyet
var — hint "boş slot yok" diyorsa, yanlış bile olsa kabul ediliyor, çünkü
`PageAddItem` de aynı hint'e inanacak:
[../src/backend/storage/page/bufpage.c:1039-1046](../src/backend/storage/page/bufpage.c#L1039-L1046)

```c
			else
			{
				/*
				 * Although the hint might be wrong, PageAddItem will believe
				 * it anyway, so we must believe it too.
				 */
				space = 0;
			}
```

Türetilmiş sınırlar (8 kB sayfa, 64-bit `MAXALIGN` ile):

| Sabit | Formül | Değer |
|---|---|---|
| `SizeOfPageHeaderData` | `offsetof(PageHeaderData, pd_linp)` | 24 bayt |
| `MaxHeapTupleSize` | `BLCKSZ - MAXALIGN(24 + 4)` | 8160 bayt |
| `MaxHeapTuplesPerPage` | `(8192 - 24) / (MAXALIGN(23) + 4)` | 291 |
| `MinHeapTupleSize` | `MAXALIGN(SizeofHeapTupleHeader)` | 24 bayt |

[../src/include/access/htup_details.h:601-617](../src/include/access/htup_details.h#L601-L617)

```c
#define MaxHeapTupleSize  (BLCKSZ - MAXALIGN(SizeOfPageHeaderData + sizeof(ItemIdData)))
#define MinHeapTupleSize  MAXALIGN(SizeofHeapTupleHeader)
...
#define MaxHeapTuplesPerPage	\
	((int) ((BLCKSZ - SizeOfPageHeaderData) / \
			(MAXALIGN(SizeofHeapTupleHeader) + sizeof(ItemIdData))))
```

**8160 baytlık sınır TOAST'ın varlık sebebidir** — daha büyük bir satır sayfaya
sığmadığı için sıkıştırılır ya da parçalanıp yan tabloya taşınır. Ayrıntısı
[TOAST-NASIL-CALISIR.md](TOAST-NASIL-CALISIR.md) içinde.

## 7. Fragmentasyon ve defragmentasyon

Tuple'lar silindikçe sayfanın gövde bölgesinde delikler oluşur. `pd_upper` ile
`pd_special` arasındaki alan parça parça boşalır ama `pd_upper` yerinde kalır —
yani **boş bayt var, ama tek parça hâlinde kullanılamaz**.

`PageRepairFragmentation` bunu toplar. VACUUM ve HOT pruning çağırır, ve
**cleanup lock** ister (kimse sayfa içine pointer tutuyor olmamalı):
[../src/backend/storage/page/bufpage.c:702-740](../src/backend/storage/page/bufpage.c#L702-L740)

Akış:
1. Line pointer dizisi taranır, gövdesi olan girişler toplanır.
2. `compactify_tuples` gövdeleri sayfanın sonuna doğru sıkıştırır.
3. `pd_upper` yukarı çekilir, `PD_HAS_FREE_LINES` hint'i yeniden hesaplanır.

`compactify_tuples`'ın iki yolu var ve ayrım performans içindir:
[../src/backend/storage/page/bufpage.c:462-478](../src/backend/storage/page/bufpage.c#L462-L478)

```c
 * Callers may pass 'presorted' as true if the 'itemidbase' array is sorted in
 * descending order of itemoff.  When this is true we can just memmove()
 * tuples towards the end of the page.  This is quite a common case as it's
 * the order that tuples are initially inserted into pages.
```

Sıralıysa tek `memmove` yeter. Değilse geçici tampona kopyalanıp geri yazılır —
ve **geri yazarken sıra düzeltilir**, böylece bir sonraki defragmentasyon hızlı
yolu yakalar. Kendi kendini iyileştiren bir tasarım.

Ayrı bir işlem daha var: `PageTruncateLinePointerArray` dizinin **sonundaki**
bitişik `LP_UNUSED` grubunu tamamen keser, `pd_lower`'ı geri çeker
([../src/backend/storage/page/bufpage.c:843-868](../src/backend/storage/page/bufpage.c#L843-L868)). Bu, madde
5'teki "her zaman baştaki slot'u kullan" kuralının karşılığı — kural olmasa
dizinin sonu hiç boşalmaz ve dizi hiç küçülmezdi.

**Heap ile index farkı:** heap'te silinen tuple'ın line pointer'ı kalır
(`LP_DEAD`), çünkü index'lerden ona `ctid` ile işaret ediliyor olabilir.
Index'te ise line pointer da sıkıştırılıp atılır — `PageIndexTupleDelete`'in
başındaki yorum bunu söylüyor:
[../src/backend/storage/page/bufpage.c:1053-1058](../src/backend/storage/page/bufpage.c#L1053-L1058)

```c
 * This routine does the work of removing a tuple from an index page.
 *
 * Unlike heap pages, we compact out the line pointer for the removed tuple.
```

## 8. Special space — AM'in kendi köşesi

`pd_special`'dan sayfa sonuna kadar olan bölge access method'a aittir. Boyutu
`PageInit`'e parametre olarak verilir ve `MAXALIGN`'lanır.

| AM | Special space içeriği | Boyut |
|---|---|---|
| heap | yok | 0 |
| B-tree | `BTPageOpaqueData` — kardeş bloklar, seviye, flag'ler | 16 bayt |
| hash | `HashPageOpaqueData` | 16 bayt |
| GIN / GiST / SP-GiST | kendi opaque yapıları | değişken |

B-tree örneği ([../src/include/access/nbtree.h:63-74](../src/include/access/nbtree.h#L63-L74)):

```c
typedef struct BTPageOpaqueData
{
	BlockNumber btpo_prev;		/* left sibling, or P_NONE if leftmost */
	BlockNumber btpo_next;		/* right sibling, or P_NONE if rightmost */
	uint32		btpo_level;		/* tree level --- zero for leaf pages */
	uint16		btpo_flags;		/* flag bits, see below */
	BTCycleId	btpo_cycleid;	/* vacuum cycle ID of latest split */
} BTPageOpaqueData;

#define BTPageGetOpaque(page) ((BTPageOpaque) PageGetSpecialPointer(page))
```

`btpo_next` (right link) Lehman-Yao protokolünün temeli — ayrıntısı
[BTREE-INDEX-NASIL-CALISIR.md](BTREE-INDEX-NASIL-CALISIR.md) içinde.

Erişim makrosu Assert'lerle korunuyor, çünkü `PageInit` öncesi kullanım sık
yapılan bir hata: [../src/include/storage/bufpage.h:347-368](../src/include/storage/bufpage.h#L347-L368)

```c
#define PageGetSpecialPointer(page) \
( \
	PageValidateSpecialPointer(page), \
	((page) + ((PageHeader) (page))->pd_special) \
)
```

## 9. Checksum ve sayfa doğrulama

Checksum diskten okumada kontrol edilir, diske yazmadan hemen önce hesaplanır.

**Yazarken** — [../src/backend/storage/page/bufpage.c:1517-1530](../src/backend/storage/page/bufpage.c#L1517-L1530)

```c
void
PageSetChecksum(Page page, BlockNumber blkno)
{
	HOLD_INTERRUPTS();
	/* If we don't need a checksum, just return */
	if (PageIsNew(page) || !DataChecksumsNeedWrite())
	{
		RESUME_INTERRUPTS();
		return;
	}

	((PageHeader) page)->pd_checksum = pg_checksum_page(page, blkno);
	RESUME_INTERRUPTS();
}
```

Yorumdaki tarihsel not önemli: eskiden sayfanın **kopyası** üzerinde
hesaplanırdı, çünkü hint bit'ler eşzamanlı değişebiliyordu; artık I/O sürerken
hint bit yazılmadığı için kopya gerekmiyor
([../src/backend/storage/page/bufpage.c:1513-1515](../src/backend/storage/page/bufpage.c#L1513-L1515)).

**Okurken** — `PageIsVerified` üç şeyi ayrı ayrı kontrol eder:
[../src/backend/storage/page/bufpage.c:93-172](../src/backend/storage/page/bufpage.c#L93-L172)

1. **Checksum** — hesaplanan değer `pd_checksum`'a eşit mi?
2. **Başlık tutarlılığı** — dört eşitsizlik ve bir hizalama:
   [../src/backend/storage/page/bufpage.c:137-142](../src/backend/storage/page/bufpage.c#L137-L142)

```c
		if ((p->pd_flags & ~PD_VALID_FLAG_BITS) == 0 &&
			p->pd_lower <= p->pd_upper &&
			p->pd_upper <= p->pd_special &&
			p->pd_special <= BLCKSZ &&
			p->pd_special == MAXALIGN(p->pd_special))
			header_sane = true;
```

3. **Tümü-sıfır özel durumu** — sıfır sayfa **geçerli** kabul edilir:
   [../src/backend/storage/page/bufpage.c:149-152](../src/backend/storage/page/bufpage.c#L149-L152)

```c
	pagebytes = (size_t *) page;

	if (pg_memory_is_all_zeros(pagebytes, BLCKSZ))
		return true;
```

Neden? Kaynakta senaryo tarif edilmiş: bir backend relation'ı genişletir, WAL
kaydını yazamadan çöker; çekirdek sıfırlanmış sayfayı zaten dosyaya yazmıştır ve
yeniden başlatmadan sonra orada durur. Bu sayfa hata vermemeli, VACUUM zamanla
temizleyip kullanılabilir hâle getirmeli
([../src/backend/storage/page/bufpage.c:71-79](../src/backend/storage/page/bufpage.c#L71-L79)).

**Algoritma.** FNV-1a türevi, 32 paralel toplam üzerinden — SIMD'e uygun olsun
diye: [../src/include/storage/checksum_impl.h:108-140](../src/include/storage/checksum_impl.h#L108-L140)

```c
#define N_SUMS 32
#define FNV_PRIME 16777619
...
#define CHECKSUM_COMP(checksum, value) \
	...
	(checksum) = __tmp * FNV_PRIME ^ (__tmp >> 17); \
```

Blok numarası sonunda XOR'lanır — böylece **yanlış yere yazılmış** (aksi hâlde
kendi içinde tutarlı) bir sayfa da yakalanır:
[../src/include/storage/checksum_impl.h:193](../src/include/storage/checksum_impl.h#L193)

```c
	checksum ^= blkno;
```

Bu ağaçta AVX2 hızlandırması çalışma zamanında seçiliyor
([../src/backend/storage/page/checksum.c:51-64](../src/backend/storage/page/checksum.c#L51-L64)):

```c
static uint32
pg_checksum_choose(const PGChecksummablePage *page)
{
	pg_checksum_block = pg_checksum_block_fallback;

#ifdef USE_AVX2_WITH_RUNTIME_CHECK
	if (x86_feature_available(PG_AVX2))
		pg_checksum_block = pg_checksum_block_avx2;
#endif

	return pg_checksum_block(page);
}
```

Bir tasarım kararı özellikle belirtilmiş: **sayfada "checksum geçerli mi" diyen
bir bayrak yoktur.** Olsaydı, bozuk olabilecek sayfanın içeriğine bakıp o
sayfayı doğrulayıp doğrulamayacağımıza karar vermek zorunda kalırdık — döngüsel
bir bağımlılık ([../src/include/storage/bufpage.h:156-165](../src/include/storage/bufpage.h#L156-L165)).

## 10. Sayfaların üstündeki iki harita: FSM ve VM

Her ikisi de ayrı fork'lardır ve **kendileri de normal page'dir** — aynı başlık,
aynı `PageInit`.

**FSM (Free Space Map)** — hangi heap sayfasında ne kadar yer var. Bayt bayt
değil, 256 kategoriye yuvarlanmış:
[../src/backend/storage/freespace/freespace.c:64-65](../src/backend/storage/freespace/freespace.c#L64-L65)

```c
#define FSM_CATEGORIES	256
#define FSM_CAT_STEP	(BLCKSZ / FSM_CATEGORIES)
```

Yani çözünürlük **32 bayt**; `fsm_space_avail_to_cat` aşağı yuvarlar
([../src/backend/storage/freespace/freespace.c:401-412](../src/backend/storage/freespace/freespace.c#L401-L412)) — FSM
asla olduğundan fazla yer vaat etmez. Sayfa başına slot sayısı
`SlotsPerFSMPage = LeafNodesPerPage`
([../src/include/storage/fsm_internals.h:51-61](../src/include/storage/fsm_internals.h#L51-L61)) — 8 kB'de
`(8192 - 24 - 4) - (4096 - 1) = 4069`. Her FSM sayfası içinde bir max-heap ağacı
var, üstünde de sayfalar arası **3 seviyeli** bir ağaç:
[../src/backend/storage/freespace/freespace.c:75](../src/backend/storage/freespace/freespace.c#L75)

```c
#define FSM_TREE_DEPTH	((SlotsPerFSMPage >= 1626) ? 3 : 4)
```

**VM (Visibility Map)** — heap sayfası başına **2 bit**:
[../src/include/access/visibilitymapdefs.h:17-21](../src/include/access/visibilitymapdefs.h#L17-L21)

```c
#define BITS_PER_HEAPBLOCK 2
#define VISIBILITYMAP_ALL_VISIBLE	0x01
#define VISIBILITYMAP_ALL_FROZEN	0x02
```

Bir VM sayfası `(8192 - 24) * 4 = 32672` heap bloğunu, yani ~255 MB'lık heap'i
özetler ([../src/backend/access/heap/visibilitymap.c:119-125](../src/backend/access/heap/visibilitymap.c#L119-L125)).
Index-only scan'in heap'e gitmeden cevap verebilmesi ve VACUUM'un sayfa atlaması
bu iki bite dayanır.

## 11. İzleme — sayfaları gözle görmek

`pageinspect` eklentisi sayfa içini SQL'e açar.

```sql
CREATE EXTENSION pageinspect;

-- Başlık: pd_lower / pd_upper / pd_special ve flag'ler
SELECT lower, upper, special, pagesize, version, prune_xid, flags
FROM page_header(get_raw_page('benim_tablom', 0));

-- Line pointer'lar ve durumları (0=UNUSED, 1=NORMAL, 2=REDIRECT, 3=DEAD)
SELECT lp, lp_off, lp_flags, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('benim_tablom', 0));

-- LP_DEAD birikmiş mi? Sayfa temizlik istiyor mu?
SELECT lp_flags, count(*)
FROM heap_page_items(get_raw_page('benim_tablom', 0))
GROUP BY lp_flags;

-- B-tree sayfasının special space'inden okunan bilgiler
SELECT * FROM bt_page_stats('benim_indexim', 1);
```

Sayfa seviyesinde boş alan ve şişkinlik:

```sql
-- FSM'in gördüğü boş alan (32 baytlık kategorilere yuvarlı)
SELECT * FROM pg_freespace('benim_tablom') LIMIT 20;

-- VM bitleri
SELECT * FROM pg_visibility('benim_tablom') LIMIT 20;

-- Tablo kaç blok?
SELECT relpages, reltuples FROM pg_class WHERE relname = 'benim_tablom';
SELECT pg_relation_size('benim_tablom') / 8192 AS gercek_blok_sayisi;
```

Checksum durumu:

```sql
SHOW data_checksums;

SELECT datname, checksum_failures, checksum_last_failure
FROM pg_stat_database WHERE datname = current_database();
```

## Tek sayfalık özet

```
                        BİR SAYFANIN HAYATI

  smgrextend / ReadBuffer(P_NEW)
        │
        ▼
  ┌───────────────────────────────────────────────────────────┐
  │ PageInit(page, BLCKSZ, specialSize)                       │
  │   pd_lower = 24                                           │
  │   pd_upper = pd_special = 8192 - MAXALIGN(specialSize)    │
  │   tüm sayfa sıfır, checksum YOK                           │
  └───────────────────────────────────────────────────────────┘
        │
        ▼  PageAddItemExtended  (her INSERT / index girişi)
  ┌───────────────────────────────────────────────────────────┐
  │  0    24                                            8192  │
  │  ┌────┬──────────┬─────────────────┬──────────┬────────┐  │
  │  │ hdr│ linp[] → │   BOŞ ALAN      │ ← tuple[]│special │  │
  │  └────┴──────────┴─────────────────┴──────────┴────────┘  │
  │            ▲ pd_lower        pd_upper ▲  pd_special ▲     │
  │        +4 bayt/slot             -MAXALIGN(size)/tuple     │
  │                                                           │
  │  lower > upper  →  InvalidOffsetNumber (sayfa dolu)       │
  └───────────────────────────────────────────────────────────┘
        │
        ├── DELETE/UPDATE ──▶ lp_flags = LP_DEAD / LP_REDIRECT
        │                     gövde yerinde kalır → fragmentasyon
        │
        ▼  PageRepairFragmentation  (VACUUM / HOT prune, cleanup lock ile)
  ┌───────────────────────────────────────────────────────────┐
  │  compactify_tuples: gövdeleri sona sıkıştır, pd_upper ↑   │
  │  PageTruncateLinePointerArray: sondaki LP_UNUSED'ı kes    │
  │  PD_HAS_FREE_LINES hint'ini yeniden hesapla               │
  └───────────────────────────────────────────────────────────┘
        │
        ▼  değişiklik WAL'landı → PageSetLSN(page, lsn)
        │
        ▼  buffer manager sayfayı yazmaya karar verdi
  ┌───────────────────────────────────────────────────────────┐
  │  1. XLogFlush(pd_lsn)        ← "WAL önce yazılır" kuralı  │
  │  2. PageSetChecksum(page, blkno)                          │
  │  3. smgrwrite → md.c:                                     │
  │       segno   = blocknum / RELSEG_SIZE                    │
  │       seekpos = BLCKSZ * (blocknum % RELSEG_SIZE)         │
  └───────────────────────────────────────────────────────────┘
        │
        ▼  sonraki okuma
  ┌───────────────────────────────────────────────────────────┐
  │  PageIsVerified: checksum? başlık tutarlı? tümü sıfır?    │
  │    üçü de başarısız → sayfa reddedilir                    │
  └───────────────────────────────────────────────────────────┘


  SAYFANIN ÜSTÜNDEKİ HARİTALAR

    main fork  ─── blok 0  1  2  3  4 ... N        (8 kB sayfalar)
                        │  │  │  │  │
    fsm fork   ─────────┴──┴──┴──┴──┴──▶ blok başına 1 bayt kategori
                                         (32 bayt çözünürlük, 3 katlı ağaç)
    vm fork    ─────────┴──┴──┴──┴──┴──▶ blok başına 2 bit
                                         (ALL_VISIBLE, ALL_FROZEN)
```

## İlgili notlar

- [BUFFER-VE-AIO.md](BUFFER-VE-AIO.md) — sayfa diskten belleğe nasıl geliyor
- [WAL-VE-KURTARMA.md](WAL-VE-KURTARMA.md) — `pd_lsn`'in diğer ucu, full-page write
- [UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md) — HOT, `LP_REDIRECT`, sayfa içi budama
- [VACUUM-NASIL-CALISIR.md](VACUUM-NASIL-CALISIR.md) — `LP_DEAD` temizliği ve defragmentasyon
- [TOAST-NASIL-CALISIR.md](TOAST-NASIL-CALISIR.md) — 8160 bayta sığmayan satırlar
- [BTREE-INDEX-NASIL-CALISIR.md](BTREE-INDEX-NASIL-CALISIR.md) — special space'in en zengin kullanıcısı
