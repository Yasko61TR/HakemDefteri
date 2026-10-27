# HakemDefteri

> **Not (yapay zekâ kullanımı):** Bu belge, yapay zekâ (Claude) yardımıyla hazırlanmış bir taslaktır. Proje ilerledikçe gerçek gelişmelere göre güncellenecektir. "Doğrulanmadı" ve "ÖRNEK" etiketli yerler henüz kontrol edilmemiştir.

> Amatör ve bölge hakemleri için maç, kart, ücret ve yol masrafı defteri; sezon sonunda PostgreSQL window function'larıyla hesaplanan bir **sezon karnesi**.

![Java](https://img.shields.io/badge/Java-21-orange) ![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.x-brightgreen) ![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16%2B-blue) ![License](https://img.shields.io/badge/license-MIT-lightgrey)

> **Proje durumu: tasarım aşaması.** Aşağıdaki özellik listesindeki kutucuklar gerçek ilerlemeyi gösterir; işaretli olmayan hiçbir şey henüz çalışmıyor. Rozetlerdeki sürümler hedeftir.
> (Rozet adresleri ve depo bağlantıları, depo oluşturulunca eklenecek: `<GITHUB-KULLANICI>/<DEPO-ADI>` yer tutucusunu doldurun.)

---

## İçindekiler

1. [Neden böyle bir proje?](#neden-böyle-bir-proje)
2. [Neden farklı? (dürüst karşılaştırma)](#neden-farklı-dürüst-karşılaştırma)
3. [Özellikler ve durum](#özellikler-ve-durum)
4. [Hızlı kurulum (Docker Compose)](#hızlı-kurulum-docker-compose)
5. [API kullanım örnekleri](#api-kullanım-örnekleri)
6. [Sezon karnesi ve "SQL'i göster"](#sezon-karnesi-ve-sqli-göster)
7. [Mimari özeti](#mimari-özeti)
8. [Veri modeli (ER diyagramı)](#veri-modeli-er-diyagramı)
9. [Test ve CI](#test-ve-ci)
10. [Örnek (sentetik) veri ve kişisel veri notu](#örnek-sentetik-veri-ve-kişisel-veri-notu)
11. [Yol haritası](#yol-haritası)
12. [Ekran görüntüleri](#ekran-görüntüleri)
13. [Bilinen sınırlamalar](#bilinen-sınırlamalar)
14. [Katkı ve lisans](#katkı-ve-lisans)

---

## Neden böyle bir proje?

Amatör ve bölge liglerinde görev yapan hakemler çoğunlukla maçlarını, kartlarını, ücretlerini ve yol masraflarını defterde, telefon notlarında ya da dağınık tablolarda tutar. Sezon sonunda şu sorulara cevap vermek zorlaşır:

- Bu sezon maç başına ortalama kaç kart gösterdim, son 3 maçta eğilim nasıl?
- Hangi takımların maçlarını kaç kez yönettim, onlarla ilgili geçmişim ne?
- Ay ay ne kadar kazandım, yol masrafım bunun ne kadarı, ödenmeyen ücretlerim var mı?

HakemDefteri bu kayıtları tek bir yerde, sorgulanabilir biçimde tutar.

Bu proje aynı zamanda bir **portföy projesidir**: backend, veritabanı tasarımı, migrasyon, test ve CI pratiğini göstermek için yazılmıştır. Yazar, futbol hakemliğiyle ilgilenen bir yazılım mühendisliği öğrencisidir; alan bilgisi gerçek, ancak proje herhangi bir federasyon, kulüp ya da kurumla ilişkili değildir.

## Neden farklı? (dürüst karşılaştırma)

Yapılan sınırlı araştırmada, **Türkçe ve hakemin kendi kayıtlarına odaklanan** doğrudan bir rakip bulunamadı. Bu, "rakip yok" demek **değildir**; yalnızca arama sonucunda bulunamadı. Kapsamlı bir pazar araştırması yapılmadı.

| Ürün | Odak (araştırmada gördüğümüz kadarıyla) | HakemDefteri ile fark |
|---|---|---|
| Askf PRO, Amatör360 | Lig, fikstür ve skor odaklı | Hakemin kişisel kaydı değil, lig/maç yönetimi odaklı görünüyor. Doğrulanmadı: hakem özelliği olup olmadığı tam incelenmedi. |
| REFSIX, Ref90 (yurt dışı) | Maç içi sayaç ve olay kaydı | Maç sırasında kullanılan araçlar; HakemDefteri maç **sonrası** kayıt ve sezon analizi yapar. Türkçe değil. |

HakemDefteri'nin iddiası küçük ve somuttur:

- Hakemin **kendi** verisi (ücret, yol masrafı, kart geçmişi) tek yerde.
- Sezon karnesi, **gerçek SQL window function'larıyla** hesaplanır ve her istatistiğin hangi sorgudan geldiği görülebilir.
- Niş bir kitleye hitap eder; bu bilinçli bir tercihtir ve aynı zamanda bir risktir (bkz. [Bilinen sınırlamalar](#bilinen-sınırlamalar)).

Ürünlerin özellikleri değişebilir; yukarıdaki tablo yazarın gözlemidir, resmi bir karşılaştırma değildir.

## Özellikler ve durum

Kutucuklar yalnızca gerçekten tamamlanıp test edilen işler için işaretlenir.

### Aşama 0 - Temel
- [ ] Spring Boot iskeleti, Java 21, Maven
- [ ] Docker Compose ile tek komutla uygulama + PostgreSQL
- [ ] Flyway migrasyonları ile şema oluşturma
- [ ] GitHub Actions CI (build + test)

### Aşama 1 - MVP
- [ ] Hakem kaydı ve girişi (JWT)
- [ ] Takım, saha, lig kayıtları
- [ ] Maç ekleme (tarih, takımlar, skor, görev: orta / yardımcı / 4. hakem)
- [ ] Olay kaydı (sarı, kırmızı, penaltı, gol, dakika)
- [ ] Maç ücreti ve yol masrafı
- [ ] Listeleme, filtreleme, sayfalama
- [ ] CSV dışa aktarma
- [ ] OpenAPI / Swagger UI
- [ ] Testcontainers ile entegrasyon testleri

### Aşama 2 - Sezon karnesi (öne çıkan kısım)
- [ ] Maç başına kart ortalaması (kümülatif ve hareketli ortalama)
- [ ] Takım bazlı geçmiş (RANK, ROW_NUMBER, LAG)
- [ ] Aylık kazanç ve kümülatif toplam
- [ ] Yanıtta "bu istatistik şu sorgudan geliyor" gösterimi
- [ ] Sentetik veri üreten seed aracı

### Aşama 3 - İsteğe bağlı
- [ ] Tek sayfalık sade web arayüzü (htmx veya düz HTML) **veya** yalnızca Swagger UI

Kapsam dışı öğeler için bkz. [Yol haritası](#yol-haritası).

## Hızlı kurulum (Docker Compose)

> Bu bölüm hedeflenen kullanımı anlatır; depo kodu yazıldıkça doğrulanacaktır.

**Gereksinimler:** Docker Desktop (Windows/macOS) veya Docker Engine + Compose eklentisi (Linux). Java kurulumu **gerekmez**, çünkü derleme Docker içinde yapılır. Yerel geliştirme için ayrıca JDK 21 ve Maven gerekir.

```bash
git clone https://github.com/<GITHUB-KULLANICI>/<DEPO-ADI>.git
cd <DEPO-ADI>
cp .env.example .env        # değerleri düzenleyin; .env dosyasını commit etmeyin
docker compose up --build
```

Beklenen sonuç (hedef):

- API: `http://localhost:8080`
- Swagger UI: `http://localhost:8080/swagger-ui.html` (yol, kullanılan springdoc sürümüne göre değişebilir; kurulumda doğrulanacak)
- PostgreSQL: yalnızca konteyner ağında; yerel porta açmak isteğe bağlıdır.

Örnek veriyi yüklemek için (planlanan, `dev` profili):

```bash
docker compose --profile seed up seed
```

Durdurmak ve veriyi silmek için: `docker compose down -v` (**dikkat:** `-v` veritabanı verisini siler).

`.env.example` içinde bulunması planlanan değişkenler: `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `JWT_SECRET` (en az 32 bayt rastgele değer), `JWT_EXPIRY_MINUTES`. Gerçek değerler depoya konmaz.

## API kullanım örnekleri

Tüm örnekler **biçimi göstermek içindir**. İçindeki isimler, tarihler ve sayılar uydurmadır; gerçek bir kişi, takım veya maçı temsil etmez. Uç nokta adları taslaktır; kesin liste çalışan Swagger belgesidir.

### 1) Kayıt ve giriş

```http
POST /api/v1/auth/register
Content-Type: application/json

{ "email": "ornek.hakem@example.com", "password": "<en az 12 karakter>", "fullName": "Örnek Hakem" }
```

```http
POST /api/v1/auth/login
Content-Type: application/json

{ "email": "ornek.hakem@example.com", "password": "<parola>" }
```

Örnek yanıt:

```json
{ "accessToken": "<JWT>", "tokenType": "Bearer", "expiresInSeconds": 3600 }
```

Sonraki isteklerde: `Authorization: Bearer <JWT>`

### 2) Maç ekleme

```http
POST /api/v1/matches
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "seasonId": 1,
  "leagueId": 3,
  "venueId": 7,
  "homeTeamId": 11,
  "awayTeamId": 12,
  "kickoffAt": "2026-03-14T13:00:00+03:00",
  "homeScore": 2,
  "awayScore": 1,
  "role": "CENTER",
  "fee": { "amount": "1500.00", "currency": "TRY" },
  "travel": { "distanceKm": 42.5, "cost": "260.00" },
  "events": [
    { "type": "YELLOW", "minute": 34, "teamId": 12, "shirtNumber": 6 },
    { "type": "GOAL",   "minute": 51, "teamId": 11, "shirtNumber": 9 }
  ]
}
```

Notlar: Para tutarları JSON'da **string** olarak gönderilir (kayan nokta hatası olmaması için). Saat bilgisi saat dilimi ofsetiyle gelir.

### 3) Filtreleme ve CSV

```http
GET /api/v1/matches?from=2026-02-01&to=2026-03-31&teamId=12&role=CENTER&page=0&size=20
GET /api/v1/matches/export.csv?seasonId=1
```

### 4) Sezon karnesi (örnek yanıt biçimi)

```http
GET /api/v1/seasons/1/report?include=sql
```

```json
{
  "season": "2025-26",
  "cardAverage": {
    "value": "<örnek: 3.40>",
    "movingAverage3": [ { "matchId": 101, "value": "<örnek>" } ],
    "source": { "id": "card-moving-avg", "sql": "SELECT ... AVG(cards) OVER (... ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) ..." }
  },
  "monthlyEarnings": [ { "month": "2026-03", "net": "<örnek>", "runningTotal": "<örnek>" } ]
}
```

Yukarıdaki sayılar yer tutucudur; gerçek değerler seed verisiyle üretilir ve **sentetiktir**.

## Sezon karnesi ve "SQL'i göster"

Her istatistik, depoda `src/main/resources/sql/report/` altında duran **sabit, adı verilmiş** bir SQL dosyasından gelir. API, `include=sql` verildiğinde bu dosyanın metnini ve sorgu kimliğini yanıta ekler. İstemci hiçbir zaman SQL gönderemez; yalnızca önceden tanımlı sorgu çalıştırılır (SQL enjeksiyonu ve veri sızıntısı riskini önlemek için). Ayrıntı: [docs/TASARIM.md](docs/TASARIM.md).

## Mimari özeti

```
İstemci (Swagger UI / curl / sade web sayfası)
        |  HTTPS + JWT
        v
 Controller  ->  Service  ->  Repository  ->  PostgreSQL
  (DTO, doğrulama)  (iş kuralı,   (JPA + özel     (Flyway ile
                     yetki)        SQL raporları)   yönetilen şema)
```

- Katmanlı mimari: `controller` / `service` / `repository`, DTO'lar entity'lerden ayrı.
- Şema yalnızca Flyway ile değişir; Hibernate şemayı değiştirmez (`validate`).
- Rapor sorguları ham SQL (window function) ile yazılır; basit CRUD için JPA kullanılır.
- Ayrıntılı paket yapısı ve kararlar: [docs/TASARIM.md](docs/TASARIM.md).

## Veri modeli (ER diyagramı)

> **Yer tutucu:** ER diyagramı, şema tamamlandığında buraya eklenecek (`docs/img/er-diagram.png`).

![ER diyagramı - yer tutucu](docs/img/er-diagram.png)

Ana tablolar: `referee`, `season`, `league`, `team`, `venue`, `match`, `match_event`. Taslak `CREATE TABLE` ifadeleri [docs/TASARIM.md](docs/TASARIM.md) içindedir.

## Test ve CI

- **Birim testleri:** iş kuralları (ücret hesabı, doğrulama) veritabanı olmadan.
- **Entegrasyon testleri:** Testcontainers ile gerçek PostgreSQL; Flyway migrasyonları çalışır, rapor sorguları sentetik veriyle doğrulanır.
- **CI:** GitHub Actions; her push ve PR'de derleme + testler.
- Entegrasyon testleri için Docker çalışıyor olmalıdır. Windows notları: [docs/SORUNLAR_VE_COZUMLER.md](docs/SORUNLAR_VE_COZUMLER.md).

CI rozeti, workflow dosyası yazıldıktan sonra eklenecektir (şimdilik yok).

## Örnek (sentetik) veri ve kişisel veri notu

Gerçek maç verisi **yoktur ve depoya konmaz**. Örnek veriyi bir seed aracı üretir: takım, saha ve kişi adları kurgusaldır, gerçek kulüp veya kişilerle eşleşmesi kasıtlı değildir. Rastgelelik sabit bir tohumla (seed) üretilir, yani herkes aynı veriyi alır.

Olay kayıtlarında oyuncu adı **tutulmaz**; yalnızca forma numarası saklanır. Bu, kişisel veri riskini azaltmak için bilinçli bir tasarım kararıdır. Gerçek veriyle kullanım yapacaksanız [docs/TASARIM.md](docs/TASARIM.md) içindeki anonimleştirme yönergesine bakın. Bu proje hukuki tavsiye vermez.

## Yol haritası

**v1 (bu depo):** Aşama 0-2 yukarıdaki gibi.

**v2 fikirleri (bilinçli olarak ilk sürümün dışında):**

- Futbol kuralları sınavı
- Takım geçmişi uyarısı ("bu takımı son 60 günde yönettin" gibi)
- Canlı maç takibi
- Mobil uygulama
- Çoklu kullanıcı rolleri (ör. gözlemci, klasman yetkilisi)
- Yenileme (refresh) token'ı, e-posta doğrulama, parola sıfırlama

Bu liste, kapsam kaymasını önlemek için kasıtlı olarak ayrı tutulur.

## Ekran görüntüleri

| Swagger UI | Sezon karnesi yanıtı | (varsa) Sade web sayfası |
|---|---|---|
| ![yer tutucu](docs/img/swagger.png) | ![yer tutucu](docs/img/report.png) | ![yer tutucu](docs/img/web.png) |

*Yer tutucular; görseller uygulama çalışınca eklenecek.*

## Bilinen sınırlamalar

- Niş bir kitleye hitap eder; kullanıcı sayısı küçük olabilir.
- Gerçek maç verisiyle doğrulanmadı; tüm örnek sayılar sentetiktir.
- Kart, ücret ve yol masrafı kurallarının resmi tarifeleri bölgeye ve lige göre değişir; uygulama bu tutarları **kullanıcının girdiği** değerler olarak saklar, resmi bir hesaplama yapmaz.
- Ticari ürün değildir; üretim ortamı güvenlik denetiminden geçmemiştir.

## Katkı ve lisans

Bu bir öğrenme / portföy projesidir. Hata ve öneriler için issue açabilirsiniz.

Lisans: **MIT** (önerilen). Depoya `LICENSE` dosyası eklenmelidir; yazar adı ve yılı yazarın kendisi tarafından doldurulur. Üçüncü taraf bağımlılıkların kendi lisansları geçerlidir.
