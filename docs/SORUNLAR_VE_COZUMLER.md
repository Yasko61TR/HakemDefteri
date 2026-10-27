# HakemDefteri - Sorunlar ve Çözümler

Bu belge, projede karşılaşılması muhtemel sorunları **belirti / neden / çözüm / nasıl test edilir** biçiminde listeler. "Doğrulandı" yazanlar bu çalışmada kaynaktan teyit edilmiştir; "doğrulanmadı" yazanlar genel bilgi veya olasılıktır, kendi ortamınızda sınayın. Her başlığın sonunda öncelik (Y: yüksek, O: orta, D: düşük) belirtilmiştir.

## Özet tablo

| # | Sorun | Alan | Öncelik |
|---|---|---|---|
| 1 | Flyway checksum / sürüm çakışması | Migrasyon | Y |
| 2 | Flyway hiç çalışmıyor veya "Unsupported Database" | Migrasyon | Y |
| 3 | Hibernate ve Flyway şemayı birlikte yönetmeye çalışıyor | Migrasyon | Y |
| 4 | Testcontainers Windows'ta konteyneri başlatamıyor | Test / Docker | Y |
| 5 | Docker Engine 29 API sürümü uyuşmazlığı | Test / Docker | O |
| 6 | Entegrasyon testleri çok yavaş | Test | O |
| 7 | Docker Compose'ta uygulama veritabanından önce açılıp çöküyor | Docker | Y |
| 8 | N+1 sorgu | Performans | Y |
| 9 | Zaman dilimi ve ay sınırı hataları | Veri | Y |
| 10 | Para için `double` kullanımı | Veri | Y |
| 11 | JWT güvenlik hataları | Güvenlik | Y |
| 12 | Başkasının verisine erişim (IDOR) | Güvenlik | Y |
| 13 | "SQL'i göster" özelliğinin güvenlik açığına dönüşmesi | Güvenlik | Y |
| 14 | Seed verisi gerçekçi görünmüyor | Veri | O |
| 15 | Kartsız maçlar ortalamayı bozuyor | SQL | Y |
| 16 | Window function çerçevesi (ROWS / RANGE) ve eşitlik | SQL | O |
| 17 | CSV Excel'de bozuk görünüyor / formül enjeksiyonu | Dışa aktarma | O |
| 18 | Kişisel veri sızıntısı (veri, log, git geçmişi) | KVKK / Güvenlik | Y |
| 19 | Kapsam kayması | Süreç | Y |
| 20 | Proje "sadece CRUD" gibi görünüyor | Portföy | O |
| 21 | Windows satır sonları ve betikler | Ortam | D |
| 22 | Eski Boot 3 öğreticileri Boot 4'te çalışmıyor | Ortam | O |

---

## 1. Flyway checksum hatası ve sürüm çakışması

- **Belirti:** Uygulama açılırken "checksum mismatch" benzeri doğrulama hatası verir; ya da iki dalda aynı `V5__...` numarası oluşur ve birleştirince "Found more than one migration with version 5" türü hata çıkar.
- **Neden:** Daha önce uygulanmış bir migrasyon dosyası sonradan düzenlendi (Flyway kayıtlı checksum ile dosyayı karşılaştırır). İki geliştirme dalı aynı sürüm numarasını aldı.
- **Çözüm:**
  - Uygulanmış migrasyona dokunmayın; düzeltme için yeni `V<n+1>` yazın.
  - Yerel geliştirme veritabanında takılırsanız: `docker compose down -v` ile veriyi silip baştan kurun (yalnızca yerelde!). Paylaşılan/üretim ortamında `flyway repair` ancak nedenini anlayarak kullanılır.
  - Dallar birleşmeden önce `main` ile rebase edip numarayı güncelleyin. Tek kişilik projede bile bu alışkanlık iyidir.
