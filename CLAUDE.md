# PostgreSQL Kod İnceleme Reposu

Bu depo PostgreSQL kaynak kodunu **geliştirmek için değil, incelemek için** tutuluyor.
Buradaki çalışma kaynak kodu okumak ve anlaşılır Türkçe notlar üretmektir.
Upstream PostgreSQL koduna değişiklik yapılmaz.

## Kural 0 — Her oturumda `i-have-adhd` skilli kullanılır

Bu depoda çalışan Claude, **ilk yanıtından önce** `i-have-adhd:i-have-adhd`
skillini çağırır ve kurallarını oturum boyunca uygular.

Kullanıcı ADHD'li. Yanıtlar buna göre şekillenmeli:

- İlk satır **yapılacak eylem** olsun — bağlam veya plan değil.
- Çok adımlı iş **numaralı liste** olsun, listeler en fazla 5 madde.
- Sonda **iki dakikada yapılabilecek tek bir eylem** olsun.
- Giriş cümlesi, kapanış nezaketi ve özet tekrarı yok.
- Süre tahminleri somut birimle verilsin ("15 dakika", "bir öğleden sonra").
- Konu dışı tespitler araya sıkıştırılmaz, sona ayrı soru olarak eklenir.

İstisna: kullanıcı "anlat" / "açıkla" derse gövde uzun olabilir, biçim aynı kalır.
Kullanıcı "stop adhd mode" derse kural düşer.

## Kural 1 — "İncele" denince otomatik dosya oluşturulur

Kullanıcı bir konu ya da senaryo verip **"incele"**, **"araştır"**, **"nasıl
çalışıyor"**, **"anlat"** derse: sonucu sohbete yazmakla yetinme.
**Sormadan** `review/` altına bir md dosyası oluştur.

Tetikleyen ifadeler: *"X'i incele"*, *"X nasıl çalışıyor"*, *"X'i araştır"*,
*"X hakkında bulabildiğini bul"*, *"X'i anlat"*.

Akış — her seferinde aynı:

1. Kodu oku, iddiaları kaynaktan doğrula.
2. `review/KONU-ADI.md` dosyasını yaz.
3. [README.md](README.md) tablosuna satırı ekle.
4. Sohbete **kısa** özet yaz: dosya linki + içindeki başlıklar + tek bir sonraki adım.

Uzun içeriği sohbete dökme — dosyaya yaz, sohbette özetle.

**Dosya yoksa iş bitmemiştir.** Kullanıcı "dosyaya kaydet" demek zorunda değil;
varsayılan davranış budur.

İstisna: kullanıcı açıkça *"dosya oluşturma"* / *"sadece söyle"* derse yazma.
Tek satırlık olgusal soru (*"bu fonksiyon kaçıncı satırda"*) not gerektirmez.

## Kural 2 — Her inceleme `review/` klasörüne gider

Yeni bir inceleme notu yazıldığında dosya **`review/`** altına konur.
Depo kök dizinine md dosyası bırakılmaz.

```
review/
  POSTGRES-NASIL-CALISIR.md      motorun genel turu
  SELECT-NASIL-CALISIR.md        sorgu boru hattı
  KILIT-MEKANIZMALARI.md         kilit mekanizmaları
```

Dosya adı **BÜYÜK-HARF-TIRE** biçiminde ve Türkçe olur (`WAL-NASIL-CALISIR.md` gibi).

## Kural 3 — Her inceleme README.md'ye eklenir

[README.md](README.md) bu deponun **içindekiler sayfasıdır**. Başka bir şey içermez.
Yeni bir not yazıldığında README'deki tabloya bir satır eklenir:

```markdown
| [review/DOSYA-ADI.md](review/DOSYA-ADI.md) | **Kısa başlık.**  Bir iki cümlelik özet. |
```

Not yazılıp README güncellenmediyse iş bitmiş sayılmaz.

## Kural 4 — Kaynak koduna satır numarasıyla link verilir

Notlar `review/` içinde olduğu için linkler **`../` ile başlar**:

```markdown
[../src/backend/storage/lmgr/lock.c:833](../src/backend/storage/lmgr/lock.c#L833)
[../src/include/storage/lockdefs.h:35-46](../src/include/storage/lockdefs.h#L35-L46)
```

Kök dizindeki README.md'de ise `../` **kullanılmaz** — oradan yollar `review/...` diye başlar.

Bir iddia kodla desteklenmiyorsa yazılmaz. Kod alıntıları gerçek dosyadan
kopyalanır, ezberden yazılmaz.

## Yazım biçimi

Mevcut üç dosya referans biçimdir. Yeni notlar da onlara benzemeli:

- **"30 saniyelik özet"** ile başla — ASCII diyagram + birkaç maddelik model.
- **"Kaynak haritası"** tablosu — hangi dizin/dosya ne yapar.
- Sonra derine in. Her bölümde kod alıntısı + satır linki.
- Sonunda **tek sayfalık özet diyagramı**.
- İzleme/hata ayıklama bölümü faydalıysa ekle (`pg_locks` sorguları gibi).
- Dil Türkçe, terimler İngilizce bırakılır (heavyweight lock, fast path, tuple).

## Sürüm

Bu ağaç PostgreSQL **20devel** ([meson.build](meson.build) `version: '20devel'`).
Notlarda sürüm belirtilir, çünkü satır numaraları sürüme bağlıdır.

## Commit

İnceleme notları tek başına commit'lenir. Commit mesajı İngilizce.
Kaynak kodu değişikliği ile inceleme notu aynı commit'e konmaz.
