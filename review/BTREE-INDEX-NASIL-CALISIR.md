# PostgreSQL B-tree (nbtree) index'i nasıl çalışır?

Örnek — dosya boyunca bunu takip ediyoruz:

```sql
CREATE INDEX users_id_idx ON users (id);
SELECT * FROM users WHERE id = 42;
INSERT INTO users VALUES (43, 'ali');
```

Kaynak ağacı: PostgreSQL **20devel** ([meson.build](../meson.build) `version: '20devel'`).
Satır numaraları sürüme bağlıdır; buradaki her numara bu ağaçtan doğrulanmıştır.

Zorunlu ön okuma: [../src/backend/access/nbtree/README](../src/backend/access/nbtree/README)
(1082 satır, tasarım belgesi). Bu not onun Türkçe okuma rehberidir.

---

## 30 saniyelik özet

nbtree, **Lehman & Yao**'nun yüksek eşzamanlılıklı B-tree algoritmasının bir
uyarlamasıdır. Klasik B-tree'den iki farkı vardır: her sayfanın bir **right link**'i
ve bir **high key**'i vardır.

```
                          blok 0
                     ┌──────────────┐
                     │  META SAYFA  │  btm_root, btm_fastroot, btm_level
                     └──────┬───────┘
                            │ btm_fastroot
                            ▼
                     ┌─────────────────────────┐
        level 1      │  ROOT / INTERNAL        │  hepsi pivot tuple
                     │ [-inf|→] [50|→] [90|→]  │  (yalnızca ayırıcı anahtar + downlink)
                     └───┬────────┬────────┬───┘
             ┌───────────┘        │        └───────────┐
             ▼                    ▼                    ▼
   ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
   │ LEAF   hikey=50  │ │ LEAF   hikey=90  │ │ LEAF  (rightmost)│
   │ 10,20,30,40      │◄┼─prev  10..49     │◄┼─prev  90..       │  level 0
   │            next──┼─►│            next──┼─►│      next=P_NONE│
   └──────────────────┘ └──────────────────┘ └──────────────────┘
        ▲ btpo_prev / btpo_next: sol ve sağ kardeş (çift yönlü zincir)
```

Zihinsel model — beş madde:

- **Her sayfada iki komşu linki var.** `btpo_next` (right link) ve `btpo_prev`
  (left link). L&Y yalnızca right link ister; PostgreSQL geri yönlü tarama için
  left link'i de ekler. Bu sayede **okuyucu ağacı kilitsiz gezebilir**: aradığı
  anahtar bulunduğu sayfanın high key'inden büyükse, sayfa eşzamanlı bölünmüş
  demektir; sağa yürüyerek düzeltir.
- **High key = sayfadaki anahtarların üst sınırı.** Yaprak olmayan sayfalarda
  offset 1 (`P_HIKEY`) her zaman high key'dir, veri offset 2'den (`P_FIRSTKEY`)
  başlar. Rightmost sayfada high key yoktur, veri offset 1'den başlar.
- **Her anahtar benzersizdir.** Heap TID (`t_tid`) gizli bir tiebreaker kolondur
  (BTREE_VERSION 4). Böylece L&Y'nin "Ki < v <= Ki+1" değişmezi, tekrarlı
  değerlerde bile bozulmaz.
- **Pivot tuple ≠ non-pivot tuple.** İç sayfalardaki tüm tuple'lar ve yaprak
  sayfaların high key'i pivot'tur: heap satırına işaret etmezler, sadece anahtar
  uzayını bölerler. **Suffix truncation** ile gereksiz kolonları atılır → fan-out artar.
- **Sayfa bölmesi son çaredir.** Bölmeden önce sırayla üç şey denenir:
  *simple deletion* (LP_DEAD girdileri sil) → *bottom-up deletion* (versiyon
  çöpünü tara) → *deduplication* (aynı anahtarları tek posting list'e topla).

Ekleme boru hattı:

```
[1] btinsert              index tuple'ı üret (index_form_tuple), t_tid = heap TID
[2] _bt_doinsert          insertion scankey kur (_bt_mkscankey)
[3] _bt_search_insert     fastpath cache dene, yoksa _bt_search ile kökten in
[4] _bt_check_unique      UNIQUE ise: eşit anahtarları gez, canlı çakışma var mı?
[5] _bt_findinsertloc     doğru yaprak sayfayı bul; yer yoksa temizle/dedup et
[6] _bt_insertonpg        ├─ sığıyorsa: PageAddItem + WAL, bitti
                          └─ sığmıyorsa: _bt_split → _bt_insert_parent (özyineleme)
                                                   → kök ise _bt_newlevel
```

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/access/nbtree/README](../src/backend/access/nbtree/README) | **Tasarım belgesi.** L&Y farkları, sayfa silme, dedup, WAL. Önce bu okunur |
| [../src/include/access/nbtree.h](../src/include/access/nbtree.h) | `BTPageOpaqueData`, `BTMetaPageData`, tuple biçimleri, tüm makrolar |
| [../src/backend/access/nbtree/nbtree.c](../src/backend/access/nbtree/nbtree.c) | AM giriş noktaları: `btinsert`, `btgettuple`, `btgetbitmap`, `btbulkdelete` |
| [../src/backend/access/nbtree/nbtsearch.c](../src/backend/access/nbtree/nbtsearch.c) | `_bt_search`, `_bt_moveright`, `_bt_binsrch`, `_bt_compare`, `_bt_first` |
| [../src/backend/access/nbtree/nbtinsert.c](../src/backend/access/nbtree/nbtinsert.c) | `_bt_doinsert`, `_bt_check_unique`, `_bt_insertonpg`, `_bt_split` |
| [../src/backend/access/nbtree/nbtsplitloc.c](../src/backend/access/nbtree/nbtsplitloc.c) | `_bt_findsplitloc` — bölme noktası seçimi, üç strateji |
| [../src/backend/access/nbtree/nbtdedup.c](../src/backend/access/nbtree/nbtdedup.c) | Deduplication, posting list, bottom-up deletion |
| [../src/backend/access/nbtree/nbtpage.c](../src/backend/access/nbtree/nbtpage.c) | Meta sayfa, buffer kilitleri, sayfa silme (`_bt_pagedel`), FSM |
| [../src/backend/access/nbtree/nbtreadpage.c](../src/backend/access/nbtree/nbtreadpage.c) | `_bt_readpage` — bir yaprak sayfayı tarayıp eşleşenleri belleğe alır |
| [../src/backend/access/nbtree/nbtutils.c](../src/backend/access/nbtree/nbtutils.c) | `_bt_mkscankey`, `_bt_truncate` (suffix truncation), `_bt_killitems` |
| [../src/backend/access/nbtree/nbtxlog.c](../src/backend/access/nbtree/nbtxlog.c) | WAL redo |
| [../src/include/access/nbtxlog.h](../src/include/access/nbtxlog.h) | WAL kayıt tipleri (`XLOG_BTREE_*`) |
| [../src/backend/executor/nodeIndexonlyscan.c](../src/backend/executor/nodeIndexonlyscan.c) | `IndexOnlyNext` — index-only scan + visibility map |
| [../contrib/pageinspect/btreefuncs.c](../contrib/pageinspect/btreefuncs.c) | `bt_metap`, `bt_page_stats`, `bt_page_items` |
| [../contrib/amcheck/verify_nbtree.c](../contrib/amcheck/verify_nbtree.c) | `bt_index_check`, `bt_index_parent_check` |

---

# 1. Sayfa düzeni

## 1.1 Sayfa sonundaki opaque alan

Her nbtree sayfasının special area'sında bu yapı durur:

