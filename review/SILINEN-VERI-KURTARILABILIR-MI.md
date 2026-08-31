# Silinen bir veri WAL kayıtlarından çıkarılabilir mi?

> PostgreSQL **20devel** ağacı üzerinde okundu.  Satır numaraları bu sürüme aittir.

Kısa cevap: **evet, ama DELETE kaydından değil.**
`DELETE`'in WAL kaydı silinen satırın içeriğini yazmaz — sadece "şu sayfanın şu
offset'indeki tuple'ın `xmax`'ı şu oldu" der.  Veriyi geri getiren şey başka üç
yerdir: satırın **INSERT kaydı**, WAL'daki **full-page image**'lar ve VACUUM
gelmediyse **heap dosyasının kendisi**.

---

## 30 saniyelik özet

```
      DELETE tablo WHERE id = 42
                │
                ▼
  ┌──────────────────────────────────────────────┐
  │ heap_delete: sayfada sadece xmax + infomask  │  veri hâlâ sayfada!
  │ değişir; tuple'ın byte'ları yerinde durur    │
  └──────────────────────────────────────────────┘
                │
                ▼
  ┌──────────────────────────────────────────────┐
  │ xl_heap_delete = { xmax, offnum, infobits,   │  ← 8 byte. veri YOK.
  │                    flags }                   │
  └──────────────────────────────────────────────┘
                │
       checkpoint'ten sonraki ilk dokunuş ise
                ▼
  ┌──────────────────────────────────────────────┐
  │ FULL-PAGE IMAGE: 8 KB'lık sayfanın tamamı    │  ← ASIL HAZİNE
  │ (silinmiş tuple'ın byte'ları dahil)          │
  └──────────────────────────────────────────────┘

  Ayrıca geçmişte bir yerde:
  ┌──────────────────────────────────────────────┐
  │ xl_heap_insert + xl_heap_header + TUPLE DATA │  ← satırın doğum kaydı
  └──────────────────────────────────────────────┘
```

Model dört maddede:

1. **`DELETE` veriyi silmez, işaretler.**  `heap_delete` sayfada yalnızca
   `xmax`/`infomask` yazar; tuple'ın byte'ları sayfada durmaya devam eder.
   Gerçek silme `VACUUM`'un işi.
2. **`DELETE`'in WAL kaydı içeriği taşımaz.**  `xl_heap_delete` dört alandan
   ibarettir.  İçerik ancak `wal_level = logical` **ve** uygun `REPLICA IDENTITY`
   varsa eklenir.
3. **Ama `INSERT`'in WAL kaydı tuple'ın tamamını taşır.**  Satır ne zaman
   yazıldıysa, o kayıt bütün kolon byte'larını içerir.
4. **Full-page write her şeyi kopyalar.**  Checkpoint'ten sonra bir sayfaya ilk
   dokunuşta 8 KB'lık sayfanın tamamı WAL'a yazılır — o sayfadaki silinmiş
   tuple'lar dahil.  `pg_waldump --save-fullpage` bunları dosyaya çıkarır.

---

## Kaynak haritası

| Yer | Ne yapar |
|---|---|
| [../src/backend/access/heap/heapam.c](../src/backend/access/heap/heapam.c) | `heap_delete`, `heap_insert`, `ExtractReplicaIdentity` — WAL kaydı burada kurulur |
| [../src/include/access/heapam_xlog.h](../src/include/access/heapam_xlog.h) | `xl_heap_delete`, `xl_heap_insert`, `xl_heap_header` yapıları ve bayrakları |
| [../src/backend/access/transam/xloginsert.c](../src/backend/access/transam/xloginsert.c) | Full-page image kararı ve sayfadaki "delik"in atlanması |
| [../src/backend/access/transam/xlog.c](../src/backend/access/transam/xlog.c) | Segment geri dönüştürme — eski WAL'ın ne kadar yaşadığı |
| [../src/bin/pg_waldump/pg_waldump.c](../src/bin/pg_waldump/pg_waldump.c) | `--save-fullpage`, `-R` / `-B` / `-w` filtreleri |
| [../src/backend/access/rmgrdesc/heapdesc.c](../src/backend/access/rmgrdesc/heapdesc.c) | `pg_waldump`'ın ekrana bastığı açıklama metni |
| [../contrib/pageinspect](../contrib/pageinspect) | Ham 8 KB sayfayı satırlara çevirir |

