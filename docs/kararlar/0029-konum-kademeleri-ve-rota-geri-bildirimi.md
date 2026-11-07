# ADR-0029: Konum sıklığı kademeleri, panel konum yayınında birleştirme ve "Rota yanlış" geri bildirimi

- **Durum:** Kabul edildi
- **Tarih:** 2026-11-07
- **İlgili:** [ADR-0005](0005-konum-takibi-ve-kvkk.md) (konum ve KVKK), [ADR-0015](0015-uygulama-ici-navigasyon.md) (navigasyon), [ADR-0017](0017-saha-verisinden-ogrenen-harita.md) (saha verisi), [ADR-0020](0020-cevrimdisi-eylem-kuyrugu-ve-idempotency.md) (çevrimdışı kuyruk), [ADR-0024](0024-navigasyon-render-performansi.md) (render); spesifikasyon: SPEC-4b

## Bağlam

Ürün sahibinin hedefleri (MVP+ S2 ve S4): kuryenin haritadaki konumu **doğru ve akıcı** olsun; **25 eşzamanlı kuryede** pil ve ağ kullanımı düşük kalsın; yanlış rotalar raporlanabilsin ki harita verisi **ücretsiz** düzeltilebilsin.

Bugün konum hattı tek bir profille çalışır: GPS 5 sn (`timeInterval`), seyreltme >= 15 m veya >= 60 sn, gönderim 15 sn / >= 10 nokta, boşta kalp atışı 30 sn. Bu profil üç durumun hiçbirine uymaz: navigasyonda panel 15 sn'lik adımlarla zıplar; kurye restoranda beklerken GPS yine 5 sn'de uyanır ve her 30 sn'de istek gider; panel yayını batch başına bir `courier.position` olayıdır (sınırsız, çevrimdışı birikim boşaltılırken ve kısa batch'lerde panel dolar). Yanlış rotayı bildirmenin de yolu yoktur: kurye sessizce kendi bildiği yoldan gider, hata sonraki kuryeye de tekrar eder.

## Karar

**1. Kademeli konum profili (istemci, saf mantık + hysteresis).** Dört kademe; kademeyi istemci seçer, sunucu sözleşmesi (`POST /courier/locations`) değişmez:

