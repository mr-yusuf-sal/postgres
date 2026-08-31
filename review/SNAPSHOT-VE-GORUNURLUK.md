# Snapshot ve görünürlük — bir satırı kim görür?

*PostgreSQL **20devel** ağacı. Satır numaraları bu sürüme aittir
([meson.build:11](../meson.build#L11)).*

[TRANSACTION-YONETIMI.md](TRANSACTION-YONETIMI.md) bir transaction'ın xid'i nasıl
aldığını ve commit'in CLOG'a nasıl yazıldığını anlatıyor. Bu not bir sonraki
soruyu cevaplıyor: **CLOG'da "committed" yazan bir xid'in ürettiği satırı ben
görmeli miyim?** Cevap CLOG'da değil, okuyanın elindeki *snapshot*'ta.

[VACUUM-NASIL-CALISIR.md](VACUUM-NASIL-CALISIR.md) ikinci yarıya bağlanıyor:
VACUUM'un `OldestXmin` cutoff'u buradaki xmin horizon hesabından geliyor. Uzun
bir transaction'ın "VACUUM'u durdurması" tam olarak burada olur.

---

## 30 saniyelik özet

```
   GetSnapshotData()  ── ProcArrayLock SHARED ──► ProcArray'i gez
        │
        ▼
   ┌──────────────────────────────────────────────────────┐
   │  SnapshotData                                        │
   │    xmin ───► bundan küçük her xid BİTMİŞ              │
   │    xmax ───► bundan büyük/eşit her xid ÇALIŞIYOR      │
   │    xip[] ──► aradakilerden ÇALIŞANLARIN listesi       │
   │    curcid ─► kendi transaction'ımın komut sınırı      │
   └────────────────────────┬─────────────────────────────┘
                            ▼
        HeapTupleSatisfiesMVCC(tuple, snapshot)
             t_xmin görünür mü?   hint bit → snapshot → CLOG
             t_xmax görünür mü?   aynı sıra

   xid ekseni:
   ───────┬─────────────────┬────────────────────┬────────►
     eski │      xmin       │   xip[] taranır    │  xmax   yeni
          │◄─ hepsi bitmiş ►│◄─── karışık ──────►│◄ hiçbiri
                                                    görünmez
```

Zihinsel model — altı madde:

1. **Snapshot bir zaman damgası değil, bir küme.** İçinde "o an çalışan
   transaction'ların xid'leri" var. Karar hep aynı soruya iner: *bu xid o anda
   çalışıyor muydu?* — çalışıyorduysa etkisi görünmez.
2. **İki sayı işin %99'unu bitirir.** `xmin`'den küçükse kesin bitmiş, `xmax`'tan
   büyük/eşitse kesin çalışıyor. Sadece aradakiler için `xip[]` taranır
   ([../src/include/utils/snapshot.h:148-151](../src/include/utils/snapshot.h#L148-L151)).
3. **CLOG "commit etti mi", snapshot "ben görmeli miyim" der.** İki ayrı soru.
   Commit etmiş ama benim snapshot'ımda çalışan görünen bir transaction'ın
   satırını göremem.
4. **Hint bit'ler CLOG'a gitmeyi önlemek için var.** İlk okuyan CLOG'a bakar ve
   sonucu tuple header'ına yazar — bu yüzden *salt okuma bile sayfayı kirletir*.
5. **Herkesin snapshot'ı VACUUM'u sınırlar.** Her backend en eski snapshot'ının
   xmin'ini `PGPROC.xmin`'de ilan eder; hepsinin minimumu xmin horizon'dur.
6. **READ COMMITTED her komutta yeni snapshot alır, REPEATABLE READ bir kez.**
   Tek fark budur; kod olarak da tek bir `if`
   ([../src/backend/utils/time/snapmgr.c:337-338](../src/backend/utils/time/snapmgr.c#L337-L338)).

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/include/utils/snapshot.h](../src/include/utils/snapshot.h) | `SnapshotData` yapısı ve yedi snapshot türü. 212 satır, tamamı okunmalı |
| [../src/backend/storage/ipc/procarray.c](../src/backend/storage/ipc/procarray.c) | `GetSnapshotData`, xmin horizon, grup xid temizliği, `KnownAssignedXids`. **En önemli dosya** |
| [../src/backend/utils/time/snapmgr.c](../src/backend/utils/time/snapmgr.c) | Snapshot yaşam döngüsü: kim alır, kim tutar, ne zaman bırakır; export/import |
| [../src/backend/access/heap/heapam_visibility.c](../src/backend/access/heap/heapam_visibility.c) | Asıl karar: `HeapTupleSatisfies*` ailesi, hint bit yazımı |
| [../src/include/utils/snapmgr.h](../src/include/utils/snapmgr.h) | `SnapshotSelf` / `SnapshotAny` makroları, `InitDirtySnapshot` |
| [../src/backend/access/transam/README](../src/backend/access/transam/README) | Kilit protokolünün *neden* böyle olduğu (satır 224-340) |
| [../src/backend/storage/ipc/standby.c](../src/backend/storage/ipc/standby.c) | Standby çatışmaları, `max_standby_streaming_delay`, `LogStandbySnapshot` |
| [../src/backend/replication/walreceiver.c](../src/backend/replication/walreceiver.c) | `hot_standby_feedback` mesajının gönderimi |
| [../src/include/storage/proc.h](../src/include/storage/proc.h) | `PGPROC.xmin`, `PGPROC.xid`, `PROC_IN_VACUUM` gibi status flag'ler |

---

# 1. `SnapshotData` — bir yapı, yedi anlam

Tamamı tek struct'ta ([../src/include/utils/snapshot.h:138](../src/include/utils/snapshot.h#L138)).
Dosyanın kendi yorumu bunun ideal olmadığını itiraf ediyor: "bunu NodeTag ile
bölmek muhtemelen iyi bir fikir" diye bir TODO duruyor
([satır 133-136](../src/include/utils/snapshot.h#L133-L136)).

```c
	TransactionId xmin;			/* all XID < xmin are visible to me */
	TransactionId xmax;			/* all XID >= xmax are invisible to me */
	...
	TransactionId *xip;
	uint32		xcnt;			/* # of xact ids in xip[] */
	...
	TransactionId *subxip;
	int32		subxcnt;		/* # of xact ids in subxip[] */
	bool		suboverflowed;	/* has the subxip array overflowed? */
```
— [../src/include/utils/snapshot.h:153-178](../src/include/utils/snapshot.h#L153-L178)

`xmin` doğruluk için gerekli değil, bir **optimizasyon**: yorum onu "çoğu tuple
için XID dizilerini aramaya gerek kalmasın diye saklanıyor" diye tanımlıyor
([satır 148-151](../src/include/utils/snapshot.h#L148-L151)). `xip[]` içindeki her
id `xmin <= xip[i] < xmax` aralığındadır
([satır 162](../src/include/utils/snapshot.h#L162)).

`subxip[]` subtransaction'lar için; PGPROC'taki subxid cache sınırlı olduğundan
taşabilir. Taşarsa `suboverflowed = true` olur ve görünürlük kontrolü
`pg_subtrans`'a inmek zorunda kalır (ayrıntısı
[TRANSACTION-YONETIMI.md](TRANSACTION-YONETIMI.md) bölüm 3.4).

Snapshot başkalarını `xip[]` ile eler, *kendi* komutlarımı `curcid` ile
([satır 183](../src/include/utils/snapshot.h#L183)) — `UPDATE t SET x = x + 1`
kendi yazdığı satırı tekrar okuyup sonsuz döngüye girmesin diye. Komut sınırında
güncellenir ([../src/backend/utils/time/snapmgr.c:743-766](../src/backend/utils/time/snapmgr.c#L743-L766)).

İki referans sayacı var — `active_count` (yığında, şu an kullanılıyor) ve
`regd_count` (bir portal/ResourceOwner ileride kullanacak); ikisi de sıfırlanınca
snapshot serbest bırakılır ([snapshot.h:200-202](../src/include/utils/snapshot.h#L200-L202),
[snapmgr.c:785-787](../src/backend/utils/time/snapmgr.c#L785-L787)). Son alan
`snapXactCompletionCount` bölüm 4'ün konusu
([snapshot.h:209](../src/include/utils/snapshot.h#L209)).

---

# 2. Yedi snapshot türü

`SnapshotType` enum'ı ([../src/include/utils/snapshot.h:31](../src/include/utils/snapshot.h#L31)):

| Tür | Satır | Ne görür |
|---|---|---|
| `SNAPSHOT_MVCC` | [46](../src/include/utils/snapshot.h#L46) | Snapshot anında commit'li olanlar + kendi önceki komutlarım |
| `SNAPSHOT_SELF` | [60](../src/include/utils/snapshot.h#L60) | *Şu an* commit'li her şey + kendi geçerli komutum. Çalışanlar hariç |
| `SNAPSHOT_ANY` | [65](../src/include/utils/snapshot.h#L65) | Her tuple görünür. Filtre yok |
| `SNAPSHOT_TOAST` | [70](../src/include/utils/snapshot.h#L70) | TOAST satırı olarak geçerliyse görünür |
| `SNAPSHOT_DIRTY` | [98](../src/include/utils/snapshot.h#L98) | `SELF` + **çalışan** transaction'ların etkileri |
| `SNAPSHOT_HISTORIC_MVCC` | [105](../src/include/utils/snapshot.h#L105) | Logical decoding için "zamanda yolculuk" MVCC |
| `SNAPSHOT_NON_VACUUMABLE` | [114](../src/include/utils/snapshot.h#L114) | "Herhangi birine görünebilir mi?" — yani vacuum edilebilir mi? |

Dağıtım tek `switch` ile:

```c
	switch (snapshot->snapshot_type)
	{
		case SNAPSHOT_MVCC:
			return HeapTupleSatisfiesMVCC(htup, snapshot, buffer, NULL);
		case SNAPSHOT_SELF:
			return HeapTupleSatisfiesSelf(htup, snapshot, buffer);
```
— [../src/backend/access/heap/heapam_visibility.c:1734-1739](../src/backend/access/heap/heapam_visibility.c#L1734-L1739)

Enum'un callback yerine tür olmasının sebebi yorumda: aynı snapshot'ı farklı
table AM'lerde kullanabilmek ([snapshot.h:27-29](../src/include/utils/snapshot.h#L27-L29)).

**`ANY`, `SELF`, `TOAST` global tekil**, çünkü içlerinde durum yok
([../src/backend/utils/time/snapmgr.c:144-146](../src/backend/utils/time/snapmgr.c#L144-L146)).
**`DIRTY` global olamaz**, çünkü çıktı parametresi taşıyor — makro ile yerel
değişkende kuruluyor ([snapmgr.h:42-43](../src/include/utils/snapmgr.h#L42-L43)):

```c
	snapshot->xmin = snapshot->xmax = InvalidTransactionId;
	snapshot->speculativeToken = 0;
```
— [../src/backend/access/heap/heapam_visibility.c:767-768](../src/backend/access/heap/heapam_visibility.c#L767-L768)

Bu alanlar geri dönüş değeri: "bu tuple'a hangi çalışan transaction dokunuyor".
Unique index çakışmasında beklenecek transaction bu yolla bulunur.

**`TOAST` neden ayrı?** TOAST tablosu kendi başına vacuum ediliyor ve o vacuum
yarıda kesilebiliyor. Kontrol asgariye indirilmiş (`return true;`,
[satır 477-478](../src/backend/access/heap/heapam_visibility.c#L477-L478)) —
mantık şu: ana tablodaki satırı görebiliyorsan TOAST parçalarını da
görebilmelisin ([satır 442-446](../src/backend/access/heap/heapam_visibility.c#L442-L446)).
Erişim `get_toast_snapshot()` üzerinden ve en az bir aktif snapshot şartıyla
([../src/backend/access/common/toast_internals.c:643-644](../src/backend/access/common/toast_internals.c#L643-L644)).

---

# 3. `GetSnapshotData` — ProcArray taraması

Giriş noktası: [../src/backend/storage/ipc/procarray.c:2114](../src/backend/storage/ipc/procarray.c#L2114).

## 3.1. `xmax` bedava geliyor

```c
	/* xmax is always latestCompletedXid + 1 */
	xmax = XidFromFullTransactionId(latest_completed);
	TransactionIdAdvance(xmax);
```
— [../src/backend/storage/ipc/procarray.c:2194-2196](../src/backend/storage/ipc/procarray.c#L2194-L2196)

Neden doğru olduğu README'de: `ProcArrayEndTransaction`, `latestCompletedXid`'i
**ProcArrayLock'u exclusive tutarken** ilerletir, bu yüzden snapshot alınırken bu
değerin ötesinde tamamlanmış bir transaction olamaz
([../src/backend/access/transam/README:259-263](../src/backend/access/transam/README#L259-L263)).

## 3.2. Döngü — sadece xid'lere bakar

```c
		for (int pgxactoff = 0; pgxactoff < numProcs; pgxactoff++)
		{
			/* Fetch xid just once - see GetNewTransactionId */
			TransactionId xid = UINT32_ACCESS_ONCE(other_xids[pgxactoff]);
```
— [../src/backend/storage/ipc/procarray.c:2220-2223](../src/backend/storage/ipc/procarray.c#L2220-L2223)

Taranan şey PGPROC dizisi değil, **`ProcGlobal->xids[]` yoğun dizisi** — cache
satırı başına daha çok xid. `UINT32_ACCESS_ONCE` zorunlu, çünkü xid'ler
ProcArrayLock olmadan yazılabiliyor
([README:286-294](../src/backend/access/transam/README#L286-L294)).

Dört erken çıkış var:

| Kod | Satır | Neden |
|---|---|---|
| `xid == InvalidTransactionId` | [2232](../src/backend/storage/ipc/procarray.c#L2232) | Xid almamış (salt okunur) backend snapshot'ı etkilemez |
| `pgxactoff == mypgxactoff` | [2240](../src/backend/storage/ipc/procarray.c#L2240) | Kendi xid'im listeye girmez; `TransactionIdIsCurrentTransactionId` halleder |
| `!NormalTransactionIdPrecedes(xid, xmax)` | [2256](../src/backend/storage/ipc/procarray.c#L2256) | `>= xmax` zaten "çalışıyor" sayılacak |
| `statusFlags & (PROC_IN_LOGICAL_DECODING \| PROC_IN_VACUUM)` | [2264](../src/backend/storage/ipc/procarray.c#L2264) | Aşağıda |

```c
			statusFlags = allStatusFlags[pgxactoff];
			if (statusFlags & (PROC_IN_LOGICAL_DECODING | PROC_IN_VACUUM))
				continue;
```
— [../src/backend/storage/ipc/procarray.c:2263-2265](../src/backend/storage/ipc/procarray.c#L2263-L2265)

**Sonuncusu kritik.** `PROC_IN_VACUUM` taşıyan bir lazy VACUUM başkalarının
snapshot'ında görünmez: işi MVCC görünürlüğüne tabi değil ve saatlerce sürebilir
— snapshot'ta dursa herkesin xmin'ini geriye çakardı.

Subxid'ler `memcpy` ile toplu kopyalanıyor; bir kez overflow görüldü mü daha fazla
kopyalama yapılmaz, zaten `pg_subtrans`'a inilecek
([satır 2288-2306](../src/backend/storage/ipc/procarray.c#L2288-L2306)).

## 3.3. Sonda ilan edilen `xmin`

```c
	if (!TransactionIdIsValid(MyProc->xmin))
		MyProc->xmin = TransactionXmin = xmin;

	LWLockRelease(ProcArrayLock);
```
— [../src/backend/storage/ipc/procarray.c:2360-2363](../src/backend/storage/ipc/procarray.c#L2360-L2363)

`if` şartı önemli: transaction içinde ikinci snapshot alınırsa `MyProc->xmin`
**ileri gitmez**, çünkü ilk snapshot hâlâ kullanımda olabilir ve gördüğü
satırların silinmemesi gerekir. Geri düşürme ayrıca yapılır (bölüm 8.3).

## 3.4. Neden shared lock yetiyor?

README'nin formülasyonu: *"bir snapshot alınırken hiçbir transaction çalışanlar
kümesinden çıkamaz"*.

```
implementation of this is that GetSnapshotData takes the ProcArrayLock in
shared mode (so that multiple backends can take snapshots in parallel),
but ProcArrayEndTransaction must take the ProcArrayLock in exclusive mode
while clearing the ProcGlobal->xids[] entry at transaction end
```
— [../src/backend/access/transam/README:251-255](../src/backend/access/transam/README#L251-L255)

Okuma paralel, çıkış seri. Bu asimetri bölüm 4 ve 9'un sebebi.

---

# 4. `GetSnapshotDataReuse` — hiç taramadan snapshot

[../src/backend/storage/ipc/procarray.c:2034](../src/backend/storage/ipc/procarray.c#L2034).
`GetSnapshotData` ilk iş olarak buna soruyor
([satır 2178-2184](../src/backend/storage/ipc/procarray.c#L2178-L2184)). Kontrol
iki satır:

```c
	curXactCompletionCount = TransamVariables->xactCompletionCount;
	if (curXactCompletionCount != snapshot->snapXactCompletionCount)
		return false;
```
— [../src/backend/storage/ipc/procarray.c:2043-2045](../src/backend/storage/ipc/procarray.c#L2043-L2045)

`xactCompletionCount`, xid almış bir top-level transaction her bittiğinde artan
global sayaç ([../src/include/access/transam.h:241-248](../src/include/access/transam.h#L241-L248)).
ProcArrayLock exclusive tutulurken artırılıyor:

```c
	/* Also advance global latestCompletedXid while holding the lock */
	MaintainLatestCompletedXid(latestXid);

	/* Same with xactCompletionCount  */
	TransamVariables->xactCompletionCount++;
```
— [../src/backend/storage/ipc/procarray.c:764-768](../src/backend/storage/ipc/procarray.c#L764-L768)

**Neden işe yarıyor?** Kodun kendi gerekçesi:

```
	 * As explained in transam/README, the set of xids considered running by
	 * GetSnapshotData() cannot change while ProcArrayLock is held. Snapshot
	 * contents only depend on transactions with xids and xactCompletionCount
	 * is incremented whenever a transaction with an xid finishes (while
	 * holding ProcArrayLock exclusively). Thus the xactCompletionCount check
	 * ensures we would detect if the snapshot would have changed.
```
— [../src/backend/storage/ipc/procarray.c:2053-2058](../src/backend/storage/ipc/procarray.c#L2053-L2058)

Snapshot'ın içeriği sadece **biten** transaction'lara bağlı; yeni başlayanlar
zaten `>= xmax` olur ve listeye girmez. Sayaç değişmediyse tarama yapılsa da aynı
sonuç çıkardı. Geriye sadece kendi durumumu tazelemek kalıyor
([satır 2067-2076](../src/backend/storage/ipc/procarray.c#L2067-L2076)).

**Pratik etkisi:** okuma ağırlıklı yüklerde ardışık `SELECT`'ler arasında hiçbir
yazan transaction bitmez; her `SELECT` için ProcArray taranmaz, sadece bir
`uint64` karşılaştırılır. Yüzlerce bağlantıda bu, ProcArrayLock üzerindeki cache
satırı trafiğini ciddi biçimde düşürür.

---

# 5. READ COMMITTED ile REPEATABLE READ arasındaki tek fark

`GetTransactionSnapshot` ([../src/backend/utils/time/snapmgr.c:272](../src/backend/utils/time/snapmgr.c#L272))
içinde fark tek bir `if`:

```c
	if (IsolationUsesXactSnapshot())
		return CurrentSnapshot;

	/* Don't allow catalog snapshot to be older than xact snapshot. */
	InvalidateCatalogSnapshot();

	CurrentSnapshot = GetSnapshotData(&CurrentSnapshotData);
```
— [../src/backend/utils/time/snapmgr.c:337-343](../src/backend/utils/time/snapmgr.c#L337-L343)

Makronun tamamı bir karşılaştırma:

```c
#define IsolationUsesXactSnapshot() (XactIsoLevel >= XACT_REPEATABLE_READ)
```
— [../src/include/access/xact.h:52](../src/include/access/xact.h#L52)

İlk snapshot alınırken de fark var: RR/SERIALIZABLE'da kopya alınıp kaydediliyor,
çünkü `CurrentSnapshotData` statik bir tampon ve üzerine yazılabilir. Kopya
`FirstXactSnapshot`'a bağlanıp pairing heap'e ekleniyor
([satır 316-329](../src/backend/utils/time/snapmgr.c#L316-L329)).

**`GetLatestSnapshot`** ([satır 354](../src/backend/utils/time/snapmgr.c#L354))
RR içinde bile taze snapshot verir:

```c
	/* If first call in transaction, go ahead and set the xact snapshot */
	if (!FirstSnapshotSet)
		return GetTransactionSnapshot();

	SecondarySnapshot = GetSnapshotData(&SecondarySnapshotData);
```
— [../src/backend/utils/time/snapmgr.c:370-374](../src/backend/utils/time/snapmgr.c#L370-L374)

Kullanıcısı referential integrity kontrolleri: bir FK doğrulaması eski snapshot'la
yapılırsa araya giren silme kaçırılır.

**`CatalogSnapshot` üçüncü kanal.** Katalog taramaları kullanıcı snapshot'ını
kullanmaz; ayrı bir snapshot var ve katalog değişikliğinde geçersizleniyor
([satır 421-438](../src/backend/utils/time/snapmgr.c#L421-L438)). Bu yüzden
REPEATABLE READ içinde bile `ALTER TABLE` sonrası yeni sütun tanımı görünür —
veri snapshot'ı donmuş olsa da katalog snapshot'ı tazelenir. Pairing heap'e elle
ekleniyor ki xmin hesabında sayılsın.

**Snapshot yığını.** Sorgu başlarken snapshot yığına itilir
([../src/backend/tcop/postgres.c:1184-1188](../src/backend/tcop/postgres.c#L1184-L1188),
portal yolunda [../src/backend/tcop/pquery.c:471-477](../src/backend/tcop/pquery.c#L471-L477)).
`HeapTupleSatisfiesMVCC` bunu Assert ile zorunlu tutuyor:

```c
	Assert(snapshot->regd_count > 0 || snapshot->active_count > 0);
```
— [../src/backend/access/heap/heapam_visibility.c:951](../src/backend/access/heap/heapam_visibility.c#L951)

Kayıtsız bir snapshot kullanımda geçersizlenebilir; bu Assert onu debug
derlemelerinde yakalar.

---

# 6. `HeapTupleSatisfiesMVCC` — karar ağacı

[../src/backend/access/heap/heapam_visibility.c:939](../src/backend/access/heap/heapam_visibility.c#L939).
İki soru sırayla: **yazan görünür mü**, sonra **silen görünür mü**.

```
      HeapTupleSatisfiesMVCC(tuple, snapshot)
                    │
     ┌──────────────┴──────────────┐
     │ HEAP_XMIN_COMMITTED set?    │
     └──────┬───────────────┬──────┘
         hayır            evet
            │               └─► frozen değil && XidInMVCCSnapshot? → GÖRÜNMEZ
     ┌──────▼──────────────────────────┐
     │ HEAP_XMIN_INVALID? → GÖRÜNMEZ   │
     │ benim xid'im?      → curcid ile │
     │ XidInMVCCSnapshot? → GÖRÜNMEZ   │
     │ TransactionIdDidCommit?         │
     │    evet → HEAP_XMIN_COMMITTED   │ ◄── hint bit YAZILIR
     │    hayır→ HEAP_XMIN_INVALID     │ ◄── hint bit YAZILIR
     └──────┬──────────────────────────┘
            │  (yazan görünür)
     ┌──────▼──────────────────────────┐
     │ HEAP_XMAX_INVALID? → GÖRÜNÜR    │
     │ LOCKED_ONLY?       → GÖRÜNÜR    │
     │ IS_MULTI?          → update xid │
     │ sonra aynı sıra tekrar          │
     └─────────────────────────────────┘
```

## 6.1. `t_xmin` dalı

Kendi yazdığım satır için CLOG'a gerek yok — `curcid` yeter
([satır 963-966](../src/backend/access/heap/heapam_visibility.c#L963-L966)).
Başkasınınki için sıra:

```c
		else if (XidInMVCCSnapshot(HeapTupleHeaderGetRawXmin(tuple), snapshot))
			return false;
		else if (TransactionIdDidCommit(HeapTupleHeaderGetRawXmin(tuple)))
			SetHintBitsExt(tuple, buffer, HEAP_XMIN_COMMITTED,
						   HeapTupleHeaderGetRawXmin(tuple), state);
		else
		{
			/* it must have aborted or crashed */
			SetHintBitsExt(tuple, buffer, HEAP_XMIN_INVALID,
						   InvalidTransactionId, state);
			return false;
		}
```
— [../src/backend/access/heap/heapam_visibility.c:1005-1016](../src/backend/access/heap/heapam_visibility.c#L1005-L1016)

**Önce snapshot, sonra CLOG.** Dosya baş yorumu bunun neden zorunlu olduğunu
anlatıyor: tersi yapılırsa commit etmiş ama snapshot'ta çalışan görünen bir
transaction'ın satırı görünür hale gelir ve uygulama seviyesinde tutarsızlık
doğar ([satır 13-27](../src/backend/access/heap/heapam_visibility.c#L13-L27)).

Hint bit zaten set ise dal kısa — ve donmuş tuple bu kontrolü de atlar:

```c
		/* xmin is committed, but maybe not according to our snapshot */
		if (!HeapTupleHeaderXminFrozen(tuple) &&
			XidInMVCCSnapshot(HeapTupleHeaderGetRawXmin(tuple), snapshot))
			return false;		/* treat as still in progress */
```
— [../src/backend/access/heap/heapam_visibility.c:1020-1023](../src/backend/access/heap/heapam_visibility.c#L1020-L1023)

`HEAP_XMIN_FROZEN`, `COMMITTED` ve `INVALID` bit'lerinin birlikte set olması
demek ([../src/include/access/htup_details.h:206](../src/include/access/htup_details.h#L206))
ve "herkese görünür" anlamına gelir. VACUUM'un freeze işi tam olarak bu satırı
kısa devre yaptırmak için ([VACUUM-NASIL-CALISIR.md](VACUUM-NASIL-CALISIR.md)).

## 6.2. `t_xmax` dalı

```c
	/* by here, the inserting transaction has committed */

	if (tuple->t_infomask & HEAP_XMAX_INVALID)	/* xid invalid or aborted */
		return true;

	if (HEAP_XMAX_IS_LOCKED_ONLY(tuple->t_infomask))
		return true;
```
— [../src/backend/access/heap/heapam_visibility.c:1026-1032](../src/backend/access/heap/heapam_visibility.c#L1026-L1032)

`SELECT ... FOR UPDATE` de `t_xmax` yazar ama silmez; `LOCKED_ONLY` bu ayrımı
yapar. `HEAP_XMAX_IS_MULTI` durumunda gerçek silen xid MultiXact'ten çıkarılır
([satır 1034-1059](../src/backend/access/heap/heapam_visibility.c#L1034-L1059);
ayrıntısı [TRANSACTION-YONETIMI.md](TRANSACTION-YONETIMI.md) bölüm 8). Kalan
durumda `t_xmin` ile birebir aynı sıra tekrarlanır
([satır 1061-1091](../src/backend/access/heap/heapam_visibility.c#L1061-L1091)).

## 6.3. `XidInMVCCSnapshot` — asıl arama

[../src/backend/utils/time/snapmgr.c:1868](../src/backend/utils/time/snapmgr.c#L1868).
Önce iki karşılaştırma:

```c
	/* Any xid < xmin is not in-progress */
	if (TransactionIdPrecedes(xid, snapshot->xmin))
		return false;
	/* Any xid >= xmax is in-progress */
	if (TransactionIdFollowsOrEquals(xid, snapshot->xmax))
		return true;
```
— [../src/backend/utils/time/snapmgr.c:1878-1883](../src/backend/utils/time/snapmgr.c#L1878-L1883)

Sonra dizi taraması (`pg_lfind32`, SIMD kullanan doğrusal arama) — ya da overflow
varsa `pg_subtrans`:

```c
		if (!snapshot->suboverflowed)
		{
			/* we have full data, so search subxip */
			if (pg_lfind32(xid, snapshot->subxip, snapshot->subxcnt))
				return true;
		}
		else
		{
			xid = SubTransGetTopmostTransaction(xid);
```
— [../src/backend/utils/time/snapmgr.c:1898-1912](../src/backend/utils/time/snapmgr.c#L1898-L1912)

**Overflow'un maliyeti tam burada:** `SubTransGetTopmostTransaction`,
`pg_subtrans` SLRU'suna disk erişimi anlamına gelebilir — hem de *her tuple için*.
Binlerce SAVEPOINT açan bir uygulamada tam taramanın neden çöktüğünün cevabı bu.

---

# 7. Hint bit'ler — okuma neden yazma üretir

```c
#define HEAP_XMIN_COMMITTED		0x0100	/* t_xmin committed */
#define HEAP_XMIN_INVALID		0x0200	/* t_xmin invalid/aborted */
#define HEAP_XMIN_FROZEN		(HEAP_XMIN_COMMITTED|HEAP_XMIN_INVALID)
#define HEAP_XMAX_COMMITTED		0x0400	/* t_xmax committed */
#define HEAP_XMAX_INVALID		0x0800	/* t_xmax invalid/aborted */
```
— [../src/include/access/htup_details.h:204-208](../src/include/access/htup_details.h#L204-L208)

## 7.1. Neden ilk okuma sayfayı kirletir

Bir `INSERT` commit ettiğinde tuple'ın hint bit'i **set edilmez** — commit anında
sayfa buffer'da olmayabilir, hepsini dolaşmak pahalıdır. Bit'i ilk okuyan yazar
ve buffer'ı kirletir
([../src/backend/access/heap/heapam_visibility.c:174-179](../src/backend/access/heap/heapam_visibility.c#L174-L179)).
`MarkBufferDirtyHint` normal kirletmeden farklı, sonuç aynı:

```
 *	Mark a buffer dirty for non-critical changes.
 *
 * 1. The caller does not write WAL; so if checksums are enabled, we may need
 *	  to write an XLOG_FPI_FOR_HINT WAL record to protect against torn pages.
```
— [../src/backend/storage/buffer/bufmgr.c:5831-5836](../src/backend/storage/buffer/bufmgr.c#L5831-L5836)

**Pratik sonuç:** büyük bir `INSERT`'ten sonraki ilk `SELECT count(*)` beklenenden
yavaştır ve I/O üretir. `data_checksums` açıksa ayrıca full page image WAL kaydı
doğar. İkinci `SELECT` hızlıdır.

## 7.2. LSN kilidi — hint bit ne zaman yazılamaz

```c
		if (BufferIsPermanent(buffer))
		{
			/* NB: xid must be known committed here! */
			XLogRecPtr	commitLSN = TransactionIdGetCommitLSN(xid);

			if (XLogNeedsFlush(commitLSN) &&
				BufferGetLSNAtomic(buffer) < commitLSN)
			{
				/* not flushed and no LSN interlock, so don't set hint */
				return;
			}
		}
```
— [../src/backend/access/heap/heapam_visibility.c:154-165](../src/backend/access/heap/heapam_visibility.c#L154-L165)

Senaryo: async commit yapan bir transaction'ın commit WAL kaydı henüz diskte
değil. Hint bit'i yazıp sayfayı diske indirir ve sonra çökersek, kurtarma sonrası
sayfada "commit etti" yazacak ama CLOG'da o transaction yok. Abort için bu kısıt
yok ([satır 124-125](../src/backend/access/heap/heapam_visibility.c#L124-L125)).

## 7.3. Toplu mod

20devel'de sayfa başına izin alma mekanizması var; sebebi bir sayfa üzerinde IO
sürerken hint bit yazmanın checksum'u bozabilmesi
([satır 106-108](../src/backend/access/heap/heapam_visibility.c#L106-L108)):

```c
typedef enum SetHintBitsState
{
	SHB_INITIAL,	/* not yet checked if hint bits may be set */
	SHB_DISABLED,	/* failed to get permission, don't check again */
	SHB_ENABLED,	/* allowed to set hint bits */
} SetHintBitsState;
```
— [../src/backend/access/heap/heapam_visibility.c:91-99](../src/backend/access/heap/heapam_visibility.c#L91-L99)

Seq scan bir sayfadaki tüm tuple'ları tek geçişte kontrol ediyor ve izni bir kez
alıyor ([satır 1705-1716](../src/backend/access/heap/heapam_visibility.c#L1705-L1716)).

---

# 8. xmin horizon — VACUUM neyi silebilir

## 8.1. Dört ayrı horizon

`ComputeXidHorizons` tek sayı değil, bir demet hesaplıyor
([../src/backend/storage/ipc/procarray.c:1674](../src/backend/storage/ipc/procarray.c#L1674)).
Başlangıç `latestCompletedXid + 1`
([satır 1697-1703](../src/backend/storage/ipc/procarray.c#L1697-L1703)), sonra
ProcArray taranıp minimum alınıyor:

```c
		xid = UINT32_ACCESS_ONCE(other_xids[index]);
		xmin = UINT32_ACCESS_ONCE(proc->xmin);
		...
		xmin = TransactionIdOlder(xmin, xid);
```
— [../src/backend/storage/ipc/procarray.c:1740-1750](../src/backend/storage/ipc/procarray.c#L1740-L1750)

İkisine birden bakmak zorunlu: bir transaction xmin'e sahip olup henüz xid
almamış olabilir, ya da tersi
([satır 1743-1748](../src/backend/storage/ipc/procarray.c#L1743-L1748)). Filtreler
`PROC_IN_VACUUM`/`PROC_IN_LOGICAL_DECODING`
([satır 1770-1771](../src/backend/storage/ipc/procarray.c#L1770-L1771)) ve
veritabanı eşleşmesi
([satır 1796-1803](../src/backend/storage/ipc/procarray.c#L1796-L1803)).

| Horizon | Kimi dikkate alır |
|---|---|
| `shared_oldest_nonremovable` | Tüm veritabanlarındaki backend'ler + slot xmin + slot catalog_xmin |
| `catalog_oldest_nonremovable` | Kendi veritabanım + slot xmin + slot catalog_xmin |
| `data_oldest_nonremovable` | Kendi veritabanım + slot xmin |
| `temp_oldest_nonremovable` | Sadece bu oturum |

**Ayrımın kazancı somut:** başka bir veritabanındaki uzun sorgu, benim
veritabanımdaki normal tabloların vacuum'unu engellemez — ama `pg_authid` gibi
paylaşımlı bir katalogunkini engeller. VACUUM'un aldığı değer ilişkinin türüne
göre seçiliyor:

```c
	switch (GlobalVisHorizonKindForRel(rel))
	{
		case VISHORIZON_SHARED:
			return horizons.shared_oldest_nonremovable;
		case VISHORIZON_CATALOG:
			return horizons.catalog_oldest_nonremovable;
		case VISHORIZON_DATA:
			return horizons.data_oldest_nonremovable;
		case VISHORIZON_TEMP:
			return horizons.temp_oldest_nonremovable;
	}
```
— [../src/backend/storage/ipc/procarray.c:1950-1960](../src/backend/storage/ipc/procarray.c#L1950-L1960)

## 8.2. `GlobalVisState` — yaklaşık sınırlar

`GetSnapshotData` bu hesabı **yapmaz** — çok pahalı olurdu. İki kaba sınır
günceller:

```c
struct GlobalVisState
{
	/* XIDs >= are considered running by some backend */
	FullTransactionId definitely_needed;

	/* XIDs < are not considered to be running by any backend */
	FullTransactionId maybe_needed;
};
```
— [../src/backend/storage/ipc/procarray.c:184-191](../src/backend/storage/ipc/procarray.c#L184-L191)

Aradaki gri bölgeye düşen bir xid sorulursa `ComputeXidHorizons` yeniden
çalıştırılıp kesin cevap alınır
([satır 136-139](../src/backend/storage/ipc/procarray.c#L136-L139)). README bu
değişikliği tarihiyle veriyor:

```
As GetSnapshotData is performance critical, it does not perform an accurate
oldest-xmin calculation (it used to, until v14).
```
— [../src/backend/access/transam/README:320-321](../src/backend/access/transam/README#L320-L321)

Sebep: bir backend'in `xmin`'i, `xid`'inden çok daha sık değişir; her snapshot
alışında tüm xmin'leri okumak gereksiz cache satırı ping-pong'u üretiyordu
([README:322-324](../src/backend/access/transam/README#L322-L324)).

## 8.3. `PGPROC.xmin` ne zaman düşer

```c
	if (ActiveSnapshot != NULL)
		return;

	if (pairingheap_is_empty(&RegisteredSnapshots))
	{
		MyProc->xmin = TransactionXmin = InvalidTransactionId;
		return;
	}

	minSnapshot = pairingheap_container(SnapshotData, ph_node,
										pairingheap_first(&RegisteredSnapshots));

	if (TransactionIdPrecedes(MyProc->xmin, minSnapshot->xmin))
		MyProc->xmin = TransactionXmin = minSnapshot->xmin;
```
— [../src/backend/utils/time/snapmgr.c:941-954](../src/backend/utils/time/snapmgr.c#L941-L954)

Kayıtlı snapshot'lar `xmin`'e göre sıralı pairing heap'te tutuluyor
([satır 909-921](../src/backend/utils/time/snapmgr.c#L909-L921)), en eskisi
O(1)'de bulunuyor. Aktif snapshot varken hiç uğraşılmıyor — yorum bunun bilinçli
bir sadeleştirme olduğunu söylüyor
([satır 930-934](../src/backend/utils/time/snapmgr.c#L930-L934)).

## 8.4. Uzun transaction'ın maliyeti

```
  idle in transaction backend
        │  PGPROC.xmin = 10000  (asla düşmez)
        ▼
  ComputeXidHorizons → data_oldest_nonremovable = 10000
        ▼
  VACUUM: xmax >= 10000 olan ölü satırlar SİLİNEMEZ
        ▼
  tablo şişer, index şişer, plan bozulur
```

Transaction'ın **hiçbir şey yapmıyor** olması bir şeyi değiştirmez. Aynı etkiyi
üç şey daha yaratır, hepsi ayrı kanaldan:

1. **Unutulmuş prepared transaction** — 2PC gxact'ı ProcArray'de kalır
   ([TRANSACTION-YONETIMI.md](TRANSACTION-YONETIMI.md) bölüm 7.2).
2. **Replication slot** — horizonu geri çeker
   ([../src/backend/storage/ipc/procarray.c:1838-1841](../src/backend/storage/ipc/procarray.c#L1838-L1841)).
3. **`hot_standby_feedback` açık standby** — standby'deki sorgunun xmin'i
   primary'ye taşınır (bölüm 10.3).

---

# 9. `ProcArrayGroupClearXid` — commit'te grup temizliği

Sorun: `GetSnapshotData` shared lock alır (paralel), ama transaction sonu
**exclusive** ister. Yüzlerce backend aynı anda commit edince ProcArrayLock elden
ele geçen bir darboğaza döner.

Önce iyimser deneme:

```c
		if (LWLockConditionalAcquire(ProcArrayLock, LW_EXCLUSIVE))
		{
			ProcArrayEndTransactionInternal(proc, latestXid);
			LWLockRelease(ProcArrayLock);
		}
		else
			ProcArrayGroupClearXid(proc, latestXid);
```
— [../src/backend/storage/ipc/procarray.c:680-686](../src/backend/storage/ipc/procarray.c#L680-L686)

Kilit hemen alınamazsa backend kendini kilitsiz bir listeye ekler
([../src/backend/storage/ipc/procarray.c:784](../src/backend/storage/ipc/procarray.c#L784)):

```c
	nextidx = pg_atomic_read_u32(&procglobal->procArrayGroupFirst);
	while (true)
	{
		pg_atomic_write_u32(&proc->procArrayGroupNext, nextidx);

		if (pg_atomic_compare_exchange_u32(&procglobal->procArrayGroupFirst,
										   &nextidx,
										   (uint32) pgprocno))
			break;
	}
```
— [../src/backend/storage/ipc/procarray.c:797-806](../src/backend/storage/ipc/procarray.c#L797-L806)

**Listeye ilk giren lider olur**, diğerleri semafor üzerinde uyur
([satır 808-836](../src/backend/storage/ipc/procarray.c#L808-L836)). Lider kilidi
bir kez alır ve listedeki herkesin xid'ini temizler
([satır 838-861](../src/backend/storage/ipc/procarray.c#L838-L861)). Uyandırma
**kilit bırakıldıktan sonra** yapılıyor; sistem çağrıları bellek yazmalarından
çok daha yavaş ([satır 866-872](../src/backend/storage/ipc/procarray.c#L866-L872)).

Ölçülebilir izi `ProcArrayGroupUpdate` wait event'i
([satır 819](../src/backend/storage/ipc/procarray.c#L819)). Çok görünüyorsa commit
hızı ProcArrayLock'a takılıyor demektir. **Salt okunur transaction'lar bu yolun
hiçbirine girmez** — xid'leri olmadığı için kilitsiz çıkarlar
([satır 688-700](../src/backend/storage/ipc/procarray.c#L688-L700)).

---

# 10. Hot standby — `KnownAssignedXids`

## 10.1. Neden ayrı bir mekanizma

Standby'de ProcArray, *standby'nin kendi* backend'lerini tutar; primary'deki
çalışan transaction'lar orada yoktur.

```
 * During hot standby, we also keep a list of XIDs representing transactions
 * that are known to be running on the primary (or more precisely, were running
 * as of the current point in the WAL stream).  This list is kept in the
 * KnownAssignedXids array, and is updated by watching the sequence of
 * arriving XIDs.  This is necessary because if we leave those XIDs out of
 * snapshots taken for standby queries, then they will appear to be already
 * complete, leading to MVCC failures.
```
— [../src/backend/storage/ipc/procarray.c:22-28](../src/backend/storage/ipc/procarray.c#L22-L28)

## 10.2. Standby'de snapshot alma

`GetSnapshotData` recovery'de tamamen farklı bir dala girer:

```c
		subcount = KnownAssignedXidsGetAndSetXmin(snapshot->subxip, &xmin,
												  xmax);

		if (TransactionIdPrecedesOrEquals(xmin, procArray->lastOverflowedXid))
			suboverflowed = true;
```
— [../src/backend/storage/ipc/procarray.c:2344-2348](../src/backend/storage/ipc/procarray.c#L2344-L2348)

**Bütün xid'ler `subxip[]`'e konur, `xip[]` boş kalır.** Sebebi yorumda:

```
		 * In recovery we don't know which xids are top-level and which are
		 * subxacts, a design choice that greatly simplifies xid processing.
		 ...
		 * A simpler
		 * way is to just store all xids in the subxip array because this is
		 * by far the bigger array. We just leave the xip array empty.
```
— [../src/backend/storage/ipc/procarray.c:2320-2332](../src/backend/storage/ipc/procarray.c#L2320-L2332)

`XidInMVCCSnapshot` bu yüzden `takenDuringRecovery` bayrağına bakıp farklı arama
yapıyor ([../src/backend/utils/time/snapmgr.c:1926-1959](../src/backend/utils/time/snapmgr.c#L1926-L1959)).
Dizi sıralı tutulduğu için en eski xid ilk elemandır
([../src/backend/storage/ipc/procarray.c:5194-5200](../src/backend/storage/ipc/procarray.c#L5194-L5200)).

## 10.3. Kurulum — dört durumlu makine

```c
	STANDBY_DISABLED,
	STANDBY_INITIALIZED,
	STANDBY_SNAPSHOT_PENDING,
	STANDBY_SNAPSHOT_READY,
```
— [../src/include/access/xlogutils.h:52-55](../src/include/access/xlogutils.h#L52-L55)

Primary düzenli olarak `XLOG_RUNNING_XACTS` kaydı yazar
([../src/backend/storage/ipc/standby.c:1284](../src/backend/storage/ipc/standby.c#L1284),
`XLogInsert` [satır 1378](../src/backend/storage/ipc/standby.c#L1378)); standby
bunu `ProcArrayApplyRecoveryInfo` ile işler
([../src/backend/storage/ipc/procarray.c:1045](../src/backend/storage/ipc/procarray.c#L1045)).

Kritik durum: gelen ilk kayıt **overflow'lu** ise standby sorgu kabul edemez,
çünkü eksik subxid var:

```c
			if (TransactionIdPrecedes(standbySnapshotPendingXmin,
									  running->oldestRunningXid))
			{
				standbyState = STANDBY_SNAPSHOT_READY;
				elog(DEBUG1,
					 "recovery snapshots are now enabled");
			}
```
— [../src/backend/storage/ipc/procarray.c:1111-1117](../src/backend/storage/ipc/procarray.c#L1111-L1117)

Log'da `"recovery snapshot waiting for non-overflowed snapshot"` görüyorsan
standby, primary'deki SAVEPOINT bolluğu yüzünden açılamıyor demektir
([satır 1119-1123](../src/backend/storage/ipc/procarray.c#L1119-L1123)).

## 10.4. `hot_standby_feedback` — xmin'i geri taşımak

Standby'deki uzun sorgu, primary'de vacuum edilmiş bir satıra ihtiyaç duyabilir.
İki çözüm var, ikisi de bedava değil.

**Çözüm A: feedback.** Standby kendi horizon'unu primary'ye bildirir:

```c
	if (hot_standby_feedback)
	{
		GetReplicationHorizons(&xmin, &catalog_xmin);
	}
	else
	{
		xmin = InvalidTransactionId;
		catalog_xmin = InvalidTransactionId;
	}
```
— [../src/backend/replication/walreceiver.c:1328-1336](../src/backend/replication/walreceiver.c#L1328-L1336)

Gönderilen değer slot catalog_xmin'in etkisi *çıkarılmış* hali, çünkü katalog
için ayrı alan gönderiliyor
([../src/backend/storage/ipc/procarray.c:1998-1999](../src/backend/storage/ipc/procarray.c#L1998-L1999)).
**Bedeli:** standby'deki bir `idle in transaction`, primary'nin tablolarını
şişirir. Sorun standby'de görünür, şişme primary'de olur.

**Çözüm B: çatışmayı kabul et.** Feedback kapalıysa primary vacuum eder, standby
replay ederken çakışan sorguyu iptal eder:

```c
	backends = GetConflictingVirtualXIDs(snapshotConflictHorizon,
										 locator.dbOid);
	ResolveRecoveryConflictWithVirtualXIDs(backends,
										   RECOVERY_CONFLICT_SNAPSHOT,
```
— [../src/backend/storage/ipc/standby.c:492-495](../src/backend/storage/ipc/standby.c#L492-L495)

Ne kadar bekleneceğini GUC belirler
([../src/backend/storage/ipc/standby.c:41-42](../src/backend/storage/ipc/standby.c#L41-L42)):

```c
	if (fromStream)
	{
		if (max_standby_streaming_delay < 0)
			return 0;			/* wait forever */
		return TimestampTzPlusMilliseconds(rtime, max_standby_streaming_delay);
	}
```
— [../src/backend/storage/ipc/standby.c:212-217](../src/backend/storage/ipc/standby.c#L212-L217)

Süre dolunca sorgu `ERROR: canceling statement due to conflict with recovery`
alır. Değer `-1` ise replay sonsuza kadar bekler — bu sefer standby geri kalır.

**Üçgen:** primary'de şişme yok / standby'de iptal yok / standby'de gecikme yok.
Üçünü birden alamazsın.

---

# 11. Snapshot dışa aktarma — paralel pg_dump'ın temeli

## 11.1. `pg_export_snapshot`

Snapshot bir dosyaya metin olarak yazılıyor
([../src/backend/utils/time/snapmgr.c:1115](../src/backend/utils/time/snapmgr.c#L1115)):

```c
	appendStringInfo(&buf, "vxid:%d/%u\n", MyProc->vxid.procNumber, MyProc->vxid.lxid);
	appendStringInfo(&buf, "pid:%d\n", MyProcPid);
	appendStringInfo(&buf, "dbid:%u\n", MyDatabaseId);
	appendStringInfo(&buf, "iso:%d\n", XactIsoLevel);
	appendStringInfo(&buf, "ro:%d\n", XactReadOnly);

	appendStringInfo(&buf, "xmin:%u\n", snapshot->xmin);
	appendStringInfo(&buf, "xmax:%u\n", snapshot->xmax);
```
— [../src/backend/utils/time/snapmgr.c:1197-1204](../src/backend/utils/time/snapmgr.c#L1197-L1204)

İhraç eden kendi xid'ini de listeye ekliyor, çünkü `GetSnapshotData` onu asla
koymaz ve ithal eden çalışırken ihraç eden hâlâ çalışıyor olacak
([satır 1217-1218](../src/backend/utils/time/snapmgr.c#L1217-L1218)). Snapshot
kendi transaction'ında kayıtlı tutulur ki xmin'i korunsun
([satır 1187-1188](../src/backend/utils/time/snapmgr.c#L1187-L1188)).
Subtransaction'dan ihraç yasak — ithal eden onun hâlâ çalışıp çalışmadığını
doğrulayamaz ([satır 1152-1155](../src/backend/utils/time/snapmgr.c#L1152-L1155)).

## 11.2. `SET TRANSACTION SNAPSHOT`

`ImportSnapshot` ([../src/backend/utils/time/snapmgr.c:1386](../src/backend/utils/time/snapmgr.c#L1386))
üç kontrol yapıyor: alan doğrulaması
([satır 1524-1530](../src/backend/utils/time/snapmgr.c#L1524-L1530)), serializable
uyumu ([satır 1538-1548](../src/backend/utils/time/snapmgr.c#L1538-L1548)) ve
veritabanı eşleşmesi:

```c
	if (src_dbid != MyDatabaseId)
		ereport(ERROR,
				(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
				 errmsg("cannot import a snapshot from a different database")));
```
— [../src/backend/utils/time/snapmgr.c:1559-1562](../src/backend/utils/time/snapmgr.c#L1559-L1562)

Sebep tam bu notun konusu: *"vacuum OldestXmin'i veritabanı bazında hesaplar; bu
yüzden kaynak transaction'ın xmin'i bizi veri kaybından korumaz"*
([satır 1551-1557](../src/backend/utils/time/snapmgr.c#L1551-L1557)).

Asıl kritik adım xmin'in atomik kurulması:

```c
	else if (!ProcArrayInstallImportedXmin(CurrentSnapshot->xmin, sourcevxid))
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
				 errmsg("could not import the requested snapshot"),
```
— [../src/backend/utils/time/snapmgr.c:572-575](../src/backend/utils/time/snapmgr.c#L572-L575)

`ProcArrayInstallImportedXmin` kaynak transaction'ın hâlâ çalıştığını **kilit
altında** doğrular ([../src/backend/storage/ipc/procarray.c:2479](../src/backend/storage/ipc/procarray.c#L2479));
aksi halde global xmin geriye giderdi ve VACUUM'un zaten sildiği satırlar
istenirdi.

## 11.3. `pg_dump -j` bunu nasıl kullanıyor

Lider bağlantı REPEATABLE READ açar ve snapshot'ı ihraç eder
([../src/bin/pg_dump/pg_dump.c:1588](../src/bin/pg_dump/pg_dump.c#L1588)); her
worker aynı token'ı ithal eder:

```c
		appendPQExpBufferStr(query, "SET TRANSACTION SNAPSHOT ");
		appendStringLiteralConn(query, AH->sync_snapshot_id, conn);
```
— [../src/bin/pg_dump/pg_dump.c:1560-1561](../src/bin/pg_dump/pg_dump.c#L1560-L1561)

Sonuç: N ayrı bağlantı, tek tutarlı görüntü. `--snapshot=` seçeneği aynı
mekanizmayı dışarıya açar ([../src/bin/pg_dump/pg_dump.c:1372](../src/bin/pg_dump/pg_dump.c#L1372)).

Paralel plan worker'ları için aynı iş daha ucuz: dosya yerine DSM segmentine
serileştirme ([../src/backend/utils/time/snapmgr.c:1735](../src/backend/utils/time/snapmgr.c#L1735)
ve [:1792](../src/backend/utils/time/snapmgr.c#L1792)), dar bir alan kümesiyle
([satır 251-260](../src/backend/utils/time/snapmgr.c#L251-L260)). Bu yüzden
paralel modda snapshot'ı değiştirmek yasak
([satır 763-764](../src/backend/utils/time/snapmgr.c#L763-L764)).

---

# İzleme ve hata ayıklama

## Xmin horizon'u kim tutuyor — tek sorgu

```sql
SELECT 'backend' AS kaynak, pid::text AS kim, backend_xmin AS xmin,
       now() - xact_start AS yas, state, left(query, 60) AS sorgu
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
UNION ALL
SELECT 'slot', slot_name, xmin, NULL, active::text, plugin
FROM pg_replication_slots
WHERE xmin IS NOT NULL OR catalog_xmin IS NOT NULL
UNION ALL
SELECT 'prepared', gid, transaction::text::xid, now() - prepared, NULL, NULL
FROM pg_prepared_xacts
ORDER BY age(xmin) DESC NULLS LAST;
```

`age(xmin)` en büyük olan satır suçludur. `backend_xmin`
[../src/backend/catalog/system_views.sql:959](../src/backend/catalog/system_views.sql#L959),
slot xmin'leri [satır 1113-1114](../src/backend/catalog/system_views.sql#L1113-L1114).

## Uzun transaction avı

```sql
SELECT pid, now() - xact_start AS xact_yas, now() - state_change AS son_hareket,
       state, backend_xid, backend_xmin, wait_event_type, wait_event,
       left(query, 80)
FROM pg_stat_activity
WHERE state <> 'idle' OR backend_xmin IS NOT NULL
ORDER BY xact_start;
```

| Görünen | Anlamı |
|---|---|
| `backend_xid` **dolu** | Yazmış. CLOG'da yeri var, WAL üretmiş |
| `backend_xid` boş, `backend_xmin` **dolu** | Salt okunur ama snapshot tutuyor — VACUUM'u yine de engelliyor |
| ikisi de boş | Zararsız |
| `idle in transaction` + `backend_xmin` dolu | **Klasik arıza.** `idle_in_transaction_session_timeout` bunun içindir |

## Snapshot'ı canlı görmek

```sql
BEGIN;
SELECT pg_current_snapshot();          -- xmin:xmax:xip_list biçiminde
SELECT * FROM pg_snapshot_xip(pg_current_snapshot());
COMMIT;
```

READ COMMITTED'de iki kez çağırınca `xmax` ilerler; REPEATABLE READ'te sabit
kalır — bölüm 5'teki tek `if`'in doğrudan gözlemi.

## Hint bit etkisini ölçmek

```sql
CREATE TABLE hb_test AS SELECT g FROM generate_series(1, 500000) g;

EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM hb_test;
--   "Buffers: shared hit=... dirtied=2213"  ← dirtied > 0, hint bit yazıldı
CHECKPOINT;
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM hb_test;
--   "dirtied=0"
```

`dirtied` düşüşü bölüm 7.1'in doğrudan kanıtıdır. Bit'lerin kendisine bakmak için:

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT lp, t_xmin, t_xmax,
       (t_infomask & 256)::bool  AS xmin_committed,   -- 0x0100
       (t_infomask & 512)::bool  AS xmin_invalid,     -- 0x0200
       (t_infomask & 1024)::bool AS xmax_committed,   -- 0x0400
       (t_infomask & 2048)::bool AS xmax_invalid      -- 0x0800
FROM heap_page_items(get_raw_page('hb_test', 0));
```

İlk ikisi birlikte true ise tuple donmuştur (`HEAP_XMIN_FROZEN`,
[../src/include/access/htup_details.h:206](../src/include/access/htup_details.h#L206)).

## Standby tarafı

```sql
-- Primary'de: standby'lerin dayattığı xmin
SELECT application_name, state, backend_xmin, age(backend_xmin) AS ne_kadar_geride
FROM pg_stat_replication;

-- Standby'de: biriken çatışmalar
SELECT datname, confl_snapshot, confl_bufferpin, confl_lock, confl_deadlock
FROM pg_stat_database_conflicts WHERE confl_snapshot > 0;
```

`pg_stat_replication.backend_xmin` dolu ise `hot_standby_feedback` açıktır
([../src/backend/catalog/system_views.sql:977](../src/backend/catalog/system_views.sql#L977));
`age()` büyükse primary'de vacuum engelleniyor. Tersine `confl_snapshot`
artıyorsa feedback kapalı ve standby sorguları iptal ediliyor.

## ProcArrayLock darboğazı

```sql
SELECT wait_event, count(*) FROM pg_stat_activity
WHERE wait_event IN ('ProcArrayGroupUpdate', 'ProcArray') GROUP BY 1;
```

`ProcArrayGroupUpdate` yoğunsa bölüm 9'daki lider/takipçi mekanizması sürekli
devrede — commit hızı ProcArrayLock'a takılıyor.

---

# Tek sayfalık özet

```
┌───────────────────────────────────────────────────────────────────────┐
│                      SNAPSHOT NASIL DOĞAR                             │
│  GetTransactionSnapshot()   snapmgr.c:272                             │
│      ├─ RR/SERIALIZABLE + ilk değil → CurrentSnapshot (donmuş)        │
│      └─ diğer                       → GetSnapshotData()  (:2114)      │
│         ┌──────────────────────────────────────────────────┐          │
│         │ ProcArrayLock SHARED                             │          │
│         │  ├─ GetSnapshotDataReuse? xactCompletionCount    │          │
│         │  │     aynı ise → HAZIR, tarama YOK    (:2034)   │          │
│         │  ├─ xmax = latestCompletedXid + 1                │          │
│         │  ├─ ProcArray gez → xip[]                        │          │
│         │  │    atla: xid yok / benim / >=xmax / VACUUM    │          │
│         │  └─ MyProc->xmin = xmin   (ilan et)              │          │
│         └──────────────────────────────────────────────────┘          │
├───────────────────────────────────────────────────────────────────────┤
│                    SNAPSHOT NASIL KULLANILIR                          │
│  HeapTupleSatisfiesMVCC()   heapam_visibility.c:939                   │
│      t_xmin: hint bit? → benim mi? → snapshot? → CLOG → hint YAZ      │
│      t_xmax: aynı sıra                                                │
│                  └─► XidInMVCCSnapshot()   snapmgr.c:1868             │
│                         < xmin  → bitmiş                              │
│                         >= xmax → çalışıyor                           │
│                         arada   → xip[]/subxip[] tara                 │
├───────────────────────────────────────────────────────────────────────┤
│                    SNAPSHOT NEYİ ENGELLER                             │
│  MyProc->xmin ─► ComputeXidHorizons()  procarray.c:1674               │
│                     ├─ shared_oldest_nonremovable   (tüm db'ler)      │
│                     ├─ catalog_oldest_nonremovable  (+ slot catalog)  │
│                     ├─ data_oldest_nonremovable     (kendi db)        │
│                     └─ temp_oldest_nonremovable     (kendi oturum)    │
│                            ▼                                          │
│              GetOldestNonRemovableTransactionId()      (:1944)        │
│                            ▼  VACUUM: bundan eskiyi silebilir         │
│  Horizonu geri çeken dört kanal:                                      │
│    1. uzun transaction (PGPROC.xmin)     3. prepared transaction      │
│    2. replication slot xmin              4. hot_standby_feedback      │
└───────────────────────────────────────────────────────────────────────┘
```

**Akılda kalması gereken 5 şey:**

1. **Snapshot = {xmin, xmax, çalışanların listesi}.** Görünürlük sorusu hep
   "bu xid o anda çalışıyor muydu"ya iner. CLOG "commit etti mi"yi, snapshot
   "ben görmeli miyim"i cevaplar — iki ayrı soru.

2. **Sıra sabittir: hint bit → kendi transaction'ım → snapshot → CLOG.** Bu sıra
   performans değil **doğruluk** meselesi. CLOG'a snapshot'tan önce bakmak,
   commit etmiş ama snapshot'ta çalışan görünen bir transaction'ın satırını
   görünür yapar ([heapam_visibility.c:13-27](../src/backend/access/heap/heapam_visibility.c#L13-L27)).

3. **`xactCompletionCount` sayesinde çoğu snapshot bedava.** Xid'li bir
   transaction bitmediyse ProcArray hiç taranmaz; okuma ağırlıklı yükte bu
   ProcArrayLock trafiğini neredeyse sıfırlar
   ([procarray.c:2043-2058](../src/backend/storage/ipc/procarray.c#L2043-L2058)).

4. **İlk okuma yazma üretir.** Hint bit commit anında değil, ilk okumada set
   edilir. Toplu `INSERT` sonrası ilk `SELECT`'in yavaş ve I/O'lu olmasının,
   `EXPLAIN (BUFFERS)` çıktısında `dirtied` görülmesinin sebebi budur.

5. **Elde tutulan her snapshot VACUUM'a fren yapar.** `PGPROC.xmin` sadece
   `idle in transaction` ile değil; slot, prepared transaction ve
   `hot_standby_feedback` ile de geriye çakılır. Şişme sorununu ararken
   `pg_stat_activity` tek başına yetmez — dört kanalın dördüne birden bak.

---

*Devamı: [VACUUM-NASIL-CALISIR.md](VACUUM-NASIL-CALISIR.md) horizon'un tüketildiği
yeri, [TRANSACTION-YONETIMI.md](TRANSACTION-YONETIMI.md) ise xid'in ve CLOG'un
doğduğu yeri anlatıyor.*