---

## 1. `DELETE` kaydı ne içeriyor: neredeyse hiçbir şey

Yapı dört alan, toplam 8 byte:

```c
typedef struct xl_heap_delete
{
	TransactionId xmax;			/* xmax of the deleted tuple */
	OffsetNumber offnum;		/* deleted tuple's offset */
	uint8		infobits_set;	/* infomask bits */
	uint8		flags;
} xl_heap_delete;
```

[../src/include/access/heapam_xlog.h:118-124](../src/include/access/heapam_xlog.h#L118-L124)

Replay tarafı bu kayıtla sayfayı bulup ilgili offset'teki tuple'ın `xmax`'ını
yazar — çünkü **sayfadaki veri zaten yerinde**, değişen tek şey görünürlük
alanları.  Bu yüzden kayda içerik koymaya gerek yok.

`pg_waldump` çıktısı da bunu doğrular; basılan tek şey `xmax`, `off`, infobit'ler
ve bayraklar:

```c
	else if (info == XLOG_HEAP_DELETE)
	{
		xl_heap_delete *xlrec = (xl_heap_delete *) rec;

		appendStringInfo(buf, "xmax: %u, off: %u, ",
						 xlrec->xmax, xlrec->offnum);
		infobits_desc(buf, xlrec->infobits_set, "infobits");
		appendStringInfo(buf, ", flags: 0x%02X", xlrec->flags);
	}
```

[../src/backend/access/rmgrdesc/heapdesc.c:199-207](../src/backend/access/rmgrdesc/heapdesc.c#L199-L207)

### 1.1 Tek istisna: logical replication

İki bayrak içerik taşındığını söyler:

```c
#define XLH_DELETE_ALL_VISIBLE_CLEARED			(1<<0)
#define XLH_DELETE_CONTAINS_OLD_TUPLE			(1<<1)
#define XLH_DELETE_CONTAINS_OLD_KEY				(1<<2)
```

[../src/include/access/heapam_xlog.h:102-104](../src/include/access/heapam_xlog.h#L102-L104)

Bunlar ancak `ExtractReplicaIdentity` bir tuple döndürürse set edilir:

```c
	old_key_tuple = walLogical ?
		ExtractReplicaIdentity(relation, &tp, true, &old_key_copied) : NULL;
```

[../src/backend/access/heap/heapam.c:3011-3012](../src/backend/access/heap/heapam.c#L3011-L3012)

```c
		if (old_key_tuple != NULL)
		{
			if (relation->rd_rel->relreplident == REPLICA_IDENTITY_FULL)
				xlrec.flags |= XLH_DELETE_CONTAINS_OLD_TUPLE;
			else
				xlrec.flags |= XLH_DELETE_CONTAINS_OLD_KEY;
		}
```

[../src/backend/access/heap/heapam.c:3103-3109](../src/backend/access/heap/heapam.c#L3103-L3109)

`ExtractReplicaIdentity` ilk satırda kapıyı kapatıyor:

```c
	if (!RelationIsLogicallyLogged(relation))
		return NULL;

	if (replident == REPLICA_IDENTITY_NOTHING)
		return NULL;
```

[../src/backend/access/heap/heapam.c:9350-9354](../src/backend/access/heap/heapam.c#L9350-L9354)

Yani varsayılan kurulumda (`wal_level = replica`) **hiçbir DELETE kaydı silinen
satırın içeriğini taşımaz**.  `REPLICA IDENTITY FULL` varsa tuple'ın tamamı,
normal replica identity varsa sadece anahtar kolonlar (gerisi NULL) loglanır:

```c
	for (int i = 0; i < desc->natts; i++)
	{
		if (bms_is_member(i + 1 - FirstLowInvalidHeapAttributeNumber,
						  idattrs))
			Assert(!nulls[i]);
		else
			nulls[i] = true;
	}
```

[../src/backend/access/heap/heapam.c:9394-9401](../src/backend/access/heap/heapam.c#L9394-L9401)

---

## 2. `INSERT` kaydı ne içeriyor: her şey

Satırın doğduğu andaki kayıt bambaşka.  Yapının kendi yorumu söylüyor:

```c
typedef struct xl_heap_insert
{
	OffsetNumber offnum;		/* inserted tuple's offset */
	uint8		flags;

	/* xl_heap_header & TUPLE DATA in HEAP_INSERT_BLKREF_HEAP */
} xl_heap_insert;
```

[../src/include/access/heapam_xlog.h:168-174](../src/include/access/heapam_xlog.h#L168-L174)

Ve `heap_insert` gerçekten tuple'ın gövdesini kayda ekliyor:

```c
		XLogRegisterBufData(HEAP_INSERT_BLKREF_HEAP, &xlhdr,
							SizeOfHeapHeader);
		/* PG73FORMAT: write bitmap [+ padding] [+ oid] + data */
		XLogRegisterBufData(HEAP_INSERT_BLKREF_HEAP,
							(char *) heaptup->t_data + SizeofHeapTupleHeader,
							heaptup->t_len - SizeofHeapTupleHeader);
```

[../src/backend/access/heap/heapam.c:2165-2171](../src/backend/access/heap/heapam.c#L2165-L2171)

**Sonuç:** silinen satırın içeriği, o satırın INSERT'inin WAL kaydında yazılıdır.
INSERT yeterince eskiyse (segment geri dönüştürülmüşse) o kayıt artık `pg_wal`
içinde olmayabilir; ama arşive gidiyorsa (`archive_command` / `pg_receivewal`)
oradadır.

> Not: `XLogRegisterBufData` ile eklenen veri, aynı kayıt full-page image de
> alıyorsa ayrıca saklanmaz — çünkü FPI zaten sayfanın tamamını taşır.
> `RelationIsLogicallyLogged` durumunda `REGBUF_KEEP_DATA` ile ikisi birden
> tutulur ([../src/backend/access/heap/heapam.c:2141-2149](../src/backend/access/heap/heapam.c#L2141-L2149)).
> Her iki durumda da byte'lar WAL'dadır; sadece hangi kutuda oldukları değişir.

---

## 3. Full-page image: asıl hazine

Bir sayfaya checkpoint'ten sonraki **ilk** dokunuşta, o sayfanın 8 KB'lık
kopyası olduğu gibi WAL'a girer (torn page koruması — ayrıntısı için
[WAL-VE-KURTARMA.md](WAL-VE-KURTARMA.md) §2).

Kritik nokta: FPI'dan atlanan tek bölge, sayfanın **boş** kısmıdır:

```c
			if (regbuf->flags & REGBUF_STANDARD)
			{
				/* Assume we can omit data between pd_lower and pd_upper */
				uint16		lower = ((const PageHeaderData *) page)->pd_lower;
				uint16		upper = ((const PageHeaderData *) page)->pd_upper;
```

[../src/backend/access/transam/xloginsert.c:732-736](../src/backend/access/transam/xloginsert.c#L732-L736)

`pd_lower` ile `pd_upper` arasındaki "delik", line pointer'ların bittiği yerle
tuple verisinin başladığı yer arasındaki **boşluktur**.  Silinmiş ama henüz
vacuum'lanmamış tuple'lar `pd_upper`'ın üstünde, yani **kopyalanan bölgededir**.

Bu yüzden bir kayıt full-page image içeriyorsa, o FPI o sayfadaki silinmiş
satırların byte'larını da taşır — DELETE kaydının kendisi taşımasa bile.
(FPI, sayfanın o kayıttan *önceki* hâlidir; DELETE için `xmax` bile henüz
yazılmamıştır.)

---

## 4. Pratik: WAL'dan sayfa çıkarıp okuma

Ağaçtaki iki araç bunu uçtan uca yapıyor.

### 4.1 `pg_waldump --save-fullpage`

```
  --save-fullpage=DIR    save full page images to DIR
```

[../src/bin/pg_waldump/pg_waldump.c:922](../src/bin/pg_waldump/pg_waldump.c#L922)

Kayıttaki her FPI, sıkıştırma açılıp **delik yeniden sıfırlarla doldurularak**
tam 8 KB'lık gerçek bir sayfa olarak diske yazılır:

```c
		/* Full page exists, so let's save it */
		if (!RestoreBlockImage(record, block_id, page))
			pg_fatal("%s", record->errormsg_buf);
		...
		snprintf(filename, MAXPGPATH, "%s/%08X-%08X-%08X.%u.%u.%u.%u%s", savepath,
				 record->seg.ws_tli,
				 LSN_FORMAT_ARGS(record->ReadRecPtr),
				 rnode.spcOid, rnode.dbOid, rnode.relNumber, blk, forkname);
		...
		if (fwrite(page, BLCKSZ, 1, file) != 1)
```

[../src/bin/pg_waldump/pg_waldump.c:626-651](../src/bin/pg_waldump/pg_waldump.c#L626-L651)

Dosya adı `TLI-LSN.spcOid.dbOid.relNumber.blkno_fork` biçiminde — yani hangi
tablonun hangi bloğu olduğu adından okunur.

Filtreleme için `-R` (relation), `-B` (block) ve `-w` / `--fullpage` (yalnızca
FPI içeren kayıtlar) vardır:

```c
		{"fullpage", no_argument, NULL, 'w'},
```

[../src/bin/pg_waldump/pg_waldump.c:949](../src/bin/pg_waldump/pg_waldump.c#L949)

### 4.2 Adım adım

```bash
# 1) Tablonun relfilenode'unu öğren
psql -c "SELECT pg_relation_filepath('musteriler'), relfilenode
         FROM pg_class WHERE relname = 'musteriler';"

# 2) O tabloya ait full-page image'ları çıkar
mkdir /tmp/fpi
pg_waldump -p /var/lib/postgresql/20/main/pg_wal \
           -R 1663/16384/24576 \
           --fullpage --save-fullpage=/tmp/fpi \
           000000010000000000000012 000000010000000000000030
```

Sonra sayfayı `pageinspect` ile aç:

```sql
CREATE EXTENSION pageinspect;

-- ham byte'lar
SELECT lp, lp_off, lp_flags, t_xmin, t_xmax, t_data
FROM heap_page_items(
       pg_read_binary_file('/tmp/fpi/00000001-0000000012ABCDEF.1663.16384.24576.7_main'));

-- kolonlara ayrılmış hâli (tablonun tanımı hâlâ duruyorsa)
SELECT lp, t_xmin, t_xmax, t_attrs
FROM heap_page_item_attrs(
       pg_read_binary_file('/tmp/fpi/...'),
       'musteriler'::regclass);
```

`heap_page_item_attrs(page bytea, rel_oid regclass, do_detoast bool)` imzası:
[../contrib/pageinspect/pageinspect--1.12--1.13.sql:8-11](../contrib/pageinspect/pageinspect--1.12--1.13.sql#L8-L11)

`t_xmax` dolu olan satırlar silinmiş olanlardır.

> `pg_read_binary_file` sunucu tarafı dosya okur; superuser ya da
> `pg_read_server_files` rolü gerekir.

### 4.3 Daha basit yol: VACUUM gelmediyse heap dosyası

Silinen satır sayfada duruyorsa WAL'a hiç gitmeye gerek yok:

```sql
SELECT lp, t_xmin, t_xmax, t_attrs
FROM heap_page_item_attrs(get_raw_page('musteriler', 3), 'musteriler'::regclass)
WHERE t_xmax <> '0'::xid;
```

Bu yalnızca `VACUUM` (veya HOT pruning / sayfa doldurma baskısı) o satırı
temizlemeden önce çalışır — ayrıntısı [VACUUM-NASIL-CALISIR.md](VACUUM-NASIL-CALISIR.md).

---

## 5. Zaman penceresi: veri ne kadar yaşıyor?

Üç ayrı saat işliyor ve **en hızlı olan kazanıyor**.

| Saat | Ne siler | Ne kadar sürer |
|---|---|---|
| `VACUUM` / autovacuum | Heap'teki tuple byte'larını | Dakikalar (autovacuum eşiği) |
| Checkpoint + WAL geri dönüştürme | `pg_wal`'daki eski segmentleri | `max_wal_size` dolana kadar |
| Arşiv saklama politikası | Arşivlenmiş segmentleri | Sizin belirlediğiniz süre |

### 5.1 WAL segmenti silinmez, geri dönüştürülür

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
```

[../src/backend/access/transam/xlog.c:4070-4080](../src/backend/access/transam/xlog.c#L4070-L4080)

`InstallXLogFileSegment` dosyayı **yeniden adlandırır**, içeriğini sıfırlamaz.
Yani geri dönüştürülmüş bir segment, üzerine yeni WAL yazılana kadar eski
byte'ları taşımaya devam eder.  Adli inceleme açısından bu, `pg_waldump`'ın
düzgün okuyamayacağı ama ham arama ile bulunabilecek bir gri bölge yaratır.
(Ayrıntı: [WAL-VE-KURTARMA.md](WAL-VE-KURTARMA.md) §5.5)

---

## 6. Ne zaman hiç çalışmaz

| Durum | Neden |
|---|---|
| `UNLOGGED` tablo | `RelationNeedsWAL` false → hiç WAL yok |
| `wal_level = minimal` + aynı transaction'da `CREATE TABLE` | Aynı makro; WAL atlanır |
| `TRUNCATE` | Yeni relfilenode yaratılır; tuple'lar oraya hiç girmez |
| Büyük değerler (TOAST) | Ana tuple'da sadece pointer var; içerik TOAST tablosunun ayrı WAL kayıtlarında |
| Segment üzerine yeniden yazılmış | Geri dönüştürülen dosya artık gerçekten dolmuş |

```c
#define RelationNeedsWAL(relation)										\
	(RelationIsPermanent(relation) && (XLogIsNeeded() ||				\
	  (relation->rd_createSubid == InvalidSubTransactionId &&			\
	   relation->rd_firstRelfilelocatorSubid == InvalidSubTransactionId)))
```

[../src/include/utils/rel.h:639-642](../src/include/utils/rel.h#L639-L642)

---

## 7. Doğru yöntem: PITR

Yukarıdakiler adli / son çare teknikleridir.  Yanlışlıkla silinen veriyi geri
almanın desteklenen yolu **point-in-time recovery**: base backup + arşivlenmiş
WAL'ı `DELETE`'ten hemen önceki LSN'e kadar oynatmak.

```
recovery_target_lsn = '0/12ABCDEF'   -- pg_waldump ile bulunan DELETE kaydının LSN'i
recovery_target_inclusive = off
```

`pg_waldump` burada iki iş yapar: hangi LSN'de durulacağını bulmak ve FPI
çıkarmak.  Replay mekanizması [WAL-VE-KURTARMA.md](WAL-VE-KURTARMA.md) §7'de.

---

## Tek sayfalık özet

```
 SORU: "silinen satırın içeriği loglarda var mı?"

 ┌─ xl_heap_delete ────────────────────────────────┐
 │ xmax, offnum, infobits, flags       → HAYIR     │
 │   + REPLICA IDENTITY FULL           → EVET      │
 │   + normal replica identity         → sadece PK │
 │     (ikisi de wal_level=logical ister)          │
 └─────────────────────────────────────────────────┘

 ┌─ Full-page image (aynı ya da komşu kayıtta) ────┐
 │ 8 KB sayfanın tamamı, delik hariç   → EVET      │
 │ pd_upper üstündeki ölü tuple'lar dahil          │
 │ çıkar: pg_waldump --save-fullpage               │
 │ oku:   heap_page_item_attrs(page, rel)          │
 └─────────────────────────────────────────────────┘

 ┌─ xl_heap_insert (satırın doğum kaydı) ──────────┐
 │ xl_heap_header + TUPLE DATA         → EVET      │
 │ ama INSERT eskiyse segment geri dönüşmüş olur   │
 └─────────────────────────────────────────────────┘

 ┌─ Heap dosyasının kendisi ───────────────────────┐
 │ VACUUM gelmediyse tuple aynen orada → EVET      │
 │ get_raw_page + heap_page_item_attrs             │
 └─────────────────────────────────────────────────┘

 KOŞUL: tablo permanent + wal_level >= replica
 SAAT:  autovacuum < WAL recycle < arşiv saklama
 DOĞRU YOL: base backup + PITR (recovery_target_lsn)
```