- **Nasıl test edilir:** CI'da her çalıştırmada sıfır veritabanına tüm migrasyonlar uygulanır (Testcontainers). Ek olarak bir kontrol: depodaki `db/migration` dosya adlarında yinelenen sürüm var mı (basit betik veya test).

## 2. Flyway çalışmıyor veya "Unsupported Database"

- **Belirti A:** Uygulama PostgreSQL'e bağlanırken Flyway "Unsupported Database: PostgreSQL ..." hatası verir. **Doğrulandı:** Flyway 10 ile PostgreSQL desteği ayrı modüle (`flyway-database-postgresql`) taşındı; Spring Boot PR #41574 bunu dokümana ekledi.
- **Belirti B:** Hata yok ama tablolar oluşmuyor; testlerde "table not found". Bir Boot 3'ten 4'e geçiş rehberine göre Boot 4'te `flyway-core` yerine `spring-boot-starter-flyway` gerekiyor ve yoksa Flyway sessizce çalışmıyor (**kısmen doğrulandı**, ikincil kaynak; kendi sürümünüzde sınayın).
- **Neden:** Eksik bağımlılık.
- **Çözüm:** `spring-boot-starter-flyway` ve `org.flywaydb:flyway-database-postgresql` bağımlılıklarını ekleyin (sürüm Boot'un yönetimine bırakılır; gerekirse start.spring.io çıktısına bakın). Başlangıç logunda "Successfully applied N migrations" satırını arayın.
- **Nasıl test edilir:** Entegrasyon testi: `flyway_schema_history` tablosunda beklenen sayıda satır var mı ve `referee`, `match` tabloları mevcut mu. Test hiç migrasyon çalışmazsa **başarısız** olmalıdır (sessiz geçmemeli).

## 3. Hibernate ve Flyway şemayı birlikte yönetiyor

- **Belirti:** Şema Flyway'in yazdığından farklı; ya da başlangıçta "missing table / wrong column type" doğrulama hatası.
- **Neden:** `ddl-auto=update` veya `create` açık kaldı; ya da entity ile migrasyon uyumsuz (örn. `NUMERIC(12,2)` kolonuna `double` alan).
- **Çözüm:** `spring.jpa.hibernate.ddl-auto=validate`. Şema kaynağı yalnızca Flyway. Entity alan tiplerini şemayla eşleyin (`BigDecimal`, `OffsetDateTime`/`Instant`).
- **Nasıl test edilir:** Aynı entegrasyon testi, `validate` ile uygulama bağlamının ayağa kalkmasıdır; uyumsuzlukta test kırılır.

## 4. Testcontainers Windows'ta çalışmıyor

- **Belirti:** "Could not find a valid Docker environment", "Cannot connect to the Docker daemon", ya da testler uzun süre takılıp zaman aşımına uğrar.
- **Neden (olasılıklar, hepsi genel bilgi, sürümünüze göre **doğrulanmadı**):**
  - Docker Desktop kapalı ya da henüz tam başlamamış.
  - WSL2 arka ucu kapalı / kurulu değil; sanallaştırma BIOS'ta kapalı.
  - IDE ve terminal farklı Docker bağlamı (context) kullanıyor.
  - Kurumsal/okul bilgisayarında proxy veya güvenlik yazılımı engeli.
  - Disk/RAM yetersizliği (PostgreSQL imajı indirilemiyor).
- **Çözüm (sırayla):**
  1. Docker Desktop'u açın, "Engine running" görene kadar bekleyin; terminalde `docker ps` ve `docker run hello-world` çalıştırın.
  2. Aynı terminalden/IDE'den `docker context ls` ile geçerli bağlamı kontrol edin.
  3. WSL2 ve sanallaştırmanın açık olduğunu doğrulayın.
  4. Testcontainers dokümanındaki "özel Docker ortamı" ve ortam değişkeni yöntemlerine bakın (ayrıntı sürüme göre değişir; resmi dokümanı okuyun).
  5. Docker'ın çalışmadığı makinede testleri CI'da çalıştırmaya güvenin ve yerelde birim testlerle devam edin; bunu README'ye "Geliştirme notu" olarak yazın.
- **Nasıl test edilir:** Projeye en küçük "konteyner başlıyor mu" testi koyun (yalnızca PostgreSQL konteyneri başlatıp bağlanan). Bu test Docker sorunlarını uygulama sorunlarından ayırır.

## 5. Docker Engine 29 API sürümü uyuşmazlığı

- **Belirti:** "client version 1.xx is too old. Minimum supported API version is 1.44" benzeri hata (Docker'a bağlanan bir istemcide).
- **Neden:** **Doğrulandı:** Docker Engine 29.0 minimum API sürümünü 1.44'e yükseltti; Spring Boot'un buildpack kodu bundan etkilendi (Spring Boot issue #48050; 3.5.8+ ile düzeltildi). Bu sorun doğrudan Testcontainers için bildirilmedi; Testcontainers'ın etkilenip etkilenmediği ve düzeltme sürümü **doğrulanmadı** (aynı konuda ayrı bir Testcontainers sorununa yönlendirme var).
- **Çözüm:** İstemci kütüphanesini (Boot, Testcontainers) güncel tutun. Geçici çözüm olarak Docker'ı 28.x'e sabitlemek ya da daemon'da minimum API sürümünü düşürmek (güvenlik ve bakım açısından son çare) anlatılıyor; kendi riskinizle ve yalnızca geliştirme makinesinde uygulayın.
- **Nasıl test edilir:** Sorun 4'teki "konteyner başlıyor mu" testi. CI'da Docker sürümünü loga yazdırmak, hatanın hangi sürümde çıktığını bulmayı kolaylaştırır.

## 6. Entegrasyon testleri çok yavaş

- **Belirti:** Her test sınıfı için PostgreSQL konteyneri yeniden başlıyor; `mvn verify` dakikalar sürüyor.
- **Neden:** Sınıf başına konteyner; her testte Flyway'in sıfırdan çalışması; testler arası veri temizleme için konteyner yeniden yaratma.
- **Çözüm:** Tüm testler için paylaşılan tek konteyner (tek yapılandırma sınıfı), Spring bağlam önbelleğinden yararlanma; testler arası temizlik için transaction geri alma ya da `TRUNCATE`. Bağlamı bozan (`@DirtiesContext`) kullanımdan kaçının.
- **Nasıl test edilir:** Test süresini CI loglarında izleyin; hedef (öneri, kural değil): tüm entegrasyon paketi birkaç dakikanın altında. Konteyner başlatma sayısını loga yazdırıp 1 olduğunu doğrulayın.

## 7. Docker Compose: uygulama veritabanından önce açılıyor

- **Belirti:** `docker compose up` sırasında uygulama "Connection refused" ile çöküp yeniden başlıyor; bazen ikinci denemede çalışıyor.
- **Neden:** `depends_on` yalnızca konteynerin **başlamasını** bekler, PostgreSQL'in **hazır** olmasını beklemez.
- **Çözüm:** PostgreSQL servisine sağlık kontrolü (`pg_isready`) ekleyin ve uygulamada `depends_on` ile `condition: service_healthy` kullanın. Bağlantı bilgileri `.env`'den gelsin. Veri kalıcılığı için isimli volume; README'de `down -v` uyarısı.
- **Nasıl test edilir:** Temiz makinede (veya `docker compose down -v` sonrası) üç kez üst üste `docker compose up --build`; her seferinde uygulama ilk denemede sağlıklı olmalı. CI'da bir "compose smoke test" adımı eklenebilir (opsiyonel).

## 8. N+1 sorgu

- **Belirti:** `GET /matches` 20 maç için 1 yerine 40-60 SQL çalıştırıyor (maç, takım, saha, lig, olaylar için ayrı ayrı); sayfa büyüdükçe yavaşlıyor.
- **Neden:** İlişkiler tembel (lazy) yükleniyor ve DTO'ya çevrilirken tek tek erişiliyor.
- **Çözüm:** Listeleme için DTO projeksiyonu (sorguda doğrudan ihtiyaç duyulan sütunlar), ya da `JOIN FETCH`/entity graph. Koleksiyon (olaylar) için sayfalama ile `JOIN FETCH` birlikte kullanılırsa bellekte sayfalama uyarısı çıkabilir; olayları ayrı toplu sorgu (`IN`) ile çekmek daha güvenlidir. `open-in-view` ayarını bilinçli seçin (kapatmak sorunu erken görünür kılar).
- **Nasıl test edilir:** Testte Hibernate istatistikleri ya da bir sorgu sayacı ile "20 maç listesi en fazla N sorgu" iddiasını yazın; N'i sabitleyin. Seed verisiyle (ör. 200 maç) süre ölçün.

## 9. Zaman dilimi ve tarih/saat hataları

- **Belirti:** Gece 23:30'da oynanan maç yanlış aya düşüyor; CSV'deki saat 3 saat kayık; "2026-03-14" filtresi bir maçı dışarıda bırakıyor.
- **Neden:** Zamanın `timestamp` (saat dilimsiz) tutulması; sunucunun UTC, kullanıcının Türkiye saatinde olması; ay gruplamasının UTC'ye göre yapılması; filtrede `to` gününün 00:00'da kesilmesi.
- **Çözüm:**
  - Veritabanında `timestamptz`; JSON'da ISO-8601 ve ofset.
  - Ay/gün gruplamasında açıkça `AT TIME ZONE 'Europe/Istanbul'`.
  - `to` filtresi **dahil** gün ise üst sınırı ertesi günün 00:00'ı **hariç** olarak kullanın (yarı açık aralık).
  - Türkiye'nin yaz saati uygulamasının şu an kalıcı olarak UTC+3 olduğu genel bilgidir (**bu çalışmada yeniden doğrulanmadı**); yine de saat dilimi adını kullanmak, kuralı kütüphaneye bırakır.
  - Java'da `LocalDateTime` yerine `Instant`/`OffsetDateTime`/`ZonedDateTime`.
- **Nasıl test edilir:** Sınır testi: 31 Ocak 23:30 İstanbul saatinde bir maç; aylık kazançta Ocak'ta görünmeli (UTC'de 20:30 ama aynı gün; bir de 1 Şubat 00:30 İstanbul = 31 Ocak 21:30 UTC örneğiyle Şubat'ta görünmeli). Testi, JVM saat dilimini değiştirerek (örn. UTC ve Europe/Istanbul) tekrar edin; sonuç aynı olmalı.

## 10. Para için `double` kullanımı

- **Belirti:** Toplam `1499.9999999` veya `0.1 + 0.2` farkı; CSV'de `1500.0000001`.
- **Neden:** İkili kayan noktalı sayılar ondalık kesirleri tam temsil edemez. `double` ya da `float` kullanıldı; JSON'da sayı olarak gidip geldi.
- **Çözüm:** Veritabanında `NUMERIC(12,2)`, Java'da `BigDecimal`, JSON'da string (örn. `"1500.00"`) ya da tamsayı kuruş. Yuvarlama kuralı (`HALF_UP` vb.) bir yerde tanımlı. Para birimi sütunu (`currency`) ve karışık para birimini toplama yasağı.
- **Nasıl test edilir:** Birim test: 0,10 TL x 3 = 0,30 TL tam eşit; 10.000 küçük tutarın toplamı beklenen değere eşit. Entegrasyon: yazılan ve okunan tutar bire bir aynı.

## 11. JWT güvenlik hataları

- **Belirti:** Kodda sabit gizli anahtar; token'ın süresi yıllarca; `alg: none` kabul ediliyor; token localStorage'da; çıkış yapınca token hâlâ geçerli.
- **Neden:** Öğretici kodunun olduğu gibi kopyalanması; sürenin geliştirmeyi kolaylaştırmak için uzatılması.
- **Çözüm:**
  - Gizli anahtar ortam değişkeninden, yeterince uzun ve rastgele; depoda yok, `.env` `.gitignore`'da. Anahtar bir kez bile commit edildiyse **değiştirin** (git geçmişi bunu tutar).
  - Beklenen imza algoritmasını doğrulayıcıda sabitleyin; `none` kabul edilmez.
  - Kısa süre (örn. 60 dk), `exp` zorunlu; refresh/iptal v2'dir ve README'de açıkça "eksik" yazılır.
  - Token'ı JSON gövdesinde dönün, istemcide bellekte tutun.
  - Parola hash'i güçlü algoritmayla; giriş hata mesajı tek tip; kaba kuvvet için sınırlama.
  - Token içine hassas veri koymayın (JWT şifreli değil, kodlanmıştır; herkes okuyabilir).
- **Nasıl test edilir:** Güvenlik testleri: süresi dolmuş token 401; imzası bozuk token 401; farklı anahtarla imzalı token 401; `alg: none` token 401; tokensiz istek 401. Depoda gizli anahtar taraması (örn. bir gizli bilgi tarama aracı CI'da; araç seçimi yazarın) yapın.

## 12. Başkasının verisine erişim (IDOR)

- **Belirti:** `GET /matches/42` başka hakemin maçını döndürüyor; `POST /matches` içinde başkasının `teamId`'si kabul ediliyor.
- **Neden:** Yalnızca "giriş yapılmış mı" denetlendi, "bu kayıt sana ait mi" denetlenmedi. Şema da (bkz. TASARIM 4.3) takımın aynı hakeme ait olmasını garanti etmiyor.
- **Çözüm:** Her sorgu `referee_id` koşuluyla; servis katmanında merkezi bir sahiplik denetimi; bulunamayan ve sahibi olmayan kayıt için aynı `404`. İlişkili kayıtlar (takım, saha, lig) için de denetim. İleri aşama: bileşik yabancı anahtar.
- **Nasıl test edilir:** İki kullanıcı oluşturan entegrasyon testi: A'nın kaydına B ile GET/PUT/DELETE denemesi 404; B'nin maçına A'nın takımını bağlama denemesi 4xx.

## 13. "SQL'i göster" özelliği güvenlik açığına dönüyor

- **Belirti:** İstemci sorgu metni gönderebiliyor; yanıtta başka kullanıcıların parametre değerleri veya şema ayrıntıları görünüyor; hata mesajında ham SQL ve tablo adları sızıyor.
- **Neden:** "Görünür SQL" fikri, dinamik SQL çalıştıran bir uç noktaya genişletildi; hata işleyici istisna mesajını olduğu gibi döndürdü.
- **Çözüm:** Sorgular sabit dosyalardan, `queryId` ile çağrılır; istemciden SQL alınmaz. Parametreler bağlanır (string birleştirme yok). Yanıtta parametre **adı** gösterilir, **değeri** gösterilmez. Genel hata işleyici ayrıntıyı sunucu loguna yazar, istemciye genel mesaj döner.
- **Nasıl test edilir:** `queryId` yerine rastgele metin gönderen test → 400/404. Hata durumu testi: yanıt gövdesinde "SELECT", tablo adı veya yığın izi aranmaz.

## 14. Seed verisi gerçekçi görünmüyor

- **Belirti:** Her maçta tam 3 kart; tüm ücretler aynı; tüm maçlar aynı saatte; takımlar tekrar etmiyor; karne grafikleri düz çizgi; ya da tersine, abartılı uç değerler.
- **Neden:** Düzgün (uniform) rastgele sayı kullanımı; sınır durumlarının hiç üretilmemesi; ilişkili alanların (mesafe-masraf, görev-ücret) bağımsız üretilmesi.
- **Çözüm:** TASARIM Bölüm 10'daki kurallar: dağılımlı kart sayısı, görev bazlı ücret aralığı, mesafe ile bağlı masraf, tekrar eden takımlar, maçsız aylar, kartsız maçlar, ödenmemiş son maçlar. Sabit tohum ile tekrarlanabilirlik. Üretim sonrası otomatik tutarlılık sorguları (negatif skor, aynı takım iki kez, aynı saatte iki maç).
- **Nasıl test edilir:** Seed sonrası "dağılım testi": kartsız maç sayısı > 0, kırmızı kart > 0 ama az, en az bir ayın maçı yok, en az bir takımla 3+ karşılaşma. Aynı tohumla iki çalıştırma aynı özeti (satır sayıları, toplam ücret) vermeli. README'de verinin sentetik olduğu belirtilir.

## 15. Kartsız maçlar ortalamayı bozuyor

- **Belirti:** "Maç başına ortalama kart" beklenenden yüksek; hareketli ortalamada kartsız maçlar hiç görünmüyor.
- **Neden:** `match` ile `match_event` `INNER JOIN` yapıldı; olayı olmayan maçlar sonuçtan düştü. Ya da `COUNT(*)` ile `LEFT JOIN` birlikte kullanıldı ve kartsız maç "1" sayıldı (NULL satırı da sayılır).
- **Çözüm:** `LEFT JOIN` + `COUNT(*) FILTER (WHERE e.event_type IN (...))` (olay yoksa 0 verir) ya da `COUNT(e.id)`. Ortalamayı maç bazında hesaplayıp sonra pencere uygulayın.
- **Nasıl test edilir:** Elle hesaplı veri: 4 maç, kartlar 0, 0, 3, 5 → ortalama 2,0 (INNER JOIN ile 4,0 çıkar). Test bu iki değeri ayırt eder.

## 16. Window function çerçevesi ve eşit değerler

- **Belirti:** Hareketli ortalama aynı günkü iki maçta aynı değeri veriyor ya da ilk maçlarda "yanlış" görünüyor; `RANK` sonuçlarında 1, 1, 3 gibi atlamalar kullanıcıyı şaşırtıyor.
- **Neden:** `RANGE` çerçevesi ve sıralama sütunundaki eşitlikler eş satırları birlikte işler (genel SQL davranışı); `ROWS` kullanılmadı ya da ikincil sıralama yok. `RANK` eşitlikten sonra sıra atlar; `DENSE_RANK` atlamaz. İlk iki maçta 3'lük pencere dolmaz.
- **Çözüm:** Hareketli ortalamada `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` ve deterministik sıralama (`ORDER BY kickoff_at, match_id`). `RANK` mı `DENSE_RANK` mı seçimini belgede açıklayın. Yanıtta pencere boyutu (kaç maçtan hesaplandı) gösterilsin.
- **Nasıl test edilir:** Aynı saatte iki maç içeren elle hesaplı veri; beklenen hareketli ortalama dizisi testte yazılı. `EXPLAIN` ile sıralama için indeks kullanımı gözlemlenir (performans için; doğruluk için değil).

## 17. CSV Excel'de bozuk görünüyor ve formül enjeksiyonu

- **Belirti:** Türkçe karakterler bozuk (ş, ğ, İ); tüm sütunlar tek hücreye yığılıyor; ondalık ayırıcı virgül/nokta karışıyor; hücre `=` ile başlayan metin (örn. not alanı) Excel'de formül olarak çalışıyor.
- **Neden:** Kodlamanın BOM'suz UTF-8 olması ve Excel'in bunu tanımaması (yaygın davranış, **sürüme göre doğrulanmadı**); Türkçe yerel ayarda Excel'in ayırıcı olarak noktalı virgül beklemesi (yaygın davranış, **doğrulanmadı**); kullanıcı girdisinin olduğu gibi yazılması.
- **Çözüm:** Karar verin ve belgeleyin: (a) Excel odaklı: UTF-8 BOM + `;` ayırıcı, (b) genel: UTF-8 + `,` ayırıcı. Seçenek parametresi (`?format=excel`) iyi bir yol. Metin hücrelerinde `=`, `+`, `-`, `@` ile başlayanların başına tek tırnak ekleme veya benzer önlem. Tarihleri ISO biçiminde, tutarları noktalı ondalıkla yazın ve başlık satırında birimi belirtin. Virgül, tırnak ve satır sonu içeren alanlar tırnaklanır.
- **Nasıl test edilir:** Birim test: not alanında `;`, `"`, satır sonu, `=1+1` içeren kayıt → beklenen CSV metni bire bir. Elle kontrol: dosyayı gerçek Excel veya LibreOffice'te açıp Türkçe karakterlere bakın.

## 18. Kişisel veri sızıntısı (veri, log, git geçmişi)

- **Belirti:** Depoda gerçek isimli bir CSV/ekran görüntüsü; loglarda e-posta, token veya istek gövdesi; hata yanıtında kullanıcı bilgisi; `.env` commit edilmiş.
- **Neden:** "Önce bir bakayım" diye gerçek verinin depo klasörüne konması; debug logları; `.gitignore` eksikliği.
- **Çözüm:** (Hukuki tavsiye değildir.) Gerçek veri depo dışında kalır. Oyuncu adı hiç saklanmaz (tasarım kararı). Loglarda parola, token ve gövde yok; e-posta mümkünse maskeli. `.gitignore`: `.env`, `*.csv` (örnek hariç), `data/`. Commit öncesi gizli bilgi/PII taraması. Ekran görüntüsü çekmeden önce sentetik veriye geçin. Sızıntı olduysa: sırları değiştirin, geçmişi temizlemek yetmeyebilir (çatallanmış/önbelleklenmiş kopyalar olabilir) ve sızan veri kişisel veri ise ne yapılacağı hukuki değerlendirme gerektirir. Ayrıntılı genel noktalar: TASARIM Bölüm 11.
- **Nasıl test edilir:** CI'da gizli bilgi taraması; testte log çıktısında token/parola dizisi aranması (örn. giriş isteği sonrası log yakalanıp kontrol edilir); depoda `*.csv` ve `.env` için basit bir denetim betiği.

## 19. Kapsam kayması

- **Belirti:** Hafta 3'te "bir de futbol kuralları sınavı ekleyelim" ya da "canlı maç sayacı yapalım" düşüncesi; Hafta 4'te hâlâ CRUD bitmemiş; README vaat ettiğinden az çalışıyor.
- **Neden:** Konu ilginç (hakemlik ilgisi); fikirler ucuz görünür; "bitmiş" tanımı yok.
- **Çözüm:**
  - "Bitti" tanımı (TASARIM 1.4) ve haftalık bitiş kriterleri.
  - v2 listesi: yeni fikir önce oraya yazılır (TASARIM Bölüm 13), v1 bitmeden kodlanmaz.
  - Taşma politikası (TASARIM Bölüm 12): hangi şeyin sırayla kesileceği önceden bellidir.
  - Her özellik için tek soru: "Bu, karneyi veya testi/CI'ı güçlendiriyor mu?" Hayırsa v2.
  - Haftalık 15 dakikalık gözden geçirme: planlanan / yapılan / kesilen.
- **Nasıl test edilir (ölçülebilir):** Her hafta sonunda bitiş kriterini karşılayıp karşılamadığını işaretleyin; iki hafta üst üste karşılanmazsa kapsamı küçültün. GitHub'da issue/milestone sayısı v1 etiketli olanlarla sınırlı kalmalı; v2 etiketli issue'ların kodlanmaması takip edilir.

## 20. Proje "sadece CRUD" gibi görünüyor

- **Belirti:** Staj için bakan kişi README'yi ve depoyu görüp "yine bir CRUD" diye geçiyor; farkı göremiyor.
- **Neden:** Güçlü yönler (SQL derinliği, test, CI) depoda görünür değil; README özellik listesi gösterişsiz; testler ve CI rozeti yok.
- **Çözüm:**
  - README'nin ilk ekranında karne çıktısının ekran görüntüsü ve "bu sayı şu SQL'den geliyor" örneği.
  - Window function sorgularının `sql/report/` altında okunabilir ve yorumlu olması; TASARIM'da gerekçeleri.
  - Testcontainers testleri, elle hesaplı beklenen değerlerle; CI'ın yeşil olması.
  - `docker compose up` ile tek komutla kurulum; kısa bir "mimari kararlar" bölümü.
  - Dürüstlük: sentetik veri, bilinen sınırlamalar. Abartılı iddia güveni düşürür.
  - Mülakat için hazırlık: "Neden NUMERIC? Neden ROWS? N+1'i nasıl yakaladın?" gibi sorulara kısa cevaplar.
- **Nasıl test edilir:** Bir arkadaşa/hocaya README'yi 2 dakika okutup "bu proje ne yapıyor, neresi zor?" diye sorun; cevapta SQL/test/CI geçmiyorsa README'yi düzeltin.

## 21. Windows satır sonları ve betikler

- **Belirti:** Konteyner içinde `.sh` betiği "bad interpreter" ya da `\r` hatası verir; `git diff` her satırı değişmiş gösterir.
- **Neden:** Windows'ta dosyaların CRLF satır sonuyla kaydedilmesi (genel bilgi).
- **Çözüm:** `.gitattributes` ile `*.sh text eol=lf`; editörde LF; betik sayısını az tutun. SQL migrasyonlarında satır sonu değişikliğinin Flyway checksum'ını etkileyip etkilemediği **doğrulanmadı**; güvenli olan, dosyaları tutarlı satır sonuyla saklamak ve uygulanmış dosyayı yeniden biçimlendirmemektir.
- **Nasıl test edilir:** Temiz klonu Windows'ta ve Linux'ta (veya CI'da) çalıştırıp Compose'un ayağa kalktığını gözlemleyin.

## 22. Eski Boot 3 öğreticileri Boot 4'te çalışmıyor

- **Belirti:** Öğreticideki bağımlılık adı bulunamıyor, `@MockBean` ya da Jackson import'ları derlenmiyor, otomatik yapılandırma "çalışmıyor".
- **Neden:** Boot 4 ile starter adları ve bazı paketler değişti (**kısmen doğrulandı:** bir geçiş rehberi `spring-boot-starter-web` yerine `spring-boot-starter-webmvc`, Jackson 3 paket değişikliği, `@MockBean` yerine `@MockitoBean` gibi farkları listeliyor). Spring Boot 3.5 desteği ise 30 Haziran 2026'da bitti (**doğrulandı**, HeroDevs).
- **Çözüm:** Resmi Boot 4.1 dokümanı ve start.spring.io çıktısı temel alınır; üçüncü taraf blog yazısı yalnızca ipucu olarak kullanılır. Karar: Boot 4.x ile başlayın; zaman kaybederseniz geçici olarak bilinen bir 4.x örnek projeyle karşılaştırın.
- **Nasıl test edilir:** Hafta 1 bitiş kriteri (iskelet + Flyway + Testcontainers + CI) bu uyumsuzlukları erken yakalar.

---

## Ek: Haftalık güvenlik ve kalite kontrol listesi (Hafta 5 öncesi)

- [ ] Depoda gizli anahtar, `.env`, gerçek veri yok
- [ ] Swagger UI üretim/halka açık dağıtımda kapalı ya da korumalı (açık bırakmanın riski: tüm API yüzeyi görünür)
- [ ] `/actuator` yalnızca `health` açık
- [ ] Tüm sorgularda `referee_id` denetimi test edilmiş
- [ ] Hata yanıtlarında yığın izi yok
- [ ] README'deki sayılar "örnek" etiketli
- [ ] Temiz makinede kurulum provası yapıldı
