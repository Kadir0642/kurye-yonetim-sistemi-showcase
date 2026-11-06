# ADR-0018: Harita karolarının ve stillerin sunumu (API yolu, PMTiles + HTTP Range, sürümlü değişmez dosyalar, kendi glyph ve sprite)

- **Durum:** Önerildi (İterasyon 3a spesifikasyonuyla birlikte onaya sunuldu)
- **Tarih:** 2026-11-05
- **Ayrıntılandırdığı karar:** [ADR-0016](0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md) madde 1 "Sunum" (ayrı karo sunucusu konteyneri yerine API yolu + üretimde ters vekilin dosya sunucusu)
- **İlgili:** [ADR-0008](0008-acik-kaynak-harita.md), [ADR-0019](0019-kurye-cevrimdisi-harita-ve-navigasyon.md), [ADR-0021](0021-haftalik-osm-guncelleme-hatti.md), [ADR-0022](0022-uretim-dagitimi-ucretsiz-katman.md), SPEC-3a §4, §6

## Bağlam

- ADR-0016, Eskişehir PMTiles dosyasının "Caddy / nginx range veya `go-pmtiles serve`" ile sunulmasını, stil, sprite ve glyph'in aynı sunucudan gelmesini söyler; somut yol, kimlik doğrulama ve önbellek kuralı açık bırakılmıştı.
- İstemciler üç farklı ortamdan erişir: panel (tarayıcı, Vite proxy'si arkasında), Android emülatörü (`10.0.2.2`), gerçek cihaz (LAN IP veya üretim alan adı). Karo sunucusu ayrı bir port / alan adında olursa her ortam için ikinci bir taban adres yapılandırmak gerekir; uygulama bugün yalnızca API taban adresini bilir.
- MapLibre stil dosyası mutlak adresler ister (Native'de göreli adres yok; GL JS'te güvenilir değil). Aynı stil çevrimiçi (HTTP) ve çevrimdışı (`file://`) kullanılacak.
- Harita verisi kişisel veri içermez (OSM türevi); ancak kuryenin IP'si ve gördüğü bölge üçüncü tarafa gitmemelidir (ADR-0016).
- Tüm iş ücretsiz kalmalıdır (ürün sahibi kuralı): ek SaaS, CDN veya ücretli eklenti yok.

## Karar

**1. Tek köken: harita dosyaları API'nin yol ağacında sunulur.** `/api/v1/map/...` altında:
- `GET /map/manifest` (kimlik doğrulamalı; sürüm, stil adresleri, çevrimdışı paket listesi, `ETag`),
- `GET /map/styles/{styleId}` (herkese açık; şablon belirteçleri mutlak adreslerle doldurulmuş stil),
- `GET /map/files/{version}/{fileName}` (PMTiles / MBTiles / ham stil şablonu; `Range` -> 206),
- `GET /map/glyphs/{version}/{fontstack}/{range}`, `GET /map/sprites/{version}/{fileName}`.
Böylece her istemci zaten bildiği API kökünü kullanır; ikinci bir taban adres yoktur. Geliştirmede bu yolları API (Fastify, `reply.sendFile` ile Range / ETag) sunar; **üretimde aynı yollar ters vekil (Caddy `file_server`) tarafından doğrudan diskten** sunulur ([ADR-0022](0022-uretim-dagitimi-ucretsiz-katman.md)), Node'a yük binmez. İstemci farkı görmez. Ayrı karo sunucusu konteyneri (`go-pmtiles serve`, tileserver) **yoktur**.

**2. Sürümlü, değişmez dosya yolları.** Her yayın `MAP_DATA_DIR/releases/<sürüm>/` dizinidir (`2026-w45`, aynı haftada yeniden yayın `2026-w45.1`). Dosyalar yayından sonra değişmez: `Cache-Control: public, max-age=31536000, immutable`, `ETag` = sha256. Güncel sürüm yalnızca `MAP_DATA_DIR/current.json` işaretçisiyle belirlenir (geçici dosyaya yaz + yeniden adlandır; Windows'ta sembolik bağ kullanılmaz). Son 2 sürüm saklanır ([ADR-0021](0021-haftalik-osm-guncelleme-hatti.md)).

**3. Stil şablonları ve belirteçler.** `infra/map/style/` altında dört stil: `admin`, `courier`, `courier-nav-day`, `courier-nav-night` (OpenMapTiles şeması, OpenFreeMap "liberty" türevi, sadeleştirilmiş). Şablonda üç belirteç bulunur: `{{TILES_URL}}`, `{{GLYPHS_URL}}`, `{{SPRITE_URL}}`. API çevrimiçi değerleri (`pmtiles://https://<kök>/api/v1/map/files/<sürüm>/eskisehir.pmtiles` vb.) doldurur; kurye uygulaması çevrimdışı pakette aynı şablonu yerel `file://` yollarıyla doldurur. Tek kaynak, iki hedef. Her `text-font` dizisi **tek** yazı tipidir (bileşik glyph yığını sunmayız).

**4. Kendi glyph ve sprite.** Noto Sans (Regular, Bold, Italic; OFL) glyph PBF'leri yalnızca gerekli aralıklarla (0-255, 256-511, 512-767, 7680-7935, 8192-8447; Türkçe karakterler ilk ikisinde) sürüme kopyalanır. Kaynaklar (glyph paketi, sprite) derleme zamanında sabitlenmiş sürüm ve sha256 ile indirilir (`infra/map/assets.lock.json`); çalışma zamanında üçüncü taraf istek yoktur. Sprite, stilin kullandığı az sayıda simgeyle (tek yön oku, temel POI) sınırlıdır.

**5. Erişim ve kötüye kullanım.** Dosya ve stil uçları herkese açıktır (MapLibre başlıksız ister; veri kamuya açık OSM'dir). Manifest kimlik doğrulamalıdır (kiracı kullanıcıları). Dosya adları sürümün `release.json` listesine karşı doğrulanır (yol geçişi yok). Geliştirmede API'nin mevcut hız sınırı geçerlidir; üretimde dosyalar Caddy'den sunulduğu için IP başına bağlantı sınırı sunucu güvenlik duvarındadır ([ADR-0022](0022-uretim-dagitimi-ucretsiz-katman.md)).

**6. Panel ve eski istemciler config ile geçer.** `MAP_STYLE_URL=self` iken `GET /tenant` yanıtındaki `mapStyleUrl` çağıranın rolüne göre kendi stilimizi gösterir (panel `admin`, kurye `courier`) ve yeni `mapStyles` alanı dört adresi verir. Panel `pmtiles` JS protokolünü kaydeder. Üretimde `MAP_STYLE_URL` üçüncü taraf bir alan adı olamaz (`MAP_ALLOW_THIRD_PARTY=false`, ortam doğrulaması).

**7. ZXY yedeği yoktur (karar, 3a entegrasyonu).** `GET /map/tiles/{version}/{z}/{x}/{y}` sözleşmeden **kaldırıldı**: cihaz spike'ı ([ADR-0019](0019-kurye-cevrimdisi-harita-ve-navigasyon.md)) PMTiles'ın `pmtiles://` ile hem yerel dosyadan hem `Range` ile çizildiğini gösterdi; ZXY yedeğine gerek kalmadı. Sözleşmede hiçbir `x-planned` işareti kalmadı.

## Sonuçlar

- (+) Ek konteyner, port veya alan adı yok; emülatör, LAN ve üretim aynı API köküyle çalışır.
- (+) Değişmez sürüm yolları: istemci ve ters vekil önbelleği sonsuz; geçiş yalnızca işaretçi değişimi, geri dönüş işaretçiyi geri almak.
- (+) Çevrimiçi ve çevrimdışı stil aynı şablondan; "çevrimdışında farklı görünüyor" hatası sınıfı kalkar.
- (+) Sıfır maliyet: Caddy, MapLibre, planetiler, Noto Sans açık kaynak ve ücretsiz.
- (-) Geliştirmede 15-40 MB dosya Node üzerinden sunulur (yerel ağda önemsiz); üretimde Caddy'nin aynı yolu diskten sunması P görevinin yapılandırmasına bağlıdır (unutulursa çalışır ama Node yüklenir).
- (-) API, `Host` / `X-Forwarded-*` başlıklarından kök türettiği için `TRUST_PROXY` doğru ayarlanmalıdır; yanlışsa stil içindeki adresler yanlış köke işaret eder (`MAP_PUBLIC_BASE_URL` ile sabitlenebilir).
- (-) Herkese açık dosya uçları kazınabilir; veri zaten kamuya açık OSM olduğundan risk düşük, bağlantı sınırı yeterli.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Ayrı karo sunucusu konteyneri (`go-pmtiles serve`, tileserver-gl, martin) | Ek servis ve port; istemcide ikinci taban adres; çevrimdışı paket için yine dosya listesi gerekir |
| Statik dosyaları ayrı alt alan adından (tiles.example) sunmak | Ücretli / ek alan adı yönetimi, CORS; tek köken daha basit |
| Stili derleme anında mutlak adresle üretmek | Emülatör / LAN / üretim için ayrı derleme gerekir; çalışma zamanında doldurma tek dosyayla çözer |
| Ücretli CDN / barındırılan karo (MapTiler, Stadia, Mapbox) | Ücretli ve üçüncü taraf; ürün sahibinin kuralına ve ADR-0016'ya aykırı |
| Caddy `pmtiles` eklentisi (xcaddy ile özel derleme) | Özel Caddy derlemesi bakım yükü; düz `file_server` Range'i zaten destekler |

## Uygulama notları

### Karo derlemesi, stiller ve panel (Görev T)

Uygulama: `infra/map/` (README), panel `apps/admin-web/src/lib/mapStyle.ts`. Ölçümler Windows 11 + Docker Desktop (WSL2, 20 çekirdek, 15,8 GB), girdi `infra/routing/data/eskisehir.osm.pbf` (3,83 MiB, 281 z14 karosu) üzerindedir.

| Ölçüt | Sonuç |
|---|---|
| PMTiles boyutu | **2,72 MiB** (2.848.636 bayt; 390 karo, z0-z14; hedef <= 60 MB, tahmin 15-40 MB idi: çıkarım küçük ve z14 üstü karo yok) |
| Çevrimdışı paket (23 dosya) | **4,18 MiB** (4.382.429 bayt): karo 2,72 + glyph (Regular, Bold; 10 dosya) 0,91 + üç kurye stili 0,09 + courier sprite 0,006 |
| Tüm sürüm (28 dosya) | 4,22 MiB (admin stil + admin sprite + Italic glyph dahil) |
| Derleme süresi, sıcak önbellek | **91-94 sn** (`npm run map:build`: planetiler adımı 58-64 sn [JVM + Natural Earth açma + planetiler'in kendi 30 sn'si], pmtiles convert 4-12 sn, stil doğrulayıcı (npx) 7-11 sn) |
| Derleme süresi, soğuk | yaklaşık 7 dk: kaynak indirme (Natural Earth 414 MB ağırlıklı, bu ağda ~5 dk) + yukarıdaki 1,5 dk. Sonraki derlemelerde kaynaklar `infra/map/data/sources/` önbelleğinde (551 MB, git dışı) |
| planetiler tepe bellek | **2,55-2,65 GB** (konteyner cgroup `memory.peak`, sayfa önbelleği dahil; JVM `-Xmx3g`) |
| Tekrar üretilebilirlik | **Aynı**: PowerShell ile iki ve Git Bash ile bir derleme **aynı `tilesContentHash` ve aynı PMTiles dosya sha256'sını** verdi (`ee1d1b0f…c1579`, `572b1510…a51dfe`). PMTiles üst verisindeki `planetiler:buildtime` aracın derleme zamanıdır (çalışma zamanı değil), bu yüzden dosya baytları da aynı kalır |

Kararlar ve bulgular:

- **`tilesContentHash` tanımı:** karolar (z, x, y) artan sırayla, her karonun sıkıştırması açılmış baytlarının sha256'sı `z/x/y:<hex>
` satırı olarak ikinci bir sha256'ya beslenir. Bu, gzip sürümü / dizin yerleşimi farklarından bağımsızdır; dosya sha256'sı da aynı çıktığı için ek koruma sağlar.
- **Windows G/Ç bulgusu:** planetiler'in Natural Earth zip'ini (849 MB SQLite) Windows bağlama dizinindeki `/tmp`'ye açması 10+ dk sürdü; konteyner içinde `jar xf` 7 sn. Betik Natural Earth'ü ve geçici dosyaları konteyner içinde (`/tmp`) tutar; PMTiles / MBTiles son adımda bağlama dizinine kopyalanır (derleme 5 dk+ yerine 1,5 dk).
- **Su poligonları (okyanus) yer tutucu:** `osmdata.openstreetmap.de` geliştirme ağından erişilemedi (TLS sıfırlama; Docker içinden de). Eskişehir iç bölge olduğundan okyanus poligonu karoyu etkilemez; planetiler için okyanus ortasında tek küçük poligonlu geçerli bir shapefile üretilir. Kıyı şehri eklenirse gerçek dosya `assets.lock.json`'a sha256'sıyla eklenmelidir.
- **Derleme zamanı kaynakları sha256 kilitli** (`infra/map/assets.lock.json`): Natural Earth 414 MB (`naciscdn.org`, 2022'den beri değişmedi), göl orta hatları 77 MB (`acalcutt/osm-lakelines v12`), glyph paketi 59 MB (`openmaptiles/fonts v2.0`). Uyuşmazlık derlemeyi durdurur. Uzak dosyalar değişebilir; bu durumda bilinçli yeniden kilitleme (`--pin`) gerekir ve sürüm notu yazılır.
- **Glyph:** OpenMapTiles fonts v2.0 `noto-sans.zip` (OFL); `Noto Sans Regular / Bold / Italic` x 5 aralık = 15 dosya, 1,37 MiB. Kurye stilleri yalnızca Regular + Bold kullanır (çevrimdışı pakette 10 dosya).
- **Sprite:** üçüncü taraf paket yerine bağımsız üretilen küçük sprite (7 simge: iki tek yön oku, eczane / hastane / akaryakıt / cami / okul rozetleri); lisans yükü yok, PNG belirlenimci. Aynı küme `courier` ve `admin` adlarıyla yayınlanır.
- **Stiller bir üreticiden** (`generate-styles.mjs`) gelir; `courier-nav-night`, `courier-nav-day`'den uygulamadaki gece dönüşümünün birebir formülüyle türetilir. Katman sayıları: courier 36, nav 34, admin 40. Doğrulayıcı `@maplibre/maplibre-gl-style-spec@26.4.4`: dört stil (doldurulmuş sürümde) 0 hata. Şablonun `glyphs` değeri `{fontstack}` içermediği için doğrulayıcı şablonu reddeder; doğrulama doldurulmuş sürümde yapılır.
- **Karo içeriği (z14):** `housenumber` 498, `poi` 8.828 öğe (karolar arası tekrar dahil), `transportation_name` 4.693; `poi` ve `place` ad alanları `name:tr` içerir. Bina katmanında ad **yoktur** (OpenMapTiles şeması); "bina adı" gereksinimi `poi.name` ile (okul, AVM, yurt, cami) karşılanır. Eskişehir'de OSM kapı numarası kapsamı seyrektir (ADR-0016 "dürüst sınır"): z17 görünümünde ekranda birkaç numara görünür.
- **Panel:** `pmtiles` 4.5.0 protokolü `main.tsx`'te bir kez kaydedilir; tüm haritalar `tenant.mapStyles?.admin ?? tenant.mapStyleUrl` kullanır. Gerçek API (Q) + `MAP_STYLE_URL=self` + `MAP_PUBLIC_BASE_URL=http://localhost:5183` ile canlı harita Eskişehir'i kendi karolarımızdan çizdi; tarayıcı ağ günlüğünde (311 istek) yalnızca `localhost:5183` (Vite vekili) alan adı vardı: `/api/v1/map/styles/admin`, `/map/files/<sürüm>/eskisehir.pmtiles` (206 Partial Content), `/map/sprites`, `/map/glyphs`. Üçüncü taraf harita alan adına istek yok.
- Karşılaştırma ve görüntüler: [`docs/assets/iterasyon-3a/`](../assets/iterasyon-3a/) (`t-openfreemap-karsi-kendi.jpg`, `t-panel-canli-harita-kendi-karo.jpg`, `t-stil-*.jpg`).
- **Uygulanmayan / kaldırılan:** `GET /map/tiles/...` (ZXY yedeği) M'nin spike sonucuyla gereksiz kaldı ve sözleşmeden silindi (3a entegrasyonu). MBTiles için `--with-mbtiles` ve `publish --offline-format mbtiles` hazırdır.


### Sunum uçları (Görev Q, API)

Uygulama: `services/api/src/map/` (`MapReleaseService`, `MapService`, `MapController`). Sunum davranışı ve ölçülen sonuçlar:

- **Okuyucu.** `MAP_DATA_DIR/current.json` ve `releases/<sürüm>/release.json` her istekte `stat` ile denetlenir; mtime / boyut değişince yeniden okunur, değişmediyse bellekteki ayrıştırılmış kopya kullanılır (hat `current.json`'u atomik değiştirdiğinde API yeniden başlatma gerektirmez). Dosya yoksa, bozuksa veya şemaya uymuyorsa (zod; yol ayırıcı, `..`, mutlak yol reddedilir) `503 MAP_UNAVAILABLE` + `Retry-After: 30`; sunucu çökmez.
- **Dosyalar.** Ek bağımlılık eklenmedi (`@fastify/static` kullanılmadı): `fs.createReadStream` ile bayt aralığı akıtılır. `Range` tek aralık -> 206 + `Content-Range`; karşılanamaz / bozuk aralık -> 416 + `Content-Range: bytes */<boyut>`; çoklu aralık -> tam gövde 200 (sözleşme "tek aralık yeterli"). `ETag` = dosyanın sha256'sı (`If-None-Match` -> 304), `Cache-Control: public, max-age=31536000, immutable`, `Cross-Origin-Resource-Policy: cross-origin`. Dosya sistemi yolu kullanıcı girdisinden değil, `release.json` listesindeki kayıttan kurulur (ad beyaz listesi); listede olmayan ad, `release.json`'ın kendisi ve alt dizin dosyalarının kök uçtan istenmesi 404.
- **Stil.** `style.<id>.json` şablonundaki `{{TILES_URL}}`, `{{GLYPHS_URL}}`, `{{SPRITE_URL}}` istek köküyle doldurulur; `glyphs` içindeki `{fontstack}` / `{range}` yer tutucuları aynen kalır; sprite `admin` / `courier` ayrımı stil kimliğinden gelir. `?version=` yalnızca güncel ve önceki sürüm için (aksi 404), `immutable`; sürümsüz istek `public, max-age=300`.
- **Kök.** `MAP_PUBLIC_BASE_URL` doluysa her zaman o; değilse Fastify'ın `request.protocol` / `request.host` değeri (`TRUST_PROXY=true` iken `X-Forwarded-*`, aksi hâlde `Host`; sahte `X-Forwarded-*` başlığı güvenilmeyen modda yok sayılır, test denetler). Ana makine adı karakter kümesi ve `new URL()` ile doğrulanır (stil JSON'una enjeksiyon / önbellek zehirlenmesi önlemi); kök yoksa `Vary: Host` döner.
- **Manifest.** `current.json` + `release.json`'dan üretilir; `files[].path` ilk segmentine göre `files|sprites|glyphs` uç adresi seçilir; yalnızca `offline: true` dosyalar `offlinePackage`'tadır; `ETag` gövdenin sha256'sı (ilk 32 hex), `private, max-age=300`, `If-None-Match` -> 304.
- **Tenant.** `MAP_STYLE_URL=self` iken `mapStyleUrl` rol bazlıdır (OWNER / DISPATCHER `admin`, COURIER `courier`) ve `mapStyles` dört adres verir; yayınlanmış sürüm yokken geliştirmede `MAP_FALLBACK_STYLE_URL` + uyarı logu (dakikada en çok bir), üretimde adres yine döner ve stil ucu 503 verir. Eski değer (tam URL) değişmeden döner.
- **Test fikstürü.** `services/api/test/fixtures/map-release/` (`generate.mjs` iki kurgusal sürüm üretir: `2026-w44`, `2026-w45`; gerçek karo içermez).
- **`GET /map/tiles/...` sözleşmeden kaldırıldı** (spike gereksiz kıldı).
