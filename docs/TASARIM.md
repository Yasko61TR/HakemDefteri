# HakemDefteri - Tasarım Dokümanı

> **Not (yapay zekâ kullanımı):** Bu belge, yapay zekâ (Claude) yardımıyla hazırlanmış bir taslaktır. Proje ilerledikçe gerçek gelişmelere göre güncellenecektir. "Doğrulanmadı" ve "ÖRNEK" etiketli yerler henüz kontrol edilmemiştir.

Sürüm: taslak 0.1 | Durum: tasarım aşaması | Dil: Türkçe

Bu belge, README'de verilen sözün teknik karşılığıdır. Kod içermez; yalnızca şema taslağı, örnek sorgular ve kararlar içerir.

## İçindekiler

1. Kapsam ve gereksinimler
2. Teknoloji seçimleri ve doğrulama durumu
3. Katmanlı mimari ve paket yapısı
4. Veri modeli
5. Flyway migrasyon planı
6. REST uç noktaları
7. Window function'lı örnek sorgular ("SQL'i göster")
8. Kimlik doğrulama tasarımı
9. Test stratejisi
10. Seed (sentetik veri) tasarımı
11. Gerçek veriyi anonimleştirme yönergesi
12. 5 haftalık plan
13. v2 fikirleri
14. Açık sorular

---

## 1. Kapsam ve gereksinimler

### 1.1 Fonksiyonel gereksinimler (MVP)

| Kod | Gereksinim | Öncelik |
|---|---|---|
| F1 | Hakem kayıt olur ve giriş yapar | Zorunlu |
| F2 | Hakem kendi takım, saha ve lig kayıtlarını yönetir | Zorunlu |
| F3 | Hakem maç ekler: tarih/saat, takımlar, skor, görev (orta / yardımcı / 4. hakem) | Zorunlu |
| F4 | Maça olay ekler: sarı, ikinci sarı, kırmızı, penaltı, gol; dakika ile | Zorunlu |
| F5 | Maça ücret ve yol masrafı girilir | Zorunlu |
| F6 | Maçlar tarih, takım, lig, görev, sezon ile filtrelenip sayfalanır | Zorunlu |
| F7 | Maçlar CSV olarak dışa aktarılır | Zorunlu |
| F8 | Sezon karnesi: kart ortalaması, hareketli ortalama, takım bazlı geçmiş, aylık kazanç | Zorunlu (öne çıkan) |
| F9 | Karne yanıtında her istatistiğin SQL'i görülebilir | Zorunlu (öne çıkan) |
| F10 | Ücretin ödenip ödenmediği izlenir | Olursa iyi |

### 1.2 Fonksiyonel olmayan gereksinimler

- **Kurulum:** `docker compose up --build` ile tek komut.
- **Taşınabilirlik:** Windows, macOS ve Linux'ta çalışmalı.
- **Güvenlik:** parolalar hash'li, JWT doğru yapılandırılmış, her kayıt sahibine ait (başkasının verisi okunamaz).
- **Doğruluk:** para `NUMERIC`, zaman `timestamptz`.
- **Gözlemlenebilirlik:** temel sağlık ucu ve yapılandırılmış log (kişisel veri loglanmaz).
- **Bakım:** şema yalnızca Flyway ile değişir.

### 1.3 Kapsam dışı (v1)

Futbol kural sınavı, takım geçmişi uyarısı, canlı maç takibi, mobil uygulama, çoklu kullanıcı rolleri, JavaFX masaüstü arayüz. Bölüm 13'e bakın.

### 1.4 Kabul ölçütü ("bitti" tanımı)

v1 şu durumda bitmiştir: temiz bir makinede `docker compose up --build` çalışır, seed yüklenir, Swagger'dan giriş yapıp maç eklenir, karne endpoint'i SQL ile birlikte yanıt verir, CI yeşildir ve README'deki kutucuklar gerçeği yansıtır.

---

## 2. Teknoloji seçimleri ve doğrulama durumu

| Konu | Karar | Doğrulama |
|---|---|---|
| Dil | Java 21 | Spring Boot 4.1 dokümanı minimum Java 17, Java 27'ye kadar uyumlu diyor; Java 21 desteklenir. **Doğrulandı** (docs.spring.io, 4.1 sistem gereksinimleri). |
| Çatı | Spring Boot 4.x (4.1 güncel kararlı hat) | Boot 3.5.x açık kaynak desteği 30 Haziran 2026'da bitmiş; 4.0.x 31 Aralık 2026'ya kadar; 4.1 haziran 2026'da çıkmış. **Doğrulandı** (HeroDevs yazısı, nisan 2026 tarihli; 4.1 sürüm tarihi oradan alındı). Tam yama sürümünü start.spring.io'dan seçin; sürüm numaralarını burada sabitlemiyoruz. |
| Maven | 3.6.3+ | **Doğrulandı** (4.1 sistem gereksinimleri). Maven wrapper (`mvnw`) kullanılacak. |
| Gömülü sunucu | Tomcat 11 | **Doğrulandı** (aynı sayfa). |
| Flyway | Boot 4'te `spring-boot-starter-flyway`; PostgreSQL için ayrıca `flyway-database-postgresql` | Starter gereği: bir Boot 3→4 geçiş yazısında `flyway-core` yerine starter gerektiği, yoksa Flyway'in sessizce çalışmadığı belirtiliyor (**kısmen doğrulandı**, tek ikincil kaynak). `flyway-database-postgresql` gereği: Flyway 10 ile geldi, Spring Boot PR #41574 bunu belgeliyor (**doğrulandı**). |
| API belgesi | springdoc-openapi 3.x | springdoc 3.1.0 sürüm notunda "Boot 4.1.0'a yükseltildi" yazıyor (**doğrulandı**: 3.1.0 için). Diğer 3.x sürümlerinin hangi Boot ile uyumlu olduğu **doğrulanmadı**; springdoc'un uyumluluk tablosuna bakın. Swagger UI yolu **doğrulanmadı**. |
| Entegrasyon testi | Testcontainers 2.x + PostgreSQL modülü | `org.testcontainers:testcontainers-postgresql` 2.0.x artifact'i Maven Central'da var (**doğrulandı**). Boot 4.1'in hangi Testcontainers sürümünü yönettiği **doğrulanmadı**; `@ServiceConnection` kullanımı Boot 4.1 dokümanından kontrol edilmeli. |
| Veritabanı | PostgreSQL 16 veya üstü, Docker imajı ile | Window function'lar standarttır ve uzun süredir desteklenir (genel bilgi). Hedef sürüm yazarın seçimi. |
| CI | GitHub Actions | GitHub barındırmalı Ubuntu runner'larında Docker bulunur (genel bilgi, **bu çalışmada yeniden doğrulanmadı**); ilk workflow çalışınca teyit edin. |
| Docker Engine 29 | Minimum API sürümü 1.44'e çıkmış | **Doğrulandı** (Spring Boot issue #48050, Docker Engine 29.0). Testcontainers'ın bundan etkilenip etkilenmediği ve düzeltme sürümü **doğrulanmadı**. |