| Kademe | Koşul | GPS aralığı (Android `timeInterval`) | Seyreltme | Gönderim | Kalp atışı |
|---|---|---|---|---|---|
| `NAVIGATING` | Navigasyon açık **ve** hareketli (hız > 2 m/sn; son hareketten 20 sn'e kadar korunur) | 2,5 sn, yüksek doğruluk | >= 8 m veya >= 10 sn | 5 sn'de bir (>= 10 nokta birikirse hemen) | - |
| `ON_SHIFT_MOVING` | Vardiya açık, hareketli, navigasyon yok (bugünkü profil) | 5 sn, dengeli | >= 15 m veya >= 60 sn | 15 sn / >= 10 nokta | 30 sn |
| `ON_SHIFT_STATIONARY` | Vardiya açık, > 60 sn boyunca başlangıç noktasından < 15 m | 5 sn (yazılım kademesi), dengeli | >= 15 m veya >= 60 sn | 60 sn (nokta ya da boş kalp atışı) | 60 sn |
| `OFF_SHIFT` | Vardiya kapalı | GPS görevi **yok** | - | - | - |

- **Geçişler:** Yukarı geçiş hızlıdır, aşağı geçiş yavaştır. Navigasyon açılınca hemen `NAVIGATING` (açık sinyal; hız beklenmez). Duran kurye yalnızca > 60 sn **sürekli** başlangıç noktasına < 15 m kalınca `STATIONARY` olur; çıkış için `max(30 m, doğruluk)` uzaklaşma ya da hız > 2 m/sn gerekir (kapı aralığı 15 m giriş / 30 m çıkış: GPS titreşimi gidip gelmeyi tetiklemez). `NAVIGATING` içinde kırmızı ışık gibi duruşlar 20 sn tolere edilir; sonra `ON_SHIFT_MOVING` (navigasyon sürüyor, GPS 5 sn), 60 sn durursa `STATIONARY`. Bir kademe değişince en az 10 sn korunur (GPS görevini yeniden yapılandırmak ucuz değildir).
- **Uygulama ve arka plan kısıtı:** `expo-location` arka plan görevi (`startLocationUpdatesAsync`, ön plan servisi, ADR-0005) yalnızca GPS aralığı / doğruluk değişirken (NAVIGATING <-> diğerleri) aynı görev adıyla yeniden başlatılır. Android arka planda ön plan servisini yeniden başlatmaya izin vermez (`ForegroundServiceStartNotAllowedException`; bağımsız inceleme ve emülatörde görüldü): ilk tasarımdaki "duranken GPS 30 sn" arka planda uygulanamadı, kurye hareketlenince 30 sn'de takılı kalıyordu. Bu yüzden **MOVING <-> STATIONARY yalnızca yazılım kademesidir** (OS isteği 5 sn kalır; seyreltme ve gönderim 60 sn); arka planda başarısız yeniden yapılandırma görevi eski ayarda bırakır, dakikada bir ve ön plana dönüşte yeniden denenir. Navigasyon ön plandadır, orada geçiş çalışır. Android'de `timeInterval` etkindir. iOS `timeInterval`'ı yok sayar; orada `distanceInterval` (NAV 5 m, MOVING 10 m, STATIONARY 25 m) ve `activityType` kullanılır (pilot Android; iOS TestFlight kapsamında ayrıca ölçülür). Navigasyon ekranı kendi 1 sn'lik `watchPositionAsync` akışını (ADR-0015) korur; görev ikinci bir GPS motoru açmaz (Android füzyonlu sağlayıcı istekleri birleştirir).
- **Kuyruk:** ADR-0020 anlamı korunur (kalıcı SQLite, 5000 nokta, en eskiyi at, 200'lük batch, üstel geri çekilme). Seyreltme eşikleri kademeye göre değişir; kademe, noktayı toplayan anda geçerli olan profildir. Hareketsizken noktalar zaten seyrelir (yalnızca 60 sn'de bir kayıt).
- **Hata modları:** Kademe seçici giriş yoksa (konum yok / eski) `ON_SHIFT_MOVING`'e düşer; görev yeniden yapılandırması hata verirse mevcut ayar sürer ve bir sonraki fix'te yeniden denenir. Uygulama ölüp işletim sistemi görevi başsız (React ağacı olmadan) başlatırsa navigasyon bilgisi yoktur: `ON_SHIFT_MOVING` / `STATIONARY` çalışır.

**2. Panel yayınında birleştirme (sunucu).** `courier.position` olayı, kiracı + kurye başına **en çok 3 sn'de bir** yayınlanır (`POSITION_COALESCE_MS`, varsayılan 3000; 0 kapatır). Kural "ilk olay hemen, pencere içindekiler tek olaya": pencere dışındaki ilk konum anında gider (gecikme yok); pencere içindeki sonrakiler yalnızca **en yeni** konum tutulur ve pencere bitince bir kez gönderilir. Her nokta yine veritabanına yazılır; Redis son konum, rota kilidi ve devir kancası bugünkü gibi her batch'te çalışır. Olay, `speedMps` ve `headingDeg` alanlarını taşır (bunlar `CourierPosition` içinde zaten vardır; yeni alan yoktur). Vardiya bitince bekleyen yayın iptal edilir. Birleştirme API örneği başınadır (bellek içi); kurye istekleri ters vekille aynı örneğe yapışkan gitmese bile en kötü durum örnek sayısı kadar olaydır, doğruluk etkilenmez.

**3. "Rota yanlış" geri bildirimi.** `POST /courier/route-feedback` (tür, konum, isteğe bağlı kısa not, rota / paket kimliği; kişisel veri yok), `route_feedback` tablosu (kiracı kapsamlı, migration 0007), panel için `GET /route-feedback` + `PATCH /route-feedback/{feedbackId}` (OPEN / FIXED_OSM / IGNORED) ve OWNER için `GET /route-feedback.geojson` (OpenStreetMap düzenleyicilerine verilebilecek, kimliksiz nokta koleksiyonu). Mobilde navigasyon talimat panelinin **menü** düğmesinden (kurye durakları değiştirirken sık dokunduğu yerde değil, uzun basma / kaza dokunması riskine karşı iki adımlı) tek dokunuşlu tür seçimi; eylem ADR-0020 çevrimdışı kuyruğuna girer (`Idempotency-Key`), bildirim fırlatma/render maliyeti ADR-0024'ü bozmaz (yalnızca panel açılınca bir alt sayfa).

## Sonuçlar

- (+) Navigasyonda panel konumu 15 sn yerine 5 sn'de bir, 2,5 sn aralıklı noktalarla akıcı; hareketsizken istek 121/sa -> 61/sa (%50 az, bayt %46 az); GPS uyanması azalmaz (arka plan kısıtı, yukarıda).
- (+) Panel WS yayını kurye başına <= 0,33 olay/sn ile sınırlı; çevrimdışı birikim boşalması ve sık batch'ler paneli doldurmaz.
- (+) Sözleşme ve sunucu alımı değişmez; eski APK'lar bozulmadan çalışır (yalnızca eski profil, birleştirme onlar için de geçerlidir).
- (+) Yanlış rota bildirimleri kimliksiz GeoJSON olarak OSM düzenlemesine verilebilir: harita verisi düzeltmesi ücretsiz.
- (−) Navigasyon kademesi daha çok istek üretir (241 -> 721 istek/sa, nokta 721 -> 1441/sa). Vardiya karışımına göre ortalama istek sayısı artar (bkz. SPEC-4b §2.5); bedel bilinçli: doğruluk ve akıcılık istendi. Navigasyon ön planda olduğundan ek GPS maliyeti ihmal edilebilir; veri hacmi kurye başına birkaç yüz KB/sa.
- (−) Duranken GPS 5 sn'de uyanmaya devam eder: pil kazancı yalnızca ağ ve işlemde (seyreltme, gönderim). Gerçek GPS gevşetmesi ancak Android arka plan kısıtını aşan bir yöntemle (ör. görevi hiç yeniden başlatmadan `deferredUpdates` veya yerel modül) mümkün; ayrı iş.
- (−) Çevrimdışı kuyruk 5000 nokta: navigasyonda ~3,5 sa, hareketlide ~7 sa tutar; dolunca önce eski yarı seyreltilir, sonra en eskiler atılır.
- (−) `locationStaleMinutes` en az 2 (API 400 verir); duran kuryenin kalp atışı 60-90 sn ölçüldü. Duran kuryenin "konum kesildi" eşiği için kiracı `locationStaleMinutes` en az 2 olmalıdır (varsayılan 3; kalp atışı 60 sn).
- (−) iOS'ta zaman aralığı ayarlanamaz: kademeler yalnızca mesafe + `activityType` ile yaklaşıktır.
- (−) Birleştirme API örneği başınadır; çok örnekli dağıtımda kurye başına en çok örnek sayısı kadar olay olur.
- (−) Geri bildirim `note` serbest metindir: uygulama kişisel veri yazılmaması için uyarır, sunucu içeriği yorumlamaz; GeoJSON dışa aktarımı notu da taşıdığından panel kullanıcısı paylaşmadan önce gözden geçirir.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Sunucunun kuryeye "şimdi şu sıklıkta gönder" demesi | Ek gidiş-dönüş ve durum; çevrimdışıyken çalışmaz. İstemci kendi hareketini en iyi bilir |
| Android Activity Recognition (hareket algılama) | Ek izin (ACTIVITY_RECOGNITION) ve Play Services bağımlılığı; hız + yer değiştirme aynı sonucu izinsiz verir |
| Her zaman 2-2,5 sn GPS | Beklerken pil ve ağ israfı; 25 kuryede panelin yarısı gereksiz olay |
| Yalnızca sondaki olay (debounce) ile birleştirme | Her olay >= 3 sn gecikir (5 sn batch'te sürekli); ilk-hemen + pencere-sonu aynı üst sınırı gecikmesiz verir |
| Redis ile dağıtık birleştirme | Pilot tek API örneği; her olay için ek Redis gidiş-dönüşü. Çok örneğe geçilirse `SET NX PX` kapısı eklenir |
| Geri bildirimi paket olayı yapmak | Rota hatası pakete değil yola aittir; paket olmadan da (boşta, yol tarifinde) bildirilebilmeli |
| Geri bildirimi doğrudan OSM Notes'a göndermek | Dış hesap / API gerektirir (ücretsiz ama ürün sahibi kuralı: dış hesap yok); önce kendi kuyruğumuz, düzenleyiciye GeoJSON ile elle verilir |
