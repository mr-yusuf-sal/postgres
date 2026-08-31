# TOAST PostgreSQL içinde nasıl çalışır?

*PostgreSQL **20devel** ağacı. Satır numaraları bu sürüme aittir
([meson.build:11](../meson.build#L11)).*

[INSERT-DELETE-NASIL-CALISIR.md](INSERT-DELETE-NASIL-CALISIR.md) `heap_insert`
içindeki tek `if`'i, [UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md) ise
`heap_update`'in TOAST için buffer kilidini bırakmasını anlatıyor. Bu not o
`if`'in arkasına giriyor. TOAST = **T**he **O**versized-**A**ttribute
**S**torage **T**echnique.

---

## 30 saniyelik özet

```
  INSERT ... (metin: 500 KB)
       │   t_len > TOAST_TUPLE_THRESHOLD (~2 KB) ?  → toaster devreye girer
  ┌────────────────────────────────────────────────────────────┐
  │ heap_toast_insert_or_update — dört tur; her turda "kalan   │
  │ en BÜYÜK varlena kolonu bul, ona müdahale et":             │
  │  1: EXTENDED sıkıştır   2: EXTENDED+EXTERNAL dışarı taşı   │
  │  3: MAIN sıkıştır       4: MAIN dışarı taşı (son çare)     │
  └────────────────────────┬───────────────────────────────────┘
                           ▼  dışarı taşınan her değer için
  ┌────────────────────────────────────────────────────────────┐
  │ toast_save_datum: 1996 baytlık chunk'lara böl              │
  │   pg_toast.pg_toast_16384    chunk_id | chunk_seq | data   │
  │   + UNIQUE btree              24601   |  0, 1, 2  | <=1996 │
  └────────────────────────┬───────────────────────────────────┘
                           ▼  ana tabloya yazılan artık 18 bayt:
  varatt_external{ va_rawsize, va_extinfo, va_valueid, va_toastrelid }
```

1. **8 KB sayfa sınırı mutlaktır.** Tuple sayfalar arasına bölünemez; sayfaya en
   az 4 tuple sığsın diye ~2 KB'lık bir eşik konur.
2. **Toaster açgözlüdür, en büyükten başlar** — amaç minimum kolona dokunmak.
3. **Önce sıkıştır, sonra dışarı taşı.** Sıkıştırma satırı sayfada tutar; dışarı
   taşıma bir index araması ekler. `MAIN` bu tercihi abartır.
4. **Dışarı taşınan değer 1996 baytlık chunk'lara bölünür**;
   `(chunk_id, chunk_seq)` üzerinde UNIQUE btree ile aranır.
5. **Ana tabloda kalan 18 baytlık işaretçidir**; `substr()` bu sayede tüm değeri
   okumadan **dilim (slice)** çekebilir — değer sıkışık değilse.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/access/heap/heaptoast.c](../src/backend/access/heap/heaptoast.c) | Karar motoru: dört turluk döngü + chunk okuma. **En önemli dosya** |
| [../src/backend/access/table/toast_helper.c](../src/backend/access/table/toast_helper.c) | Tur adımlarının gövdesi: en büyüğü bul, sıkıştır, dışarı taşı |
| [../src/backend/access/common/toast_internals.c](../src/backend/access/common/toast_internals.c) | Asıl yazma/silme: `toast_save_datum`, `toast_delete_datum`, `toast_compress_datum` |
| [../src/backend/access/common/detoast.c](../src/backend/access/common/detoast.c) | Geri okuma: `detoast_attr`, `detoast_attr_slice` |
| [../src/backend/access/common/toast_compression.c](../src/backend/access/common/toast_compression.c) | pglz ve LZ4 sarmalayıcıları |
| [../src/include/varatt.h](../src/include/varatt.h) | `varlena` bit düzeni, `varatt_external`, `VARATT_*` makroları |
| [../src/include/access/heaptoast.h](../src/include/access/heaptoast.h) | Eşikler: `TOAST_TUPLE_THRESHOLD`, `TOAST_MAX_CHUNK_SIZE` |
| [../src/backend/catalog/toasting.c](../src/backend/catalog/toasting.c) | TOAST tablosunu ve index'ini yaratır |
| [../src/include/access/detoast.h](../src/include/access/detoast.h) | `VARATT_EXTERNAL_GET_POINTER`, `TOAST_POINTER_SIZE` |

---

# 1. Neden var: 8 KB'lık duvar

Heap tuple sayfalar arasına bölünemez; PostgreSQL sayfa boyutuna değil onun
**dörtte birine** göre müdahale eder:

```c
#define TOAST_TUPLES_PER_PAGE	4
#define TOAST_TUPLE_THRESHOLD	MaximumBytesPerTuple(TOAST_TUPLES_PER_PAGE)
#define TOAST_TUPLE_TARGET		TOAST_TUPLE_THRESHOLD
```
— [../src/include/access/heaptoast.h:46-50](../src/include/access/heaptoast.h#L46-L50)

`MaximumBytesPerTuple` sayfaya N tuple sığması için tuple başına azami boyutu
verir ([satır 23-26](../src/include/access/heaptoast.h#L23-L26)); 8 KB blokta
`MAXALIGN_DOWN((8192 − MAXALIGN(24 + 4×4)) / 4)` = **2032**. `THRESHOLD`
"işleme başla", `TARGET` "buraya kadar in" demektir; ikisi şu an aynı ama zorunlu
değil ([satır 32-35](../src/include/access/heaptoast.h#L32-L35)). Hedef tablo
bazında `toast_tuple_target` reloption'ıyla 128–8160 arasına çekilebilir
([../src/backend/access/common/reloptions.c:365-370](../src/backend/access/common/reloptions.c#L365-L370)).

**Her tablonun TOAST tablosu yoktur.** Karar tablo access method'a devredilir
([toasting.c:432-433](../src/backend/catalog/toasting.c#L432-L433)); heap'in
cevabı üç adımlı
([heapam_handler.c:2040-2047](../src/backend/access/heap/heapam_handler.c#L2040-L2047)):
toast'lanabilir kolon yoksa hayır, sınırsız uzunlukta bir kolon varsa evet, hepsi
sınırlıysa en kötü durum uzunluğu eşikle karşılaştırılır — sadece `varchar(10)`
kolonlarından oluşan bir tablo TOAST tablosu almaz. Partitioned tablolar, shared
kataloglar ve initdb sonrası kataloglar da elenir
([407-430](../src/backend/catalog/toasting.c#L407-L430)).

---

# 2. Dört storage strategy

`pg_attribute.attstorage` tek harflik bir stratejidir
([../src/include/catalog/pg_type.h:311-314](../src/include/catalog/pg_type.h#L311-L314)):
`'p'` plain, `'e'` external, `'x'` extended, `'m'` main.

| Strateji | Sıkışır | Dışarı taşınır | Ne zaman |
|---|---|---|---|
| `plain` | hayır | hayır | Sabit boyutlu tipler; varlena olmayan her kolon zorunlu olarak burada |
| `extended` | evet | evet | **Varsayılan.** `text`, `bytea`, `jsonb`, `numeric` |
| `external` | **hayır** | evet | Sıkıştırmadan vazgeç, karşılığında `substr()` dilim okusun (bölüm 8) |
| `main` | evet | son çare | Satırı sayfada tutmaya çalış; başka çare kalmazsa taşı |

`plain` **hiç TOAST'lanmaz** — değer sığmazsa INSERT hata verir; `external`
dışarı taşınır, sadece sıkıştırılmaz.

**`ALTER TABLE ... ALTER COLUMN ... SET STORAGE` sadece bir katalog
güncellemesidir** — `GetAttributeStorage` harfi çözer, `attstorage`
`CatalogTupleUpdate` ile yazılır, mevcut satırlara dokunulmaz
([tablecmds.c:9345-9347](../src/backend/commands/tablecmds.c#L9345-L9347)). Yeni
değer bundan sonraki INSERT/UPDATE'lerde geçerlidir; eski satırlar için
`VACUUM FULL` ya da dummy bir `UPDATE` gerekir. Toast'lanamayan bir tipe `plain`
dışında strateji verilemez
([22822-22846](../src/backend/commands/tablecmds.c#L22822-L22846)).

---

# 3. Giriş kapıları

**INSERT** — `heap_prepare_insert` içinde tek bir `if`:

```c
	else if (HeapTupleHasExternal(tup) || tup->t_len > TOAST_TUPLE_THRESHOLD)
		return heap_toast_insert_or_update(relation, tup, NULL, options);
```
— [../src/backend/access/heap/heapam.c:2264-2265](../src/backend/access/heap/heapam.c#L2264-L2265)

Sadece "büyük" değil, "**zaten dışarıda değer taşıyor**" da tetikleyicidir —
başka bir tablodan gelen TOAST işaretçisi olduğu gibi kopyalanamaz, başka bir
TOAST tablosuna aittir. **UPDATE** aynı fonksiyonu üç koşullu çağırır
([3839-3842](../src/backend/access/heap/heapam.c#L3839-L3842)): eski tuple'ın
dışarıda değeri varsa da çağrılır, çünkü o değerin silinip silinmeyeceğine karar
verilmeli (bölüm 9). TOAST tablolarının kendisi elenir — özyineleme felaket
olurdu ([2260-2262](../src/backend/access/heap/heapam.c#L2260-L2262)).

---

# 4. Karar akışı: `heap_toast_insert_or_update`

[../src/backend/access/heap/heaptoast.c:96](../src/backend/access/heap/heaptoast.c#L96).
Algoritmayı kendi yorumu tarif ediyor:

```
	 *	1: Inline compress attributes with attstorage EXTENDED, and store very
	 *	   large attributes with attstorage EXTENDED or EXTERNAL external
	 *	   immediately
	 *	2: Store attributes with attstorage EXTENDED or EXTERNAL external
	 *	3: Inline compress attributes with attstorage MAIN
	 *	4: Store attributes with attstorage MAIN external
```
— [heaptoast.c:162-167](../src/backend/access/heap/heaptoast.c#L162-L167)

## 4.1 Hedef boyut ve hazırlık

`maxDataLen = RelationGetToastTupleTarget(rel, TOAST_TUPLE_TARGET) - hoff;`
([satır 177](../src/backend/access/heap/heaptoast.c#L177)) — `hoff` başlık +
NULL bitmap, `maxDataLen` arta kalan **veri** bütçesi; dört turun hepsi buna
karşı `heap_compute_data_size()` çağırır.

`toast_tuple_init`
([../src/backend/access/table/toast_helper.c:41](../src/backend/access/table/toast_helper.c#L41))
NULL'ları, varlena olmayanları ve `PLAIN` kolonları `TOASTCOL_IGNORE` ile eler
([satır 111-160](../src/backend/access/table/toast_helper.c#L111-L160)). Tuple'da
duran **yabancı** bir TOAST işaretçisi bulursa geri getirir — ama `PLAIN`
kolonlar dışında `detoast_external_attr` ile, yani **sıkıştırmayı açmadan**
([satır 140-143](../src/backend/access/table/toast_helper.c#L140-L143)); değer
böylece yeniden sıkıştırılmadan yeni TOAST tablosuna yazılabilir.

## 4.2 "En büyük kolon" seçimi

Her turun ilk işi `toast_tuple_find_biggest_attribute`
([../src/backend/access/table/toast_helper.c:181](../src/backend/access/table/toast_helper.c#L181)).
Arama `biggest_size = MAXALIGN(TOAST_POINTER_SIZE)`, yani **24 bayttan** başlar
([187](../src/backend/access/table/toast_helper.c#L187)): 24 bayttan küçük bir
değeri dışarı taşımak boşuna, yerine yazılacak işaretçi zaten o kadar yer tutuyor.
`for_compression` ayrıca `TOASTCOL_INCOMPRESSIBLE` kolonları eler
([191-192](../src/backend/access/table/toast_helper.c#L191-L192)); zaten dışarıda
olan, zaten sıkışık ve `check_main`'e uymayan `attstorage`'lar da atlanır
([198-209](../src/backend/access/table/toast_helper.c#L198-L209)).

## 4.3 Tur 1 — sıkıştır, çok büyükse hemen at

```c
		if (TupleDescAttr(tupleDesc, biggest_attno)->attstorage == TYPSTORAGE_EXTENDED)
			toast_tuple_try_compression(&ttc, biggest_attno);
		else
			toast_attr[biggest_attno].tai_colflags |= TOASTCOL_INCOMPRESSIBLE;
```
— [../src/backend/access/heap/heaptoast.c:196-204](../src/backend/access/heap/heaptoast.c#L196-L204)
(yorumlar çıkarıldı)

`EXTERNAL` kolonlar burada "sıkıştırılamaz" damgası yer. Turun ikinci yarısı bir
kısayoldur — tek bir kolon tek başına bütçeden büyükse hemen dışarı atılır
([215-217](../src/backend/access/heap/heaptoast.c#L215-L217)); yoksa "bir uzun ve
birkaç kısa kolon" senaryosunda diğerleri boşuna sıkıştırılır
([208-213](../src/backend/access/heap/heaptoast.c#L208-L213)).

## 4.4 Tur 2, 3, 4

Tur 2: `EXTENDED` + `EXTERNAL` kolonlar en büyükten başlayarak dışarı taşınır
([satır 220-235](../src/backend/access/heap/heaptoast.c#L220-L235)). Tur 3:
`MAIN` kolonlar sıkıştırılır
([satır 241-251](../src/backend/access/heap/heaptoast.c#L241-L251)). Tur 4 son
çaredir ve **hedefi değiştirir**:
`maxDataLen = TOAST_TUPLE_TARGET_MAIN - hoff;`
([satır 258](../src/backend/access/heap/heaptoast.c#L258)). Bu değer
`MaximumBytesPerTuple(1)` = 8160, yani **tam sayfa**; `MAIN` kolonlar ancak satır
bir sayfaya bile sığmıyorsa taşınır — kullanıcıya verilen sözün tutulması
([heaptoast.h:52-61](../src/include/access/heaptoast.h#L52-L61)). Her turda
`if (biggest_attno < 0) break;` var; iş kalmadığında döngü sessizce biter. Bir
kolon değiştiyse `TOAST_NEEDS_CHANGE` yanar ve tuple sıfırdan kurulur
([277](../src/backend/access/heap/heaptoast.c#L277)); `natts` ve `t_hoff` yeniden
hesaplanır, çünkü `ALTER TABLE ADD COLUMN` sonrası eski tuple'ların natts'ı
farklı olabilir ([288-293](../src/backend/access/heap/heaptoast.c#L288-L293)).

---

# 5. Sıkıştırma: pglz ve LZ4

`toast_compress_datum`
([../src/backend/access/common/toast_internals.c:46](../src/backend/access/common/toast_internals.c#L46))
yöntemi kolonun `attcompression`'ından, geçersizse `default_toast_compression`
GUC'undan alır ([satır 57-59](../src/backend/access/common/toast_internals.c#L57-L59)).
GUC tanımı
[guc_parameters.dat:784-789](../src/backend/utils/misc/guc_parameters.dat#L784-L789);
seçenekler ve varsayılan derleme zamanında belirlenir — LZ4 derlendiyse o kazanır
([../src/backend/utils/misc/guc_tables.c:471-475](../src/backend/utils/misc/guc_tables.c#L471-L475),
[../src/include/access/toast_compression.h:59-63](../src/include/access/toast_compression.h#L59-L63)).

**İki farklı "başarısız" var.** Kütüphane pes eder — pglz girdi çok kısa ya da
çok uzunsa hiç denemez
([../src/backend/access/common/toast_compression.c:52-54](../src/backend/access/common/toast_compression.c#L52-L54)),
LZ4 çıktı girdiden büyükse `NULL` döner
([166-170](../src/backend/access/common/toast_compression.c#L166-L170)). Ya da
PostgreSQL kazancı yeterli bulmaz — kütüphane "başardım" dese bile ikinci
kez ölçülür ve `if (VARSIZE(tmp) < valsize - 2)` şartını geçemeyen sonuç atılıp
`NULL` dönülür
([../src/backend/access/common/toast_internals.c:91-103](../src/backend/access/common/toast_internals.c#L91-L103)).
Gerekçe üstteki yorumda: sıkıştırılmış format 4 baytlık başlık + hizalama dolgusu
gerektirir, sıkıştırılmamış kısa değer 1 baytlık başlıkla yetinir — bu yüzden
**2 bayttan fazla** kazanç şart koşulur
([81-90](../src/backend/access/common/toast_internals.c#L81-L90)). `NULL` dönerse
kolon `TOASTCOL_INCOMPRESSIBLE` damgası yer ve bir daha denenmez
([toast_helper.c:247-248](../src/backend/access/table/toast_helper.c#L247-L248)).
JPEG, ZIP, şifreli veri bu dala düşer; bu tür kolonlar için `EXTERNAL` doğrudur.

**Yöntem nerede saklanır?** Kolonun tanımında değil, **değerin kendisinde**:
sıkıştırılmış inline bir varlena'nın ikinci 4 baytı `va_tcinfo`'dur — üst 2 bit
yöntem, alt 30 bit açılmış boyut
([toast_internals.h:39-46](../src/include/access/toast_internals.h#L39-L46)). Bu
yüzden `default_toast_compression`'ı değiştirmek eski satırları etkilemez; aynı
kolonda pglz ile lz4 değerler yan yana durabilir.

---

# 6. Dışarı taşıma: `toast_save_datum`

[../src/backend/access/common/toast_internals.c:119](../src/backend/access/common/toast_internals.c#L119) —
varlena'yı TOAST tablosuna yaz, 18 baytlık işaretçi döndür. Başlık alanları üç
duruma göre hesaplanır (kısa varlena / sıkışık / düz,
[161-187](../src/backend/access/common/toast_internals.c#L161-L187)); sıkışık
olan en öğreticisi:

```c
		/* rawsize in a compressed datum is just the size of the payload */
		toast_pointer.va_rawsize = VARDATA_COMPRESSED_GET_EXTSIZE(dval) + VARHDRSZ;

		/* set external size and compression method */
		VARATT_EXTERNAL_SET_SIZE_AND_COMPRESS_METHOD(toast_pointer, data_todo,
													 VARDATA_COMPRESSED_GET_COMPRESS_METHOD(dval));
```
— [../src/backend/access/common/toast_internals.c:172-177](../src/backend/access/common/toast_internals.c#L172-L177)

Sıkıştırılmış değer **açılmadan** dışarı yazılır; yöntem `va_extinfo`'nun üst 2
bitine kopyalanır, böylece "sıkışık + dışarıda" değer tek okumada hem getirilir
hem açılır. Value ID, TOAST index'i üzerinden kullanılmayan bir OID olarak seçilir
([217-220](../src/backend/access/common/toast_internals.c#L217-L220)); tablo
yeniden yazılırken (`CLUSTER`, `VACUUM FULL`) eski ID korunur ve aynı değere
işaret eden birden fazla satır versiyonu varsa veri **ikinci kez yazılmaz**
([255-260](../src/backend/access/common/toast_internals.c#L255-L260)).

## 6.1 Chunk döngüsü

```c
	while (data_todo > 0)
	{
		chunk_size = Min(TOAST_MAX_CHUNK_SIZE, data_todo);
		t_values[0] = ObjectIdGetDatum(toast_pointer.va_valueid);
		t_values[1] = Int32GetDatum(chunk_seq++);
		memcpy(VARDATA(&chunk_data), data_p, chunk_size);
		t_values[2] = PointerGetDatum(&chunk_data);
		toasttup = heap_form_tuple(toasttupDesc, t_values, t_isnull);
		heap_insert(toastrel, toasttup, mycid, options, NULL);
```
— [toast_internals.c:283-314](../src/backend/access/common/toast_internals.c#L283-L314)
(yorumlar, boş satırlar ve `SET_VARSIZE` çıkarıldı)

Chunk boyutu sabittir ve yine "sayfaya 4 tuple sığsın" mantığından türetilir:
`TOAST_MAX_CHUNK_SIZE = EXTERN_TUPLE_MAX_SIZE − MAXALIGN(SizeofHeapTupleHeader)
− sizeof(Oid) − sizeof(int32) − VARHDRSZ`
([../src/include/access/heaptoast.h:84-90](../src/include/access/heaptoast.h#L84-L90)),
8 KB blokta `2032 − 24 − 4 − 4 − 4 = **1996**`; değiştirmek `initdb` gerektirir
([78](../src/include/access/heaptoast.h#L78)).

**Chunk'lar `heap_insert` ile yazılır** — TOAST satırları normal MVCC kurallarına
tabidir: kendi `xmin`/`xmax`'ı, kendi WAL kaydı, kendi VACUUM ihtiyacı vardır.
Index girişi elle atılır, `FormIndexDatum` kullanılmaz
([327-336](../src/backend/access/common/toast_internals.c#L327-L336)); birden
fazla index olabilmesinin sebebi `REINDEX CONCURRENTLY`'dir, okuma tarafı
`indisvalid` olanı seçer
([576-586](../src/backend/access/common/toast_internals.c#L576-L586)). Sonunda
`palloc(TOAST_POINTER_SIZE)` + `SET_VARTAG_EXTERNAL(result, VARTAG_ONDISK)` ile
işaretçi kurulur
([362-364](../src/backend/access/common/toast_internals.c#L362-L364)).
`TOAST_POINTER_SIZE` = 2 + 16 = **18 bayt**
([detoast.h:31](../src/include/access/detoast.h#L31)): 1 GB'lık bir `text` de,
3 KB'lık bir `text` de ana tabloda 18 bayt yer kaplar.

## 6.2 TOAST tablosunun şeması

`create_toast_table`
([../src/backend/catalog/toasting.c:128](../src/backend/catalog/toasting.c#L128))
üç kolonluk (`chunk_id` OID, `chunk_seq` int4, `chunk_data` bytea) bir tablo kurar
([205-217](../src/backend/catalog/toasting.c#L205-L217)). Ad **ana tablonun
OID'inden** türetilir (`pg_toast_<relOid>`), adından değil — o yüzden
`ALTER TABLE ... RENAME` TOAST adını değiştirmez
([199-202](../src/backend/catalog/toasting.c#L199-L202)).

TOAST tablosunun kendisinin TOAST'lanmaması hayati; yorum aynen şöyle:
*"Ensure that the toast table doesn't itself get toasted, or we'll be toast :-(.
This is essential for chunk_data because type bytea is toastable"*
([220-222](../src/backend/catalog/toasting.c#L220-L222)). Üç kolonun da
`attstorage`'ı `TYPSTORAGE_PLAIN` yapılır ve sıkıştırma kapatılır
([224-231](../src/backend/catalog/toasting.c#L224-L231)).

Kalıcı tablolar `pg_toast` şemasına (OID 99,
[pg_namespace.dat:18](../src/include/catalog/pg_namespace.dat#L18)), geçici
tablolar backend'e özel geçici toast şemasına gider
([243-246](../src/backend/catalog/toasting.c#L243-L246)). Index
`(chunk_id, chunk_seq)` üzerinde UNIQUE btree'dir
([328-335](../src/backend/catalog/toasting.c#L328-L335)); yorum gerekçeyi veriyor:
tek kolon da işe yarardı ama **dilim erişimi için ikinci kolon şart**, UNIQUE
olması da OID çakışmalarına karşı sigorta
([286-292](../src/backend/catalog/toasting.c#L286-L292)). Bağlantı `reltoastrelid`
ile kurulur ([351](../src/backend/catalog/toasting.c#L351)),
`DEPENDENCY_INTERNAL` ana tabloya bağlar
([393](../src/backend/catalog/toasting.c#L393)).

---

# 7. `varlena` gösterimi

Bir değişken uzunluklu değerin **ilk baytı**, o değerin ne olduğunu söyler
([../src/include/varatt.h:164-169](../src/include/varatt.h#L164-L169), küçük
endian):

| Hâl | Başlık biti | Ne demek |
|---|---|---|
| 4 baytlık, düz | `xxxxxx00` | Sıradan uzun değer, satırın içinde (1 GB'a kadar) |
| 4 baytlık, sıkışık | `xxxxxx10` | Satırın içinde ama sıkışık; 4 bayt daha `va_tcinfo` gelir |
| 1 baytlık | `xxxxxxx1` | **Kısa varlena** — 126 bayta kadar; 3 bayt tasarruf, hizalama yok |
| 1 baytlık, tag `0x01` | `00000001` | **TOAST pointer** — asıl veri başka yerde |

1 baytlık başlık ("short varlena") TOAST'ın az bilinen ama en çok işe yarayan
kısmıdır: kısa `text` değerleri 1 baytlık başlık kullanır ve hizalama dolgusu
yemez (struct tanımları [../src/include/varatt.h:126-154](../src/include/varatt.h#L126-L154)).

**Asıl TOAST işaretçisi** dört alandır
([../src/include/varatt.h:32-39](../src/include/varatt.h#L32-L39)):

```c
	int32		va_rawsize;		/* Original data size (includes header) */
	uint32		va_extinfo;		/* External saved size + compression method */
	Oid			va_valueid;		/* Unique ID of value within TOAST table */
	Oid			va_toastrelid;	/* RelID of TOAST table containing it */
```

16 bayt, **dolgu (padding) yok** — çünkü iki işaretçi bazen `memcmp` ile
karşılaştırılıyor (bölüm 9). Tuple içinde hizalanmamış durur; alanlarına bakmadan
önce yerel bir değişkene kopyalanması zorunludur
([../src/include/access/detoast.h:22-28](../src/include/access/detoast.h#L22-L28)).
`va_extinfo` alt 30 bitte dışarıdaki gerçek boyutu, üst 2 bitte sıkıştırma
yöntemini taşır
([../src/include/varatt.h:45-46](../src/include/varatt.h#L45-L46)). Bir dış
değerin sıkışık olup olmadığı ayrı bayrakla değil, **iki boyutu karşılaştırarak**
anlaşılır — `VARATT_EXTERNAL_GET_EXTSIZE(tp) < tp.va_rawsize - VARHDRSZ`
([satır 538-539](../src/include/varatt.h#L538-L539)). Bu ancak sıkıştırma **her zaman** yer kazandırdığı için güvenli — bölüm 5'teki
"2 bayttan fazla kazanç" kuralı burayı garanti eder.

**Üç tür işaretçi vardır** (`va_tag` baytı,
[varatt.h:84-90](../src/include/varatt.h#L84-L90)): `VARTAG_ONDISK = 18` bu notun
konusu, veri TOAST tablosunda; `VARTAG_INDIRECT` veriyi bellekte başka bir yerde
gösterir; `VARTAG_EXPANDED_RO/RW` genişletilmiş nesnedir (PL/pgSQL array'leri
gibi). Diskteki tuple'larda sadece `VARTAG_ONDISK` bulunur, ama `detoast_attr`
üçünü de karşılamak zorundadır.

---

# 8. Geri okuma ve dilim (slice) okuma

`detoast_attr`
([../src/backend/access/common/detoast.c:116](../src/backend/access/common/detoast.c#L116))
beş dallı bir `if` zinciridir. İlk dal en sık kullanılanı: `toast_fetch_datum` ile
chunk'lar toplanır, sonuç hâlâ `VARATT_IS_COMPRESSED` ise `toast_decompress_datum`
ile açılır ([118-132](../src/backend/access/common/detoast.c#L118-L132)). Kalan
dallar: indirect pointer (dereference + özyineleme), expanded object (düzleştir),
inline sıkışık (aç), kısa başlıklı (4 baytlığa çevir)
([133-188](../src/backend/access/common/detoast.c#L133-L188)). Sonuç **her zaman**
düz, 4 baytlık başlıklı bir varlena'dır — bu yüzden `PG_GETARG_TEXT_PP` yerine
`PG_GETARG_TEXT_P` kullanan fonksiyonlar her çağrıda tam kopya öder. Kardeşi
`detoast_external_attr` ([45](../src/backend/access/common/detoast.c#L45)) getirir
ama **sıkıştırmayı açmaz**; bölüm 4.1'deki yol bunu kullanır.

## 8.1 Chunk'ları toplamak

`toast_fetch_datum` işaretçiden boyutu okur, tampon ayırır, tablo access method'a
devreder; sıkışıksa sonuca sıkışık damgası basılır ki üstteki `detoast_attr` onu
açsın ([356-363](../src/backend/access/common/detoast.c#L356-L363)). Heap
tarafında asıl iş `heap_fetch_toast_slice`
([../src/backend/access/heap/heaptoast.c:626](../src/backend/access/heap/heaptoast.c#L626)),
zekâsı scan key kurulumunda: tümü isteniyorsa sadece `chunk_id` eşitliği (1 key),
tek chunk isteniyorsa `chunk_seq = N` (2 key), aralık isteniyorsa
`chunk_seq BETWEEN start AND end` (3 key)
([663-683](../src/backend/access/heap/heaptoast.c#L663-L683)).
**Dilim okuma gerçekten sadece gereken chunk'ları okur.** Okurken dört bozulma
kontrolü var: chunk numarası beklenen mi
([737-742](../src/backend/access/heap/heaptoast.c#L737-L742)), aralıkta mı
([743-749](../src/backend/access/heap/heaptoast.c#L743-L749)), boyutu doğru mu
([752-758](../src/backend/access/heap/heaptoast.c#L752-L758)), sonda eksik chunk
var mı ([781-786](../src/backend/access/heap/heaptoast.c#L781-L786)).

TOAST okuması normal snapshot da kullanmaz — `SnapshotToastData` kullanılır ve
aktif snapshot yoksa hata verilir
([toast_internals.c:643-646](../src/backend/access/common/toast_internals.c#L643-L646)).
Kural yorumda: **detoast, işaretçiyi getiren transaction ile aynı transaction'da
olmalı** ([625-626](../src/backend/access/common/toast_internals.c#L625-L626)) —
toast'lı değeri değişkene alıp `COMMIT` eden bir prosedür chunk'ları aradan
sildirmiş olabilir.

## 8.2 `substr()` neden dilim okuyabiliyor?

Zincir: `substr` → `text_substring`
([varlena.c:586](../src/backend/utils/adt/varlena.c#L586)) → `DatumGetTextPSlice`
([fmgr.h:305](../src/include/fmgr.h#L305)) → `pg_detoast_datum_slice`
([fmgr.c:1823](../src/backend/utils/fmgr/fmgr.c#L1823)) → `detoast_attr_slice`
([detoast.c:205](../src/backend/access/common/detoast.c#L205)). Oradaki
*"fast path for non-compressed external datums"* dalı doğrudan
`toast_fetch_datum_slice` çağırır
([232-234](../src/backend/access/common/detoast.c#L232-L234)).

**Sıkıştırılmamışsa** ofset ve uzunluk doğrudan chunk numaralarına çevrilir —
`substr(x, 1000000, 10)` üç dört sayfaya dokunur. **Sıkıştırılmışsa** akışın
ortasından başlanamaz; pglz için "bu kadar açılmış veri için en fazla ne kadar
sıkışık veri gerekir" hesaplanabiliyor
([254-256](../src/backend/access/common/detoast.c#L254-L256),
[../src/common/pg_lzcompress.c:857](../src/common/pg_lzcompress.c#L857)) ama
**LZ4 için mümkün değil** — uygun bir API çağrısı olmadığı için tüm değer
getirilir ([249-252](../src/backend/access/common/detoast.c#L249-L252)). Her
hâlükârda sadece **prefix** hızlandırılabilir.

**`EXTERNAL` ne zaman mantıklı?** Sıkıştırılmayacağı belli olan **ve** dilim
dilim okunan büyük kolonlar için — tipik örnek JPEG/PNG/ZIP saklayan bir `bytea`.
Sıkıştırma zaten işe yaramaz (bölüm 5) ve `EXTERNAL` ile değer düz yazıldığı için
`substr()` hızlı yolu kullanır; bedeli, sıkıştırılabilir veride `EXTENDED`'in
kazandıracağı yerden vazgeçmektir. Uzun düz metin (log, JSON, HTML) tutan ve
bütün okunan kolonlarda ise `EXTENDED` bırakılmalı.

---

# 9. UPDATE'te TOAST: değişmeyen kolon yeniden yazılmaz

TOAST'ın en önemli performans özelliği ve tek bir `memcmp`'ye dayanıyor:

```c
				if (ttc->ttc_isnull[i] ||
					!VARATT_IS_EXTERNAL_ONDISK(new_value) ||
					memcmp(old_value, new_value,
						   VARSIZE_EXTERNAL(old_value)) != 0)
					ttc->ttc_attr[i].tai_colflags |= TOASTCOL_NEEDS_DELETE_OLD;
				else
					ttc->ttc_attr[i].tai_colflags |= TOASTCOL_IGNORE;
```
— [toast_helper.c:76-97](../src/backend/access/table/toast_helper.c#L76-L97)
(yorumlar ve yardımcı satırlar çıkarıldı)

- **İşaretçiler aynıysa**: kolon `TOASTCOL_IGNORE` alır, yeni tuple aynı 18
  baytlık işaretçiyi taşır, TOAST tablosuna **hiç dokunulmaz** — 1 GB'lık bir
  `bytea` kolonu olan satırda başka kolonu güncellemek o 1 GB'ı yeniden yazmaz.
- **Farklıysa**: eski chunk'lar `toast_tuple_cleanup` içinde silinir
  ([299-310](../src/backend/access/table/toast_helper.c#L299-L310)).

`varatt_external`'ın dolgusuz olmak zorunda olmasının sebebi bu `memcmp`'dir
([varatt.h:24-30](../src/include/varatt.h#L24-L30)). Silme fiziksel değil —
`simple_heap_delete` sadece `xmax` yazar. `heap_update`'in TOAST için buffer
kilidini bırakması [UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md) bölüm
4.5'te.

**DELETE'te** zincir: `heap_delete` → `heap_toast_delete`
([../src/backend/access/heap/heapam.c:3184](../src/backend/access/heap/heapam.c#L3184),
[../src/backend/access/heap/heaptoast.c:43](../src/backend/access/heap/heaptoast.c#L43))
→ `toast_delete_external` (her varlena kolonu tarar,
[../src/backend/access/table/toast_helper.c:325-335](../src/backend/access/table/toast_helper.c#L325-L335))
→ `toast_delete_datum` (chunk chunk siler,
[../src/backend/access/common/toast_internals.c:420-429](../src/backend/access/common/toast_internals.c#L420-L429));
speculative insertion iptalinde `heap_abort_speculative` ile
([heapam.c:6512](../src/backend/access/heap/heapam.c#L6512)).

---

# 10. TOAST tablosunun kendi VACUUM'u

TOAST satırları normal heap satırları olduğuna göre temizlenmeleri gerekir. Ama
**TOAST tablosu ana tabloyla birlikte otomatik vacuum'lanmaz.**

**Elle `VACUUM`** ana tablodan sonra TOAST'a iner
([vacuum.c:2284-2287](../src/backend/commands/vacuum.c#L2284-L2287)) —
`VACUUM FULL`'da atlanır, çünkü `cluster_rel` TOAST'ı zaten baştan kurar. Ana
tablonun session kilidi hâlâ tutulurken `vacuum_rel` TOAST için ikinci kez
çağrılır; `toast_parent` alanı yetki kontrollerinin ana tabloya bakması içindir
([2355-2367](../src/backend/commands/vacuum.c#L2355-L2367)).
`VACUUM (PROCESS_TOAST false) tablom` bu adımı kapatır
([267-268](../src/backend/commands/vacuum.c#L267-L268)).

**Autovacuum'da TOAST tabloları ayrı bir geçişte** değerlendirilir —
*"second pass: check TOAST tables"* diyerek `RELKIND_TOASTVALUE` için ikinci bir
`pg_class` taraması yapılır
([autovacuum.c:2133-2137](../src/backend/postmaster/autovacuum.c#L2133-L2137)).
Sebep hemen üstteki yorumda: *"we don't automatically vacuum toast tables along
the parent table"*
([2098-2102](../src/backend/postmaster/autovacuum.c#L2098-L2102)).
**TOAST tablosunun eşiği bağımsızdır**: ana tabloda hiç ölü tuple olmasa bile
TOAST'ta milyonlarca ölü chunk birikmişse autovacuum sadece TOAST'ı vacuum'lar —
ya da tersi. Ayarlar TOAST tablosunun kendi reloption'larından, yoksa ana
tablonun `toast.*` ayarlarından gelir
([2848-2857](../src/backend/postmaster/autovacuum.c#L2848-L2857)):
`ALTER TABLE tablom SET (toast.autovacuum_vacuum_scale_factor = 0.05);`

**ANALYZE hiç çalışmaz** — TOAST tablosu istatistik tutmaz
([3314-3315](../src/backend/postmaster/autovacuum.c#L3314-L3315)). Gerekçe
`vacuum.c` yorumunda: *"the toaster always uses hardcoded index access and
statistics are totally unimportant for toast relations"*
([../src/backend/commands/vacuum.c:2351-2353](../src/backend/commands/vacuum.c#L2351-L2353));
toaster planner kullanmaz, doğrudan index'e gider.

---

# 11. İzleme ve hata ayıklama

| Fonksiyon | Heap | FSM/VM | TOAST + TOAST index | Diğer index'ler |
|---|---|---|---|---|
| `pg_relation_size()` | ✔ | fork parametresiyle | ✘ | ✘ |
| `pg_table_size()` | ✔ | ✔ | **✔** | ✘ |
| `pg_indexes_size()` | ✘ | ✘ | ✘ | ✔ |
| `pg_total_relation_size()` | ✔ | ✔ | ✔ | ✔ |

`pg_table_size` TOAST'ı içerir — `calculate_table_size` içinde
`if (OidIsValid(rel->rd_rel->reltoastrelid)) size += calculate_toast_table_size(...)`
([dbsize.c:457-458](../src/backend/utils/adt/dbsize.c#L457-L458)); bu fonksiyon
TOAST heap'ini **ve index'ini** toplar
([396-431](../src/backend/utils/adt/dbsize.c#L396-L431)), yani
`pg_total_relation_size − pg_indexes_size` TOAST index'ini hâlâ içerir.

```sql
-- TOAST'ın payı + reltoastrelid ile TOAST tablosunu bulmak
SELECT c.relname, t.relname AS toast_tablosu,
       pg_size_pretty(pg_relation_size(c.oid))                 AS heap,
       pg_size_pretty(pg_total_relation_size(c.reltoastrelid)) AS toast,
       pg_size_pretty(pg_total_relation_size(c.oid))           AS toplam
FROM pg_class c JOIN pg_class t ON t.oid = c.reltoastrelid
WHERE c.relkind = 'r' ORDER BY pg_total_relation_size(c.oid) DESC LIMIT 20;

-- storage stratejileri:  p=plain  e=external  x=extended  m=main
SELECT attname, atttypid::regtype, attstorage, attcompression FROM pg_attribute
WHERE attrelid = 'tablom'::regclass AND attnum > 0 AND NOT attisdropped;

-- bir değerin diskteki hâli
SELECT pg_column_size(icerik)           AS diskte,   -- sıkışık / dış boyut
       octet_length(icerik)             AS acilmis,  -- gerçek uzunluk
       pg_column_compression(icerik)    AS yontem,   -- 'pglz' | 'lz4' | NULL
       pg_column_toast_chunk_id(icerik) AS chunk_id  -- dışarıdaysa OID
FROM tablom WHERE id = 42;

-- chunk'ları doğrudan görmek + TOAST I/O sayaçları
SELECT chunk_seq, length(chunk_data)
FROM pg_toast.pg_toast_16384 WHERE chunk_id = 24601 ORDER BY chunk_seq;
SELECT relname, toast_blks_read, toast_blks_hit, tidx_blks_read
FROM pg_statio_user_tables WHERE toast_blks_read > 0;
```

`attcompression` boşsa (`\0`) `default_toast_compression` geçerlidir
([toast_compression.h:49-53](../src/include/access/toast_compression.h#L49-L53)).
`pg_column_size` → `toast_datum_size`
([varlena.c:4186](../src/backend/utils/adt/varlena.c#L4186)); dış değer için
`va_extinfo`'daki boyutu döner ve **18 baytlık işaretçiyi saymaz**
([detoast.c:608-611](../src/backend/access/common/detoast.c#L608-L611)).
`pg_column_compression` yöntemi değerin kendisinden okur, kolon tanımından değil
([varlena.c:4234-4237](../src/backend/utils/adt/varlena.c#L4234-L4237)); `NULL`
üç şey demek olabilir: değer sıkıştırılmamış, varlena değil, ya da sıkıştırılmadan
dışarı yazılmış (`EXTERNAL`). `pg_column_toast_chunk_id` `NULL` değilse değer
diskte dışarıdadır ([4260](../src/backend/utils/adt/varlena.c#L4260)). TOAST I/O
kolonları `reltoastrelid` üzerinden LEFT JOIN ile gelir
([system_views.sql:826-833](../src/backend/catalog/system_views.sql#L826-L833)).

| Belirti | Bakılacak yer |
|---|---|
| `pg_total_relation_size` büyük, `pg_relation_size` küçük | Şişkinlik TOAST'ta; TOAST'ın kendi VACUUM'u (bölüm 10) |
| Büyük kolon değişmediği hâlde UPDATE yavaş | Kolon gerçekten değişiyor mu? `memcmp` yolu (bölüm 9) sadece işaretçi aynıysa çalışır |
| CPU yüksek, sıkıştırma kazancı yok | Veri zaten sıkışık (JPEG/ZIP) → `SET STORAGE EXTERNAL` |
| `substr()` büyük değerde yavaş | Değer sıkıştırılmış; LZ4 ise tamamı getiriliyor (bölüm 8.2) |
| `row is too big for size ...` | Kolonlar `PLAIN` ya da çok fazla kolon var — toaster kurtaramıyor |
| TOAST tablosu sürekli büyüyor | `toast.autovacuum_vacuum_scale_factor` yüksek; ya da uzun transaction `OldestXmin`'i tutuyor |
| `unexpected chunk number ...` | Diskte bozulma; `heap_fetch_toast_slice` doğrulaması patladı |
| `toast_blks_read` yüksek | Sorgular TOAST'a iniyor; `SELECT *` yerine gereken kolonları seç |

---

# 12. Tek sayfalık özet

```
 INSERT [heapam.c:2264] / UPDATE [heapam.c:3839]
      │  t_len > TOAST_TUPLE_THRESHOLD (2032)  ya da  HasExternal
      ▼
 ┌─ heap_toast_insert_or_update [heaptoast.c:96] ──────────────────────┐
 │ toast_tuple_init → NULL/PLAIN/fixed kolonları ele; UPDATE'te memcmp:│
 │                    aynı işaretçi = DOKUNMA                          │
 │ maxDataLen = toast_tuple_target (2032) − başlık                     │
 │ while data_size > maxDataLen:   (her turda en BÜYÜK kolon seçilir)  │
 │   TUR 1 EXTENDED ise sıkıştır; tek başına bütçeden büyükse hemen at │
 │   TUR 2 EXTENDED/EXTERNAL → dışarı at    TUR 3 MAIN → sıkıştır      │
 │   TUR 4 hedef 8160'a yükselir → MAIN → dışarı at (son çare)         │
 └──────────────┬──────────────────────────────────────────────────────┘
   sıkıştır     │    dışarı at
     ▼          │       ▼
 toast_compress_datum  toast_save_datum [toast_internals.c:119]
 [toast_internals.c:46]  │ chunk_size = 1996, chunk_id = yeni OID
   pglz | lz4            │ heap_insert + index_insert  ← MVCC'ye tabi
   kazanç ≤ 2 bayt       ▼
    → INCOMPRESSIBLE   pg_toast.pg_toast_<oid> (chunk_id, chunk_seq,
                       chunk_data) + UNIQUE btree (chunk_id, chunk_seq)
                         ▼
           varatt_external (18 bayt) ana tabloya yazılır:
           { va_rawsize | va_extinfo | va_valueid | va_toastrelid }
             va_extinfo = 30 bit boyut + 2 bit sıkıştırma yöntemi
 ── OKUMA ──  detoast_attr [detoast.c:116] → chunk'ları topla, sıkışıksa aç
              detoast_attr_slice [detoast.c:205] → sıkışık DEĞİL: sadece
                gereken chunk | pglz: prefix kadarı | lz4: tamamı
 ── SİLME ──  DELETE → heap_toast_delete [heaptoast.c:43] → chunk'lara xmax
              UPDATE → işaretçi değiştiyse eski chunk'lara xmax, aynıysa hiç
              VACUUM → TOAST AYRI eşikle, AYRI geçişte [autovacuum.c:2133]
```

**Akılda kalması gereken beş şey:**

1. Eşik ~2 KB'dır, 8 KB değil: `TOAST_TUPLE_THRESHOLD` sayfaya 4 tuple sığsın
   diye seçilmiştir, çok daha erken devreye girer.
2. Toaster her turda **en büyük kolonu** seçer ve sırayla dener: EXTENDED
   sıkıştır → EXTENDED/EXTERNAL dışarı at → MAIN sıkıştır → MAIN dışarı at.
3. Sıkıştırma yöntemi kolonun tanımında değil, **değerin kendi başlığında**
   saklanır. `default_toast_compression`'ı değiştirmek eski satırları etkilemez.
4. UPDATE'te değişmeyen bir TOAST kolonunun işaretçisi `memcmp` ile tanınır ve
   olduğu gibi kopyalanır — 1 GB'lık kolon yeniden yazılmaz.
5. TOAST tablosunun **kendi autovacuum eşiği** vardır ve ana tabloyla birlikte
   otomatik vacuum'lanmaz. Şişkinlik çoğu zaman burada gizlenir.
