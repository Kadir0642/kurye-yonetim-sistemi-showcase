# ADR-0022: Pilot öncesi üretim dağıtımı, yalnızca ücretsiz bileşenlerle (tek sunucu compose, Caddy + Let's Encrypt, saatlik pg_dump, Uptime Kuma, yalın GitHub Actions, sırlar ve APK imzalama)

- **Durum:** Önerildi (İterasyon 3a spesifikasyonuyla birlikte onaya sunuldu; ücretli kalemler ürün sahibi kararıdır, SPEC-3a §18)
- **Tarih:** 2026-11-05
- **İlgili:** [ADR-0003](0003-dagitim-apk-self-update.md), [ADR-0005](0005-konum-takibi-ve-kvkk.md), [ADR-0007](0007-backend-ve-web-yigini.md), [ADR-0012](0012-rls-iterasyon-2b-ye-ertelendi.md), [ADR-0018](0018-harita-karo-ve-stil-sunumu.md), [ADR-0021](0021-haftalik-osm-guncelleme-hatti.md), PLAN §7 P-1 ... P-8

## Bağlam

- **Ürün sahibinin kesin kuralı:** hiçbir ücretli hizmet satın alınmaz, ücretli plan veya faturalama açılmaz, ücretli bulut kaynağı oluşturulmaz. Ücretli bir şey işe yarayacaksa karar olarak yazılır ve ücretsiz alternatifi verilir.
- Bugün yalnızca geliştirme altyapısı var (`infra/docker-compose.yml`: Postgres, Redis, `routing` profili). Üretim Dockerfile'ı, ters vekil, HTTPS, yedek, izleme, CI yok. Kurye uygulaması demo için düz HTTP'ye izin verir (`usesCleartextTraffic: true`).
- Depo özeldir (private); GitHub Actions'ın özel depolar için aylık ücretsiz dakika kotası vardır (GitHub Free planında 2.000 dk / ay, yalnızca Linux çalıştırıcı 1x sayılır; kota planla değişebilir, P görevi güncel değeri doğrular). GHCR özel paket depolaması da kotalıdır.
- Pilot ölçeği küçüktür (yaklaşık 25 kurye, tek kiracı, tek şehir); boyutlandırma PLAN P-1 tablosunda tahmin edilmiştir.

## Karar

**1. Tek sunucu, Docker Compose.** `infra/prod/compose.prod.yml`: `caddy` (tek açık servis: 80 / 443), `api`, `postgres` (PostGIS, aynı sürüm), `redis`, `backup` (zamanlanmış `pg_dump`), `uptime-kuma`; OSRM, Photon, iç geçit ve VROOM [ADR-0021](0021-haftalik-osm-guncelleme-hatti.md) mavi / yeşil hat compose dosyasından gelir (`-f compose.prod.yml -f ../pipeline/compose.pipeline.yml`). Tüm iç servisler yalnızca iç Docker ağında; hiçbir veritabanı / OSRM / Photon portu dışa açılmaz. Panel, Caddy tarafından statik dosya olarak sunulur (ayrı Node süreci yok).

**2. Ters vekil ve HTTPS: Caddy + Let's Encrypt (ücretsiz).** Otomatik sertifika ve yenileme, HTTP -> HTTPS, HSTS, güvenlik başlıkları, `encode zstd gzip` (rota bacakları ve stil JSON'u için önemli), `/api/v1/map/files|glyphs|sprites/*` doğrudan diskten (`file_server`, Range) ([ADR-0018](0018-harita-karo-ve-stil-sunumu.md)), `/ws` WebSocket vekili. Caddy çekirdeğinde hız sınırı yoktur (eklenti özel derleme ister); API'nin mevcut hız sınırı korunur, harita dosyaları için sunucu güvenlik duvarında (nftables / fail2ban) IP başına bağlantı sınırı yeterlidir. Alan adı ücretli olduğundan **karar**dır; ücretsiz alternatif DuckDNS / sslip.io gibi alt alan adlarıyla da Let's Encrypt çalışır.