[../src/backend/access/nbtree/../../../include/access/nbtree.h:63-70](../src/include/access/nbtree.h#L63-L70)

```c
typedef struct BTPageOpaqueData
{
	BlockNumber btpo_prev;		/* left sibling, or P_NONE if leftmost */
	BlockNumber btpo_next;		/* right sibling, or P_NONE if rightmost */
	uint32		btpo_level;		/* tree level --- zero for leaf pages */
	uint16		btpo_flags;		/* flag bits, see below */
	BTCycleId	btpo_cycleid;	/* vacuum cycle ID of latest split */
} BTPageOpaqueData;
```

Dört alanın her biri ayrı bir mekanizmayı besler:

| Alan | Kim kullanır |
|---|---|
| `btpo_prev` | Geri yönlü tarama (`_bt_lock_and_validate_left`), sayfa silme |
| `btpo_next` | **Lehman-Yao right link** — eşzamanlı bölmeden kurtulma, ileri tarama |
| `btpo_level` | Yaprak = 0. Yukarı doğru artar. Kök bölünmesinde seviye eşleşmesi için |
| `btpo_cycleid` | VACUUM'un "bu sayfa ben başladıktan sonra mı bölündü?" testi |

Bayraklar:

[../src/include/access/nbtree.h:77-85](../src/include/access/nbtree.h#L77-L85)

```c
#define BTP_LEAF		(1 << 0)	/* leaf page, i.e. not internal page */
#define BTP_ROOT		(1 << 1)	/* root page (has no parent) */
#define BTP_DELETED		(1 << 2)	/* page has been deleted from tree */
#define BTP_META		(1 << 3)	/* meta-page */
#define BTP_HALF_DEAD	(1 << 4)	/* empty, but still in tree */
#define BTP_SPLIT_END	(1 << 5)	/* rightmost page of split group */
#define BTP_HAS_GARBAGE (1 << 6)	/* page has LP_DEAD tuples (deprecated) */
#define BTP_INCOMPLETE_SPLIT (1 << 7)	/* right sibling's downlink is missing */
#define BTP_HAS_FULLXID	(1 << 8)	/* contains BTDeletedPageData */
```

`P_IGNORE()` makrosu `BTP_DELETED | BTP_HALF_DEAD` demektir — arayıcı böyle bir
sayfaya düşerse sağa yürür:

[../src/include/access/nbtree.h:226](../src/include/access/nbtree.h#L226)

```c
#define P_IGNORE(opaque)		(((opaque)->btpo_flags & (BTP_DELETED|BTP_HALF_DEAD)) != 0)
```

## 1.2 Meta sayfa (blok 0)

[../src/include/access/nbtree.h:104-120](../src/include/access/nbtree.h#L104-L120)

```c
typedef struct BTMetaPageData
{
	uint32		btm_magic;		/* should contain BTREE_MAGIC */
	uint32		btm_version;	/* nbtree version (always <= BTREE_VERSION) */
	BlockNumber btm_root;		/* current root location */
	uint32		btm_level;		/* tree level of the root page */
	BlockNumber btm_fastroot;	/* current "fast" root location */
	uint32		btm_fastlevel;	/* tree level of the "fast" root page */
	...
	bool		btm_allequalimage;	/* are all columns "equalimage"? */
} BTMetaPageData;
```

[../src/include/access/nbtree.h:149-151](../src/include/access/nbtree.h#L149-L151)

```c
#define BTREE_METAPAGE	0		/* first page is meta */
#define BTREE_MAGIC		0x053162	/* magic number in metapage */
#define BTREE_VERSION	4		/* current version number */
```

**Neden iki kök var?** Rightmost sayfa asla silinmez, dolayısıyla ağacın boyu
asla kısalmaz. Kitlesel silmelerden sonra kök ile veri arasında tek çocuklu
"zayıf" seviyeler kalabilir. `btm_fastroot` en alttaki tek sayfalık seviyeyi
gösterir; normal aramalar **oradan** başlar. `btm_root` ise gerçek köktür.

`btm_allequalimage`: tüm kolonlar "equalimage" opclass'ına sahipse
deduplication güvenlidir (iki eşit değerin ikili gösterimi de aynıdır).

Meta sayfa her aramada okunmasın diye relcache'te (`rd_amcache`) tutulur:

[../src/backend/access/nbtree/nbtpage.c:361-365](../src/backend/access/nbtree/nbtpage.c#L361-L365)

```c
	if (rel->rd_amcache != NULL)
	{
		metad = (BTMetaPageData *) rel->rd_amcache;
		/* We shouldn't have cached it if any of these fail */
		Assert(metad->btm_magic == BTREE_MAGIC);
```

Cache bayat olabilir; `_bt_getroot` bunu kabullenir ve sayfayı doğrular
(silinmiş değil, kendi seviyesinde yalnız):

[../src/backend/access/nbtree/nbtpage.c:392-396](../src/backend/access/nbtree/nbtpage.c#L392-L396)

```c
		if (!P_IGNORE(rootopaque) &&
			rootopaque->btpo_level == rootlevel &&
			P_LEFTMOST(rootopaque) &&
			P_RIGHTMOST(rootopaque))
		{
```

## 1.3 Sayfa içi yerleşim

[../src/include/access/nbtree.h:368-370](../src/include/access/nbtree.h#L368-L370)

```c
#define P_HIKEY				((OffsetNumber) 1)
#define P_FIRSTKEY			((OffsetNumber) 2)
#define P_FIRSTDATAKEY(opaque)	(P_RIGHTMOST(opaque) ? P_HIKEY : P_FIRSTKEY)
```

```
  NON-RIGHTMOST YAPRAK              RIGHTMOST YAPRAK
  ┌────────────────────┐            ┌────────────────────┐
  │ off 1: HIGH KEY    │  pivot     │ off 1: veri        │
  │ off 2: veri        │            │ off 2: veri        │
  │ off 3: veri        │            │ ...                │
  │ ...                │            │                    │
  ├────────────────────┤            ├────────────────────┤
  │ BTPageOpaqueData   │            │ BTPageOpaqueData   │
  └────────────────────┘            └────────────────────┘

  İÇ SAYFA
  ┌───────────────────────────────────────────────┐
  │ off 1: HIGH KEY                               │
  │ off 2: "minus infinity" (0 attribute) + downlink   ← her zaman ilk veri
  │ off 3: ayırıcı anahtar + downlink             │
  │ ...                                           │
  └───────────────────────────────────────────────┘
```

İç sayfanın ilk veri öğesi her zaman "eksi sonsuz" kabul edilir — `_bt_compare`
onu karşılaştırmadan geçer:

[../src/backend/access/nbtree/nbtsearch.c:709-714](../src/backend/access/nbtree/nbtsearch.c#L709-L714)

```c
	/*
	 * Force result ">" if target item is first data item on an internal page
	 * --- see NOTE above.
	 */
	if (!P_ISLEAF(opaque) && offnum == P_FIRSTDATAKEY(opaque))
		return 1;
```

## 1.4 1/3 sayfa kuralı

Algoritma her sayfaya en az üç öğe (high key + iki veri) sığmasını varsayar:

[../src/include/access/nbtree.h:165-169](../src/include/access/nbtree.h#L165-L169)

```c
#define BTMaxItemSize \
	(MAXALIGN_DOWN((BLCKSZ - \
					MAXALIGN(SizeOfPageHeaderData + 3*sizeof(ItemIdData)) - \
					MAXALIGN(sizeof(BTPageOpaqueData))) / 3) - \
					MAXALIGN(sizeof(ItemPointerData)))
```

8 KB blokta bu ~2704 bayttır. Aşılırsa `_bt_check_third_page`
([../src/backend/access/nbtree/nbtutils.c:1118](../src/backend/access/nbtree/nbtutils.c#L1118))
"index row size ... exceeds maximum" hatası verir.

## 1.5 Tuple biçimleri

[../src/include/access/nbtree.h:383-386](../src/include/access/nbtree.h#L383-L386)

```
 * Non-pivot tuple format (plain/non-posting variant):
 *
 *  t_tid | t_info | key values | INCLUDE columns, if any
```

[../src/include/access/nbtree.h:406-408](../src/include/access/nbtree.h#L406-L408)

```
 * Pivot tuple format:
 *
 *  t_tid | t_info | key values | [heap TID]
```

[../src/include/access/nbtree.h:444-446](../src/include/access/nbtree.h#L444-L446)

```
 * Posting tuple format (alternative non-pivot tuple representation):
 *
 *  t_tid | t_info | key values | posting list (TID array)
```

Üçünü ayırmak için `t_info`'daki `INDEX_ALT_TID_MASK` biti ve `t_tid`'in offset
alanındaki durum bitleri kullanılır:

[../src/include/access/nbtree.h:466-471](../src/include/access/nbtree.h#L466-L471)

```c
/* Item pointer offset bit masks */
#define BT_OFFSET_MASK				0x0FFF
#define BT_STATUS_OFFSET_MASK		0xF000
/* BT_STATUS_OFFSET_MASK status bits */
#define BT_PIVOT_HEAP_TID_ATTR		0x1000
#define BT_IS_POSTING				0x2000
```

---

# 2. Lehman-Yao: okuyucu ağaç bölünürken neden kaybolmaz

Klasik B-tree'de bir okuyucu, parent'tan downlink'i okuyup çocuğa inerken araya
bir bölme girerse aradığı satır artık o çocukta olmayabilir. L&Y'nin çözümü:
**bölünen sayfanın verisi asla sola gitmez**, hep sağa taşınır ve sol yarı yeni
sağ yarıya bir right link bırakır.

README bunu tek paragrafta anlatır
([../src/backend/access/nbtree/README:24-29](../src/backend/access/nbtree/README#L24-L29)):

```
When a search follows a downlink to a child page, it compares the page's
high key with the search key.  If the search key is greater than the high
key, the page must've been split concurrently, and you must follow the
right-link to find the new page containing the key range you're looking
for.  This might need to be repeated, if the page has been split more than
once.
```

```
  ZAMAN T0                          ZAMAN T1 (bölmeden sonra)
  parent ─┐                         parent ─┐            (downlink henüz yok!)
          ▼                                 ▼
     ┌─────────────┐                   ┌──────────┐   ┌──────────┐
     │ 10..90      │                   │ 10..49   │──►│ 50..90   │
     │ hikey=100   │                   │ hikey=50 │   │ hikey=100│
     └─────────────┘                   └──────────┘   └──────────┘
                                       INCOMPLETE_SPLIT

  T0'da downlink'i okuyup uykuya dalan okuyucu, T1'de sol sayfaya iner.
  Aradığı 70 > hikey 50 olduğu için right link'i izler ve doğru sayfayı bulur.
```

Bu, `_bt_moveright` içinde tek bir karşılaştırmadır:

[../src/backend/access/nbtree/nbtsearch.c:275](../src/backend/access/nbtree/nbtsearch.c#L275)

```c
	cmpval = key->nextkey ? 0 : 1;
```

[../src/backend/access/nbtree/nbtsearch.c:309-314](../src/backend/access/nbtree/nbtsearch.c#L309-L314)

```c
		if (P_IGNORE(opaque) || _bt_compare(rel, key, page, P_HIKEY) >= cmpval)
		{
			/* step right one page */
			buf = _bt_relandgetbuf(rel, buf, opaque->btpo_next, access);
			continue;
		}
```

Aynı mekanizma iki başka durumu da bedavaya çözer:

1. **Kök bölünmesi.** Kök de sıradan bir sayfa gibi bölünür, sonra yeni bir kök
   kurulur. Meta sayfayı güncellemeden önce eski kökü okumuş bir arayıcı,
   right link'i izleyerek yine doğru veriye ulaşır
   ([README:112-123](../src/backend/access/nbtree/README#L112-L123)).
2. **Sayfa silme.** Silinen sayfa hemen geri dönüştürülmez; bir süre "mezar taşı"
   olarak kalır. Ona düşen okuyucu `P_IGNORE` görüp sağa yürür.

PostgreSQL'in L&Y'den üç farkı vardır
([README:57-164](../src/backend/access/nbtree/README#L57-L164)):

| L&Y | PostgreSQL |
|---|---|
| Okuma kilidi yok (in-memory kopyalar paylaşılmaz) | Buffer'lar paylaşıldığı için **page-level read lock** var |
| Sadece right link | Geri yönlü tarama için **left link** de var |
| Ascent sırasında ayırıcı anahtarla eşleşme | **Blok numarasıyla** eşleşme (`_bt_getstackbuf`) — daha az kilit |
| Sabit boyutlu anahtar | Değişken boyut; bölmede öğe değil **bayt** dengelenir |

---

# 3. Arama

## 3.1 `_bt_search` — kökten yaprağa iniş

[../src/backend/access/nbtree/nbtsearch.c:102](../src/backend/access/nbtree/nbtsearch.c#L102)

```c
BTStack
_bt_search(Relation rel, Relation heaprel, BTScanInsert key, Buffer *bufP,
		   int access, bool returnstack)
```

Döngü seviye başına bir kez döner:

[../src/backend/access/nbtree/nbtsearch.c:141-156](../src/backend/access/nbtree/nbtsearch.c#L141-L156)

```c
		*bufP = _bt_moveright(rel, heaprel, key, *bufP, (access == BT_WRITE),
							  stack_in, page_access);

		/* if this is a leaf page, we're done */
		page = BufferGetPage(*bufP);
		opaque = BTPageGetOpaque(page);
		if (P_ISLEAF(opaque))
			break;

		/*
		 * Find the appropriate pivot tuple on this page.  Its downlink points
		 * to the child page that we're about to descend to.
		 */
		offnum = _bt_binsrch(rel, key, *bufP);
		itemid = PageGetItemId(page, offnum);
		itup = (IndexTuple) PageGetItem(page, itemid);
```

Üç ayrıntı önemli:

- **Kilit hiçbir zaman iki seviyede birden tutulmaz.** İniş
  `_bt_relandgetbuf` ile yapılır: önce çocuk kilitlenmez, önce ebeveyn bırakılır.
  Yorumun dediği gibi *"drop the read lock on the page, then acquire one on its child"*
  ([nbtsearch.c:184](../src/backend/access/nbtree/nbtsearch.c#L184)).
- **Stack.** `returnstack` true ise her seviyede seçilen pivot'un
  (blok, offset) çifti yığına konur. Bölme olursa ebeveyni yeniden bulmak için
  kullanılır.
- **Level 1 optimizasyonu.** Yazma amaçlı inişte, bir alt seviye yaprak olduğu
  için kilit doğrudan BT_WRITE alınır:

[../src/backend/access/nbtree/nbtsearch.c:177-179](../src/backend/access/nbtree/nbtsearch.c#L177-L179)

```c
		if (opaque->btpo_level == 1 && access == BT_WRITE)
			page_access = BT_WRITE;
```

## 3.2 `_bt_binsrch` — sayfa içi ikili arama

[../src/backend/access/nbtree/nbtsearch.c:346](../src/backend/access/nbtree/nbtsearch.c#L346)

Değişmez, yorumda açıkça yazılı
([nbtsearch.c:378-386](../src/backend/access/nbtree/nbtsearch.c#L378-L386)):

```
	 * For nextkey=false (cmpval=1), the loop invariant is: all slots before
	 * 'low' are < scan key, all slots at or after 'high' are >= scan key.
```

Döngü:

[../src/backend/access/nbtree/nbtsearch.c:394-406](../src/backend/access/nbtree/nbtsearch.c#L394-L406)

```c
	while (high > low)
	{
		OffsetNumber mid = low + ((high - low) / 2);

		/* We have low <= mid < high, so mid points at a real slot */

		result = _bt_compare(rel, key, page, mid);

		if (result >= cmpval)
			low = mid + 1;
		else
			high = mid;
	}
```

Yaprakta `low` döner (ilk >= anahtar). İç sayfada `OffsetNumberPrev(low)` döner
(son < anahtar) — çünkü inilecek çocuk, anahtarı **kapsayan** aralığın sahibidir.

Ekleme yolunda bunun cache'li bir varyantı kullanılır:
`_bt_binsrch_insert`
([../src/backend/access/nbtree/nbtinsert.c:446](../src/backend/access/nbtree/nbtinsert.c#L446)
ve [:1002](../src/backend/access/nbtree/nbtinsert.c#L1002)),
`insertstate->low` / `stricthigh` sınırlarını saklayarak
`_bt_check_unique` ile `_bt_findinsertloc` arasında karşılaştırmaları paylaşır.

## 3.3 `_bt_compare` — üç yönlü karşılaştırma

[../src/backend/access/nbtree/nbtsearch.c:691](../src/backend/access/nbtree/nbtsearch.c#L691)

NULL'lar sıralanabilir değer sayılır; `SK_BT_NULLS_FIRST` yönü belirler:

[../src/backend/access/nbtree/nbtsearch.c:742-750](../src/backend/access/nbtree/nbtsearch.c#L742-L750)

```c
		if (scankey->sk_flags & SK_ISNULL)	/* key is NULL */
		{
			if (isNull)
				result = 0;		/* NULL "=" NULL */
			else if (scankey->sk_flags & SK_BT_NULLS_FIRST)
				result = -1;	/* NULL "<" NOT_NULL */
			else
				result = 1;		/* NULL ">" NOT_NULL */
		}
```

Karşılaştırma fonksiyonu `sk_func`'tır (opclass'ın `BTORDER_PROC`'u), sonucun
işareti çevrilir çünkü çağıran "scankey'i tuple'a kıyasla" diye düşünür:

[../src/backend/access/nbtree/nbtsearch.c:773-774](../src/backend/access/nbtree/nbtsearch.c#L773-L774)

```c
			if (!(scankey->sk_flags & SK_BT_DESC))
				INVERT_COMPARE_RESULT(result);
```

## 3.4 Tarama başlangıcı: `_bt_first`

[../src/backend/access/nbtree/nbtsearch.c:885](../src/backend/access/nbtree/nbtsearch.c#L885)

`btrescan`'den gelen **search scankey** (`int4lt` gibi boolean dönen operatörler)
burada geçici bir **insertion scankey**'e çevrilir ve `_bt_search` çağrılır:

[../src/backend/access/nbtree/nbtsearch.c:1515](../src/backend/access/nbtree/nbtsearch.c#L1515)

```c
	_bt_search(rel, NULL, &inskey, &so->currPos.buf, BT_READ, false);
```

İki scankey türünün farkı README'de anlatılır
([README:797-819](../src/backend/access/nbtree/README#L797-L819)). Kısaca:
insertion scankey **yeri bulmak** için, search scankey **satırları elemek** için.

---

# 4. Ekleme

## 4.1 `btinsert` — AM giriş noktası

[../src/backend/access/nbtree/nbtree.c:206](../src/backend/access/nbtree/nbtree.c#L206)

```c
bool
btinsert(Relation rel, Datum *values, bool *isnull,
		 ItemPointer ht_ctid, Relation heapRel,
		 IndexUniqueCheck checkUnique,
		 bool indexUnchanged,
		 IndexInfo *indexInfo)
{
	bool		result;
	IndexTuple	itup;

	/* generate an index tuple */
	itup = index_form_tuple(RelationGetDescr(rel), values, isnull);
	itup->t_tid = *ht_ctid;

	result = _bt_doinsert(rel, itup, checkUnique, indexUnchanged, heapRel);
```

`indexUnchanged`, executor'dan gelen bir **ipucudur**: "bu UPDATE indexlenen
değeri mantıksal olarak değiştirmedi, ama MVCC için yeni bir girdi lazım".
Bottom-up deletion'ı tetikleyen sinyal budur (bölüm 9.3).

## 4.2 `_bt_doinsert`

[../src/backend/access/nbtree/nbtinsert.c:105](../src/backend/access/nbtree/nbtinsert.c#L105)

Unique kontrol yapılacaksa `scantid` **geçici olarak kaldırılır**:

[../src/backend/access/nbtree/nbtinsert.c:120-123](../src/backend/access/nbtree/nbtinsert.c#L120-L123)

```c
	if (checkingunique)
	{
		if (!itup_key->anynullkeys)
		{
			/* No (heapkeyspace) scantid until uniqueness established */
			itup_key->scantid = NULL;
```

Neden? Heap TID tiebreaker'ı devrede kalırsa arama, aynı anahtarın *ilk*
sayfasına değil, kendi TID'sinin ait olduğu yere iner ve eşit anahtarları göremez.
Benzersizlik kanıtlandıktan sonra geri konur:

[../src/backend/access/nbtree/nbtinsert.c:239-240](../src/backend/access/nbtree/nbtinsert.c#L239-L240)

```c
		/* Uniqueness is established -- restore heap tid as scantid */
		if (itup_key->heapkeyspace)
			itup_key->scantid = &itup->t_tid;
```

## 4.3 Fastpath: artan anahtarlar için kısayol

Sequence / timestamp kolonlarında ekleme hep rightmost yaprağa gider. O sayfa
backend-yerel olarak cache'lenir:

[../src/backend/access/nbtree/nbtinsert.c:326-329](../src/backend/access/nbtree/nbtinsert.c#L326-L329)

```c
	if (RelationGetTargetBlock(rel) != InvalidBlockNumber)
	{
		/* Simulate a _bt_getbuf() call with conditional locking */
		insertstate->buf = ReadBuffer(rel, RelationGetTargetBlock(rel));
		if (_bt_conditionallockbuf(rel, insertstate->buf))
```

Kabul koşulu dört testtir — hâlâ rightmost, hâlâ yaprak, yer var, anahtar
sayfadaki ilk tuple'dan büyük:

[../src/backend/access/nbtree/nbtinsert.c:347-352](../src/backend/access/nbtree/nbtinsert.c#L347-L352)

```c
			if (P_RIGHTMOST(opaque) &&
				P_ISLEAF(opaque) &&
				!P_IGNORE(opaque) &&
				PageGetFreeSpace(page) > insertstate->itemsz &&
				PageGetMaxOffsetNumber(page) >= P_HIKEY &&
				_bt_compare(rel, insertstate->itup_key, page, P_HIKEY) > 0)
```

Cache `_bt_insertonpg` sonunda tazelenir — ve yalnızca ağaç yeterince yüksekse:

[../src/backend/access/nbtree/nbtinsert.c:1447-1449](../src/backend/access/nbtree/nbtinsert.c#L1447-L1449)

```c
		if (BlockNumberIsValid(blockcache) &&
			_bt_getrootheight(rel) >= BTREE_FASTPATH_MIN_LEVEL)
			RelationSetTargetBlock(rel, blockcache);
```

Neden kilitleme *conditional*? README'nin dediği gibi: çekişme varsa cache
zaten uzun süre geçerli kalmaz, beklemeye değmez
([README:491-509](../src/backend/access/nbtree/README#L491-L509)).

## 4.4 `_bt_findinsertloc`

[../src/backend/access/nbtree/nbtinsert.c:829](../src/backend/access/nbtree/nbtinsert.c#L829)

Unique index'te `_bt_search` scantid'siz yapıldığı için, doğru sayfa **sağdaki**
sayfa olabilir. Bu yüzden gerekirse sağa yürünür:

[../src/backend/access/nbtree/nbtinsert.c:899-904](../src/backend/access/nbtree/nbtinsert.c#L899-L904)

```c
				/* Test '<=', not '!=', since scantid is set now */
				if (P_RIGHTMOST(opaque) ||
					_bt_compare(rel, itup_key, page, P_HIKEY) <= 0)
					break;

				_bt_stepright(rel, heapRel, insertstate, stack);
```

`_bt_stepright` ([nbtinsert.c:1041](../src/backend/access/nbtree/nbtinsert.c#L1041))
sağdaki sayfayı **bırakmadan önce** kilitler — yoksa başka birinin
`_bt_check_unique` taraması bizim eklememizi kaçırabilir.

Yer yoksa, bölmeden önce kurtarma denenir:

[../src/backend/access/nbtree/nbtinsert.c:917-920](../src/backend/access/nbtree/nbtinsert.c#L917-L920)

```c
		if (PageGetFreeSpace(page) < insertstate->itemsz)
			_bt_delete_or_dedup_one_page(rel, heapRel, insertstate, false,
										 checkingunique, uniquedup,
										 indexUnchanged);
```

## 4.5 `_bt_insertonpg`

[../src/backend/access/nbtree/nbtinsert.c:1119](../src/backend/access/nbtree/nbtinsert.c#L1119)

Fonksiyonun kendi başlık yorumu işi beş maddede özetliyor
([nbtinsert.c:1088-1103](../src/backend/access/nbtree/nbtinsert.c#L1088-L1103)):

```
 *			+  if postingoff != 0, splits existing posting list tuple
 *			+  if necessary, splits the target page, using 'itup_key' for
 *			   suffix truncation on leaf pages
 *			+  inserts the new tuple (might be split from posting list).
 *			+  if the page was split, pops the parent stack, and finds the
 *			   right place to insert the new child pointer
 *			+  updates the metapage if a true root or fast root is split.
```

Dallanma noktası tek satır:

[../src/backend/access/nbtree/nbtinsert.c:1225-1233](../src/backend/access/nbtree/nbtinsert.c#L1225-L1233)

```c
	if (PageGetFreeSpace(page) < itemsz)
	{
		Buffer		rbuf;

		Assert(!split_only_page);

		/* split the buffer into left and right halves */
		rbuf = _bt_split(rel, heaprel, itup_key, buf, cbuf, newitemoff, itemsz,
						 itup, origitup, nposting, postingoff);
```

Sığıyorsa kritik bölüm çok kısadır:

[../src/backend/access/nbtree/nbtinsert.c:1296-1306](../src/backend/access/nbtree/nbtinsert.c#L1296-L1306)

```c
		/* Do the update.  No ereport(ERROR) until changes are logged */
		START_CRIT_SECTION();

		if (postingoff != 0)
			memcpy(oposting, nposting, MAXALIGN(IndexTupleSize(nposting)));

		if (PageAddItem(page, itup, itemsz, newitemoff, false, false) == InvalidOffsetNumber)
			elog(PANIC, "failed to add new item to block %u in index \"%s\"",
				 BufferGetBlockNumber(buf), RelationGetRelationName(rel));

		MarkBufferDirty(buf);
```

İç sayfaya ekleme yapılıyorsa, aynı atomik işlem çocuktaki
`BTP_INCOMPLETE_SPLIT` bayrağını da temizler:

[../src/backend/access/nbtree/nbtinsert.c:1322-1330](../src/backend/access/nbtree/nbtinsert.c#L1322-L1330)

```c
		if (!isleaf)
		{
			Page		cpage = BufferGetPage(cbuf);
			BTPageOpaque cpageop = BTPageGetOpaque(cpage);

			Assert(P_INCOMPLETE_SPLIT(cpageop));
			cpageop->btpo_flags &= ~BTP_INCOMPLETE_SPLIT;
			MarkBufferDirty(cbuf);
		}
```

---

# 5. Unique kısıt kontrolü neden ekleme yolunda?

`_bt_check_unique` ([../src/backend/access/nbtree/nbtinsert.c:411](../src/backend/access/nbtree/nbtinsert.c#L411))
ayrı bir "kontrol" adımı değil, **ekleme yolunun içindedir**. Sebebi
`_bt_doinsert`'in yorumunda yazılı
([nbtinsert.c:190-196](../src/backend/access/nbtree/nbtinsert.c#L190-L196)):

```
	 * NOTE: obviously, _bt_check_unique can only detect keys that are already
	 * in the index; so it cannot defend against concurrent insertions of the
	 * same key.  We protect against that by means of holding a write lock on
	 * the first page the value could be on, with omitted/-inf value for the
	 * implicit heap TID tiebreaker attribute.  Any other would-be inserter of
	 * the same key must acquire a write lock on the same page, so only one
	 * would-be inserter can be making the check at one time.
```

Yani **kilit protokolü kısıtın kendisidir**: aynı anahtarı ekleyecek herkes,
scantid'siz aramanın indiği *aynı ilk sayfayı* write-lock'lamak zorundadır.
O kilit, kontrol ile ekleme arasında sürekli tutulur.

Tarama, eşit anahtar kaldığı sürece ilerler; her heap TID için `SnapshotDirty`
ile heap'e bakılır:

[../src/backend/access/nbtree/nbtinsert.c:563-567](../src/backend/access/nbtree/nbtinsert.c#L563-L567)

```c
				else if (table_index_fetch_tuple_check(heapRel, &htid,
													   &SnapshotDirty,
													   &all_dead))
				{
					TransactionId xwait;
```

Üç sonuç olabilir:

| Durum | Davranış |
|---|---|
| Çakışan satır **committed canlı** | `ERRCODE_UNIQUE_VIOLATION` ([nbtinsert.c:669-676](../src/backend/access/nbtree/nbtinsert.c#L669-L676)) |
| Çakışan satır **devam eden bir xact'e ait** | `xwait` döner; çağıran kilidi bırakıp `XactLockTableWait` ile bekler, sonra baştan arar ([nbtinsert.c:592-601](../src/backend/access/nbtree/nbtinsert.c#L592-L601)) |
| Çakışan satır **herkese ölü** | LP_DEAD işaretlenir (aşağıda) |

`UNIQUE_CHECK_PARTIAL` (deferred kısıt / CREATE INDEX CONCURRENTLY) beklemez,
sadece `*is_unique = false` der ve kontrolü sonraya bırakır
([nbtinsert.c:577-583](../src/backend/access/nbtree/nbtinsert.c#L577-L583)).

**Yan fayda:** unique kontrol, ölü girdileri ücretsiz olarak işaretler. Zaten
heap'e bakıyoruz, exclusive kilit de elimizde:

[../src/backend/access/nbtree/nbtinsert.c:706-707](../src/backend/access/nbtree/nbtinsert.c#L706-L707)

```c
						ItemIdMarkDead(curitemid);
						opaque->btpo_flags |= BTP_HAS_GARBAGE;
```

Eşit anahtarlar sayfa sınırını aşabilir; o zaman high key üzerinden sağa geçilir:

[../src/backend/access/nbtree/nbtinsert.c:738-744](../src/backend/access/nbtree/nbtinsert.c#L738-L744)

```c
			/* If scankey == hikey we gotta check the next page too */
			if (P_RIGHTMOST(opaque))
				break;
			highkeycmp = _bt_compare(rel, itup_key, page, P_HIKEY);
			Assert(highkeycmp <= 0);
			if (highkeycmp != 0)
				break;
```

Pratikte bu neredeyse hiç gerekmez, çünkü `_bt_findsplitloc` tekrarlı değerleri
sayfa sınırına yaymamak için çok uğraşır (bölüm 6.1).

---

# 6. Sayfa bölme

## 6.1 `_bt_findsplitloc` — nereden bölelim?

[../src/backend/access/nbtree/nbtsplitloc.c:130](../src/backend/access/nbtree/nbtsplitloc.c#L130)

Ana hedef alan dengesi, ikincil hedef suffix truncation'ı etkili kılmak:

[../src/backend/access/nbtree/nbtsplitloc.c:88-99](../src/backend/access/nbtree/nbtsplitloc.c#L88-L99)

```
 * The main goal here is to equalize the free space that will be on each
 * split page, *after accounting for the inserted tuple*. ...
 * If the page is the rightmost page on its level, we instead try to arrange
 * to leave the left split page fillfactor% full.  In this way, when we are
 * inserting successively increasing keys (consider sequences, timestamps,
 * etc) we will end up with a tree whose pages are about fillfactor% full,
 * instead of the 50% full result that we'd get without this special case.
```

Üç strateji vardır:

[../src/backend/access/nbtree/nbtsplitloc.c:21-27](../src/backend/access/nbtree/nbtsplitloc.c#L21-L27)

```c
typedef enum
{
	/* strategy for searching through materialized list of split points */
	SPLIT_DEFAULT,				/* give some weight to truncation */
	SPLIT_MANY_DUPLICATES,		/* find minimally distinguishing point */
	SPLIT_SINGLE_VALUE,			/* leave left page almost full */
} FindSplitStrat;
```

Seçim `_bt_strategy`'de yapılır
([nbtsplitloc.c:935](../src/backend/access/nbtree/nbtsplitloc.c#L935),
atamalar [:946](../src/backend/access/nbtree/nbtsplitloc.c#L946),
[:995](../src/backend/access/nbtree/nbtsplitloc.c#L995),
[:1022](../src/backend/access/nbtree/nbtsplitloc.c#L1022)).

```
  SPLIT_DEFAULT           50/50'ye yakın, truncation'a küçük bir ağırlık
  SPLIT_MANY_DUPLICATES   dengeyi feda edip "en erken ayrışan" noktayı seç
  SPLIT_SINGLE_VALUE      sayfa tek değerden ibaret → sol %96 dolu bırak
                          (BTREE_SINGLEVAL_FILLFACTOR)
```

Fillfactor sabitleri:

[../src/include/access/nbtree.h:200-203](../src/include/access/nbtree.h#L200-L203)

```c
#define BTREE_MIN_FILLFACTOR		10
#define BTREE_DEFAULT_FILLFACTOR	90
#define BTREE_NONLEAF_FILLFACTOR	70
#define BTREE_SINGLEVAL_FILLFACTOR	96
```

## 6.2 `_bt_split` — bölmenin kendisi

[../src/backend/access/nbtree/nbtinsert.c:1489](../src/backend/access/nbtree/nbtinsert.c#L1489)

Sol yarı **geçici bir bellek arabelleğinde** kurulur, sonra orijinal sayfanın
üstüne kopyalanır. Sebep: sol sayfa asla yer değiştirmemeli, ama hata olursa
orijinal bozulmamış kalmalı:

[../src/backend/access/nbtree/nbtinsert.c:1523-1534](../src/backend/access/nbtree/nbtinsert.c#L1523-L1534)

```
	 * origpage is the original page to be split.  leftpage is a temporary
	 * buffer that receives the left-sibling data, which will be copied back
	 * into origpage on success.  rightpage is the new page that will receive
	 * the right-sibling data.
```

Sıra:

```
 1. firstrightoff = _bt_findsplitloc(...)                       nbtinsert.c:1567
 2. leftpage'i kur; BTP_INCOMPLETE_SPLIT bayrağını koy          nbtinsert.c:1582
 3. sol için yeni high key üret (yaprakta suffix truncation)    nbtinsert.c:1682
 4. rbuf = _bt_allocbuf()   → yeni sağ sayfa                    nbtinsert.c:1741
 5. linkleri kur: lopaque->btpo_next = right                    nbtinsert.c:1762
                  ropaque->btpo_prev = orig                     nbtinsert.c:1771
                  ropaque->btpo_next = eski sağ komşu           nbtinsert.c:1772
 6. veri öğelerini iki sayfaya dağıt
 7. ESKİ SAĞ KOMŞUYU kilitle, onun btpo_prev'ini güncelle       nbtinsert.c:1914
 8. START_CRIT_SECTION → leftpage'i origpage üstüne kopyala     nbtinsert.c:1952
 9. WAL: XLOG_BTREE_SPLIT_L / XLOG_BTREE_SPLIT_R
```

Adım 3, suffix truncation'ın yapıldığı yerdir:

[../src/backend/access/nbtree/nbtinsert.c:1682-1683](../src/backend/access/nbtree/nbtinsert.c#L1682-L1683)

```c
		lefthighkey = _bt_truncate(rel, lastleft, firstright, itup_key);
		itemsz = IndexTupleSize(lefthighkey);
```

İç sayfada truncation **yasaktır** — ayırıcı anahtarların her seviyede aynı
"dikiş"i oluşturması gerekir:

[../src/backend/access/nbtree/nbtinsert.c:1692-1701](../src/backend/access/nbtree/nbtinsert.c#L1692-L1701)

```
		 * Each distinct separator key value originates as a leaf level high
		 * key; all other separator keys/pivot tuples are copied from one
		 * level down. ... There must always be an unbroken
		 * "seam" of identical separator keys that guide index scans at every
		 * level, starting from the grandparent.  That's why suffix truncation
		 * is unsafe here.
```

Adım 7'de kilit sırası sabittir (sol → sağ), bu yüzden deadlock olmaz:

[../src/backend/access/nbtree/nbtinsert.c:1908-1917](../src/backend/access/nbtree/nbtinsert.c#L1908-L1917)

```c
	/*
	 * We have to grab the original right sibling (if any) and update its prev
	 * link.  We are guaranteed that this is deadlock-free, since we couple
	 * the locks in the standard order: left to right.
	 */
	if (!isrightmost)
	{
		sbuf = _bt_getbuf(rel, oopaque->btpo_next, BT_WRITE);
```

## 6.3 Suffix truncation

[../src/backend/access/nbtree/nbtutils.c:692](../src/backend/access/nbtree/nbtutils.c#L692)

Amaç: high key (ve dolayısıyla ebeveyne gidecek downlink) mümkün olduğunca kısa
olsun → fan-out artsın → ağaç alçalsın.

[../src/backend/access/nbtree/nbtutils.c:666-671](../src/backend/access/nbtree/nbtutils.c#L666-L671)

```
 * Returns truncated pivot index tuple allocated in caller's memory context,
 * with key attributes copied from caller's firstright argument.  If rel is
 * an INCLUDE index, non-key attributes will definitely be truncated away,
 * since they're not part of the key space.  More aggressive suffix
 * truncation can take place when it's clear that the returned tuple does not
 * need one or more suffix key attributes.
```

```
   lastleft  = (ali,   34, TID)      ← sol yarının son satırı
   firstright= (ayşe,  12, TID)      ← sağ yarının ilk satırı
                ▲
                └─ ilk ayrışan kolon burası → high key = (ayşe) yeter
                   ikinci kolon ve heap TID atılır
```

Hiç ayrışmıyorlarsa (tam tekrar), tiebreaker olarak heap TID eklenir. Bu yüzden
`BTMaxItemSize` hesabında `sizeof(ItemPointerData)` payı ayrılmıştır
([nbtree.h:169](../src/include/access/nbtree.h#L169)).

## 6.4 Kök bölünmesi ve incomplete split

Bölme bittiğinde ebeveyne downlink eklenir:

[../src/backend/access/nbtree/nbtinsert.c:2130](../src/backend/access/nbtree/nbtinsert.c#L2130)

```c
static void
_bt_insert_parent(Relation rel, Relation heaprel, Buffer buf, Buffer rbuf,
				  BTStack stack, bool isroot, bool isonly)
```

Kök bölündüyse yeni bir seviye kurulur:

[../src/backend/access/nbtree/nbtinsert.c:2153-2161](../src/backend/access/nbtree/nbtinsert.c#L2153-L2161)

```c
	if (isroot)
	{
		Buffer		rootbuf;

		Assert(stack == NULL);
		Assert(isonly);
		/* create a new root node one level up and update the metapage */
		rootbuf = _bt_newlevel(rel, heaprel, buf, rbuf);
```

`_bt_newlevel` ([nbtinsert.c:2492](../src/backend/access/nbtree/nbtinsert.c#L2492))
iki downlink'li yeni bir kök yazar. Sol downlink "eksi sonsuz"dur (0 attribute):

[../src/backend/access/nbtree/nbtinsert.c:2533-2537](../src/backend/access/nbtree/nbtinsert.c#L2533-L2537)

```c
	left_item_sz = sizeof(IndexTupleData);
	left_item = (IndexTuple) palloc(left_item_sz);
	left_item->t_info = left_item_sz;
	BTreeTupleSetDownLink(left_item, lbkno);
	BTreeTupleSetNAtts(left_item, 0, false);
```

Kilit sırası burada da kritiktir — okuyucular meta sayfayı kökten **önce**
bırakır, yazarlar kökü meta sayfadan **önce** kilitler:

[../src/backend/access/nbtree/nbtinsert.c:2477-2481](../src/backend/access/nbtree/nbtinsert.c#L2477-L2481)

```
 *		In order to do this, we add a new root page to the file, then lock
 *		the metadata page and update it.  This is guaranteed to be deadlock-
 *		free, because all readers release their locks on the metadata page
 *		before trying to lock the root, and all writers lock the root before
 *		trying to lock the metadata page.
```

**Bölme iki WAL kaydıdır**, dolayısıyla arada çökme olabilir. O zaman sağ
sayfanın downlink'i eksik kalır. Çözüm: sol sayfaya `BTP_INCOMPLETE_SPLIT`
konur ve eksik downlink **sonraki eklemede** tamamlanır:

[../src/backend/access/nbtree/README:665-672](../src/backend/access/nbtree/README#L665-L672)

```
To identify missing downlinks, when a page is split, the left page is
flagged to indicate that the split is not yet complete (INCOMPLETE_SPLIT).
When the downlink is inserted to the parent, the flag is cleared atomically
with the insertion.
```

Tamamlama `_bt_moveright` içinde, yazma modunda geçerken yapılır:

[../src/backend/access/nbtree/nbtsearch.c:287-291](../src/backend/access/nbtree/nbtsearch.c#L287-L291)

```c
			if (P_INCOMPLETE_SPLIT(opaque))
				_bt_finish_split(rel, heaprel, buf, stack);
			else
				_bt_relbuf(rel, buf);
```

WAL kayıt tipleri:

[../src/include/access/nbtxlog.h:27-43](../src/include/access/nbtxlog.h#L27-L43)

```c
#define XLOG_BTREE_INSERT_LEAF	0x00	/* add index tuple without split */
#define XLOG_BTREE_INSERT_UPPER 0x10	/* same, on a non-leaf page */
#define XLOG_BTREE_INSERT_META	0x20	/* same, plus update metapage */
#define XLOG_BTREE_SPLIT_L		0x30	/* add index tuple with split */
#define XLOG_BTREE_SPLIT_R		0x40	/* as above, new item on right */
#define XLOG_BTREE_INSERT_POST	0x50	/* add index tuple with posting split */
#define XLOG_BTREE_DEDUP		0x60	/* deduplicate tuples for a page */
#define XLOG_BTREE_DELETE		0x70	/* delete leaf index tuples for a page */
#define XLOG_BTREE_UNLINK_PAGE	0x80	/* delete a half-dead page */
#define XLOG_BTREE_UNLINK_PAGE_META 0x90	/* same, and update metapage */
#define XLOG_BTREE_NEWROOT		0xA0	/* new root page */
#define XLOG_BTREE_MARK_PAGE_HALFDEAD 0xB0	/* mark a leaf as half-dead */
```

---

# 7. Deduplication ve posting list

## 7.1 Fikir

Aynı anahtarın N kopyası yerine, anahtarı bir kez + N heap TID'i bir dizide sakla:

```
  ÖNCE                                SONRA (posting list tuple)
  ┌──────────────┐                    ┌──────────────────────────────┐
  │ 'ali' (1,4)  │                    │ 'ali' │ (1,4)(1,9)(3,2)(7,1) │
  │ 'ali' (1,9)  │       →            └──────────────────────────────┘
  │ 'ali' (3,2)  │                     tek t_info + tek anahtar kopyası
  │ 'ali' (7,1)  │
  └──────────────┘
```

README'nin dediği gibi bu **tembel** bir işlemdir — yalnızca sayfa bölünmek
üzereyken yapılır:

[../src/backend/access/nbtree/README:906-914](../src/backend/access/nbtree/README#L906-L914)

```
Deduplication is always
applied lazily, at the point where it would otherwise be necessary to
perform a page split.  It occurs only when LP_DEAD items have been
removed, as our last line of defense against splitting a leaf page
(bottom-up index deletion may be attempted first, as our second last line
of defense).
```

## 7.2 `_bt_dedup_pass`

[../src/backend/access/nbtree/nbtdedup.c:59](../src/backend/access/nbtree/nbtdedup.c#L59)

Yeni bir geçici sayfaya, ardışık eşit tuple'lar birleştirilerek yazılır:

[../src/backend/access/nbtree/nbtdedup.c:143-160](../src/backend/access/nbtree/nbtdedup.c#L143-L160)

```c
		if (offnum == minoff)
		{
			/*
			 * No previous/base tuple for the data item -- use the data item
			 * as base tuple of pending posting list
			 */
			_bt_dedup_start_pending(state, itup, offnum);
		}
		else if (state->deduplicate &&
				 _bt_keep_natts_fast(rel, state->base, itup) > nkeyatts &&
				 _bt_dedup_save_htid(state, itup))
		{
			/*
			 * Tuple is equal to base tuple of pending posting list.  Heap
			 * TID(s) for itup have been saved in state.
			 */
		}
```

Posting list boyutu sayfanın 1/6'sı ile sınırlanır — ileride yapılacak bölmeye
iyi bir nokta bırakmak için:

[../src/backend/access/nbtree/nbtdedup.c:89](../src/backend/access/nbtree/nbtdedup.c#L89)

```c
	state->maxpostingsize = Min(BTMaxItemSize / 2, INDEX_SIZE_MASK);
```

`allequalimage` şart: `_bt_delete_or_dedup_one_page` bunu kontrol eder
([nbtinsert.c:2826](../src/backend/access/nbtree/nbtinsert.c#L2826)).
`numeric` gibi "eşit ama farklı gösterimli" tipler dedup edilemez.

Kapatmak için index reloption'ı var:

[../src/backend/access/common/reloptions.c:157](../src/backend/access/common/reloptions.c#L157)

```c
			"deduplicate_items",
			"Enables \"deduplicate items\" feature for this btree index",
```

## 7.3 Posting list split

Yeni bir tuple'ın heap TID'i mevcut bir posting list'in aralığına düşerse,
o posting list bölünür. `_bt_swap_posting`
([nbtdedup.c:1022](../src/backend/access/nbtree/nbtdedup.c#L1022)) yeni tuple
ile posting list'in bir TID'ini takas eder; işlem `_bt_insertonpg`'nin
kritik bölümüne bir adım olarak eklenir:

[../src/backend/access/nbtree/nbtinsert.c:1208-1214](../src/backend/access/nbtree/nbtinsert.c#L1208-L1214)

```c
		/* use a mutable copy of itup as our itup from here on */
		origitup = itup;
		itup = CopyIndexTuple(origitup);
		nposting = _bt_swap_posting(itup, oposting, postingoff);
		/* itup now contains rightmost/max TID from oposting */

		/* Alter offset so that newitem goes after posting list */
```

## 7.4 Unique index'te dedup

Amaç yer kazanmak değil, **zaman kazanmaktır**:

[../src/backend/access/nbtree/README:962-966](../src/backend/access/nbtree/README#L962-L966)

```
Deduplication in unique indexes helps to prevent these pathological page
splits.  Storing duplicates in a space efficient manner is not the goal,
since in the long run there won't be any duplicates anyway.  Rather, we're
buying time for standard garbage collection mechanisms to run before a
page split is needed.
```

---

# 8. Ölü giriş temizliği

Bir index girdisi, işaret ettiği heap satırı ölünce otomatik olarak silinmez.
nbtree'nin dört ayrı temizleme mekanizması vardır:

```
  ┌─────────────────────────────────────────────────────────────────┐
  │ 1. kill_prior_tuple → LP_DEAD işaretle     (tarama sırasında)   │  ipucu
  │ 2. simple deletion  → LP_DEAD'leri sil     (ekleme sırasında)   │  kesin
  │ 3. bottom-up deletion → versiyon çöpünü ara(ekleme sırasında)   │  sezgisel
  │ 4. VACUUM btbulkdelete → tam tarama                            │  garantili
  └─────────────────────────────────────────────────────────────────┘
```

## 8.1 `kill_prior_tuple` → LP_DEAD

Executor, index taramasında dönen bir satırın heap'te herkese ölü olduğunu
görürse `scan->kill_prior_tuple = true` der
([../src/backend/access/index/indexam.c:677](../src/backend/access/index/indexam.c#L677)).
nbtree bunu hemen uygulamaz, biriktirir:

[../src/backend/access/nbtree/nbtree.c:255-268](../src/backend/access/nbtree/nbtree.c#L255-L268)

```c
			if (scan->kill_prior_tuple)
			{
				/*
				 * Yes, remember it for later. (We'll deal with all such
				 * tuples at once right before leaving the index page.)
				 */
				if (so->killedItems == NULL)
					so->killedItems = palloc_array(int, MaxTIDsPerBTreePage);
				if (so->numKilled < MaxTIDsPerBTreePage)
					so->killedItems[so->numKilled++] = so->currPos.itemIndex;
			}
```

Sayfadan ayrılırken `_bt_killitems` çağrılır
([nbtsearch.c:1664](../src/backend/access/nbtree/nbtsearch.c#L1664),
[nbtree.c:398](../src/backend/access/nbtree/nbtree.c#L398),
[nbtree.c:464](../src/backend/access/nbtree/nbtree.c#L464)):

[../src/backend/access/nbtree/nbtutils.c:191](../src/backend/access/nbtree/nbtutils.c#L191)

İşaretleme yalnızca **hint bit** altyapısıyla, shared lock altında yapılır:

[../src/backend/access/nbtree/nbtutils.c:347-363](../src/backend/access/nbtree/nbtutils.c#L347-L363)

```c
			if (killtuple && !ItemIdIsDead(iid))
			{
				if (!killedsomething)
				{
					/*
					 * Use the hint bit infrastructure to check if we can
					 * update the page while just holding a share lock. If we
					 * are not allowed, there's no point continuing.
					 */
					if (!BufferBeginSetHintBits(buf))
						goto unlock_page;
				}

				/* found the item/all posting list items */
				ItemIdMarkDead(iid);
				killedsomething = true;
```

**Yarış koşulu.** Pin bırakan taramalarda (bkz. bölüm 10.2), aradan geçen VACUUM
TID'leri geri dönüştürmüş olabilir; aynı TID'i kullanan *yeni* bir tuple'ı
yanlışlıkla ölü işaretlemek felaket olur. Çözüm: sayfa LSN'i değiştiyse tümüyle
vazgeç:

[../src/backend/access/nbtree/nbtutils.c:243-250](../src/backend/access/nbtree/nbtutils.c#L243-L250)

```c
		latestlsn = BufferGetLSNAtomic(buf);
		Assert(so->currPos.lsn <= latestlsn);
		if (so->currPos.lsn != latestlsn)
		{
			/* Modified, give up on hinting */
			_bt_relbuf(rel, buf);
			return;
		}
```

Bu, unlogged index'lerde bile "sahte LSN" tutulmasının sebebidir
([README:483-486](../src/backend/access/nbtree/README#L483-L486)).

## 8.2 Simple deletion

LP_DEAD işareti yalnızca bir ipucudur; fiziksel silme exclusive kilit ister ve
ekleme yolunda, bölmeden hemen önce yapılır:

[../src/backend/access/nbtree/nbtinsert.c:2730](../src/backend/access/nbtree/nbtinsert.c#L2730)

```c
static void
_bt_delete_or_dedup_one_page(Relation rel, Relation heapRel,
							 BTInsertState insertstate,
							 bool simpleonly, bool checkingunique,
							 bool uniquedup, bool indexUnchanged)
```

Önce LP_DEAD'ler toplanır, sonra `_bt_simpledel_pass`
([nbtinsert.c:2859](../src/backend/access/nbtree/nbtinsert.c#L2859)) çağrılır:

[../src/backend/access/nbtree/nbtinsert.c:2768-2775](../src/backend/access/nbtree/nbtinsert.c#L2768-L2775)

```c
	if (ndeletable > 0)
	{
		_bt_simpledel_pass(rel, buffer, heapRel, deletable, ndeletable,
						   insertstate->itup, minoff, maxoff);
		insertstate->bounds_valid = false;

		/* Return when a page split has already been avoided */
		if (PageGetFreeSpace(page) >= insertstate->itemsz)
			return;
```

Kurnazlık: tableam zaten snapshotConflictHorizon üretmek için heap bloklarını
ziyaret edecek. Aynı bloklara işaret eden **LP_DEAD olmayan** komşu girdiler de
"yolda" kontrol edilir. `_bt_deadblocks`
([nbtinsert.c:2985](../src/backend/access/nbtree/nbtinsert.c#L2985)) o blok
listesini çıkarır.

## 8.3 Bottom-up deletion

[../src/backend/access/nbtree/nbtdedup.c:309](../src/backend/access/nbtree/nbtdedup.c#L309)

Tetikleyici `indexUnchanged` ipucu veya unique index'te görülen tekrar:

[../src/backend/access/nbtree/nbtinsert.c:2821-2828](../src/backend/access/nbtree/nbtinsert.c#L2821-L2828)

```c
	if ((indexUnchanged || uniquedup) &&
		_bt_bottomupdel_pass(rel, buffer, heapRel, insertstate->itemsz))
		return;

	/* Perform deduplication pass (when enabled and index-is-allequalimage) */
	if (BTGetDeduplicateItems(rel) && itup_key->allequalimage)
		_bt_dedup_pass(rel, buffer, insertstate->itup, insertstate->itemsz,
					   (indexUnchanged || uniquedup));
```

Amaç niceliksel değil **niteliksel**:

[../src/backend/access/nbtree/nbtdedup.c:284-291](../src/backend/access/nbtree/nbtdedup.c#L284-L291)

```
 * See if duplicate index tuples (plus certain nearby tuples) are eligible to
 * be deleted via bottom-up index deletion.  The high level goal here is to
 * entirely prevent "unnecessary" page splits caused by MVCC version churn
 * from UPDATEs (when the UPDATEs don't logically modify any of the columns
 * covered by the 'rel' index).  This is qualitative, not quantitative -- we
 * do not particularly care about once-off opportunities to delete many index
 * tuples together.
```

README bunu "kuşak hipotezi"ne dayandırır: çoğu nesne genç ölür
([README:592-600](../src/backend/access/nbtree/README#L592-L600)).

---

# 9. VACUUM tarafı

## 9.1 `btbulkdelete` → `btvacuumscan`

[../src/backend/access/nbtree/nbtree.c:1122](../src/backend/access/nbtree/nbtree.c#L1122)

```c
IndexBulkDeleteResult *
btbulkdelete(IndexVacuumInfo *info, IndexBulkDeleteResult *stats,
			 IndexBulkDeleteCallback callback, void *callback_state)
{
	...
		cycleid = _bt_start_vacuum(rel);

		btvacuumscan(info, stats, callback, callback_state, cycleid);
```

Tarama **fiziksel blok sırasındadır**, mantıksal anahtar sırasında değil
([nbtree.c:1240](../src/backend/access/nbtree/nbtree.c#L1240)). Blok 1'den
başlar ve read stream ile ilerler:

[../src/backend/access/nbtree/nbtree.c:1316](../src/backend/access/nbtree/nbtree.c#L1316)

```c
	p.current_blocknum = BTREE_METAPAGE + 1;
```

İlişki uzunluğu **her turda yeniden okunur** — tarama sırasında yeni sayfa
eklenmiş olabilir ([nbtree.c:1336-1340](../src/backend/access/nbtree/nbtree.c#L1336-L1340)).

## 9.2 Cleanup lock ve cycle ID

Her yaprak sayfada — silinecek bir şey olmasa bile — **cleanup lock** alınır:

[../src/backend/access/nbtree/nbtree.c:1533-1539](../src/backend/access/nbtree/nbtree.c#L1533-L1539)

```c
		/*
		 * Trade in the initial read lock for a full cleanup lock on this
		 * page.  We must get such a lock on every leaf page over the course
		 * of the vacuum scan, whether or not it actually contains any
		 * deletable tuples --- see nbtree/README.
		 */
		_bt_upgradelockbufcleanup(rel, buf);
```

Bu, VACUUM ile index taramaları arasındaki **interlock**'tur: pin tutan bir
tarama bitmeden VACUUM o sayfadaki TID'leri geri dönüştüremez
([README:169-187](../src/backend/access/nbtree/README#L169-L187)).

**Cycle ID** fiziksel tarama ile eşzamanlı bölmeyi uzlaştırır. Bölme, sağ yarıyı
*daha küçük* bir blok numarasına koyabilir — yani zaten geçtiğimiz bir yere.
Bunu yakalayıp geri dönüyoruz:

[../src/backend/access/nbtree/nbtree.c:1550-1555](../src/backend/access/nbtree/nbtree.c#L1550-L1555)

```c
		if (vstate->cycleid != 0 &&
			opaque->btpo_cycleid == vstate->cycleid &&
			!(opaque->btpo_flags & BTP_SPLIT_END) &&
			!P_RIGHTMOST(opaque) &&
			opaque->btpo_next < scanblkno)
			backtrack_to = opaque->btpo_next;
```

Test **yanlış pozitif** verebilir ama asla yanlış negatif vermez; bu yüzden
16 bitlik küçük bir sayaç yeterlidir
([README:227-230](../src/backend/access/nbtree/README#L227-L230)).

Silme kararları callback ile verilir; posting list'te kısmi silme mümkündür
([nbtree.c:1592-1620](../src/backend/access/nbtree/nbtree.c#L1592-L1620)),
sonra tek bir `_bt_delitems_vacuum` çağrısıyla uygulanır
([nbtpage.c:1162](../src/backend/access/nbtree/nbtpage.c#L1162)).

## 9.3 Sayfa silme: iki aşamalı

Sadece **tamamen boş yaprak** sayfa silinebilir; rightmost sayfa asla silinmez.

[../src/backend/access/nbtree/nbtpage.c:1812](../src/backend/access/nbtree/nbtpage.c#L1812)

```c
void
_bt_pagedel(Relation rel, Buffer leafbuf, BTVacState *vstate)
```

```
  AŞAMA 1  _bt_mark_page_halfdead()              nbtpage.c:2102
     ├─ ebeveyni bul, hedefin downlink'ini kaldır
     ├─ anahtar uzayı SAĞ kardeşe devredilir
     └─ hedefe BTP_HALF_DEAD → arayıcılar artık onu atlar
     WAL: XLOG_BTREE_MARK_PAGE_HALFDEAD

  AŞAMA 2  _bt_unlink_halfdead_page()            nbtpage.c:2329
     ├─ sol kardeş, hedef, sağ kardeş — bu SIRAYLA kilitle
     ├─ sibling linklerini birleştir
     └─ BTP_DELETED + safexid yaz
     WAL: XLOG_BTREE_UNLINK_PAGE
```

Aşama 1'in özeti README'de:

[../src/backend/access/nbtree/README:247-257](../src/backend/access/nbtree/README#L247-L257)

```
Deleting a leaf page is a two-stage process.  In the first stage, the page
is unlinked from its parent, and marked as half-dead. ... we lock the target and the parent pages, change the
target's downlink to point to the right sibling, and remove its old
downlink.  This causes the target page's key space to effectively belong to
its right sibling.
```

Bir ebeveynin **son çocuğu** siliniyorsa, ince bir alt ağaç zinciri birlikte
silinir ([README:284-317](../src/backend/access/nbtree/README#L284-L317)).

## 9.4 Geri dönüşüm — neden hemen değil?

Silinen sayfa hemen FSM'e verilemez, çünkü ona doğru "yolda" olan okuyucular
olabilir. Sayfa bir süre **mezar taşı** olarak durur (Lanin & Shasha'nın "drain
technique"i). Kurtarma için sayfa içeriğine bir safexid yazılır:

[../src/include/access/nbtree.h:232-237](../src/include/access/nbtree.h#L232-L237)

```c
/*
 * BTDeletedPageData is the page contents of a deleted page
 */
typedef struct BTDeletedPageData
{
	FullTransactionId safexid;	/* See BTPageIsRecyclable() */
} BTDeletedPageData;
```

Geri dönüşüm testi `BTPageIsRecyclable`
([nbtree.h:292](../src/include/access/nbtree.h#L292)); `btvacuumpage` bunu
kullanır:

[../src/backend/access/nbtree/nbtree.c:1496-1501](../src/backend/access/nbtree/nbtree.c#L1496-L1501)

```c
	if (!opaque || BTPageIsRecyclable(page, heaprel))
	{
		/* Okay to recycle this page (which could be leaf or internal) */
		RecordFreeIndexPage(rel, blkno);
		stats->pages_deleted++;
		stats->pages_free++;
	}
```

PostgreSQL 14'ten beri, aynı VACUUM'un taraması bitince yeni silinen sayfalar
da denenir (`_bt_pendingfsm_finalize`,
[nbtpage.c:3013](../src/backend/access/nbtree/nbtpage.c#L3013)).

Henüz dönüştürülemeyen sayfa sayısı meta sayfada tutulur ve bir sonraki
VACUUM'un "index taramasını atlayabilir miyim?" kararını besler
(`_bt_vacuum_needs_cleanup`,
[nbtpage.c:180](../src/backend/access/nbtree/nbtpage.c#L180)).

---

# 10. Tarama

## 10.1 `btgettuple` ve `btgetbitmap`

[../src/backend/access/nbtree/nbtree.c:230](../src/backend/access/nbtree/nbtree.c#L230) — `btgettuple`
[../src/backend/access/nbtree/nbtree.c:291](../src/backend/access/nbtree/nbtree.c#L291) — `btgetbitmap`

İkisi de aynı motoru kullanır: `_bt_first` → `_bt_readpage` → `_bt_next`.
Fark, `btgetbitmap`'in tüm TID'leri tek seferde bir `TIDBitmap`'e doldurmasıdır:

[../src/backend/access/nbtree/nbtree.c:324-326](../src/backend/access/nbtree/nbtree.c#L324-L326)

```c
				heapTid = &so->currPos.items[so->currPos.itemIndex].heapTid;
				tbm_add_tuples(tbm, heapTid, 1, false);
				ntids++;
```

Bitmap taramada `scan->heapRelation == NULL`'dır
([nbtree.c:297](../src/backend/access/nbtree/nbtree.c#L297)), bu yüzden
`kill_prior_tuple` optimizasyonu **bitmap taramada çalışmaz** — index ile heap
senkron gezilmediği için.

## 10.2 Sayfa başına çalışma ve pin politikası

Tarama sayfa sayfa ilerler: sayfa kilitlenir, eşleşen tüm öğeler
backend-yerel `so->currPos.items[]` dizisine kopyalanır, kilit bırakılır:

[../src/include/access/nbtree.h:936-945](../src/include/access/nbtree.h#L936-L945)

```
 * Index scans work a page at a time: we pin and read-lock the page, identify
 * all the matching items on the page and save them in BTScanPosData, then
 * release the read-lock while returning the items to the caller for
 * processing.  This approach minimizes lock/unlock traffic.  We must always
 * drop the lock to make it okay for caller to process the returned items.
 * Whether or not we can also release the pin during this window will vary.
 * We drop the pin (when so->dropPin) to avoid blocking progress by VACUUM
```

Karar `btrescan`'de verilir:

[../src/backend/access/nbtree/nbtree.c:419-421](../src/backend/access/nbtree/nbtree.c#L419-L421)

```c
	so->dropPin = (!scan->xs_want_itup &&
				   IsMVCCLikeSnapshot(scan->xs_snapshot) &&
				   scan->heapRelation != NULL);
```

| Koşul | Neden |
|---|---|
| `!xs_want_itup` | Index-only scan **pin bırakamaz** (bkz. 11.2) |
| MVCC benzeri snapshot | Snapshot, yanlış cevaba karşı yeterli koruma sağlar |
| `heapRelation != NULL` | Bitmap taramada pin zaten anlık, LSN okumaya değmez |

Uygulama:

[../src/backend/access/nbtree/nbtsearch.c:57-74](../src/backend/access/nbtree/nbtsearch.c#L57-L74)

```c
_bt_drop_lock_and_maybe_pin(Relation rel, BTScanOpaque so)
{
	if (!so->dropPin)
	{
		/* Just drop the lock (not the pin) */
		_bt_unlockbuf(rel, so->currPos.buf);
		return;
	}

	/*
	 * Drop both the lock and the pin.
	 *
	 * Have to set so->currPos.lsn so that _bt_killitems has a way to detect
	 * when concurrent heap TID recycling by VACUUM might have taken place.
	 */
	so->currPos.lsn = BufferGetLSNAtomic(so->currPos.buf);
	_bt_relbuf(rel, so->currPos.buf);
```

## 10.3 Sayfalar arası geçiş: right link hafızası

Tarama "sayfalar arasında" durur. Sağa geçerken **o an okunan** right link
kullanılır, sayfanın güncel right link'i değil — yoksa bölme ile taşınan öğeler
iki kez okunurdu:

[../src/backend/access/nbtree/README:99-101](../src/backend/access/nbtree/README#L99-L101)

```
The scan must remember the page's right-link at the time it
was scanned, since that is the page to move right to; if we move right to
the current right-link then we'd re-scan any items moved by a page split.
```

Bu yüzden `BTScanPosData` linkleri saklar:

[../src/include/access/nbtree.h:967-969](../src/include/access/nbtree.h#L967-L969)

```c
	BlockNumber currPage;		/* page referenced by items array */
	BlockNumber prevPage;		/* currPage's left link */
	BlockNumber nextPage;		/* currPage's right link */
```

Geri yönlü geçiş çok daha zordur (sol kardeş bölünmüş ya da bizim sayfamız
silinmiş olabilir); algoritma README'de dört adımda anlatılır
([README:337-354](../src/backend/access/nbtree/README#L337-L354)) ve
`_bt_lock_and_validate_left`
([nbtsearch.c:1982](../src/backend/access/nbtree/nbtsearch.c#L1982)) ile
uygulanır.

---

# 11. Index-only scan ve visibility map

## 11.1 `IndexOnlyNext`

[../src/backend/executor/nodeIndexonlyscan.c:63](../src/backend/executor/nodeIndexonlyscan.c#L63)

Kurulumda `xs_want_itup` açılır — AM'e "bana index tuple'ın kendisini de ver" der:

[../src/backend/executor/nodeIndexonlyscan.c:105-107](../src/backend/executor/nodeIndexonlyscan.c#L105-L107)

```c
		/* Set it up for index-only scan */
		node->ioss_ScanDesc->xs_want_itup = true;
		node->ioss_VMBuffer = InvalidBuffer;
```

Ana döngü tek bir karara indirgenir — **visibility map biti**:

[../src/backend/executor/nodeIndexonlyscan.c:164-175](../src/backend/executor/nodeIndexonlyscan.c#L164-L175)

```c
		if (!VM_ALL_VISIBLE(scandesc->heapRelation,
							ItemPointerGetBlockNumber(tid),
							&node->ioss_VMBuffer))
		{
			/*
			 * Rats, we have to visit the heap to check visibility.
			 */
			InstrCountTuples2(node, 1);
			if (!index_fetch_heap(scandesc, node->ioss_TableSlot))
				continue;		/* no visible tuple, try next index entry */
```

VM biti okunurken **kilit alınmaz**; sonucun biraz bayat olması sorun değildir.
Yorum bunun neden güvenli olduğunu bellek sıralaması düzeyinde açıklar
([nodeIndexonlyscan.c:132-161](../src/backend/executor/nodeIndexonlyscan.c#L132-L161)):
INSERT'te VM biti index güncellemesinden **önce** temizlenir ve index sayfası
kilit/kilit-açma tam bir bellek bariyeridir.

Bir index'in bu şekilde kullanılabilmesi `amcanreturn` ile bildirilir:

[../src/backend/access/nbtree/nbtree.c:1802](../src/backend/access/nbtree/nbtree.c#L1802)

```c
bool
btcanreturn(Relation index, int attno)
```

## 11.2 Index-only scan neden pin bırakamaz?

Çünkü heap'e bakmıyor. Elindeki TID geri dönüştürülmüşse bunu asla fark edemez:

[../src/backend/access/nbtree/README:466-476](../src/backend/access/nbtree/README#L466-L476)

```
Index-only scans can never drop their buffer pin, since they are unable to
tolerate having a referenced TID become recyclable.  Index-only scans
typically just visit the visibility map (not the heap proper), and so will
not reliably notice that any stale TID reference (for a TID that pointed
to a dead-to-all heap item at first) was concurrently marked LP_UNUSED in
the heap by VACUUM.  This could easily allow VACUUM to set the whole heap
page to all-visible in the visibility map immediately afterwards.
```

Normal index scan ise LP_UNUSED bir öğeyi "görünmez" sayarak kendini kurtarır;
bu yüzden ona MVCC snapshot yeter.

```
  Normal index scan      : snapshot yeter → pin bırakılır → VACUUM engellenmez
  Index-only scan        : pin ŞART       → cleanup lock bekleyen VACUUM bloklanır
  Bitmap scan            : pin zaten anlık
```

---

# İzleme ve hata ayıklama

## pageinspect — index'in içine bakmak

```sql
CREATE EXTENSION pageinspect;
```

**Meta sayfa** ([../contrib/pageinspect/btreefuncs.c:44](../contrib/pageinspect/btreefuncs.c#L44)):

```sql
SELECT * FROM bt_metap('users_id_idx');
-- magic                     | 340322        (0x053162)
-- version                   | 4
-- root                      | 3
-- level                     | 1             ← ağaç 2 seviyeli
-- fastroot                  | 3
-- fastlevel                 | 1
-- last_cleanup_num_delpages | 0             ← FSM'e verilemeyen silinmiş sayfa
-- allequalimage             | t             ← deduplication mümkün mü?
```

`level > 3` ise ağaç şişmiş olabilir. `root != fastroot` ise kitlesel silmeden
kalma "zayıf" seviyeler var.

**Sayfa istatistikleri**:

```sql
SELECT * FROM bt_page_stats('users_id_idx', 1);
-- type          | l          l=leaf, i=internal, r=root, d=deleted, e=ignored
-- live_items    | 224
-- dead_items    | 0          ← LP_DEAD işaretli, henüz silinmemiş
-- free_size     | 3668
-- btpo_prev     | 0
-- btpo_next     | 0
-- btpo_level    | 0
-- btpo_flags    | 3          ← BTP_LEAF|BTP_ROOT
```

Tüm index'i tek sorguda taramak
([btreefuncs.c:50](../contrib/pageinspect/btreefuncs.c#L50)):

```sql
-- Yaprak sayfalarda ortalama doluluk ve ölü oranı
SELECT type,
       count(*)                                AS pages,
       round(avg(free_size))                   AS avg_free,
       sum(dead_items)                         AS dead_items
FROM bt_multi_page_stats('users_id_idx', 1, -1)
GROUP BY type;
```

**Sayfa öğeleri** — high key ve posting list'leri görmek:

```sql
SELECT itemoffset, ctid, itemlen, dead, htid, tids[0:3] AS some_tids
FROM bt_page_items('users_id_idx', 1);
```

`itemoffset = 1` non-rightmost sayfada high key'dir (`htid` boş gelir).
`tids` dolu ise o satır bir **posting list tuple**'dır.

## amcheck — yapısal doğrulama

[../contrib/amcheck/verify_nbtree.c:176-177](../contrib/amcheck/verify_nbtree.c#L176-L177)

```sql
CREATE EXTENSION amcheck;

-- Hafif: AccessShareLock, yalnızca yaprak seviyesi tutarlılığı
SELECT bt_index_check('users_id_idx');

-- Ağır: ShareLock, parent/child ilişkilerini de doğrular
SELECT bt_index_parent_check('users_id_idx', heapallindexed => true);
```

`bt_index_parent_check` yazma engeller — production'da dikkat.

## Şişkinlik ve kullanım

```sql
-- Index kaç kez kullanıldı?
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes WHERE relname = 'users';

-- idx_tup_read >> idx_tup_fetch  → çok satır okunup eleniyor
```

```sql
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstatindex('users_id_idx');
-- leaf_fragmentation, avg_leaf_density  → şişkinlik göstergeleri
```

## EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM users WHERE id BETWEEN 100 AND 200;
```

Bakılacak satırlar:

| Çıktı | Anlamı |
|---|---|
| `Index Only Scan ... Heap Fetches: 0` | VM tamamen all-visible; ideal durum |
| `Heap Fetches: 12345` | VM bayat → `VACUUM users` çalıştır |
| `Index Cond:` vs `Filter:` | `Filter` altındaki koşullar index'te değerlendirilmedi |
| `Rows Removed by Filter` | Yüksekse index kolon sırası yanlış olabilir |
| `Bitmap Index Scan` + `Recheck Cond` | `work_mem` yetmedi, bitmap "lossy" oldu |

## İlgili GUC ve reloption'lar

| Ayar | Nerede | Etki |
|---|---|---|
| `enable_indexscan` | [guc_parameters.dat:943](../src/backend/utils/misc/guc_parameters.dat#L943) | Planlayıcıyı index scan'den caydırır (test için) |
| `enable_indexonlyscan` | [guc_parameters.dat:936](../src/backend/utils/misc/guc_parameters.dat#L936) | Index-only scan'i kapatır |
| `enable_bitmapscan` | [guc_parameters.dat:873](../src/backend/utils/misc/guc_parameters.dat#L873) | Bitmap scan'i kapatır |
| `fillfactor` (reloption) | [reloptions.c:191](../src/backend/access/common/reloptions.c#L191) | Varsayılan 90; rightmost bölmede kullanılır |
| `deduplicate_items` (reloption) | [reloptions.c:157](../src/backend/access/common/reloptions.c#L157) | Varsayılan `true`; dedup'ı kapatır |
| `maintenance_work_mem` | — | `CREATE INDEX` sıralama belleği (`nbtsort.c`) |
| `random_page_cost`, `effective_cache_size` | — | Index scan maliyet tahmini |

Sorun izleme için hızlı eşleştirme:

```
 "index şişiyor, VACUUM işe yaramıyor"
    → bt_metap: last_cleanup_num_delpages yüksek mi?
    → uzun süren transaction / replication slot safexid'leri kilitliyor olabilir
 "UPDATE'ler yavaşladı, index büyüyor"
    → bottom-up deletion çalışmıyor olabilir; indexUnchanged ipucu HOT'a bağlı
    → bt_page_stats ile dead_items ve free_size'a bak
 "index-only scan Heap Fetches yüksek"
    → VM bayat; autovacuum eşiklerini düşür
```

---

# Tek sayfalık özet

```
                            ┌──────────────────────────────────────┐
                            │  META (blok 0)                       │
                            │  btm_root / btm_fastroot / btm_level │
                            │  btm_allequalimage                   │
                            └──────────────┬───────────────────────┘
   OKUMA                                   │  (relcache: rd_amcache)
   _bt_first                               ▼
      └─ _bt_search  ──────────►  ┌──────────────────────┐
            │                     │  ROOT / INTERNAL     │  pivot tuple:
            │  her seviyede:      │  [-inf|→][k1|→][k2|→]│  ayırıcı + downlink
            │  _bt_moveright      └──┬───────────────────┘  suffix truncated
            │     (high key > key?   │
            │      → btpo_next'e git)│
            │  _bt_binsrch           ▼
            │  (ikili arama)  ┌──────────────────────────────────────┐
            └───────────────► │  LEAF                                │
                              │  off1 = HIGH KEY (pivot)             │
                              │  off2.. = non-pivot / posting list   │
                              │  btpo_prev ◄──►  btpo_next           │
                              └──────────────────────────────────────┘
                                        │
                       _bt_readpage: eşleşenleri so->currPos.items[]'a kopyala
                       kilit bırak (pin? → so->dropPin kararı)
                                        │
                          ┌─────────────┴──────────────┐
                          ▼                            ▼
                  Index Scan                    Index-Only Scan
                  heap'e git                    VM_ALL_VISIBLE?
                  ölüyse kill_prior_tuple       evet → heap'e hiç gitme
                                                hayır → heap fetch
                                                pin ASLA bırakılmaz

   ─────────────────────────────────────────────────────────────────────

   YAZMA   btinsert → _bt_doinsert
              ├─ _bt_search_insert     (fastpath: rightmost cache)
              ├─ _bt_check_unique      UNIQUE ise; kilit = kısıtın kendisi
              ├─ _bt_findinsertloc     yer yoksa ↓
              │     ┌──────────────────────────────────────────────┐
              │     │ 1. simple deletion   (LP_DEAD'leri sil)      │
              │     │ 2. bottom-up deletion(versiyon çöpü)         │
              │     │ 3. deduplication     (posting list kur)      │
              │     └──────────────────────────────────────────────┘
              └─ _bt_insertonpg
                    ├─ sığdı  → PageAddItem + XLOG_BTREE_INSERT_LEAF
                    └─ sığmadı→ _bt_findsplitloc → _bt_split
                                  │  sol: BTP_INCOMPLETE_SPLIT
                                  │  sağ: yeni sayfa, linkler kurulur
                                  ▼
                               _bt_insert_parent  (özyineleme, stack ile)
                                  └─ kök ise _bt_newlevel + meta güncelle

   ─────────────────────────────────────────────────────────────────────

   TEMİZLİK  btbulkdelete → btvacuumscan  (FİZİKSEL blok sırası)
                 └─ btvacuumpage (her yaprakta CLEANUP LOCK — istisnasız)
                       ├─ cycleid testi → gerekirse geriye dön (backtrack)
                       ├─ callback ile ölü TID'leri seç
                       ├─ _bt_delitems_vacuum
                       └─ sayfa boşaldıysa _bt_pagedel
                             ├─ AŞAMA 1: half-dead + downlink kaldır
                             └─ AŞAMA 2: sibling linkleri birleştir + safexid
                                   └─ safexid eskiyince → FSM (recycle)
```

Üç cümlelik kapanış:

1. **Right link + high key** sayesinde okuyucu hiçbir zaman kaybolmaz; ağaç
   bölünse bile sağa yürüyerek doğru sayfayı bulur.
2. **Heap TID tiebreaker'ı** her anahtarı benzersiz yapar; bu, hem L&Y
   değişmezini korur hem de tekrarlı değerlerin tek bir sayfaya "hapsedilmesini"
   mümkün kılar.
3. **Sayfa bölmesi son çaredir**; simple deletion, bottom-up deletion ve
   deduplication sırayla denenerek gereksiz bölmeler önlenir.
