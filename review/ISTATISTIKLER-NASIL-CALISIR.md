# İstatistikler PostgreSQL içinde nasıl çalışır?

*PostgreSQL **20devel** ağacı. Satır numaraları bu sürüme aittir
([meson.build:11](../meson.build#L11)).*

[SELECT-NASIL-CALISIR.md](SELECT-NASIL-CALISIR.md) planlayıcının maliyet
karşılaştırıp plan seçtiğini anlatıyor. Bu not o maliyetin *girdisini*
anlatıyor: planlayıcı "bu WHERE kaç satır döndürür" sorusunu nereden biliyor,
o bilgi kim tarafından nasıl toplanıyor, ne zaman yanlış çıkıyor.

---

## Önce bir ayrım: PostgreSQL'de iki ayrı "istatistik" var

Aynı kelime iki farklı alt sistemi anlatıyor, karıştırmak kolay:

| | **Planner istatistikleri** | **Kümülatif istatistikler** |
|---|---|---|
| Nerede | `pg_statistic`, `pg_statistic_ext_data`, `pg_class` | paylaşımlı bellek + `pg_stat_*` view'ları |
| Kim yazar | `ANALYZE` (elle veya autovacuum) | her backend, iş yaparken |
| Ne tutar | veri *dağılımı* — MCV, histogram, ndistinct | *sayaçlar* — kaç seq scan, kaç tuple okundu |
| Kim okur | planlayıcı (selectivity hesabı) | DBA, autovacuum, izleme araçları |
| Kodu | [../src/backend/commands/analyze.c](../src/backend/commands/analyze.c), [../src/backend/utils/adt/selfuncs.c](../src/backend/utils/adt/selfuncs.c) | [../src/backend/utils/activity/](../src/backend/utils/activity/) |

Bu notun gövdesi **planner istatistikleri**. Kümülatif sisteme sonda ayrı bir
bölüm var, çünkü ikisi tek yerde buluşuyor: autovacuum, kümülatif sayaçlara
bakıp `ANALYZE` tetikliyor.

---

## 30 saniyelik özet

```
   ANALYZE tablo                                   (veya autovacuum tetikler)
        │
        ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ 1. Kaç satır örnekleyeceğine karar ver                       │
   │    targrows = 300 * en yüksek stats target   (varsayılan     │
   │    100 → 30 000 satır)                                       │
   └──────────────────────┬───────────────────────────────────────┘
                          ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ 2. İKİ AŞAMALI ÖRNEKLEME                                     │
   │    a) targrows kadar rastgele BLOK seç (BlockSampler)        │
   │    b) o blokları okurken Vitter reservoir ile targrows satır │
   │    → tablo ne kadar büyürse büyüsün örnek sabit kalır        │
   └──────────────────────┬───────────────────────────────────────┘
                          ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ 3. Her kolon için compute_stats()                            │
   │    null oranı, ortalama genişlik, ndistinct (Haas-Stokes),   │
   │    MCV listesi, histogram, korelasyon                        │
   └──────────────────────┬───────────────────────────────────────┘
                          ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ 4. Yaz:  pg_statistic (kolon başına 1 satır, 5 slot)         │
   │          pg_statistic_ext_data (extended stats)              │
   │          pg_class.reltuples / relpages                       │
   └──────────────────────┬───────────────────────────────────────┘
                          ▼
        ─────────── sonraki sorgu geldiğinde ───────────
                          ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ 5. Planlayıcı: clauselist_selectivity → eqsel / scalarltsel  │
   │    MCV'de var mı? → tam frekans                              │
   │    yoksa → (1 - MCV toplamı - null) / kalan distinct         │
   │    aralık? → histogram kutusunda doğrusal interpolasyon      │
   │    → rows = reltuples * selectivity → maliyet → plan         │
   └──────────────────────────────────────────────────────────────┘
```

Zihinsel model — altı madde:

1. **İstatistik örnekten üretilir, tam taramadan değil.** Milyar satırlık
   tablo da 30 000 satır örnekleniyor. Bu bilinçli: ANALYZE ucuz olsun diye.
2. **Örnek sabit boyutlu, tablo değil.** Yani tablo büyüdükçe istatistiğin
   *göreli* kalitesi düşer. `stats target` bunu telafi etme düğmesi.
3. **Dağılım iki parçaya bölünür:** sık değerler MCV listesinde *tek tek*,
   geri kalanı histogramda *özet* olarak. Histogram MCV'ler çıkarıldıktan
   sonraki dağılımı anlatır — bu ayrım eşitlik tahminini iyi yapıyor.
4. **`ndistinct` tahmindir, sayım değil.** Örnekten Haas-Stokes formülüyle
   ekstrapole edilir ve en kırılgan istatistik odur.
5. **Kolonlar bağımsız varsayılır.** `WHERE a=1 AND b=2` selectivity'si
   `sel(a) * sel(b)`. Gerçekte korelasyon varsa tahmin çöker — `CREATE
   STATISTICS` tam olarak bu deliği kapatmak için var.
6. **İstatistik yoksa sabit sayı kullanılır.** `=` için 0.005, `<` için
   0.3333. Kötü planların klasik sebebi budur.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/commands/analyze.c](../src/backend/commands/analyze.c) | ANALYZE komutunun tamamı: örnekleme, hesaplama, katalog yazımı |
| [../src/backend/utils/misc/sampling.c](../src/backend/utils/misc/sampling.c) | `BlockSampler` ve Vitter reservoir algoritması |
| [../src/include/catalog/pg_statistic.h](../src/include/catalog/pg_statistic.h) | Katalog şeması + slot "kind" tanımları (asıl dokümantasyon burada) |
| [../src/backend/utils/adt/selfuncs.c](../src/backend/utils/adt/selfuncs.c) | Tüketici taraf: `eqsel`, `scalarltsel`, `eqjoinsel`, `estimate_num_groups` |
| [../src/backend/optimizer/path/clausesel.c](../src/backend/optimizer/path/clausesel.c) | Cümlecikleri birleştirme, bağımsızlık varsayımı, range query eşleştirme |
| [../src/backend/optimizer/util/plancat.c](../src/backend/optimizer/util/plancat.c) | `pg_class.reltuples`/`relpages` → planlayıcının satır sayısı tahmini |
| [../src/backend/statistics/](../src/backend/statistics/) | Extended statistics: ndistinct, functional dependencies, çok kolonlu MCV |
| [../src/backend/statistics/attribute_stats.c](../src/backend/statistics/attribute_stats.c) | `pg_restore_attribute_stats` — istatistiği ANALYZE olmadan içeri aktarma |
| [../src/backend/postmaster/autovacuum.c](../src/backend/postmaster/autovacuum.c) | ANALYZE ne zaman otomatik tetiklenir |
| [../src/backend/utils/activity/pgstat*.c](../src/backend/utils/activity/) | Kümülatif istatistik sistemi (`pg_stat_*`) |

---

## 1. `pg_statistic`'in şekli — neden bu kadar tuhaf görünüyor

Katalog kolon başına bir satır tutuyor
([../src/include/catalog/pg_statistic.h:32-129](../src/include/catalog/pg_statistic.h#L32-L129)).
İlk kısım her tip için ortak:

```c
	float4		stanullfrac;    /* NULL oranı */
	int32		stawidth;       /* ortalama bayt genişliği (TOAST sonrası) */
	float4		stadistinct;    /* farklı değer sayısı — işaretli! */
```

`stadistinct`'in işareti anlamı değiştiriyor
([:53-71](../src/include/catalog/pg_statistic.h#L53-L71)):

| Değer | Anlamı |
|---|---|
| `0` | bilinmiyor / hesaplanmadı |
| `> 0` | gerçek farklı değer sayısı (ör. bir enum kolonu için `7`) |
| `< 0` | satır sayısının çarpanı — `-1` = unique, `-0.5` = her değer ~2 kez |

Negatif biçim tabloyla birlikte "ölçekleniyor": `pg_class.reltuples`
`pg_statistic`'ten daha sık güncelleniyor, o yüzden unique bir kolonu sabit
sayıyla anlatmak yanlış olurdu.

Asıl ilginç kısım **slot mimarisi**. Sabit kolon yerine 5 genel slot var
([:131](../src/include/catalog/pg_statistic.h#L131)):

```
  stakind1..5   → bu slotta ne var (tür kodu)
  staop1..5     → ilgili operatör OID'i (= veya <)
  stacoll1..5   → collation
  stanumbers1..5→ float4 dizisi (frekanslar, korelasyon)
  stavalues1..5 → anyarray (gerçek veri değerleri)
```

Sebep: her veri tipi aynı tür istatistiği tutmuyor. Bir `int` için histogram
mantıklı, bir `tsvector` için "en sık geçen *element*" mantıklı, bir `int4range`
için "aralık uzunluğu histogramı" mantıklı. Slotlar boş kutu; her `typanalyze`
fonksiyonu istediğini koyuyor. Çekirdeğin tanımladığı türler:

| Kod | Ad | İçerik |
|---|---|---|
| 1 | `STATISTIC_KIND_MCV` | en sık değerler + frekansları, azalan sırada |
| 2 | `STATISTIC_KIND_HISTOGRAM` | eşit-nüfuslu kutu sınırları, **MCV'ler çıkarılmış** |
| 3 | `STATISTIC_KIND_CORRELATION` | fiziksel sıra ↔ mantıksal sıra korelasyonu, -1..+1 |
| 4 | `STATISTIC_KIND_MCELEM` | dizi/tsvector için en sık *elemanlar* |
| 5 | `STATISTIC_KIND_DECHIST` | satır başına farklı eleman sayısı histogramı |
| 6 | `STATISTIC_KIND_RANGE_LENGTH_HISTOGRAM` | range tipleri: uzunluk dağılımı |
| 7 | `STATISTIC_KIND_BOUNDS_HISTOGRAM` | range tipleri: alt/üst sınır histogramı |

100-199 PostGIS'e, 200-299 ESRI'ye ayrılmış; özel tipler kendi kodunu
seçebiliyor ([:158-180](../src/include/catalog/pg_statistic.h#L158-L180)).

**Kod okuma kuralı:** slot sırasına asla güvenilmiyor. İstenen türü bulmak için
`stakind` alanları taranıyor — `get_attstatsslot()` bunu yapıyor. `pg_stats`
view'ının o upuzun `CASE WHEN stakind1 = 1 THEN ...` zinciri de bundan
([../src/backend/catalog/system_views.sql:190-235](../src/backend/catalog/system_views.sql#L190-L235)).

`pg_statistic` doğrudan okunamaz (satır verisi sızdırır); `pg_stats` view'ı
`security_barrier` ile sadece kullanıcının okuyabildiği tabloları gösteriyor.

---

## 2. ANALYZE: kaç satır örneklenir?

Karar `do_analyze_rel()` içinde
([../src/backend/commands/analyze.c:508-534](../src/backend/commands/analyze.c#L508-L534)):

```c
	targrows = 100;
	for (i = 0; i < attr_cnt; i++)
	{
		if (targrows < vacattrstats[i]->minrows)
			targrows = vacattrstats[i]->minrows;
	}
```

Yani **tablodaki en aç kolon kazanıyor** — tek kolon `SET STATISTICS 1000`
yapıldıysa bütün tablo o hedefle örnekleniyor. Index ifadeleri ve extended
statistics nesneleri de bu yarışa katılıyor
([:520](../src/backend/commands/analyze.c#L520),
[:532](../src/backend/commands/analyze.c#L532)).

`minrows` nereden geliyor? `std_typanalyze()` içinde, akademik bir
referansla ([:1951-2016](../src/backend/commands/analyze.c#L1951-L2016)):

```c
		 * Their Corollary 1 to Theorem 5
		 * says that for table size n, histogram size k, maximum relative
		 * error in bin size f, and error probability gamma, the minimum
		 * random sample size is
		 *		r = 4 * k * ln(2*n/gamma) / f^2
		 * Taking f = 0.5, gamma = 0.01, n = 10^6 rows, we obtain
		 *		r = 305.82 * k
```

```c
		stats->minrows = 300 * stats->attstattarget;
```

Yorumdaki kritik cümle: *"because of the log function, the dependence on n is
quite weak; even at n = 10^12, a 300*k sample gives <= 0.66 bin size error"*.
Yani örnek boyutunun tablo boyutundan bağımsız olması matematiksel olarak
savunuluyor — **histogram kutuları için**. Bu garanti `ndistinct` için
geçerli değil, ve pratikteki tahmin hatalarının çoğu oradan geliyor.

Hedef nereden okunuyor: `pg_attribute.attstattarget` (NULL ise) →
`default_statistics_target` GUC'u, varsayılan **100**
([../src/backend/utils/misc/postgresql.conf.sample:490](../src/backend/utils/misc/postgresql.conf.sample#L490)),
üst sınır 10 000
([../src/include/commands/vacuum.h:350](../src/include/commands/vacuum.h#L350)).

Yani varsayılanda: `300 * 100 = 30 000` satır örnekleniyor, MCV listesi en
fazla 100 değer, histogram en fazla 101 sınır tutuyor.

`std_typanalyze` aynı zamanda hangi algoritmanın kullanılacağını seçiyor
([:1975-2016](../src/backend/commands/analyze.c#L1975-L2016)):

```
  tipin < ve = operatörü var mı?  → compute_scalar_stats   (MCV+histogram+korelasyon)
  sadece = var mı?                → compute_distinct_stats (MCV, histogram yok)
  hiçbiri yok mu?                 → compute_trivial_stats  (sadece null + genişlik)
```

---

## 3. Örnekleme: iki aşamalı, ve neden mükemmel değil

`acquire_sample_rows()` başındaki yorum yöntemi anlatıyor
([:1230-1260](../src/backend/commands/analyze.c#L1230-L1260)):

```
 * As of May 2004 we use a new two-stage method:  Stage one selects up
 * to targrows random blocks (or all blocks, if there aren't so many).
 * Stage two scans these blocks and uses the Vitter algorithm to create
 * a random sample of targrows rows
```

Ve dürüst itiraf:

```
 * Although every row has an equal chance of ending up in the final
 * sample, this sampling method is not perfect: not every possible
 * sample has an equal chance of being selected.  For large relations
 * the number of different blocks represented by the sample tends to be
 * too small.  We can live with that for now.
```

Bu cümle pratikte şu demek: **kümelenmiş veri yanlı örneklenir**. Zaman
damgasına göre sıralı yazılmış bir tabloda aynı bloktaki satırlar birbirine
benziyor; 30 000 satır 30 000 farklı bloktan değil, çok daha az bloktan
geliyorsa çeşitlilik olduğundan az görünüyor.

Akış:

```c
	nblocks = BlockSampler_Init(&bs, totalblocks, targrows, randseed);
	...
	reservoir_init_selection_state(&rstate, targrows);
```
([:1288-1294](../src/backend/commands/analyze.c#L1288-L1294))

Reservoir kısmı — ilk `targrows` satır doğrudan alınıyor, sonrası rastgele
birinin yerine geçiyor ([:1342-1358](../src/backend/commands/analyze.c#L1342-L1358)):

```c
				if (rowstoskip <= 0)
				{
					/*
					 * Found a suitable tuple, so save it, replacing one old
					 * tuple at random
					 */
					int			k = (int) (targrows * sampler_random_fract(&rstate.randstate));
					heap_freetuple(rows[k]);
					rows[k] = ExecCopySlotHeapTuple(slot);
				}
```

Reservoir rastgele yer değiştirdiği için örnek fiziksel sırayı bozuyor —
korelasyon hesabı fiziksel sıraya ihtiyaç duyduğundan sonda tekrar
sıralanıyor ([:1380-1382](../src/backend/commands/analyze.c#L1380-L1382)):

```c
	if (numrows == targrows)
		qsort_interruptible(rows, numrows, sizeof(HeapTuple),
							compare_rows, NULL);
```

Toplam satır sayısı da buradan çıkıyor — okunan blokların yoğunluğu tüm
tabloya genelleniyor ([:1387-1400](../src/backend/commands/analyze.c#L1387-L1400)):

```c
	if (bs.m > 0)
	{
		*totalrows = floor((liverows / bs.m) * totalblocks + 0.5);
		*totaldeadrows = floor((deadrows / bs.m) * totalblocks + 0.5);
	}
```

Bu `totalrows` hem `pg_class.reltuples`'a yazılıyor hem de sonraki bütün
hesapların `N`'i oluyor.

---

## 4. `compute_scalar_stats` — asıl iş

En sık kullanılan yol ([:2462](../src/backend/commands/analyze.c#L2462)).
Örnek satırlar sıralanıyor, sonra tek geçişte dört şey çıkarılıyor.

### 4a. ndistinct — Haas-Stokes tahmincisi

Önce iki kısayol ([:2662-2685](../src/backend/commands/analyze.c#L2662-L2685)):

```c
		if (nmultiple == 0)
		{
			/* hiç tekrar yok → unique varsay */
			stats->stadistinct = -1.0 * (1.0 - stats->stanullfrac);
		}
		else if (toowide_cnt == 0 && nmultiple == ndistinct)
		{
			/* her değer birden fazla geçti → tüm popülasyon bu kadar */
			stats->stadistinct = ndistinct;
		}
```

İkincisi boolean/enum gibi küçük sabit kümeler için: örnekte tek başına geçen
hiçbir değer yoksa "tablo bu değerlerden ibaret" deniyor ve **kesin** sayı
yazılıyor.

Aksi hâlde formül ([:2669-2700](../src/backend/commands/analyze.c#L2669-L2700)):

```c
			 * Estimate the number of distinct values using the estimator
			 * proposed by Haas and Stokes in IBM Research Report RJ 10025:
			 *		n*d / (n - f1 + f1*n/N)
			 * where f1 is the number of distinct values that occurred
			 * exactly once in our sample of n rows (from a total of N),
			 * and d is the total number of distinct values in the sample.
```

```c
			int			f1 = ndistinct - nmultiple + toowide_cnt;
			int			d = f1 + nmultiple;
			double		n = samplerows - null_cnt;
			double		N = totalrows * (1.0 - stats->stanullfrac);
			...
				stadistinct = (n * d) / ((n - f1) + f1 * n / N);
```

Sezgi: örnekte **bir kez** geçen değerlerin sayısı (`f1`), görülmeyen ne kadar
değer olduğunun göstergesi. Çok sayıda tek-geçişli değer varsa "daha çok
görmediğim değer var" deniyor.

Bu tahminin zayıf yönü kavramsal: 30 000 satırlık örnekten milyar satırlık
tablonun ndistinct'ini çıkarmak bilinen zor bir problem, ve düzgün dağılmamış
verilerde ciddi biçimde şaşabiliyor. Çare `ALTER TABLE ... ALTER COLUMN ...
SET (n_distinct = ...)` ile elle sabitlemek — ANALYZE bu değeri hesabın
üstüne yazıyor ([:592-604](../src/backend/commands/analyze.c#L592-L604)).

Son bir dönüşüm ([:2712-2718](../src/backend/commands/analyze.c#L2712-L2718)):

```c
		 * If we estimated the number of distinct values at more than 10% of
		 * the total row count (a very arbitrary limit), then assume that
		 * stadistinct should scale with the row count rather than be a fixed
		 * value.
		 */
		if (stats->stadistinct > 0.1 * totalrows)
			stats->stadistinct = -(stats->stadistinct / totalrows);
```

Yorumdaki "a very arbitrary limit" ifadesi kodun kendi dürüstlüğü — bu %10
sınırı teorik değil, deneysel.

### 4b. MCV listesi — kaç değer saklamaya değer?

Ham liste tarama sırasında `track[]` dizisinde toplanıyor (en sık `num_mcv`
değer). Ama listenin tamamı saklanmıyor. İki durum
([:2720-2760](../src/backend/commands/analyze.c#L2720-L2760)):

```c
		 * If we are able to generate a complete MCV list (all the values in the
		 * sample will fit, and we think these are all the ones in the table),
		 * then do so.  Otherwise, store only those values that are
		 * significantly more common than the values not in the list.
```

**Tam liste durumu** (boolean, enum, düşük kardinaliteli kolonlar): listede
kolonun bütün değerleri varsa hepsi saklanıyor — planlayıcı tam bilgi almış
oluyor, histograma gerek kalmıyor.

**Kısmi liste durumu**: `analyze_mcv_list()` alttan buduyor
([:3040-3150](../src/backend/commands/analyze.c#L3040-L3150)). Sorduğu soru şu:
*bu değeri listeden çıkarsam, ona atanacak "sıradan değer" tahmininden
istatistiksel olarak anlamlı biçimde daha sık mı?*

```c
		selec = 1.0 - sumcount / samplerows - stanullfrac;
		...
		otherdistinct = ndistinct_table - (num_mcv - 1);
		if (otherdistinct > 1)
			selec /= otherdistinct;
```

Sonra hipergeometrik dağılımın standart sapmasıyla eşik kuruluyor
([:3120-3135](../src/backend/commands/analyze.c#L3120-L3135)):

```c
		N = totalrows;
		n = samplerows;
		K = N * mcv_counts[num_mcv - 1] / n;
		variance = n * K * (N - K) * (N - n) / (N * N * (N - 1));
		stddev = sqrt(variance);

		if (mcv_counts[num_mcv - 1] > selec * samplerows + 2 * stddev + 0.5)
			break;   /* anlamlı derecede sık — tut, ve üstündekilerin hepsini tut */
		else
			num_mcv--;  /* değmez — at, bir alttakine bak */
```

Budamanın **yukarıdan değil aşağıdan** yapılmasının sebebi yorumda yazıyor:
tüm değerler benzer frekanstaysa boş listeden başlayıp eklemek hiçbir şey
eklemez, o zaman nadir değerler ciddi biçimde fazla tahmin edilir
([:3068-3076](../src/backend/commands/analyze.c#L3068-L3076)).

Frekans örnek üzerinden hesaplanıyor
([:2782](../src/backend/commands/analyze.c#L2782)):

```c
				mcv_freqs[i] = (double) track[i].count / (double) samplerows;
```

### 4c. Histogram — MCV'ler çıkarıldıktan sonra

Kritik detay katalog başlığında yazılı
([../src/include/catalog/pg_statistic.h:196-214](../src/include/catalog/pg_statistic.h#L196-L214)):

```
 * IMPORTANT POINT: if an MCV
 * slot is also provided, then the histogram describes the data distribution
 * *after removing the values listed in MCV* (thus, it's a "compressed
 * histogram" in the technical parlance).
```

Kodda MCV'ler `values[]` dizisinden fiziksel olarak siliniyor
([:2827-2860](../src/backend/commands/analyze.c#L2827-L2860)), sonra kalanlardan
eşit aralıklı sınırlar seçiliyor
([:2872-2900](../src/backend/commands/analyze.c#L2872-L2900)):

```c
			 * The object of this loop is to copy the first and last values[]
			 * entries along with evenly-spaced values in between.  So the
			 * i'th value is values[(i * (nvals - 1)) / (num_hist - 1)].
```

Sonuç: **eşit genişlikli değil, eşit nüfuslu** kutular. Her kutuda kabaca aynı
sayıda satır var; kutu *genişlikleri* değişiyor. Yoğun bölgede kutular dar,
seyrek bölgede geniş.

Histogram sadece MCV'de yer bulamayan en az iki farklı değer varsa üretiliyor
([:2804-2806](../src/backend/commands/analyze.c#L2804-L2806)) — boolean kolonda
histogram olmamasının sebebi bu.

### 4d. Korelasyon — index maliyetinin gizli kahramanı

Örnek fiziksel sırada; değerler sıralı. İkisi arasındaki Pearson korelasyonu
hesaplanıyor ([:2915-2950](../src/backend/commands/analyze.c#L2915-L2950)):

```c
			corrs[0] = (values_cnt * corr_xysum - corr_xsum * corr_xsum) /
				(values_cnt * corr_x2sum - corr_xsum * corr_xsum);
```

`+1` = tablo bu kolona göre fiziksel sıralı, `0` = rastgele, `-1` = ters sıralı.

Neden önemli: index scan maliyeti "kaç farklı heap sayfasına gidilecek"
sorusuna bağlı. Korelasyon +1'e yakınsa index taraması heap'te de sıralı
ilerliyor ve rastgele I/O neredeyse sıralı I/O'ya dönüşüyor. Aynı satır sayısı
için maliyet kat kat düşüyor. Bir tabloyu `CLUSTER` etmenin plan üzerindeki
etkisi tam olarak bu tek float4 üzerinden geçiyor.

---

## 5. Index ifadeleri ve kalıtım

**İfade indexleri.** `CREATE INDEX ON t (lower(name))` yazıldığında `lower(name)`
üzerinde istatistik tutulur — ama tabloda öyle bir kolon yok. Çözüm: index'in
kendi `pg_statistic` satırları var. `compute_index_stats()` örnek satırlardan
index ifadesini hesaplayıp ondan istatistik çıkarıyor
([:878-900](../src/backend/commands/analyze.c#L878-L900)), sonuç index relation'ının
OID'i altına yazılıyor ([:614-621](../src/backend/commands/analyze.c#L614-L621)).
`examine_attribute()` bu durumda kolonun değil ifadenin tipine güveniyor
([:1101-1120](../src/backend/commands/analyze.c#L1101-L1120)).

PG 14'ten beri aynı şey `CREATE STATISTICS ... ON (expr) FROM t` ile index'siz
de yapılabiliyor (`STATS_EXT_EXPRESSIONS`).

**Kalıtım.** `pg_statistic.stainherit` bayrağı yüzünden bir kolonun **iki** satırı
olabiliyor:

- `stainherit = false` → sadece tablonun kendi satırları
- `stainherit = true` → tablo + bütün alt tabloları (partition'lar dâhil)

`acquire_inherited_sample_rows()` alt tabloların hepsinden orantılı örnek
topluyor ([:1451](../src/backend/commands/analyze.c#L1451)). Bölümlenmiş tabloda
sorgu ebeveyn üzerinden geldiği için planlayıcının `inherited = true`
satırlarına ihtiyacı var; partition'ları tek tek ANALYZE etmek bu satırı
üretmiyor — **bölümlenmiş tablolarda plan bozukluklarının klasik sebebi budur**.

---

## 6. Yazma tarafı

`update_attstats()` beş slotu düzleştirip kataloğa yazıyor
([:1715](../src/backend/commands/analyze.c#L1715)). Var olan satır varsa
güncelleniyor, yoksa ekleniyor; **ANALYZE edilmeyen kolonların satırlarına
dokunulmuyor** ([:617-620](../src/backend/commands/analyze.c#L617-L620)) — yani
`ANALYZE t (a)` çağrısı `b`'nin eski istatistiğini bırakıyor.

Ayrıca güncellenenler:

| Ne | Nerede |
|---|---|
| `pg_class.relpages`, `reltuples` | [:656](../src/backend/commands/analyze.c#L656), [:692](../src/backend/commands/analyze.c#L692) |
| `pg_statistic_ext_data` | `BuildRelationExtStatistics` [:625](../src/backend/commands/analyze.c#L625) |
| kümülatif sayaçlar (`last_analyze`, `mod_since_analyze` sıfırlama) | `pgstat_report_analyze` [:709](../src/backend/commands/analyze.c#L709) |

Not: `BuildRelationExtStatistics` **`update_attstats`'tan sonra** çağrılıyor,
çünkü extended stats kurulurken tek kolonluk istatistiklere ihtiyaç var.

---

## 7. Tüketim: planlayıcı bu sayıları nasıl kullanıyor

### 7a. Satır sayısı — `pg_class`'tan, ama körü körüne değil

`estimate_rel_size()` ilginç bir düzeltme yapıyor
([../src/backend/optimizer/util/plancat.c:1305-1390](../src/backend/optimizer/util/plancat.c#L1305-L1390)):
`relpages`/`reltuples` ANALYZE anındaki değerler, ama dosyanın **şu anki** blok
sayısı okunabiliyor. O yüzden yoğunluk (`reltuples / relpages`) alınıp güncel
blok sayısıyla çarpılıyor:

```c
		if (reltuples >= 0 && relpages > 0)
			density = reltuples / (double) relpages;
		...
		*tuples = rint(density * (double) curpages);
```

Yani ANALYZE'dan sonra tabloya iki kat veri girse bile satır sayısı tahmini
büyük ölçüde doğru kalıyor — bozulan şey *dağılım* (MCV/histogram), *hacim*
değil. Bu, "ANALYZE unutulmuş" belirtilerinin neden hep seçicilikle ilgili
olduğunu açıklıyor.

### 7b. Eşitlik — `var_eq_const()`

[../src/backend/utils/adt/selfuncs.c:370-533](../src/backend/utils/adt/selfuncs.c#L370-L533).
Sıra:

1. Kolon unique index / DISTINCT / GROUP BY ile kanıtlanmışsa: `1 / reltuples`
   ([:405-408](../src/backend/utils/adt/selfuncs.c#L405-L408)) — istatistiğe hiç
   bakılmıyor.
2. Sabit MCV listesinde mi? Operatör gerçekten çalıştırılarak aranıyor
   ([:449-465](../src/backend/utils/adt/selfuncs.c#L449-L465)). Bulunursa
   **frekans doğrudan okunuyor** — tahmin değil, ölçüm:

```c
			selec = sslot.numbers[i];
```

3. MCV'de yoksa, kalan kütle eşit paylaştırılıyor
   ([:481-508](../src/backend/utils/adt/selfuncs.c#L481-L508)):

```c
			for (i = 0; i < sslot.nnumbers; i++)
				sumcommon += sslot.numbers[i];
			selec = 1.0 - sumcommon - nullfrac;
			...
			otherdistinct = get_variable_numdistinct(vardata, &isdefault) -
				sslot.nnumbers;
			if (otherdistinct > 1)
				selec /= otherdistinct;
```

   Ve akıllı bir üst sınır: sonuç en küçük MCV frekansından büyük olamaz
   ([:505-508](../src/backend/utils/adt/selfuncs.c#L505-L508)) — mantık, "listeye
   girememişse en az sık MCV'den daha sık olamaz".

4. Hiç istatistik yoksa: `1 / get_variable_numdistinct(...)`, ki o da
   varsayılan **200** döndürüyor → selectivity 0.005
   ([../src/include/utils/selfuncs.h:34-52](../src/include/utils/selfuncs.h#L34-L52)):

```c
#define DEFAULT_EQ_SEL	0.005
#define DEFAULT_INEQ_SEL  0.3333333333333333
#define DEFAULT_NUM_DISTINCT  200
```

`get_variable_numdistinct()` ayrıca birkaç özel durum biliyor
([:6713-6800](../src/backend/utils/adt/selfuncs.c#L6713-L6800)): boolean → 2,
`ctid` → unique, `tableoid` → 1, `VALUES` listesi → unique. Ve unique index
kanıtı `pg_statistic`'i **eziyor** — çünkü istatistik eski olabilir, unique
kısıt olamaz.

### 7c. Aralık — histogram interpolasyonu

`scalarineqsel` → `ineq_histogram_selectivity`
([:1116](../src/backend/utils/adt/selfuncs.c#L1116)). Binary search ile sabitin
hangi kutuya düştüğü bulunuyor, sonra kutu içinde **doğrusal interpolasyon**
([:1290-1320](../src/backend/utils/adt/selfuncs.c#L1290-L1320)):

```c
						binfrac = (val - low) / (high - low);
```

Buradaki örtük varsayım: *kutu içinde veri düzgün dağılmış*. Kutu sınırları
eşit nüfuslu seçildiği için bu varsayım genelde makul, ama bir kutunun içinde
sivrilik varsa tahmin şaşıyor — `stats target` artırmak kutuları daraltıp
tam olarak bu hatayı küçültüyor.

Not: histogram MCV'siz dağılımı anlattığı için sonuç `1 - sumcommon - nullfrac`
ile ölçekleniyor, MCV'lerin aralığa düşen kısmı ayrıca `mcv_selectivity()`
ile ekleniyor ([:807](../src/backend/utils/adt/selfuncs.c#L807)).

`WHERE x > 5 AND x < 10` gibi çift taraflı aralıklar `clausesel.c` içinde
`RangeQueryClause` olarak eşleştirilip tek aralık gibi hesaplanıyor — yoksa
iki bağımsız `0.33` çarpılıp saçma bir sonuç çıkardı.

### 7d. Cümlecikleri birleştirme — bağımsızlık varsayımı

`clauselist_selectivity_ext()`
([../src/backend/optimizer/path/clausesel.c:116](../src/backend/optimizer/path/clausesel.c#L116))
önce extended statistics deniyor, kalanları teker teker hesaplayıp
**çarpıyor**. `s1 = 1.0` ile başlayıp her cümlecikte `s1 *= s2`.

Bu, PostgreSQL tahmininin en büyük tek varsayımı: `P(A ∧ B) = P(A) · P(B)`.
`WHERE city = 'İstanbul' AND country = 'Türkiye'` sorgusunda ikisi %100
bağımlı olmasına rağmen selectivity'ler çarpılıyor ve sonuç dramatik biçimde
küçük çıkıyor → nested loop seçiliyor → sorgu saatlerce sürüyor.

### 7e. Join ve gruplama

- `eqjoinsel()` ([:2387](../src/backend/utils/adt/selfuncs.c#L2387)) iki tarafın
  MCV listelerini birbirine karşı eşleştiriyor; eşleşmeyen kısım için
  `1 / max(ndistinct1, ndistinct2)` klasik formülü kullanılıyor.
- `estimate_num_groups()` ([:3804](../src/backend/utils/adt/selfuncs.c#L3804))
  GROUP BY / DISTINCT / hash agg bellek tahmini için ndistinct'leri
  birleştiriyor — burada da bağımsızlık varsayımı var, ve extended
  `ndistinct` istatistiği tam bunu düzeltiyor.

---

## 8. Extended statistics — bağımsızlık varsayımına yama

`CREATE STATISTICS` ile açılıyor. README açık konuşuyor
([../src/backend/statistics/README:1-10](../src/backend/statistics/README#L1-L10)):

```
When estimating various quantities (e.g. condition selectivities) the default
approach relies on the assumption of independence. In practice that's often
not true, resulting in estimation errors.
```

Üç tür var ([../src/include/catalog/pg_statistic_ext.h:88-91](../src/include/catalog/pg_statistic_ext.h#L88-L91)):

| Kod | Tür | Neyi düzeltir | Hangi cümleciklerde |
|---|---|---|---|
| `'d'` | **ndistinct** | çok kolonlu grup sayısı | GROUP BY, hash agg boyutu |
| `'f'` | **dependencies** | `a → b` yumuşak fonksiyonel bağımlılık | `=` (AND), IS NULL |
| `'m'` | **MCV** | çok kolonlu sık değer kombinasyonları | `=`, `<`, AND/OR/NOT, IS [NOT] NULL |
| `'e'` | expressions | ifade üzerinde tek kolonluk istatistik | her şey |

Veri `pg_statistic_ext_data` içinde serileştirilmiş özel tiplerde duruyor
([../src/include/catalog/pg_statistic_ext_data.h:37-44](../src/include/catalog/pg_statistic_ext_data.h#L37-L44)):

```c
	bool		stxdinherit;
	pg_ndistinct stxdndistinct;
	pg_dependencies stxddependencies;
	pg_mcv_list stxdmcv;
	pg_statistic stxdexpr[1];
```

**Yumuşak bağımlılık** fikri README.dependencies'te
([../src/backend/statistics/README.dependencies:38-50](../src/backend/statistics/README.dependencies#L38-L50)):
gerçek veride hata olur, katı bir `a → b` tanımı işe yaramaz. Onun yerine
bağımlılığın *derecesi* (0..1) saklanıyor ve selectivity ikisi arasında
harmanlanıyor.

Uygulama sırası `statext_clauselist_selectivity()` içinde
([../src/backend/statistics/extended_stats.c:2024-2060](../src/backend/statistics/extended_stats.c#L2024-L2060)):
**önce MCV** (en hassas), sonra kalan cümlecikler için **dependencies**. MCV
tarafından hesaplanan cümlecikler `estimatedclauses` bitmap'ine işaretlenip
bir daha hesaplanmıyor — çift sayım engelleniyor.

Önemli sınır: ekstra istatistik tek tabloya ait. Join'in iki tarafı arasındaki
korelasyona çare değil.

---

## 9. ANALYZE ne zaman kendiliğinden çalışır

Autovacuum launcher her tablo için eşik hesaplıyor
([../src/backend/postmaster/autovacuum.c:3292](../src/backend/postmaster/autovacuum.c#L3292)):

```c
	anlthresh = (float4) anl_base_thresh + anl_scale_factor * reltuples;
	...
		if (av_enabled && anltuples > anlthresh)
			*dovacuum = true;
```

`anltuples` kümülatif sistemden geliyor: `mod_since_analyze`, yani son
ANALYZE'dan beri INSERT+UPDATE+DELETE sayısı
([:3263](../src/backend/postmaster/autovacuum.c#L3263)).

Varsayılanlar
([../src/backend/utils/misc/postgresql.conf.sample:740-745](../src/backend/utils/misc/postgresql.conf.sample#L740-L745)):

```
#autovacuum_analyze_threshold = 50
#autovacuum_analyze_scale_factor = 0.1
```

Yani: **50 + satır sayısının %10'u**. 100 milyon satırlık tabloda bu 10 milyon
değişiklik demek — çok geç. Büyük tablolarda `ALTER TABLE ... SET
(autovacuum_analyze_scale_factor = 0.01)` yaygın bir ayar.

Kritik boşluklar:

- `ANALYZE` bir tabloyu **`TRUNCATE`/`COPY` sonrası anında** çalıştırmıyor;
  toplu yükleme sonrası elle ANALYZE gerekiyor.
- Geçici tablolara autovacuum hiç bakmıyor.
- Yeni oluşturulan bölümler (partition) ilk yüklemeden sonra eşiğe ulaşana
  kadar istatistiksiz kalıyor.

---

## 10. Kümülatif istatistik sistemi (`pg_stat_*`) — kısa tur

Tamamen ayrı bir makine
([../src/backend/utils/activity/pgstat.c:1-80](../src/backend/utils/activity/pgstat.c#L1-L80)).
Mimarisi tek paragrafta:

```
  backend iş yapar
     │  sayaçları ÖNCE process-local "pending" alanda biriktirir
     ▼
  pgstat_report_stat()  (commit sonrası veya idle timeout)
     │  pending → paylaşımlı hash tablosu (dshash, DSM'de)
     ▼
  paylaşımlı bellek — her giriş kendi LWLock'u ile korunur
     │  checkpointer kapanışta diske yazar
     ▼
  pg_stat_* view'ları  (stats_fetch_consistency: none / cache / snapshot)
```

Tasarımın üç önemli sonucu:

1. **Sayaçlar anlık değil.** Backend commit edene ya da boşa düşene kadar
   sayaçlar paylaşımlı belleğe ulaşmıyor — `pg_stat_user_tables` birkaç saniye
   geride olabiliyor.
2. **Çökme sonrası hepsi sıfırlanıyor** ([:12-16](../src/backend/utils/activity/pgstat.c#L12-L16)):
   *"unless preceded by a crash, in which case all stats are discarded"*.
   Çökmeden sonra autovacuum eşikleri de sıfırdan başlıyor.
3. Sabit sayılı istatistikler (checkpointer, WAL, IO) düz paylaşımlı bellekte;
   değişken sayılı olanlar (tablo, index, fonksiyon, replication slot) dshash'te.

Buradaki tek bağ planner istatistiklerine: `mod_since_analyze` ve
`last_analyze` — autovacuum'un ANALYZE kararı.

---

## 11. İzleme ve hata ayıklama

**Bir kolonun istatistiği ne diyor:**

```sql
SELECT attname, null_frac, avg_width, n_distinct,
       most_common_vals[1:5]  AS mcv_ilk5,
       most_common_freqs[1:5] AS frek_ilk5,
       array_length(histogram_bounds, 1) AS hist_kutu,
       correlation
FROM pg_stats
WHERE schemaname = 'public' AND tablename = 'siparis';
```

**Ne zaman ANALYZE edilmiş, ne kadar değişmiş:**

```sql
SELECT relname, n_live_tup, n_mod_since_analyze,
       last_analyze, last_autoanalyze,
       n_mod_since_analyze::float / GREATEST(n_live_tup, 1) AS degisim_orani
FROM pg_stat_user_tables
ORDER BY degisim_orani DESC;
```

**Hiç istatistiği olmayan tablolar** (en sık kök sebep):

```sql
SELECT c.relname, c.reltuples, c.relpages
FROM pg_class c
WHERE c.relkind = 'r'
  AND NOT EXISTS (SELECT 1 FROM pg_statistic s WHERE s.starelid = c.oid);
```

**Tahmin ne kadar şaşmış:** `EXPLAIN (ANALYZE, BUFFERS)` çıktısında her düğümün
`rows=` (tahmin) ile `actual rows=` (gerçek) oranına bakılıyor. 100× ve üstü
sapmalar ya eksik ANALYZE'ı ya da bağımsızlık varsayımının kırıldığını
gösteriyor.

**Ayar düğmeleri, etkiden en dara doğru:**

| Düğme | Ne yapar |
|---|---|
| `ANALYZE tablo;` | en basit çare, çoğu vakayı çözer |
| `ALTER TABLE t ALTER c SET STATISTICS 1000;` | o kolon için MCV+histogram çözünürlüğünü 10× artırır (örnek de 10× büyür) |
| `ALTER TABLE t SET (autovacuum_analyze_scale_factor = 0.01)` | büyük tabloda ANALYZE'ı sıklaştırır |
| `CREATE STATISTICS s (dependencies, mcv) ON a, b FROM t;` | çok kolonlu korelasyonu düzeltir |
| `ALTER TABLE t ALTER c SET (n_distinct = -0.3);` | Haas-Stokes tahminini elle ezer |

`SET STATISTICS`'in bedelini unutmamak lazım: hedef tablo genelinde en yüksek
kolona göre belirlendiği için ([:511](../src/backend/commands/analyze.c#L511)),
tek kolonu 10 000 yapmak bütün tabloyu 3 milyon satır örnekletiyor.

**Test ortamına istatistik taşıma** (PG 18+): ANALYZE çalıştırmadan
üretimdeki istatistiği kopyalamak mümkün —
[../src/backend/statistics/attribute_stats.c](../src/backend/statistics/attribute_stats.c),
[../src/backend/statistics/relation_stats.c](../src/backend/statistics/relation_stats.c):

```sql
SELECT pg_restore_relation_stats(
    'relation', 'public.siparis'::regclass,
    'relpages', 12000, 'reltuples', 900000.0);
```

Boş bir test veritabanında üretim planlarını yeniden üretmenin en pratik yolu
bu. `pg_dump --statistics-only` de aynı fonksiyonları üretiyor.

---

## Tek sayfalık özet

```
┌─────────────────────────────── ÜRETİM (ANALYZE) ────────────────────────────────┐
│                                                                                 │
│  autovacuum: mod_since_analyze > 50 + 0.1*reltuples ?        autovacuum.c:3292 │
│         │                                                                       │
│         ▼                                                                       │
│  targrows = max(100, 300 * en yüksek stats target)              analyze.c:508   │
│         │                        (varsayılan → 30 000 satır)                    │
│         ▼                                                                       │
│  BlockSampler: rastgele bloklar ──► Vitter reservoir: rastgele satırlar         │
│         │                                                       analyze.c:1263  │
│         ▼  fiziksel sıraya geri sırala (korelasyon için)                        │
│  ┌──────────────────────────────────────────────────────────────────────┐       │
│  │ compute_scalar_stats                                 analyze.c:2462  │       │
│  │   stanullfrac  = null / örnek                                        │       │
│  │   stawidth     = ortalama bayt                                       │       │
│  │   stadistinct  = n*d / (n - f1 + f1*n/N)      ← Haas-Stokes  :2698   │       │
│  │   MCV          = sık değerler, alttan budanmış  (2σ testi)  :3131    │       │
│  │   HISTOGRAM    = MCV'siz veri, eşit-NÜFUSLU kutular          :2879   │       │
│  │   CORRELATION  = fiziksel sıra ↔ değer sırası, -1..+1        :2941   │       │
│  └──────────────────────────────────────────────────────────────────────┘       │
│         │                                                                       │
│         ▼                                                                       │
│  pg_statistic (5 slot: kind/op/coll/numbers/values)      analyze.c:1715         │
│  pg_statistic_ext_data (ndistinct, dependencies, MCV)    analyze.c:625          │
│  pg_class.reltuples / relpages                           analyze.c:656          │
└─────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     │   (sonraki sorgu)
                                     ▼
┌──────────────────────────────── TÜKETİM (planlayıcı) ───────────────────────────┐
│                                                                                 │
│  satır sayısı: density = reltuples/relpages, * güncel blok    plancat.c:1305    │
│                                                                                 │
│  clauselist_selectivity                                       clausesel.c:116   │
│      │                                                                          │
│      ├─► extended stats var mı? → MCV, sonra dependencies  ext_stats.c:2024     │
│      │                                                                          │
│      └─► kalanlar tek tek, sonra ÇARPILIR  ← bağımsızlık varsayımı              │
│               │                                                                 │
│               ├── x = 'a'   → MCV'de mi?  evet: frekans      selfuncs.c:477     │
│               │                           hayır: (1-Σmcv-null)/kalan_distinct   │
│               │                           istatistik yok: 0.005 (DEFAULT_EQ_SEL)│
│               │                                                                 │
│               ├── x > 5     → histogram kutusunda doğrusal    selfuncs.c:1116   │
│               │                           istatistik yok: 0.3333                │
│               │                                                                 │
│               └── a JOIN b  → iki MCV listesi eşleştir        selfuncs.c:2387   │
│                                                                                 │
│  rows = tuples * selectivity  →  maliyet (korelasyon index'i ucuzlatır)  → plan │
└─────────────────────────────────────────────────────────────────────────────────┘

   YANLIŞ GİDEBİLECEK YERLER:
   1. ANALYZE hiç çalışmamış          → sabit varsayılanlar (0.005 / 0.3333)
   2. ndistinct şaşmış                → örnekten ekstrapolasyon, kırılgan
   3. kolonlar korelasyonlu           → çarpım çok küçük çıkar → nested loop
   4. veri kümelenmiş                 → blok örneklemesi yanlı
   5. bölümlenmiş tabloda inherit yok → ebeveyn üzerinden gelen sorgu kör
   6. skew stats target'tan büyük     → MCV'ye sığmayan sık değer
```
