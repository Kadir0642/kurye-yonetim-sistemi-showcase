# ADR-0011: OSRM motosiklet profili ve rota altyapısı (OSRM MLD + VROOM, Docker, Eskişehir çıkarımı)

- **Durum:** Önerildi (İterasyon 2 spesifikasyonuyla birlikte onaya sunuldu)
- **Tarih:** 2026-11-02
- **İlgili:** [ADR-0001](0001-hibrit-harita-mimarisi.md), [ADR-0006](0006-hizmet-bolgesi-eskisehir.md), [ADR-0009](0009-rota-uyumlu-atama.md), İterasyon 2 SPEC §14

- **Güncelleme (2026-11-03):** madde 4'teki "aylık zamanlanmış iş", [ADR-0016](0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md) ile haftalık, kapılı ve kesintisiz geçişli bir hatta dönüştü (karolar ve geocoder ile birlikte). Öğrenilmiş hızlar `osrm-customize --segment-speed-file` ile eklenir ([ADR-0017](0017-saha-verisinden-ogrenen-harita.md)).

## Bağlam

- Rota uyumlu atama ([ADR-0009](0009-rota-uyumlu-atama.md)) süre matrisine (`table`) ve rota çizgisine dayanır. Öneri bütçesi 25 kurye için 300 ms'dir; matris çağrısına bunun yaklaşık 100 ms'i düşer.
- Kuryelerin büyük çoğunluğu motosikletlidir (`couriers.vehicle_type` varsayılanı `MOTORCYCLE`). OSRM'in hazır profilleri `car`, `bicycle` ve `foot`'tur; **motosiklet profili yoktur**. ADR-0009 başlangıç için "`car` profili + hız ayarı" demişti.
- Motosikleti otomobilden ayıran, şehir içinde önemli olan farklar: `motorcycle=*` erişim etiketleri (ör. yalnızca motosiklete açık geçitler veya motosiklete kapalı yollar), daha düşük dönüş ve U dönüşü maliyeti, dar sokaklarda biraz daha yüksek ortalama hız. Trafikte şerit arası geçiş gibi davranışlar OSM verisiyle modellenemez; bunlar kalibrasyon çarpanına bırakılır.
- Her şey ürün sahibinin Windows 11 makinesinde (32 GB RAM, Docker Desktop / WSL2) ve ileride küçük bir Linux VPS'te aynı biçimde çalışmalıdır. Geofabrik Türkiye dosyası yaklaşık 0,6-0,8 GB'tır; Eskişehir çıkarımı ondan çok küçüktür.

## Karar