**3. Yedek: saatlik `pg_dump` + geri yükleme tatbikatı.** Resmî `postgis/postgis` imajındaki `pg_dump -Fc` saatte bir, `age` ile şifreli; saklama: saatlik 48, günlük 35 (KVKK: silinen veri yedekte 35 günden uzun kalmaz). **RPO <= 1 sa, RTO <= 2 sa** hedefi (PLAN'daki 15 dk RPO, WAL arşivi ve sunucu dışı depolama ister; sunucu dışı depolama ücretli olabileceği için karar listesindedir). Sunucu dışı kopya: ücretsiz varsayılan, ürün sahibinin makinesinin gece SSH ile şifreli yedekleri çekmesi (`rsync` / `scp`); ücretli seçenek Türkiye'de nesne depolama. Aylık geri yükleme tatbikatı betiği (`restore-drill`) temiz bir konteynere geri yükler, satır sayıları ve `/health` ile doğrular. Harita, OSRM ve Photon artefaktları yeniden üretilebilir: yedeklenmez. Redis kalıcı veri taşımaz sayılır.

**4. İzleme: Uptime Kuma (kendi barındırma, MIT) + push betikleri.** HTTP monitörleri (`/api/v1/health`, panel, TLS bitiş tarihi), TCP / Docker konteyner monitörleri, ve push monitörleri: yedek başarılı, haftalık hat sonucu, ping özet işi, disk > %80, bellek > %85 (cron ile çalışan küçük kabuk betikleri). Bildirim: Telegram botu (ücretsiz) veya e-posta. Aynı sunucudaki izleyici sunucu tamamen düşünce uyaramaz: ücretsiz dış denetleyici (ör. bir uptime SaaS'ın ücretsiz katmanı yalnızca herkese açık `/health`'i yoklar, kişisel veri görmez) **karar**dır. Prometheus / Grafana pilot sonrasına bırakılır (bellek ve bakım).

**5. Loglar.** Uygulama zaten JSON log üretir (pino). Docker `json-file` sürücüsü `max-size` / `max-file` ile döner; uygulama logu en çok 30 gün. Konum koordinatı, adres metni, telefon ve token loglanmaz (mevcut log testleri genişletilir). Caddy erişim logu (IP) süresi hukuk görüşüne bağlıdır (P-9); varsayılan 14 gün.

**6. Sırlar.** Depoda sır yok (CLAUDE.md kuralı 3). Sunucuda `/srv/kurye/env/prod.env` (0600, yalnızca kök) **veya** `sops` + `age` ile şifreli dosya (ikisi de ücretsiz). Üretimde `dev-only` önekli sırlar ve üçüncü taraf harita / Photon adresleri ortam doğrulamasıyla reddedilir. JWT ve devir kodu anahtarı dönüş prosedürü belgelenir. APK imzalama anahtarı yalnızca ürün sahibinde ve ayrı konumda yedekli; CI'ya konmaz.

**7. CI: yalın GitHub Actions.** Yalnızca `ubuntu-latest`; yol süzgeçleri (değişen çalışma alanına göre iş), `concurrency` ile eski koşuları iptal, npm önbelleği. İşler: sözleşme denetimi (`contracts:check`), tip denetimi, lint, birim testleri (API, panel, uygulama), panel ve API derlemesi. Postgres gerektiren e2e testleri yalnızca `main`'e birleşmede ve elle (`workflow_dispatch`); Docker imaj derlemesi yalnızca elle. İmajlar GHCR'ye **itilmez** (kota); sunucuda `git pull && docker compose build` ile derlenir. APK CI'da derlenmez (dakika ve imza anahtarı); ürün sahibinin makinesinde yerel Gradle ile. Gizli taraması: ücretsiz `gitleaks` CI adımı.

**8. APK dağıtımı: imzalı sürüm + kendi sunucumuzdan indirme, isteğe bağlı Headwind MDM CE.** Sürüm APK'sı yerelde imzalanır (keystore depoda değil; parolalar ortam değişkeninde), sha256 ile birlikte Caddy'nin sunduğu `/indir/` sayfasına konur; kurye "bilinmeyen kaynak" izniyle kurar. Uygulama içi güncelleme (PackageInstaller) ve kendi `expo-updates` sunucumuz sonraki iştir ([ADR-0003](0003-dagitim-apk-self-update.md)). Sürüm APK'sında düz HTTP kapalıdır.

## Sonuçlar

- (+) Tüm bileşenler açık kaynak ve ücretsiz; tek ücretli zorunluluk sunucunun kendisi (ve isteğe bağlı alan adı), ikisi de ürün sahibi kararı.
- (+) Tek komutla kurulum ve geri dönüş; harita / rota / geocoder hattıyla aynı compose düzeni.
- (+) CI dakikaları kontrol altında; e2e ve imaj işleri yalnızca gerektiğinde.
- (-) RPO 1 sa (15 dk değil) ve sunucu dışı yedek ürün sahibinin makinesine bağlı; pilot için kabul edilebilir, büyümede gözden geçirilir.
- (-) İzleme aynı sunucuda; sunucunun tamamen düşmesi ancak dış denetleyiciyle (karar) anında fark edilir.
- (-) Tek sunucu tek arıza noktasıdır (pilot ölçeğinde bilinçli kabul).

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Yönetilen bulut (managed DB, load balancer, kapsayıcı servisi) | Ücretli; tek şehirlik pilot için gereksiz; veri Türkiye dışına çıkabilir |
| Traefik | Caddy'ye eşdeğer ve ücretsiz; Caddy'nin yapılandırması daha kısa ve hat geçidinde de Caddy kullanılıyor (tek araç) |
| pgBackRest / wal-g ile sürekli WAL arşivi | 15 dk RPO için doğru araç; sunucu dışı depolama olmadan değeri sınırlı; pilot sonrasına (karar) |
| Prometheus + Grafana + Alertmanager | Daha güçlü ama 3 servis ve bellek; pilot için Uptime Kuma yeterli |
| GHCR'ye imaj itme + sunucuda çekme | Özel paket kotası; sunucuda derleme ücretsiz |
| CI'da APK derleme veya EAS Build | Dakika kotası / EAS ücretli katmanları; imza anahtarını CI'ya koymak gerekir |
| Ücretli MDM | ADR-0003: Headwind MDM CE ücretsiz ve yeterli; şirket telefonu yoksa MDM gereksiz |

## Uygulama notları (P görevi, 2026-11-06)

Kod: `infra/prod/`, `services/api/Dockerfile`, `apps/admin-web/Dockerfile`, `.github/workflows/`; belgeler: `docs/uretim/`. Yerelde doğrulandı: `prod:smoke --seed` (19 denetim), `prod:backup`, `prod:restore-drill --strict` (18/18 tablo eşit, 20,5 sn), 3 uyarı tatbikatı (API durdur ~35 sn, yedek hatası ve disk eşiği < 1 sn), `actionlint` ve `shellcheck` temiz, `gitleaks` 155 commit temiz.

- **Sürüm sabitleme:** `node:22.23.3-bookworm-slim`, `caddy:2.11.7-alpine` (2.11.8 ve 2.11.9 iki haftadan yeni), `louislam/uptime-kuma:2.5.7-slim`, `postgis/postgis:16-3.6` ve `redis:7.4-alpine` (geliştirmeyle aynı), `postgres:16.15-trixie` (yedek istemcisi), `age` Debian paketi, `gitleaks` 8.30.1 (sha256 doğrulamalı), GitHub eylemleri commit SHA ile (`actions/checkout` v7.0.1, `actions/setup-node` v7.1.0).
- **Güncelleme (3a entegrasyonu): `postgis/postgis:16-3.5` Debian bullseye tabanlıydı (destek dışı).** Sunucu imajı `postgis/postgis:16-3.6`'ya (Debian 13 trixie, PostgreSQL 16.15, PostGIS 3.6) geçirildi; API migration'ları ve e2e (275 test) bu imajda doğrulandı. Aynı ana sürüm (PG 16): mevcut veri dizinleri açılır, ancak libc sürümü değiştiği için **eski (bullseye) bir veri biriminde `REINDEX DATABASE` ve `ALTER DATABASE ... REFRESH COLLATION VERSION` çalıştırılmalı** (geliştirmede `npm run infra:reset` yeterli; üretim henüz veri taşımıyor). Yedek imajı sunucuyla aynı aileye çekildi: `postgres:16.15-trixie` (pg_dump 16.15 + `age` + `curl`). `postgis/postgis:17-*` etiketleri mevcut ama API bu sürümde doğrulanmadığı için seçilmedi; 17'ye geçiş ayrı iştir.
- **Postgres sağlık denetimi `-h 127.0.0.1` ile yapılmalıdır:** ilk açılışta resmi imajın geçici (yalnızca soket) sunucusu `pg_isready`'e "hazır" der; bağımlı servisler (migration) bağlanamaz. `infra/docker-compose.yml` aynı kusuru taşır (bu görevde dokunulmadı; sahibi: geliştirme altyapısı).
- **npm çalışma alanları ve imaj boyutu:** `npm ci --workspace=...` ağaç kurulumunda süzgeç uygulamaz (mobil bağımlılıkları da iner, ~580 MB). Dockerfile'lar kök `workspaces` listesini ilgili klasörlere indirip `npm install --package-lock-only` ile lockfile'ı budar, üretim aşamasında `--omit=peer` kullanır: API imajı 1,1 GB -> 421 MB (diskte), `node_modules` ~80 MB. Panel imajı (Caddy + statik) 98 MB.
- **Uptime Kuma 2.x** ilk açılışta veritabanı seçim sihirbazı gösterir: `UPTIME_KUMA_DB_TYPE=sqlite` ile atlanır; yapılandırma socket.io API ile **kodla** kurulur (`monitoring/kuma-provision.mjs`; 14 monitör + push jetonları + bildirim kanalı).
- **Docker soketi hiçbir konteynere bağlanmaz** (Kuma dahil): konteyner sağlığı ana makine betiğiyle push edilir.
- **API üretim komutları:** `node dist/main.js` (başlatma), `node dist/db/migrate.js` (migration; ayrı `docker compose run --rm api ...`). Üretimde birinci kiracı / sahip hesabını oluşturan bir komut **yoktur** (`db:seed` üretimde reddedilir): pilot öncesi Q için açık iş.
- **Ölçülenler (boşta, yerel):** API 73 MiB, PostgreSQL 112 MiB, Redis 4 MiB, Caddy 16 MiB, Kuma 104 MiB; yedek turu 0,4 sn (demo veri).
- **Hat entegrasyonu (G, ADR-0021):** hat servisleri (`routing-gw`, `osrm-blue|green`, `photon-blue|green`, `vroom-pl`) `profiles: ["pipeline"]` arkasındadır ve ana makinedeki Node orkestrası tarafından başlatılır; `dc.sh up -d` onları başlatmaz. Hat `/srv/kurye/pipeline.env` ile `PIPELINE_COMPOSE_PROJECT=kurye-prod` ve **yalnızca** `compose.pipeline.yml` dosyasıyla çalışır: servisler prod ağına (`kurye-prod_default`) katılır (adı çözme doğrulandı), `compose.prod.yml` eklenmez (sır gerektirir; `pipeline down` caddy / api'yi de durdururdu). Birleşik `config` (`-f compose.prod.yml -f compose.pipeline.yml --profile pipeline`) `prod:config` ile doğrulanır. Hat portları yalnızca `127.0.0.1`'e yayınlanır; duman testi dışarıya yalnızca caddy'nin yayınladığını denetler. Haftalık zamanlama G'nin `kurye-pipeline.timer` birimidir (Pazar 03:00 Europe/Istanbul); başarı / hata bildirimi hattın `KUMA_PUSH_URL` çağrısıdır.
- **Yerel e2e:** `npm run test:e2e -w @kurye/api` (Q'nun 31 dosyası) PostGIS 16-3.5 + Redis 7.4 ile aynı ortamda geçti: 261 geçti, 11 atlandı, 217 sn (Docker, boş veritabanı).
- **Dağıtımda 503 penceresi (smoke testinin yakaladığı hata):** Caddy `reverse_proxy` aktif sağlık denetimi 15 sn aralıklıyken API konteyneri yeniden oluşturulunca, API dönse de Caddy bir sonraki denetime kadar 503 veriyordu. Çözüm: `health_interval 2s` + `lb_try_duration 20s` / `lb_try_interval 300ms` (`/api/v1/*` ve `/ws`); `prod:smoke` API yeniden başlatılırken istekleri ölçer (tümü 200, en yavaş ~5 sn).
