# ADR-0021: Haftalık OSM güncelleme hattı (bağımlılıksız Node orkestra betiği, sürüm dizinleri, iç ağ geçidiyle mavi / yeşil OSRM ve Photon, kapılar, geri alma)

- **Durum:** Önerildi (İterasyon 3a spesifikasyonuyla birlikte onaya sunuldu)
- **Tarih:** 2026-11-05
- **Ayrıntılandırdığı karar:** [ADR-0016](0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md) madde 4 (haftalık hat, kapılar, sıfır kesinti geçiş)
- **İlgili:** [ADR-0011](0011-osrm-motosiklet-profili-ve-rota-altyapisi.md), [ADR-0017](0017-saha-verisinden-ogrenen-harita.md) (3c'de hız dosyası adımı), [ADR-0018](0018-harita-karo-ve-stil-sunumu.md), [ADR-0022](0022-uretim-dagitimi-ucretsiz-katman.md), SPEC-3a §9

## Bağlam

- Bugün OSRM verisi elle üretilir (`npm run routing:build`, PowerShell + Bash ikiz betik). İterasyon 3a'da karo (planetiler) ve Photon dizini (geçici Nominatim) eklenir; üçü aynı OSM sürümünden, haftada bir, kapılı ve geri alınabilir yenilenmelidir.
- Hat hem ürün sahibinin **Windows 11 + Docker Desktop** makinesinde (bugünkü tek ortam) hem ileride **Linux VPS**'te çalışmalıdır. Karmaşık bir orkestrayı PowerShell 5.1 ve Bash'te iki kez yazmak hataya açıktır.
- API, OSRM'e tek bir adresle (`OSRM_URL`) konuşur; VROOM OSRM'e kendi yapılandırmasıyla bağlanır; Photon'a `PHOTON_BASE_URL` ile. Geçiş, API yeniden başlatması gerektirmemelidir.
- Mevcut `infra/docker-compose.yml` `routing` profilindeki tek `osrm` servisi geliştirme için yeterlidir ve İterasyon 2 demosunu bozmamak için değişmemelidir. Paylaşılan, hazır OSRM verisi (`infra/routing/data`) silinmemeli ve yeniden derlenmemelidir.

## Karar

**1. Orkestra betiği: bağımlılıksız Node.js (ESM, `.mjs`), `infra/pipeline/`.** Node 22 (depo zaten `engines.node >= 22.12` ister; VPS'e Node kurmak ücretsizdir). Yalnızca yerleşik modüller (`child_process`, `fs`, `crypto`, `fetch`); `docker` CLI'ı çağırır. Bileşen derleme adımları mevcut ikiz betik düzenini korur: `infra/routing/build-extract.{ps1,sh}`, `infra/map/build.{ps1,sh}`, `infra/geocoder/build.{ps1,sh}` her biri `--pbf <dosya> --out <dizin>` alır; orkestra işletim sistemine göre doğru betiği çağırır. Komutlar: `pipeline run`, `pipeline status`, `pipeline rollback [bileşen]`, `pipeline gc`. Kapı mantığı saf fonksiyonlardır ve `node --test` ile birim testlidir.

**2. Sürüm dizinleri ve tek işaretçi.**
```
infra/pipeline/data/            (git dışı; prod: /srv/kurye/pipeline)
  cache/turkey-latest.osm.pbf (+ .md5)     indirme önbelleği
  releases/2026-w45/
    eskisehir.osm.pbf  osrm/  photon/  build.log  gates.json  release.json
  state.json   { current: { osrm, photon, tiles }, previous: {...}, lastRun: {...} }
  lock         aynı anda tek koşu (PID + başlangıç zamanı; 6 sa'ten eski kilit bayat sayılır)
```
Karo sürümü ayrıca `infra/map/data` altına yayınlanır ([ADR-0018](0018-harita-karo-ve-stil-sunumu.md) dizin sözleşmesi) ve `current.json` işaretçisi atomik değiştirilir. Son 2 sürüm saklanır; daha eskiler `gc` ile silinir. Hat `infra/routing/data`'ya **yazmaz**; o dizindeki PBF ve md5 yalnızca salt-okunur önbellek olarak kullanılabilir.

**3. Sıfır kesinti geçiş: iç ağ geçidi + mavi / yeşil.** Ayrı compose dosyası `infra/pipeline/compose.pipeline.yml` (profil `pipeline`):
- `routing-gw`: Caddy (yalnızca iç ağ ve yerelde `127.0.0.1`), iki dinleyici: `:5000` -> etkin OSRM, `:2322` -> etkin Photon. Yerel portlar geliştirmeyle aynı (`5001`, `2322`), böylece `OSRM_URL` ve `PHOTON_BASE_URL` değişmez.
- `osrm-blue`, `osrm-green`, `photon-blue`, `photon-green`: her biri bir sürüm dizinini salt-okunur bağlar.
- Geçiş: pasif rengi yeni sürümle başlat -> sağlık denetimi + kapı istekleri doğrudan pasif konteynere -> Caddy yönetim API'siyle (yalnızca konteyner içi `localhost:2019`) upstream'i değiştir (zarif yeniden yükleme; açık istekler biter) -> eski rengi 24 sa açık bırak (geri dönüş) -> sonra durdur.
- VROOM bu modda OSRM'e `routing-gw` üzerinden bağlanır (`infra/pipeline/vroom-conf/config.yml`).
- Geliştirme modunda (`routing:up`) geçit yoktur; `pipeline:up` öncesi `routing:down` gerekir (aynı port). İki mod aynı anda çalışmaz; README bunu yazar.

**4. Kapılar (geçişten önce, hepsi geçmeli).**
- OSRM: sabit rota test seti (en az 30 çift, `infra/pipeline/gates/routes.json`); her çiftte rota bulunmalı ve süre önceki sürümün ±%15'i içinde olmalı (çiftlerin en az %90'ı); `table` 25x25 yanıt verir.
- Karo: PMTiles başlığı okunur (`pmtiles verify` / `show`), bbox ve yakınlaştırma beklenen, boyut önceki sürümün ±%20'si, stil şablonları MapLibre stil doğrulayıcısından geçer.
- Photon: altın sorgu seti (`infra/geocoder/golden-queries.json`, en az 30 sorgu); ilk-5 isabet oranı önceki sürümden en çok 2 puan düşük ve mutlak eşik üstünde (SPEC-3a §9.3).
- Kapı kalırsa geçiş yapılmaz, eski sürüm sürer, `gates.json` nedenleri yazar, uyarı gider (P görevinin izleme kancası: çıkış kodu + `state.json.lastRun`).

**5. Zamanlama ve uyarı.** Linux: systemd timer (Pazar 03:00 Europe/Istanbul, `Persistent=true`, `nice` / `ionice`). Windows (yerel): Görev Zamanlayıcı ile isteğe bağlı; elle `npm run pipeline:run`. Başarısızlık / kapı reddi Uptime Kuma push monitörüne bildirilir ([ADR-0022](0022-uretim-dagitimi-ucretsiz-katman.md)).

**6. Geri alma.** `pipeline rollback` `state.json.previous` sürümünü etkin yapar: OSRM / Photon için geçit upstream'ini eski renge (24 sa içinde hâlâ açık) veya eski sürüm diziniyle yeniden başlatılan pasif renge çevirir; karo için `current.json`'u geri alır. Hedef: 2 dk içinde.

## Sonuçlar

- (+) Tek orkestra, iki işletim sistemi; mantık testli, kopyası yok.
- (+) Geçişte API / VROOM yapılandırması değişmez; Caddy zarif yeniden yükleme sıfır başarısız istek hedefini mümkün kılar (ölçülür, iddia edilmez).
- (+) Bileşen betikleri tek başına da çalışır (geliştirici yalnızca karoyu derleyebilir).
- (+) Sıfır maliyet: Caddy, Node, Docker, OSM verisi ücretsiz.
- (-) Derleme sırasında geçici bellek tepeleri (planetiler + Nominatim, tahmin 2-4 GB her biri); hat bileşenleri sırayla derleyebilir (`--sequential`), küçük VPS'te varsayılan sıralı.
- (-) Mavi / yeşil, OSRM ve Photon için geçiş penceresinde iki kat bellek (OSRM ~0,24 GB ölçüldü; Photon ölçülecek).
- (-) Geliştirme modu ile hat modu arasında port çakışması; elle mod değiştirme gerekir.
- (-) Prod'da hat konteyner değil ana makinedeki Node ile çalışır (Docker soketini bir konteynere bağlamamak için); VPS'e Node 22 kurulmalıdır.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| PowerShell + Bash ikiz orkestra | Karmaşık durum makinesini iki dilde sürdürmek; test zor |
| TypeScript + tsx (API bağımlılıklarıyla) | Hat API'nin derlemesine ve `node_modules`'e bağlanır; VPS'te ek kurulum |
| Docker ağ takma adını (`osrm`) kopar / bağla ile değiştirmek | Kısa kesinti; zarif değil |
| Kubernetes / Nomad | Tek VPS için aşırı; işletim yükü |
| Hat konteyneri + `/var/run/docker.sock` bağlama | Kök eşdeğeri yetki; ana makinede Node daha az riskli |
| Geofabrik günlük farkları (`pyosmium-up-to-date`) | Optimizasyon; tam indirme (~7 dk) şimdilik yeterli (ADR-0016) |
| Ücretli CI / bulut derleme makinesi | Ücretli; gerek yok, derleme VPS'te gece veya ürün sahibinin makinesinde |

## Uygulama notları

Ölçüm: Windows 11, Docker Desktop (WSL2, 15,5 GB), 6 Kasım 2026; makinede başka ajan işleri de çalışıyordu (süreler üst sınır). Ayrıntı ve komutlar: infra/pipeline/README.md, CALISMA-KITABI.md, infra/geocoder/README.md.

| Ölçüm | Değer |
|---|---|
| Uçtan uca `pipeline run` (indirme hariç, Türkiye PBF girdisi, sıralı) | **2 dk 51 sn**: osmium extract 80 sn, OSRM 10 sn, Photon (Nominatim + import) 63 sn, pasif renk başlatma 12 sn, kapılar 2 sn, geçiş 1 sn (hedef <= 30 dk) |
| Aynı, Eskişehir PBF girdisiyle (osmium atlanmaz, 4 MB girdi) | 2 dk 2 sn - 2 dk 42 sn |
| Yük altında (başka işler varken) | osmium extract 284 sn'ye çıktı; yine toplam < 10 dk |
| Derleme belleği (tepe, `docker stats`) | Nominatim 0,4-1,1 GiB, Photon import 0,8-0,9 GiB; sıralı koşuda tepeler üst üste binmez |
| Çalışırken bellek | Photon 0,67-0,81 GiB (her renk), OSRM 27-39 MiB, Caddy 51 MiB; geçiş penceresinde iki renk toplamı yaklaşık 1,6 GiB |
| Photon dizini | **4,1 MiB** (8 986 belge); ADR'deki "yüzlerce MB" tahmini Eskişehir için geçersiz |
| Sürüm dizini boyutu | 42 MiB (PBF + OSRM + Photon); son 2 sürüm + önbellek birkaç yüz MB |
| Kesinti (loadtest, geçiş sırasında) | 337 istek (169 `route`, 84 `table`, 84 Photon), **0 hata**; route p95 25 ms (en fazla 149 ms), Photon p95 48 ms (en fazla 273 ms) |
| `pipeline rollback` | 19 sn (durdurulmuş rengi eski sürümle yeniden başlatma, OSRM + Photon); 1,8 sn (hedef renk canlıyken yalnızca geçit); hedef <= 2 dk |
| Bozuk girdi | `--pbf <rastgele bayt>`: osmium hatası, çıkış 1, eski sürüm sürer; `--inject-gate-failure tiles`: çıkış 2, geçiş yok, eski sürüm sürer |
| Kapı değerleri (ilk gerçek koşu) | OSRM 32 / 32 rota bulundu, `table` 25x25 35 ms; Photon altın set 50 / 55 (%90,9) |

Uygulamada doğrulanan sapmalar ve notlar:

- **Photon 1.x CLI:** `-nominatim-import` yerine `import` alt komutu; `-synonym-file` yalnızca `serve` sırasında etkili (içe aktarmada etkisiz) ve yalnızca tek sözcüklü terimler kabul eder. Photon dizini OpenSearch gömülü modda yazılabilir dizin ister; sürüm dizini konteynere salt-okunur bağlanır, giriş betiği yazılabilir bir kopya oluşturur (dizin 4 MiB olduğu için maliyetsiz).
- **Bileşen betikleri:** OSRM için `build-extract.{ps1,sh}` `--pbf` / `--out` (PowerShell: `-Pbf` / `-Out`) aldı ve bileşen kipinde `infra/routing/data`'ya yazmaz. Photon derlemesinin mantığı tek yerde (`infra/geocoder/build.mjs`), `.ps1` / `.sh` ince sarmalayıcıdır (iki dilde ikiz durum makinesi yerine); Windows ve Git Bash aynı sonucu verir. OSRM çıktıları çok iş parçacıklı olduğu için baytça aynı değildir (dosya kümesi ve profil sha256 aynı).
- **Compose:** hat servisleri ayrı proje adı `kurye-pipeline` altında (`infra` projesiyle yetim konteyner karışıklığı olmasın diye); P prod'da `-f` ile eklerken `PIPELINE_COMPOSE_FILES` / `PIPELINE_COMPOSE_PROJECT` ile yol verilir.
- **Kapılar ilk koşuda yalnızca mutlak eşiklerle** çalışır (önceki sürüm yok); `gates.json` her sürümün rota sürelerini saklar, sonraki koşu ±%15 karşılaştırmasını bundan yapar.
- **Doğrulanmayanlar:** karo adımı uçtan uca (T birleşmedi; `--skip-tiles` ve otomatik atlama), Linux systemd zamanlayıcısı, Uptime Kuma push (adres yok), 24 saat dolunca pasif renk durdurma (mantık birim testli, zamanla gözlenmedi), `--parallel` kipi.
- **Gerçek kullanım notu:** koşular arasında dizin geçişi `docker compose up --wait` ile pasif rengi yeniden oluşturur; bu sırada eski sürümün son geri alma rengi yeni sürümle değiştirilir (tek derinlikli geri alma: yalnızca bir önceki sürüm).