**1. Motor: OSRM (MLD algoritması) + VROOM, ikisi de Docker konteyneri.**
- OSRM imajı `ghcr.io/project-osrm/osrm-backend` (proje 2025'ten beri takvim sürümlemesi kullanır, ör. `v26.10.0`). Görev R, en az iki haftalık en yeni `-debian` etiketini sabitler ve `docker pull` ile doğrular. Yedek: Docker Hub'daki `osrm/osrm-backend:v5.27.1` (HTTP API aynıdır).
- VROOM imajı `ghcr.io/vroom-project/vroom-docker:v1.15.0` (vroom-express HTTP sarmalayıcısıyla).
- MLD seçilir (CH değil): profil veya hız çarpanı değiştiğinde yalnızca `osrm-customize` yeniden çalışır; `table` performansı bu ölçekte yeterlidir.
- `osrm-routed --max-table-size 1000`: canlı öneri ~230 x 27 hücre ister, benchmark matrisi ~650 x 650'dir.

**2. Profil: depoda tutulan `infra/routing/profiles/motorcycle.lua`.** Sabitlenen OSRM sürümünün `car.lua` dosyasından türetilir ve yalnızca şu farkları içerir (dosya başında liste hâlinde yazılır, R bir `diff` testiyle korur):

| Ayar | `car.lua` | `motorcycle.lua` | Gerekçe |
|---|---|---|---|
| Erişim hiyerarşisi | `motorcar, motor_vehicle, vehicle, access` | `motorcycle, motor_vehicle, vehicle, access` | `motorcycle=no/yes` etiketlerine uyulur |
| Kısıtlamalar (`restrictions`) | `motorcar, motor_vehicle, vehicle` | `motorcycle, motor_vehicle, vehicle` | Dönüş yasakları motosiklet etiketiyle okunur |
| `turn_penalty` | 7.5 | 4 | Motosiklet dönüşlerde daha az yavaşlar |
| `u_turn_penalty` | 20 | 10 | Şehir içinde U dönüşü kolay |
| `residential`, `living_street`, `service` hızları | 25, 10, 15 km/sa | 30, 15, 20 km/sa | Dar sokaklarda motosiklet daha hızlı |
| Tek yön | uyulur | uyulur | Yasal rota hesaplanır; kural dışı sürüş modellenmez |

**3. Kalibrasyon, yeniden çıkarım gerektirmez.** API, OSRM sürelerini `ROUTING_DURATION_FACTOR` (varsayılan `1.0`) ile çarpar. Pilotta gerçek teslimat süreleri (konum izinden) OSRM tahminiyle karşılaştırılır ve çarpan ayarlanır (ör. 0.85). Çarpan ileride kiracı ayarına taşınabilir.

**4. Çıkarım boru hattı, tekrar üretilebilir ve yalnızca Docker gerektirir.**
- Girdi: `https://download.geofabrik.de/europe/turkey-latest.osm.pbf` + aynı adresteki `.md5` doğrulaması (ODbL, atıf zorunlu).
- Kesim: `osmium extract --strategy smart -p infra/routing/regions/eskisehir.geojson`. Poligon, pilot kiracının hizmet bölgesi bbox'ı (`[30.40, 39.70, 30.65, 39.85]`, [ADR-0006](0006-hizmet-bolgesi-eskisehir.md)) ve her yönde yaklaşık 5 km tampondur: `[30.34, 39.655, 30.71, 39.895]`. Hizmet bölgesi değişirse bu dosya güncellenir (veri, kod değil).
- `osmium-tool`, depodaki küçük bir Dockerfile ile (`infra/routing/osmium.Dockerfile`, Debian paketi) yerel imaj olarak derlenir; harici, doğrulanmamış imaj kullanılmaz.
- Sonra `osrm-extract -p /profiles/motorcycle.lua`, `osrm-partition`, `osrm-customize`. Çıktı `infra/routing/data/` altına yazılır (git'e girmez; `.gitignore` zaten `infra/**/data/`, `*.osm.pbf`, `*.osrm*` içerir).
- İki giriş noktası aynı adımları yürütür: `infra/routing/build-extract.ps1` (Windows PowerShell 5.1) ve `infra/routing/build-extract.sh` (Git Bash / Linux). Kök betik: `npm run routing:build`.
- Tazeleme: aynı komut (md5 değiştiyse yeniden indirir). Canlıda aylık zamanlanmış iş (İterasyon 3 operasyon işi).

**5. Docker Compose.** `osrm` ve `vroom` servisleri `infra/docker-compose.yml` içine **`routing` profiliyle** eklenir; `npm run infra:up` eskisi gibi yalnızca Postgres ve Redis'i açar, `npm run routing:up` rota servislerini açar. Böylece rota verisi olmayan bir makinede demo yaklaşık modda yine çalışır ([ADR-0013](0013-rota-motoru-siniri-ve-yaklasik-mod.md)). Portlar: OSRM `5001 -> 5000`, VROOM `3001 -> 3000` (API zaten 3000'i kullanır; ROTA-UYUMLU-ATAMA taslağındaki 3000 çakışması düzeltildi).

**6. VROOM yapılandırması** `infra/routing/vroom-conf/config.yml`: `router: osrm`, `routingServers.osrm.motorcycle = { host: osrm, port: 5000 }`. Araçlar `profile: "motorcycle"` ile gönderilir.

## Sonuçlar

- (+) Motosikletin erişim kuralları ve dönüş davranışı rota sürelerine yansır; tek parametreyle (çarpan) kalibre edilir.
- (+) Tek komutla tekrar üretilebilir çıkarım; Windows ve Linux'ta aynı Docker adımları. Profil depoda olduğu için değişiklikleri incelenebilir.
- (+) `routing` profili sayesinde mevcut geliştirme akışı ve İterasyon 1 demosu yavaşlamaz.
- (-) `motorcycle.lua`, OSRM sürümü yükseltildiğinde `car.lua`'dan yeniden türetilmelidir (fark listesi ve test bunu kolaylaştırır).
- (-) Canlı trafik yoktur ([ADR-0006](0006-hizmet-bolgesi-eskisehir.md) e); çarpan ortalamayı düzeltir, yoğun saat farkını düzeltmez.
- (-) İlk kurulumda ~0,7 GB indirme ve birkaç dakikalık ön işleme.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| `car.lua` olduğu gibi + yalnızca süre çarpanı (ADR-0009'un ilk notu) | Erişim etiketleri ve dönüş maliyetleri motosiklete uymaz; çarpan bunları düzeltemez. Fark listesi küçük olduğu için özel profil ucuz |
| Valhalla (`motorcycle` / `motor_scooter` maliyet modeli hazır) | Motosiklet modeli daha zengin, ancak matris (`sources_to_targets`) bu boyutta OSRM MLD'den belirgin yavaş; 300 ms bütçesini riske atar. ADR-0001 OSRM + VROOM'u seçmişti. Pilot motosiklete özgü rota sorunları gösterirse VROOM Valhalla'yı da desteklediği için geçiş yolu açıktır |
| GraphHopper (özel model ile motosiklet) | Açık kaynak sürümünde matris API'si yok (ticari); Java yığını ek işletim yükü |
| Google Routes / Route Optimization | Paket başı maliyet ve anahtar gereksinimi ([ADR-0009](0009-rota-uyumlu-atama.md) alternatifler) |
| Tüm Türkiye çıkarımı | Gereksiz bellek ve ön işleme süresi ([ADR-0006](0006-hizmet-bolgesi-eskisehir.md) a) |
| Docker'sız, Windows'a OSRM derlemek | Tekrar üretilemez, VPS ile farklı; Docker zaten altyapı standardı |
