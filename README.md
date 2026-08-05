Kod İnceleme Notları
====================

`review/` klasörü, bu kaynak ağacının kodu takip edilerek yazılmış Türkçe
inceleme notlarını içerir.  Her not, ilgili kaynak dosya ve satır
numaralarına doğrudan bağlantı verir.

| Dosya | Ne anlatır |
|---|---|
| [review/POSTGRES-NASIL-CALISIR.md](review/POSTGRES-NASIL-CALISIR.md) | **Motorun genel turu.**  Postmaster ve süreç modeli, paylaşımlı bellek, disk düzeni, MVCC, WAL, kilitler ve bakım işleri.  Diğer iki dosyanın giriş noktası. |
| [review/SELECT-NASIL-CALISIR.md](review/SELECT-NASIL-CALISIR.md) | **Sorgu boru hattı.**  Tek bir `SELECT` komutunun soketten okunmasından satırların geri gönderilmesine kadar izlediği yol: parse → analyze → rewrite → plan → execute. |
| [review/KILIT-MEKANIZMALARI.md](review/KILIT-MEKANIZMALARI.md) | **Kilit mekanizmaları.**  Dört kilit türü (spinlock, LWLock, heavyweight, SIReadLock), çakışma matrisi, fast path, deadlock tespiti, satır kilitleri ve MultiXact, SSI. |