**Boot 4 notu:** Boot 3'ten 4'e geçişte starter adları (ör. `spring-boot-starter-web` yerine `spring-boot-starter-webmvc`), Jackson 3 paket adları ve test notasyonları değişmiş (ikincil kaynak: bir geçiş rehberi; **kısmen doğrulandı**). İnternetteki eski Boot 3 öğreticileri doğrudan çalışmayabilir. Bunu bilmek, Hafta 1'de zaman kaybını önler. Boot 3.5 öğrenme kolaylığı sağlasa da destek süresi bitmiştir; CV için güncel hat daha iyi görünür.

---

## 3. Katmanlı mimari ve paket yapısı

### 3.1 Katmanlar

```
controller   HTTP, DTO <-> servis çağrısı, @Valid doğrulama, yetki bilgisi alma
service      iş kuralı, sahiplik denetimi, transaction sınırı
repository   veri erişimi (basit CRUD: JPA; raporlar: adlandırılmış SQL dosyaları)
domain       entity'ler ve sabit tipler (enum benzeri)
```

Kurallar:

- Controller doğrudan repository çağırmaz.
- Entity'ler API'den dışarı çıkmaz; DTO kullanılır.
- Transaction sınırı service katmanındadır.
- Her service metodu "bu kayıt oturumdaki hakeme ait mi?" denetimini yapar (sahiplik).

### 3.2 Paket yapısı (öneri: önce özellik, sonra katman)

```
com.<kullanici>.hakemdefteri
 ├─ HakemDefteriApplication
 ├─ config/            güvenlik, OpenAPI, JSON ayarları
 ├─ common/            hata modeli, sayfalama, para/zaman yardımcıları
 ├─ auth/              controller, service, JWT üretim/doğrulama, DTO
 ├─ referee/           profil
 ├─ reference/         league, team, venue, season (controller/service/repository)
 ├─ match/             match + match_event (controller/service/repository/dto)
 ├─ export/            CSV üretimi
 └─ report/            karne controller/service, SqlReportRepository, ReportQuery tanımları
src/main/resources
 ├─ db/migration/      V1__..., V2__...
 ├─ sql/report/        card-cumulative-avg.sql, card-moving-avg.sql, ...
 └─ application.yml, application-dev.yml
src/test/java         birim ve entegrasyon testleri
```

Neden özellik bazlı: küçük projede ilgili dosyalar yan yana durur, kapsam kaymasını fark etmek kolaylaşır. Alternatif katman bazlı yapı da kabul edilebilir; önemli olan tutarlılıktır.

### 3.3 Tasarım kararları (ADR özeti)

