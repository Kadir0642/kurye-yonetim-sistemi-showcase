# Karar Kayıtları (ADR)

Mimari ve teknoloji kararları, bağlamı ve alternatifleriyle birlikte burada tutulur. Karar değişirse ilgili ADR güncellenir (eski karar silinmez, "Yerine geçti" olarak işaretlenir).

## Dizin

| No | Başlık | Durum |
|---|---|---|
| [0001](0001-hibrit-harita-mimarisi.md) | Hibrit harita mimarisi (Google Places + OSRM/VROOM) | Kabul edildi (harita gösterimi ADR-0008 ile değişti) |
| [0002](0002-kurye-uygulamasi-react-native-expo.md) | Kurye uygulaması: React Native + Expo, tek kod Android + iOS | Kabul edildi |
| [0003](0003-dagitim-apk-self-update.md) | Dağıtım: Android APK + OTA, iOS TestFlight / özel uygulama | Kabul edildi |
| [0004](0004-cok-kiracili-mimari.md) | Çok kiracılı (multi-tenant) mimari (RLS İterasyon 2'de) | Kabul edildi |
| [0005](0005-konum-takibi-ve-kvkk.md) | Konum takibi ve KVKK | Kabul edildi |
| [0006](0006-hizmet-bolgesi-eskisehir.md) | Hizmet bölgesi: Eskişehir pilotu, çok şehre hazır tasarım | Kabul edildi |
| [0007](0007-backend-ve-web-yigini.md) | Backend ve web yığını: uçtan uca TypeScript monorepo | Kabul edildi |
| [0008](0008-acik-kaynak-harita.md) | Açık kaynak harita (MapLibre + OpenFreeMap), sağlayıcı arkasında adres arama | Kabul edildi (karo sağlayıcısı ve genel Photon ADR-0016 ile değişti) |
| [0009](0009-rota-uyumlu-atama.md) | Rota uyumlu atama: kurye başına canlı rota, sapma süresine dayalı uygunluk, üç atama modu | Kabul edildi |
| [0010](0010-kuryeler-arasi-devir.md) | Kuryeler arası devir: yakındaki meslektaşa teklif, alıştan sonra fiziksel buluşma ve kanıt (kod + konum) | Kabul edildi |
| [0011](0011-osrm-motosiklet-profili-ve-rota-altyapisi.md) | OSRM motosiklet profili ve rota altyapısı (OSRM MLD + VROOM, Docker, Eskişehir çıkarımı) | Önerildi |
| [0012](0012-rls-iterasyon-2b-ye-ertelendi.md) | PostgreSQL RLS İterasyon 2b'ye ertelendi; İterasyon 2 tabloları RLS'ye hazır | Önerildi |
| [0013](0013-rota-motoru-siniri-ve-yaklasik-mod.md) | Rota motoru sınırı, yaklaşık (düşüş) modu ve süreç içi kesin sıralayıcı | Önerildi |
| [0014](0014-toplu-atama-regret-ekleme.md) | Toplu atama: mevcut rotalara regret-2 ekleme, gecikme sınırı, aynı işletme gruplaması | Önerildi |
| [0015](0015-uygulama-ici-navigasyon.md) | Uygulama içi adım adım navigasyon: kendi hafif navigasyonumuz (OSRM adımları + MapLibre + Türkçe sesli yönlendirme), Google Maps devri yedek | Kabul edildi |
| [0016](0016-eskisehire-ozel-kendi-barindirdigimiz-harita-platformu.md) | Eskişehir'e özel, kendi barındırdığımız harita platformu (planetiler + PMTiles, kendi Photon, haftalık OSM hattı, çevrimdışı harita); ADR-0008'i kısmen değiştirir | Kabul edildi |
| [0017](0017-saha-verisinden-ogrenen-harita.md) | Saha verisinden öğrenen harita: öğrenilmiş teslim noktaları ve öğrenilmiş hızlar | Kabul edildi |
| [0018](0018-harita-karo-ve-stil-sunumu.md) | Harita karolarının ve stillerin sunumu: API yolu (prod'da Caddy diskten), PMTiles + Range, sürümlü değişmez dosyalar, kendi glyph / sprite | Önerildi |
| [0019](0019-kurye-cevrimdisi-harita-ve-navigasyon.md) | Kurye uygulamasında çevrimdışı harita (yerel PMTiles, manifest + sha256) ve çevrimdışı navigasyon (rota bacağı önbelleği) | Önerildi |
| [0020](0020-cevrimdisi-eylem-kuyrugu-ve-idempotency.md) | Çevrimdışı eylem kuyruğu ve sunucu tarafı idempotency (Idempotency-Key, cihaz olay zamanı, çakışma sözleşmesi) | Önerildi |
| [0021](0021-haftalik-osm-guncelleme-hatti.md) | Haftalık OSM güncelleme hattı: Node orkestra, sürüm dizinleri, iç geçitle mavi / yeşil OSRM ve Photon, kapılar, geri alma | Önerildi |
| [0022](0022-uretim-dagitimi-ucretsiz-katman.md) | Pilot öncesi üretim dağıtımı, yalnızca ücretsiz bileşenlerle (Caddy + Let's Encrypt, saatlik pg_dump, Uptime Kuma, yalın CI, sırlar, APK imzalama) | Önerildi |
| [0023](0023-dagitim-algoritmasi-v2.md) | Dağıtım algoritması v2: kayan ufuk toplu atama + sınırlı kuryeler arası ALNS (bayrak arkasında); CBBA yalnızca araştırma taban çizgisi | Kabul edildi (Faz 2: bayrak arkasında) |
| [0024](0024-navigasyon-render-performansi.md) | Navigasyon ekranında render performansı: yerel öngörülü kamera, rota yönünden course-up, konum başına harita ağacı çizimi yok, rota baş / kuyruk, hedef işareti yalnızca ekrana yakınken, düşük donanım kademesi | Kabul edildi |
| [0025](0025-isletme-hazir-baglantisi.md) | İşletme "paket hazır" bağlantısı: girişsiz, kısa ömürlü opak jeton (URL parçasında), yalnızca o işletmenin açık paketleri | Kabul edildi |
| [0026](0026-teslim-geofence-politikasi.md) | Teslim kanıtı geofence politikası: ölçüm kipiyle başlangıç, etkin teslim noktasına tolerans, gerekçeli geçersiz kılma, öğrenmeden dışlama | Kabul edildi |
| [0027](0027-marka-kimligi-ve-navigasyon-gorsel-dili.md) | Marka kimliği ve navigasyon görsel dili: koyu krom + neon yeşil, gündüz açık harita, koyu krom üzerinde metin ≥ 7:1 (önce kurye uygulaması, panel ayrı görev) | Kabul edildi |
| [0028](0028-havuz-gorunurlugu-ve-atama-kabulu.md) | Havuz görünürlüğü (kiracı ayarı) ve atama kabulü: kabul paket durumu değil atamanın özelliği, veritabanı son tarihi, süre aşımı = sistem reddi (MVP+ 4a) | Kabul edildi |
| [0029](0029-konum-kademeleri-ve-rota-geri-bildirimi.md) | Konum sıklığı kademeleri (NAVIGATING / MOVING / STATIONARY, hysteresis), panel konum yayınında birleştirme (kurye başına <= 1 olay / 3 sn) ve "Rota yanlış" geri bildirimi (OSM düzeltmesi için kimliksiz GeoJSON) | Kabul edildi |
| [0030](0030-kimlik-sertlestirme.md) | Kimlik sertleştirmesi: giriş kilidi `kimlik + IP` (5 / 15 dk) + kimlik başına 50 / saat eşik + artan gecikme; yeni şifre en az 8 karakter, gömülü yaygın şifre listesi, telefon içermez; panelin verdiği şifre geçici, ilk girişte değiştirme (403 `PASSWORD_CHANGE_REQUIRED`) | Kabul edildi |
| [0031](0031-panel-csp-ve-baglanti-sinirlari.md) | Panel içerik güvenlik politikası (CSP, önce Report-Only; tek satırla zorunlu) ve bağlantı / istek sınırları: WebSocket IP başına 20 kimliksiz + kullanıcı başına 10 kimlikli bağlantı, `/health` 60/dk + 5 sn önbellek, Caddy başlık / gövde sınırı | Kabul edildi |
| [0032](0032-musteri-verisi-saklama.md) | Müşteri verisi saklama süresi: kiracı ayarı `settings.privacy.customerRetentionDays` (0 = kapalı, 7-365; yeni kiracıda varsayılan 30, mevcut kiracılar migration ile kapalı); saatlik iş + elle çalıştırma terminal paketlerde ad / telefon / açık adresi geri alınamaz biçimde anonimleştirir (mahalle-ilçe düzeyi adres, ~100 m koordinat, olay / devir notları temizlenir); istatistikler ve öğrenilmiş noktalar kalır | Kabul edildi |
| [0034](0034-kapida-tahsilat-mutabakati.md) | Kapıda tahsilat kaydı ve kasa mutabakatı: teslim eylemine isteğe bağlı `collection` (nakit / kart / alınamadı; çevrimdışı kuyrukta aynı gövde ve Idempotency-Key, sunucu teslimi asla reddetmez, kırpar), `packages.collected_*`, yalnızca eklenen `cash_handovers` (düzeltme eksi + notlu), `GET /payments/daily`, `POST /payments/handover`, `GET /courier/payments/today`; panel "Tahsilat" sayfası + CSV; anonimleştirme tutarlara dokunmaz | Kabul edildi |
| [0033](0033-yuk-testi-darbogazlari.md) | Yük testi darboğazları: panel liste yenilemeleri olay güdümlü (`package.updated` yalnızca etkilenen sorguları yeniler, yoklama 120/300 sn, sekme gizliyken durur); bekleyenler kuyruğu yeniden deneme aralığı kiracı ayarı `settings.dispatch.retryIntervalS` (15-120, varsayılan 60, pilot 30) + kapasite açılınca erken deneme; konum satırı eklemesi `synchronous_commit=off` (p99 kuyruk ucu fsync beklemesiydi) | Kabul edildi |
| [0035](0035-mola-durumu.md) | Kurye mola durumu: `shifts.break_started_at` / `break_ends_at` + `shift_breaks` geçmişi; kiracı ayarı `settings.shifts.breakMaxMin` (10-120 dk, varsayılan 45), süre dolunca iş otomatik bitirir; molada kurye `prefilter` içinde `ON_BREAK` ile aday olmaz (öneri, toplu plan, OTOMATİK, kabul süresi aşımı yeniden dağıtımı), havuz `claim` 422, elle atama ve kabul / ret serbest; uçlar `POST /courier/shifts/break/start|end`; WS `courier.shift` `BREAK_STARTED` / `BREAK_ENDED`; sözleşme 0.6.5 |
| [0036](0036-gecikme-uyarisi.md) | Gecikme ve takılma uyarıları: beş tür (geç alım, geç teslim, konum yok, takıldı; 2026-11-12 ekiyle yüksek ret oranı), kiracı eşikleri `settings.alerts` (OWNER; varsayılan 15 / 10 / 45 / 20 dk), dakikalık idempotent değerlendirme işi (konu + tür başına tek açık satır, koşul kalkınca kendiliğinden kapanır), `GET /alerts`, `/alerts/summary`, `POST /alerts/{id}/ack`, WS `alert.opened/resolved/acked` (yalnızca panel), panelde zil + çekmece + `/uyarilar` + Ayarlar › Uyarılar, işletme sayfasında "Gecikiyor" rozeti; FCM / SMS yok | Kabul edildi |

İterasyon 1 teknik spesifikasyonu: docs/iterasyon-1/SPEC.md. İterasyon 2 ana hedefi (rota uyumlu atama) tasarımı: docs/iterasyon-2/ROTA-UYUMLU-ATAMA.md. Kuryeler arası devir tasarımı: docs/iterasyon-2/KURYE-DEVIR.md. İterasyon 2 teknik spesifikasyonu: docs/iterasyon-2/SPEC.md. İterasyon 3 (harita platformu) planı: docs/iterasyon-3/PLAN.md; İterasyon 3a teknik spesifikasyonu: docs/iterasyon-3/SPEC-3a.md; İterasyon 3b teknik spesifikasyonu: docs/iterasyon-3/SPEC-3b.md. (0015 numarası başka bir çalışma kolunda ayrılmıştır.)

## Durumlar

- **Önerildi:** Taslak, müşteri cevabı / doğrulama bekliyor.
- **Kabul edildi:** Onaylandı, uygulanıyor.
- **Reddedildi:** Değerlendirildi, seçilmedi.
- **Yerine geçti:** Başka bir ADR ile değiştirildi.

## Şablon

```markdown
# ADR-NNNN: Başlık

- **Durum:** Önerildi | Kabul edildi | Reddedildi | Yerine geçti (ADR-XXXX)
- **Tarih:** YYYY-MM-DD

## Bağlam
## Karar
## Sonuçlar
## Alternatifler
```