| Karar | Seçim | Gerekçe | Bedel |
|---|---|---|---|
| Veri erişimi | CRUD için Spring Data JPA, raporlar için ham SQL (`JdbcClient` veya `JdbcTemplate`; hangisi Boot 4.1'de önerildiği doğrulanmadı) | Window function'ları JPQL ile ifade etmek zor; SQL'i görünür yapmak zaten gerekli | İki erişim yöntemi öğrenmek |
| Referans veri sahipliği | Takım/saha/lig her hakeme ait (`referee_id`) | Paylaşımlı veri, çoklu rol ve moderasyon gerektirir (v2); ayrıca isim çakışması sorunlarını önler | Aynı takım farklı hakemlerde tekrar girilir |
| Şema yönetimi | Yalnızca Flyway; `spring.jpa.hibernate.ddl-auto=validate` | Şema sürümlü ve tekrarlanabilir | Her değişiklik için migrasyon yazmak |
| Para | `NUMERIC(12,2)` + `currency char(3)` | Kayan nokta hatası yok | Java'da `BigDecimal` ile çalışmak |
| Zaman | `timestamptz` (UTC saklanır), gösterim/aylık gruplama `Europe/Istanbul` | Ay sınırı hatalarını önler | Sorgularda `AT TIME ZONE` gerekir |
| Olayda oyuncu | Ad yok, yalnızca forma numarası | Kişisel veriyi en baştan azaltır | Oyuncu bazlı istatistik yapılamaz (kapsam dışı) |
| Görev/olay türü | `varchar` + `CHECK` kısıtı | PostgreSQL `ENUM` değerini sonradan kaldırmak/yeniden adlandırmak zahmetli | Uygulamada aynı değerlerin enum olarak tekrarı |

---

## 4. Veri modeli

### 4.1 İlişkiler

```
referee 1 ─── * league
referee 1 ─── * team
referee 1 ─── * venue
season  1 ─── * match        (season global; referee_id yok)
referee 1 ─── * match
league  1 ─── * match
venue   1 ─── * match
team    1 ─── * match (home_team_id)
team    1 ─── * match (away_team_id)
match   1 ─── * match_event
team    1 ─── * match_event  (olayın ait olduğu takım; opsiyonel)
```

Not: `season` tablosu ortak olduğu için hakemler arasında paylaşılır (ör. "2025-26"). Karar: sezon sınırları sabit seed ile gelir. Alternatif: sezon de hakeme ait olsun. Bu, açık sorular bölümündedir.

### 4.2 CREATE TABLE taslakları (PostgreSQL)

Taslaktır; ilk migrasyon yazılırken gözden geçirilecektir.

```sql
-- V1__create_referee.sql
CREATE TABLE referee (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email         VARCHAR(254) NOT NULL,
    password_hash VARCHAR(100) NOT NULL,
    full_name     VARCHAR(120) NOT NULL,
    region        VARCHAR(80),
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_referee_email ON referee (lower(email));

-- V2__create_reference_tables.sql
CREATE TABLE season (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    label      VARCHAR(20) NOT NULL UNIQUE,      -- örn. '2025-26'
    start_date DATE NOT NULL,
    end_date   DATE NOT NULL,
    CONSTRAINT ck_season_dates CHECK (end_date > start_date)
);

CREATE TABLE league (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    referee_id BIGINT NOT NULL REFERENCES referee(id) ON DELETE CASCADE,
    name       VARCHAR(120) NOT NULL,
    level      VARCHAR(60)
);
CREATE UNIQUE INDEX ux_league_owner_name ON league (referee_id, lower(name));

CREATE TABLE team (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    referee_id BIGINT NOT NULL REFERENCES referee(id) ON DELETE CASCADE,
    name       VARCHAR(120) NOT NULL
);
CREATE UNIQUE INDEX ux_team_owner_name ON team (referee_id, lower(name));

CREATE TABLE venue (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    referee_id BIGINT NOT NULL REFERENCES referee(id) ON DELETE CASCADE,
    name       VARCHAR(120) NOT NULL,
    city       VARCHAR(80),
    district   VARCHAR(80)
);
CREATE UNIQUE INDEX ux_venue_owner_name ON venue (referee_id, lower(name));

-- V3__create_match.sql
CREATE TABLE match (
    id                BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    referee_id        BIGINT NOT NULL REFERENCES referee(id) ON DELETE CASCADE,
    season_id         BIGINT NOT NULL REFERENCES season(id),
    league_id         BIGINT NOT NULL REFERENCES league(id),
    venue_id          BIGINT REFERENCES venue(id),
    home_team_id      BIGINT NOT NULL REFERENCES team(id),
    away_team_id      BIGINT NOT NULL REFERENCES team(id),
    kickoff_at        TIMESTAMPTZ NOT NULL,
    home_score        SMALLINT NOT NULL DEFAULT 0,
    away_score        SMALLINT NOT NULL DEFAULT 0,
    role              VARCHAR(10) NOT NULL,        -- CENTER | ASSISTANT | FOURTH
    fee_amount        NUMERIC(12,2) NOT NULL DEFAULT 0,
    travel_distance_km NUMERIC(6,1),
    travel_cost       NUMERIC(12,2) NOT NULL DEFAULT 0,
    currency          CHAR(3) NOT NULL DEFAULT 'TRY',
    fee_paid_on       DATE,                        -- NULL = henüz ödenmedi
    notes             VARCHAR(500),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT ck_match_teams_differ CHECK (home_team_id <> away_team_id),
    CONSTRAINT ck_match_scores       CHECK (home_score >= 0 AND away_score >= 0),
    CONSTRAINT ck_match_role         CHECK (role IN ('CENTER','ASSISTANT','FOURTH')),
    CONSTRAINT ck_match_money        CHECK (fee_amount >= 0 AND travel_cost >= 0),
    CONSTRAINT ck_match_distance     CHECK (travel_distance_km IS NULL OR travel_distance_km >= 0)
);
CREATE INDEX ix_match_referee_kickoff ON match (referee_id, kickoff_at DESC);
CREATE INDEX ix_match_referee_season  ON match (referee_id, season_id);
CREATE INDEX ix_match_home_team       ON match (home_team_id);
CREATE INDEX ix_match_away_team       ON match (away_team_id);

-- V4__create_match_event.sql
CREATE TABLE match_event (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    match_id      BIGINT NOT NULL REFERENCES match(id) ON DELETE CASCADE,
    event_type    VARCHAR(15) NOT NULL,   -- YELLOW | SECOND_YELLOW | RED | PENALTY | GOAL
    minute        SMALLINT NOT NULL,
    stoppage_minute SMALLINT,
    team_id       BIGINT REFERENCES team(id),
    shirt_number  SMALLINT,               -- oyuncu adı TUTULMAZ
    note          VARCHAR(200),
    CONSTRAINT ck_event_type   CHECK (event_type IN ('YELLOW','SECOND_YELLOW','RED','PENALTY','GOAL')),
    CONSTRAINT ck_event_minute CHECK (minute BETWEEN 0 AND 130),
    CONSTRAINT ck_event_shirt  CHECK (shirt_number IS NULL OR shirt_number BETWEEN 1 AND 99)
);
CREATE INDEX ix_event_match      ON match_event (match_id);
CREATE INDEX ix_event_match_type ON match_event (match_id, event_type);
```

### 4.3 Bilinen şema sorunları (taslakta ele alınacak)

1. **Takımların aynı hakeme ait olmasını veritabanı garanti etmez.** `match.home_team_id` başka hakemin takımını gösterebilir. Çözüm seçenekleri: (a) servis katmanında denetim (MVP için yeterli, testle kapsanır), (b) bileşik yabancı anahtar `(referee_id, id)` ile veritabanında zorlama. Öneri: (a) ile başla, (b)'yi sorunlar belgesinde belgelenen iyileştirme olarak sakla.
2. **Gol skoru ile `GOAL` olayları tutarsız olabilir.** Skor `match` üzerinde ayrı tutuluyor; `GOAL` olayları isteğe bağlı. Kural: skor esastır, gol olayı isteğe bağlı ayrıntıdır; API uyuşmazlığı reddetmez ama raporlar skoru kullanır. Bu karar README'de de söylenmelidir.
3. **Para birimi:** MVP'de yalnızca `TRY` kabul edilir (`CHECK` eklenebilir); kolon ileride genişlemeye izin verir. Karışık para birimi toplamak anlamsızdır.
4. **`match` kelimesi** PostgreSQL'de ayrılmış sözcük değildir ancak bazı araçlarda karışabilir; isterseniz `football_match` adını kullanın. Karar yazarın.

---

## 5. Flyway migrasyon planı

Konum: `src/main/resources/db/migration`. Adlandırma: `V<sürüm>__<açıklama>.sql`.

| Sürüm | İçerik |
|---|---|
| V1 | `referee` |
| V2 | `season`, `league`, `team`, `venue` |
| V3 | `match` + indeksler |
| V4 | `match_event` + indeksler |
| V5 | `season` için başlangıç verisi (2024-25, 2025-26, 2026-27 gibi; sınırlar yazarın seçimi) |
| V6+ | Sonraki değişiklikler (örn. `fee_paid_on`, ek indeksler) |

Kurallar:

- Uygulanmış bir migrasyon **asla düzenlenmez**; hata varsa yeni migrasyon eklenir.
- Her PR en fazla bir yeni migrasyon ekler; sürüm numarası çakışması için PR öncesi `main` ile rebase edilir.
- Seed verisi migrasyon **değildir**; ayrı araçla `dev` profilinde yüklenir (Bölüm 10).
- Hibernate `ddl-auto=validate`.
- Testcontainers testleri sıfır veritabanından tüm migrasyonları çalıştırır, böylece "sıfırdan kurulum" sürekli test edilir.
- Geri alma (undo) migrasyonu kullanılmaz; ileri düzeltme tercih edilir.

---

## 6. REST uç noktaları

Taban yol: `/api/v1`. Kimlik doğrulama gerektirmeyenler işaretli. Tüm listeler `page`, `size`, `sort` kabul eder. Hata biçimi: RFC 9457 `application/problem+json` (Spring bunu destekler; Boot 4.1'de ayar adı **doğrulanmadı**).

| Yöntem | Yol | Açıklama | Kimlik |
|---|---|---|---|
| POST | `/auth/register` | Kayıt | Gerekmez |
| POST | `/auth/login` | Giriş, JWT döner | Gerekmez |
| GET | `/me` | Profilim | Gerekli |
| PUT | `/me` | Profili güncelle | Gerekli |
| GET/POST | `/leagues`, `/teams`, `/venues` | Liste / ekle | Gerekli |
| GET/PUT/DELETE | `/leagues/{id}`, `/teams/{id}`, `/venues/{id}` | Oku / güncelle / sil (maçta kullanılıyorsa 409) | Gerekli |
| GET | `/seasons` | Sezon listesi | Gerekli |
| GET | `/matches` | Filtreli liste: `from`, `to`, `seasonId`, `leagueId`, `teamId`, `role`, `unpaid=true` | Gerekli |
| POST | `/matches` | Maç ekle (olaylarla birlikte isteğe bağlı) | Gerekli |
| GET | `/matches/{id}` | Detay + olaylar | Gerekli |
| PUT | `/matches/{id}` | Güncelle | Gerekli |
| DELETE | `/matches/{id}` | Sil | Gerekli |
| POST | `/matches/{id}/events` | Olay ekle | Gerekli |
| DELETE | `/matches/{id}/events/{eventId}` | Olay sil | Gerekli |
| PATCH | `/matches/{id}/payment` | Ödendi olarak işaretle | Gerekli |
| GET | `/matches/export.csv` | CSV (aynı filtreler) | Gerekli |
| GET | `/seasons/{id}/report` | Karne; `include=sql` ile SQL metni | Gerekli |
| GET | `/seasons/{id}/report/{queryId}` | Tek istatistik ve kaynak sorgusu | Gerekli |
| GET | `/actuator/health` | Sağlık | Gerekmez (yalnızca health) |

Kararlar:

- Başkasının kaydına erişim denemesi `404` döner (varlığı ifşa etmemek için `403` yerine).
- Güncellemede iyimser kilit için `version` alanı v2'ye bırakılabilir (açık soru).
- CSV: ayırıcı ve kodlama kararı Sorunlar belgesindedir (Excel Türkçe yerel ayarı).

---

## 7. Window function'lı örnek sorgular

Bunlar tasarım taslağıdır; gerçek şemada çalıştırılıp `EXPLAIN` ile kontrol edilmelidir. Hepsi `src/main/resources/sql/report/` altında birer dosya olacak ve `:referee_id`, `:season_id` adlandırılmış parametre alacak.

### 7.1 Maç başına kart sayısı (yardımcı CTE)

```sql
-- card-per-match (diğer sorguların temeli)
SELECT m.id AS match_id,
       m.kickoff_at,
       m.referee_id,
       m.season_id,
       COUNT(*) FILTER (WHERE e.event_type IN ('YELLOW','SECOND_YELLOW','RED')) AS cards
FROM match m
LEFT JOIN match_event e ON e.match_id = m.id
WHERE m.referee_id = :referee_id AND m.season_id = :season_id
GROUP BY m.id;
```

`LEFT JOIN` bilerek seçildi: kartsız maçlar da ortalamaya girer; `INNER JOIN` kullanılırsa ortalama şişer (Sorunlar belgesi, madde "kartsız maç").

İş kuralı sorusu: ikinci sarı + kırmızı birlikte nasıl sayılır? Önerilen tanım: `SECOND_YELLOW` bir kart olayıdır, ardından gelen kırmızı ayrıca kaydedilmez. Bu tanım API belgesinde yazılmalı.

### 7.2 Kümülatif ortalama ve 3 maçlık hareketli ortalama

```sql
WITH per_match AS ( /* 7.1 */ )
SELECT match_id,
       kickoff_at,
       cards,
       ROUND(AVG(cards) OVER (ORDER BY kickoff_at, match_id
             ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW), 2) AS cumulative_avg,
       ROUND(AVG(cards) OVER (ORDER BY kickoff_at, match_id
             ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2)         AS moving_avg_3
FROM per_match
ORDER BY kickoff_at, match_id;
```

- `ROWS` kullanıldı (satır sayısına göre). `RANGE` aynı tarihli maçları eş sayar ve beklenmedik sonuç verebilir; ikincil sıralama olarak `match_id` eklendi.
- İlk iki maçta pencere 3 satırdan azdır; bu yüzden ortalama daha az maçtan hesaplanır. Yanıtta `window_size` alanı bu durumu göstermelidir.

### 7.3 Takım bazlı geçmiş

```sql
WITH team_matches AS (
  SELECT m.id AS match_id, m.kickoff_at, t.id AS team_id, t.name AS team_name
  FROM match m
  JOIN team t ON t.id IN (m.home_team_id, m.away_team_id)
  WHERE m.referee_id = :referee_id
)
SELECT team_name,
       kickoff_at,
       ROW_NUMBER() OVER (PARTITION BY team_id ORDER BY kickoff_at)       AS nth_time,
       LAG(kickoff_at) OVER (PARTITION BY team_id ORDER BY kickoff_at)    AS previous_time,
       kickoff_at - LAG(kickoff_at) OVER (PARTITION BY team_id ORDER BY kickoff_at) AS since_previous
FROM team_matches
ORDER BY team_name, kickoff_at;
```

### 7.4 Takımların sıralaması (RANK)

```sql
WITH team_cards AS (
  SELECT e.team_id,
         COUNT(*) AS cards,
         COUNT(DISTINCT e.match_id) AS matches
  FROM match_event e
  JOIN match m ON m.id = e.match_id
  WHERE m.referee_id = :referee_id AND m.season_id = :season_id
    AND e.event_type IN ('YELLOW','SECOND_YELLOW','RED')
    AND e.team_id IS NOT NULL
  GROUP BY e.team_id
)
SELECT t.name, tc.cards, tc.matches,
       RANK()       OVER (ORDER BY tc.cards DESC) AS rank_with_gaps,
       DENSE_RANK() OVER (ORDER BY tc.cards DESC) AS rank_dense
FROM team_cards tc JOIN team t ON t.id = tc.team_id
ORDER BY rank_with_gaps, t.name;
```

`matches` yalnızca **kart görülen** maçları sayar (olay içeren maçlar); kart görmeyen maçlar dışarıda kalır. Takım başına maç sayısı için ayrı bir CTE gerekir; bu bilinçli bir eksiklik olarak not edilmiştir ve kart/maç oranını yanıltıcı yapmamak için yanıtta `matches` alanı adlandırması "kartlı maç sayısı" olmalıdır.

### 7.5 Aylık kazanç, bir önceki aya göre fark ve kümülatif toplam

```sql
WITH monthly AS (
  SELECT date_trunc('month', m.kickoff_at AT TIME ZONE 'Europe/Istanbul') AS month,
         SUM(m.fee_amount)                 AS fee,
         SUM(m.travel_cost)                AS travel,
         SUM(m.fee_amount - m.travel_cost) AS net
  FROM match m
  WHERE m.referee_id = :referee_id AND m.season_id = :season_id
  GROUP BY 1
),
months AS (  -- maçsız ayları da göstermek için
  SELECT generate_series(date_trunc('month', s.start_date::timestamp),
                         date_trunc('month', s.end_date::timestamp),
                         interval '1 month') AS month
  FROM season s WHERE s.id = :season_id
)
SELECT to_char(mo.month, 'YYYY-MM') AS month,
       COALESCE(x.fee, 0)    AS fee,
       COALESCE(x.travel, 0) AS travel,
       COALESCE(x.net, 0)    AS net,
       SUM(COALESCE(x.net, 0)) OVER (ORDER BY mo.month) AS running_net,
       COALESCE(x.net, 0) - LAG(COALESCE(x.net, 0)) OVER (ORDER BY mo.month) AS change_vs_prev
FROM months mo LEFT JOIN monthly x ON x.month = mo.month
ORDER BY mo.month;
```

Dikkat: "net = ücret - yol masrafı" bir tanımdır. Yol masrafı bazen ayrıca ödenir/iade edilir; bu tanımı belgeye açık yazın ve ayrı `fee`/`travel` sütunlarını da gösterin.

### 7.6 "SQL'i göster" tasarımı

- Her sorgu: `id`, `title`, `description`, `sql` (dosya içeriği), `parameters`.
- Sorgu kayıt defteri (`ReportQuery` listesi) kodda sabittir; istemci yalnızca `queryId` verebilir.
- Yanıt, parametre **değerlerini** içermez, parametre **adlarını** gösterir (başka kullanıcıya ait değer sızmaz).
- Sorgu dosyaları testle doğrulanır: her dosya sentetik veride çalışır ve beklenen sonucu verir.

---

## 8. Kimlik doğrulama tasarımı

### 8.1 Seçenekler

| | Sunucu oturumu (cookie) | JWT (Bearer) |
|---|---|---|
| Swagger UI ile deneme | Çalışır ama CSRF ve cookie ayarı gerekir | "Authorize" düğmesi ile kolay |
| Sade web sayfası (aynı site) | Doğal uyum | Token'ı nerede saklayacağınız sorusu çıkar |
| Güvenlik tuzakları | CSRF, oturum sabitleme | Uzun ömürlü token, zayıf sır, XSS ile token çalma, iptal zorluğu |
| CV değeri | Orta | Yüksek (API odaklı projelerde yaygın) |
| Durumsuzluk | Hayır | Evet |

### 8.2 Öneri

**JWT (yalnızca access token), kısa ömürlü.** Gerekçe: proje REST API odaklı, Swagger ile test edilecek. Web sayfası eklenirse de token bellekte (JavaScript değişkeni) tutulur; `localStorage` önerilmez (XSS durumunda çalınır).

Tasarım:

- Parola: `BCrypt` (Spring Security'nin `DelegatingPasswordEncoder` ile) veya Argon2; en az 12 karakter kuralı.
- Token: imza algoritması sabit (HS256 ile en az 32 baytlık rastgele sır, ya da RS256 anahtar çifti). `alg` başlığına güvenilmez; doğrulayıcı beklenen algoritmayı zorlar.
- Süre: 60 dakika (yapılandırılabilir); `exp`, `iat`, `sub` (referee id) zorunlu.
- Sır: ortam değişkeninden gelir; depoda yoktur; `.env.example` yalnızca yer tutucu içerir.
- Yetkilendirme: her istek için `sub` ile sahiplik denetimi servis katmanında.
- Giriş hataları: kullanıcı yok / parola yanlış aynı mesaj ve benzer süre (kullanıcı sayımı engeli).
- Kaba kuvvet: temel hız sınırlama (örn. IP + e-posta başına dakikada N deneme). Uygulama yöntemi (filtre mi, ters proxy mi) açık soru.
- CORS: sadece gerekli kaynaklara izin; `*` kullanılmaz.
- Loglarda token ve parola yoktur.

### 8.3 Bilinçli eksikler (v2)

Refresh token, token iptali, e-posta doğrulama, parola sıfırlama, MFA. Bu eksikler README "Bilinen sınırlamalar" bölümünde dürüstçe yazılmalıdır.

### 8.4 Spring Security ayrıntıları

Spring Security'nin OAuth2 Resource Server ile JWT doğrulama kullanımı genel olarak yaygındır; ancak Boot 4.1 ile gelen Spring Security sürümünde tam yapılandırma ve starter adı **doğrulanmadı**. Hafta 2'nin başında resmi dokümandan kontrol edin; eski blog örneklerine güvenmeyin.

---

## 9. Test stratejisi

### 9.1 Katmanlar

| Tür | Ne test eder | Araç | Hız |
|---|---|---|---|
| Birim | Servis iş kuralları, DTO doğrulama, para/zaman yardımcıları, CSV biçimleyici | JUnit 5, Mockito | Saniyeler |
| Entegrasyon (repository/SQL) | Migrasyonlar, kısıtlar, rapor SQL'leri, N+1 | Testcontainers PostgreSQL | Onlarca saniye |
| Entegrasyon (API) | Uçtan uca: kayıt, giriş, maç ekle, filtre, rapor, sahiplik | Spring Boot test + Testcontainers + MockMvc veya HTTP istemcisi | Onlarca saniye |
| Güvenlik | Tokensiz 401, başkasının kaydı 404, süresi dolmuş token | Aynı | |

### 9.2 Kurallar

- Testcontainers testlerinde **tek paylaşımlı konteyner** (sınıf başına yeni konteyner yok); aksi halde süre uzar.
- Testler sıfır veritabanından Flyway'i çalıştırır; `ddl-auto=validate` şema uyumunu da sınar.
- Rapor sorguları için küçük, **elle hesaplanmış** veri kümesi kullanılır (ör. 5 maç, bilinen kartlar); beklenen değerler testte yazılıdır. Böylece "SQL doğru mu" sorusunun kanıtı olur.
- Sınır durumları: kartsız maç, aynı gün iki maç, ay sınırı (31 Ocak 23:30 gibi İstanbul saati), maçsız ay, tek maçlık sezon, bir takımla ilk karşılaşma (`LAG` null).
- Hedef: anlamlı kapsam, rakam kovalamak değil. Başlangıç hedefi önerisi: servis katmanında yaklaşık %70 satır kapsamı; bu **tercihtir**, kural değildir.
- Docker yoksa entegrasyon testleri atlanır (kullanıcıya açık mesajla) ya da başarısız olur; hangisi seçilirse README'de yazılır.

### 9.3 CI (GitHub Actions)

- Tetik: push ve pull request.
- Adımlar: checkout, JDK 21 kurulumu, Maven önbelleği, `mvnw verify`.
- Docker, GitHub barındırmalı Ubuntu runner'da bulunur (genel bilgi; ilk çalıştırmada teyit edin).
- Başarısızlıkta test raporu artifact olarak yüklenir.
- İsteğe bağlı: Docker imajı derleme adımı (yayınlamadan).

---

## 10. Seed (sentetik veri) tasarımı

### 10.1 Amaç

- Karne sorguları anlamlı görünsün, ama uydurma veri **gerçekmiş gibi** sunulmasın (README bunu açıkça söyler).
- Herkes aynı veriyi alsın: sabit rastgelelik tohumu (örn. 42) ve sabit "referans tarih".
- Sınır durumları bilerek veride bulunsun (testlerle çakışmasın diye test verisi ayrıdır).

### 10.2 Biçim

- Ayrı bir komut satırı aracı ya da `dev` profilinde çalışan yükleyici; Docker Compose'ta `seed` profili.
- Migrasyon **değildir**; üretim şemasını kirletmez.
- Idempotent: ikinci çalıştırmada çoğaltmaz (örn. demo hakem varsa atlar ya da `--reset` bayrağı ile siler).
- Parametreler yapılandırma dosyasında: sezon sayısı, hakem başına maç sayısı, dağılım oranları, ücret aralığı.

### 10.3 Veri üretim kuralları

Aşağıdaki sayılar **varsayımdır**, gerçek istatistik değildir; yapılandırılabilir olmalıdır.

| Alan | Kural |
|---|---|
| Hakemler | 1 demo hakem (`demo@example.com`), parola README'de belirtilen sabit demo parolası; isim kurgusal ("Demo Hakem") |
| Takımlar | Tamamen kurgusal adlar, bir kalıptan üretilir (örn. "<kurgusal semt> <Spor|Gençlik|İdman>"). Gerçek kulüp adı kullanılmaz; üretimden sonra bilinen kulüp adlarıyla çakışma denetimi yapılır (kara liste dosyası). |
| Ligler | 2-3 kurgusal lig adı (örn. "Örnek Amatör Küme A") |
| Sahalar | Kurgusal; şehir/ilçe alanı genel ("Merkez") veya kurgusal |
| Maç sayısı | Sezon başına yaklaşık 25-40 (varsayım); hafta sonu ağırlıklı, bazı hafta içi |
| Görev dağılımı | Yaklaşık %60 orta, %30 yardımcı, %10 dördüncü (varsayım) |
| Sezon takvimi | Ağustos-Mayıs arası ağırlık; haziran-temmuz boş (maçsız ay testi için) |
| Skor | Küçük tam sayılar, gerçekçi dağılım (çoğu 0-4 gol arası, nadir yüksek skor) |
| Kartlar | Maç başına sarı sayısı Poisson benzeri dağılım (ortalama yapılandırılabilir); kırmızı nadir (%3-5 maç, varsayım); penaltı nadir; 0 kartlı maçlar var |
| Dakika | 1-90 arası, 40-90 ağırlıklı; birkaç tane uzatma dakikası (45+2 gibi) |
| Ücret | Görev bazlı aralık (orta > yardımcı > 4.), yapılandırılabilir; **resmî tarife değildir** |
| Yol | Mesafe 5-120 km; masraf = mesafe x birim fiyat + rastgele küçük sapma; birim fiyat yapılandırılabilir |
| Ödeme | Eski maçların çoğu ödenmiş, son 1-2 ayın bir kısmı ödenmemiş (filtre testi) |
| Tekrar | Bazı takımlar birden fazla kez karşımıza çıkar (takım geçmişi sorgusu anlamlı olsun) |

Gerçekçilik denetimi (otomatik): üretimden sonra kontrol sorguları çalışır: skor negatif mi, aynı takım ev-deplasman mı, aynı hakem iki yerde aynı saatte mi, ücret aralık içinde mi. Sonuç raporlanır.

### 10.4 Sentetik verinin sınırı

Sentetik veri, gerçek hakem davranışını yansıtmaz. Karnedeki "ortalama" gibi sayılar yalnızca yöntemi gösterir. README ve ekran görüntülerinde "örnek veri" etiketi bulunmalıdır.

---

## 11. Gerçek veriyi anonimleştirme yönergesi

**Bu bölüm hukuki tavsiye değildir.** Kişisel verilerin korunması mevzuatı (Türkiye'de KVKK) karmaşıktır; gerçek veri kullanacaksanız bir hukukçuya veya ilgili kurumun veri sorumlusuna danışın. Aşağıdakiler genel dikkat noktalarıdır.

### 11.1 Neden önemli

- Hakem adı, takım/oyuncu adı, forma numarası + takım + tarih birleşimi, kişiyi doğrudan veya dolaylı tanımlayabilir.
- Disiplin olayları (kırmızı kart) ve ödeme bilgileri hassas sayılabilecek bağlam taşır.
- "Herkese açık maç sonucu" olması, o verinin kendi hâlinde her amaçla kullanılabileceği anlamına gelmeyebilir; amaç ve kapsam önemlidir (hukuki değerlendirme gerekir).

### 11.2 Genel ilkeler

1. **Önce gereksinimi sorgulayın:** Gerçek veriye gerçekten ihtiyaç var mı? Çoğu zaman sentetik veri yeterlidir. Bu projenin tasarım tercihi budur.
2. **Veri azaltma:** Oyuncu adı hiç alınmaz; forma numarası bile gerekmiyorsa kaldırılır.
3. **Kaynak ve izin:** Verinin nereden geldiğini ve kullanım iznini netleştirin. Başkasına ait defter, mesaj ya da belge ise sahibinin açık onayı olmalı.
4. **Anonimleştirme yöntemleri:**
   - İsimleri tutarlı takma adlarla değiştirme (aynı kişi hep aynı takma ad; eşleştirme tablosu depoya girmez ve ayrı güvenli yerde tutulur ya da yok edilir).
   - Tarihleri kaydırma (tüm tarihler aynı rastgele gün sayısı kadar; haftanın günü korunur) ya da haftaya yuvarlama.
   - Ücretleri yuvarlama veya gürültü ekleme.
   - Nadir kombinasyonları genelleme (ör. küçük bir ilçedeki tek saha adı).
5. **Takma ad (pseudonymization) anonimleştirme değildir:** Eşleştirme tablosu başka yerde duruyorsa veri hâlâ kişisel veri sayılabilir. Gerçek anonimleştirme geri döndürülemez olmalıdır; bunun yeterli olup olmadığı hukuki/teknik değerlendirme ister.
6. **Yeniden tanımlama riski:** Anonim görünen veri, dış bilgiyle (örn. herkese açık fikstür) eşleştirilerek çözülebilir. Küçük kümelerde bu risk yüksektir.
7. **Depoya koymayın:** Gerçek veri, anonimleştirilmiş bile olsa, doğrulanmadıkça herkese açık GitHub deposuna konmaz. Git geçmişi sonsuza kadar tutar; silmek yetmez.
8. **Saklama ve silme:** Gerçek veriyi yalnızca yerelde, şifreli diskte, gerekli süre kadar tutun; işi bitince silin.
9. **Kendi verinizi kullanıyorsanız:** Kendi maç kayıtlarınızdaki **başkalarına ait** bilgiler (oyuncu, takım yetkilisi) yine de başkalarının verisidir; aynı özen geçerlidir.
10. **Dokümantasyon:** Hangi alanın nasıl dönüştürüldüğü kısa bir not olarak yazılır (kendi kayıt defteriniz için, depoya değil).

### 11.3 Önerilen iş akışı (kendi gerçek kayıtlarınız için)

1. Kayıtları yerel, depo dışı bir klasörde tutun.
2. Dönüştürme betiği (depoda olabilir, veri olmadan) alanları yukarıdaki kurallara göre dönüştürsün.
3. Çıktıyı yalnızca yerel veritabanına yükleyin.
4. Ekran görüntüsü alırken isimleri, tarihleri ve tutarları kontrol edin.
5. Şüphe varsa kullanmayın.

---

## 12. 5 haftalık plan

Hafta başına yaklaşık 8-12 saat varsayımıyla yazıldı (okul yoğunluğuna göre ayarlayın). Her haftanın sonunda bir "bitiş kriteri" vardır; karşılanmadıysa bir sonraki haftaya yeni özellik eklenmez, kapsam kısılır.

### Hafta 1 - İskele, Docker, Flyway, CI

Yapılacaklar:

- Depoyu oluştur, MIT lisansı, `.gitignore`, `.gitattributes` (satır sonları), `.env.example`.
- Spring Boot iskeleti (start.spring.io; Java 21, Maven).
- `docker-compose.yml`: PostgreSQL + uygulama; sağlık kontrolü ile başlatma sırası.
- Flyway: V1-V4 migrasyonları; `ddl-auto=validate`.
- İlk Testcontainers testi: "migrasyonlar sıfırdan çalışıyor".
- GitHub Actions: `verify` çalışıyor.

**Bitiş kriteri:** Temiz klonda `docker compose up --build` uygulamayı kaldırır, `/actuator/health` UP döner, CI yeşil, en az bir Testcontainers testi geçer.

### Hafta 2 - Kimlik doğrulama ve referans veriler

Yapılacaklar:

- Kayıt/giriş, parola hash'leme, JWT üretimi ve doğrulaması.
- Takım, saha, lig, sezon CRUD; sahiplik denetimi.
- Ortak hata modeli ve doğrulama mesajları.
- Güvenlik testleri (401, 404, süresi dolmuş token).
- OpenAPI/Swagger'da "Authorize" çalışıyor.

**Bitiş kriteri:** Swagger üzerinden kayıt, giriş, takım ekleme yapılır; başka kullanıcının takımına erişim denemesi test ile reddedilir.

### Hafta 3 - Maç, olay, ücret, filtre, CSV

Yapılacaklar:

- Maç ekleme/okuma/güncelleme/silme; olay ekleme/silme.
- Ücret ve yol masrafı; ödendi işareti.
- Filtre + sayfalama; N+1 kontrolü.
- CSV dışa aktarma (Excel uyumu kararı).
- Doğrulama kuralları (aynı takım, negatif skor, dakika aralığı).

**Bitiş kriteri:** MVP uçtan uca çalışır; filtrelenmiş liste ve CSV doğru; listeleme isteği sorgu sayısı testte sabit (N+1 yok).

### Hafta 4 - Sezon karnesi ve "SQL'i göster"

Yapılacaklar:

- Bölüm 7'deki sorgular dosyalara yazılır, gerçek şemada çalıştırılır, `EXPLAIN` incelenir.
- Karne endpoint'i; `include=sql`.
- Seed aracı (Bölüm 10) ve sınır durumu verisi.
- Her sorgu için elle hesaplanmış küçük veri kümesiyle test.

**Bitiş kriteri:** Seed yüklenir, karne yanıtı 4 istatistiği ve kaynak SQL'lerini döndürür; her sorgunun beklenen değer testi geçer.

### Hafta 5 - Cila, belge, (isteğe bağlı) sade arayüz

Yapılacaklar:

- README gerçek duruma göre güncellenir (kutucuklar, örnek yanıtlar gerçek çıktıyla değiştirilir, "örnek veri" etiketi korunur).
- Ekran görüntüleri; ER diyagramı (şemadan üretilmiş veya elle çizilmiş).
- İsteğe bağlı: tek sayfalık sade arayüz (htmx veya düz HTML) **ya da** yalnızca Swagger.
- Güvenlik gözden geçirmesi (Sorunlar belgesindeki kontrol listesi).
- Temiz makinede kurulum provası (mümkünse başka bir bilgisayar veya sanal makinede).
- Kapsam dondurma: v2 listesi güncel.

**Bitiş kriteri:** Birinin README'yi okuyup 10 dakikada projeyi çalıştırabildiği kanıtlanır (bir arkadaşa denetirin); CI yeşil; v1 etiketi (tag) atılır.

### Taşma politikası

Bir hafta kaçarsa sırayla şunlar kesilir: (1) sade web arayüzü, (2) hız sınırlama, (3) CSV'de gelişmiş seçenekler, (4) ödeme takibi. Karne ve testler kesilmez; projeyi CRUD'dan ayıran kısım bunlardır.

---

## 13. v2 fikirleri (bilinçli olarak dışarıda)

- Futbol kuralları sınavı
- Takım geçmişi uyarısı (örn. "bu takımı yakın zamanda yönettin")
- Canlı maç takibi
- Mobil uygulama
- Çoklu kullanıcı rolleri (gözlemci, yetkili)
- Refresh token, e-posta doğrulama, parola sıfırlama, MFA
- Oyuncu bazlı istatistik (kişisel veri riski nedeniyle ayrıca değerlendirilmeli)
- Bileşik yabancı anahtarla sahiplik zorlaması
- İyimser kilitleme (`version`)
- PDF sezon karnesi

Yeni fikir geldiğinde kural: önce bu listeye yaz, v1 bitmeden koda başlama.

---

## 14. Açık sorular

1. Sezon tablosu ortak mı, hakeme özel mi? (Öneri: ortak, seed ile.)
2. `match` tablosu adı `football_match` olsun mu?
3. Karne API'sinde "net kazanç" tanımı: ücret - yol masrafı mı, yoksa yol masrafı iade sayılıp ücrete eklenmeli mi?
4. İkinci sarı + kırmızı sayım kuralı (Bölüm 7.1) onaylanmalı.
5. Hız sınırlama uygulama içinde mi, ters proxy'de mi?
6. Sade web arayüzü yapılacak mı, yoksa yalnızca Swagger mi? (Hafta 5'te karar.)
7. JWT için HS256 mı, RS256 mı? (Tek hizmet için HS256 yeterlidir; ders amaçlı RS256 denenebilir.)
8. Boot 4.1'de `@ServiceConnection`, springdoc yolu, RFC 9457 ayarı gibi ayrıntılar resmi dokümandan teyit edilmeli (Bölüm 2).
