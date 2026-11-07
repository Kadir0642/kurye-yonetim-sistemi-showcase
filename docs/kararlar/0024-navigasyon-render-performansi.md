# ADR-0024: Navigasyon ekranında render performansı (yerel kamera animasyonu, konum başına harita ağacı çizimi yok, kalite kademesi)

- **Durum:** Önerildi (ürün sahibi onayına; gerçek cihaz ölçümü bekleniyor)
- **Tarih:** 2026-11-06
- **Ayrıntılandırdığı karar:** [ADR-0015](0015-uygulama-ici-navigasyon.md) madde 3 ("kamera ve konum işaretçisi animasyonları sınırlıdır") ve N2 brief'indeki kare hızı bütçesi
- **İlgili:** [ADR-0002](0002-kurye-uygulamasi-react-native-expo.md), [ADR-0019](0019-kurye-cevrimdisi-harita-ve-navigasyon.md), ölçüm belgesi NAVIGASYON-AKICILIK.md

## Bağlam

- Navigasyon ekranı MapLibre (`@maplibre/maplibre-react-native` 11.5, MapLibre Native Android, `SurfaceView`) üzerinde course-up takip kamerası, rota çizgisi, koridor etiketleri (`within` filtresi), sokak adları, hedef işareti ve React Native bant / alt çubuktan oluşur.
- Sürüm APK'sıyla emülatörde (2 GB RAM, `-gpu host`) 1 Hz GPS oynatmasında ölçüldü (SurfaceFlinger timestats, `top -H`, uygulama içi JS sondası). Ayrıntılı sayılar belgede; özet:
  - Harita yüzeyi 30 fps tavanında (`preferredFramesPerSecond={30}`) bile ortalama 8-25 fps, kare aralığı p50 33-110 ms; **MapLibre GL iş parçacığı %65-90 CPU** ile darboğaz (emülatörün GL çeviri yolu pahalıdır; gerçek cihazda farklı olabilir).
  - Her GPS okumasında tüm `NavMap` React ağacı yeniden çiziliyordu (`NavMap.render` 1/sn; 4 kaynak + 12 katman bileşeni) ve tüm kalan rota GeoJSON'u yeniden gönderiliyordu.
  - Hedef işareti bir RN görünümü (`Marker`) olduğu için Android her harita karesinde onu yeniden konumluyor, **uygulama penceresi (HWUI) harita kare hızında yeniden çiziliyordu** (ablasyon: işaret kaldırılınca uygulama penceresi 24 fps -> 1,4 fps).
  - Kamera 1 sn doğrusal `easeTo` ile ve en az 900 ms arayla güncelleniyordu: 980 ms gibi titreşimli 1 Hz okumalar atlanıyor, animasyon biter bitmez harita bir sonraki okumaya kadar duruyordu (200-999 ms'lik "duraklama" kareleri).
  - **Ürün sahibi hatası:** yeniden rotadan sonra harita eski yönde çapraz kalıyordu (yön süzgeci adım başına 40° ile sınırlıydı ve sıfırlanmıyordu; ilk kamera hedefi harita abone olmadan yayınlanıp kayboluyordu).

## Karar

1. **Kamera yerelde, öngörülü ve kesintisiz akar.** Oturum her okumada (en az 400 ms arayla) bir kamera hedefi üretir; harita bunu MapLibre `easeTo` (doğrusal) ile verir. Süre = beklenen konum aralığı (üstel ortalama, 0,5-2 sn) x 1,5; hedef, rotaya oturmuşken hızla 0,5 aralık kadar **rota üzerinde ileri** alınmış noktadır. Kararlı durumda kamera gerçek hızla akar, gecikme bir konum aralığıdır (eskisiyle aynı) ve sonraki okuma 0,5 aralığa kadar gecikse de harita durmaz.
2. **Course-up yönü rota geometrisinden.** Yön hedefi oturtulan noktanın önündeki ~30 m'lik rota parçasının yönüdür; GPS yönü yalnızca rotaya oturmamışken yedektir. Alçak geçiren süzgeç: güncelleme başına farkın %60'ı, en çok 60°. Yeniden rotada süzgeç sıfırlanır; son kamera hedefi oturumda tutulur ve harita onunla açılır. Puk ekranda sabit (alttan ~%33, yatay ortada), eğim 45° / dururken 30°, "Ortala" aynı kameraya döner (davranış değişmedi).
3. **Konum başına harita ağacı çizimi yok.** `NavMap` ve katmanları yalnızca takip / kullanıcı etkileşiminde (ve 30 sn'lik fps örneklemesinde) çizilir; yalnızca küçük `RouteLine` bileşeni rota başı GeoJSON'unu göndermek için okuma başına çizilir (MapLibre RN'de kaynağa veriyi React çizimi olmadan gönderen bir API yok). Konuma bağlı her parça (rota başı, puk halesi, hız çipi, koridor filtresi, sokak adları, hedef etiketi, "rotaya dön" çizgisi) kendi `memo` bileşeninde oturuma abonedir ve yalnızca kendi değeri değişince çizilir (ör. hale 2 px adımla, hız tam sayı km/sa ile). Katman `paint` / `layout` nesneleri sabittir.
4. **Rota çizgisi iki kaynak:** "baş" (kuryeden ~250 m ilerideki köşeye kadar, birkaç nokta, her okumada) ve "kuyruk" (oradan hedefe, ~250 m'de bir). Baş, bir önceki okumanın rota noktasından başlar; kamera bir aralık geriden geldiği için çizgi puk'un altından çıkar.
5. **Hedef işareti (RN görünümü) yalnızca gerektiğinde:** takip kamerasında hedef kuş uçuşu 1,5 km'den yakınken ya da takip kapalıyken (genel bakış, kaydırma). Görünen davranış aynı; uzaktayken uygulama penceresi her harita karesinde yeniden çizilmez.
6. **Kalite kademesi** (`src/navigation/quality.ts`, `expo-device`): toplam bellek < 2,5 GB veya cihaz yıl sınıfı <= 2015 ise **düşük**: 30 fps tavan, eğimsiz (düz) kamera, koridor filtresi ~250 m / 10 sn'de bir (önde yine en az ~1 km), kısa geçişler. Aksi halde **yüksek**: eğik kamera, koridor ~100 m / 5 sn ve **uyarlanır kare tavanı**: 60 fps ile başlar; oturum boyunca her 30 sn'de bir ısınma + iki 3 sn'lik pencere örneklenir (kare olayı yalnızca bu 9 sn'de JS'e gelir); harita akarken ölçülen iki ardışık pencere 40 fps'in altındaysa tavan oturum boyunca 30'a iner, geri çıkmaz. **Emülatör** (`Device.isDevice === false`) belleğine bakılmadan yüksek kademedir; gerçek 2 GB telefonlar düşük kalır. Test APK'sında `EXPO_PUBLIC_NAV_FPS_OVERLAY=1` ekranda harita fps'ini gösterir (gerçek cihaz protokolü). Gerekçe: emülatörde GL iş parçacığı doygunken 60 tavanı 30'dan belirgin biçimde kötüydü (aynı koşullarda ortalama 11 fps'e karşı 18-22 fps). `EXPO_PUBLIC_NAV_QUALITY=low|high` derleme anahtarı kademeyi zorlar.
7. **Ölçüm araçları depoda:** `scripts/nav-perf-measure.mjs` (timestats + `top -H` + gfxinfo + meminfo + JS sondası, tek pencere), `scripts/gps-playback.mjs --interval-ms 1000 --max-s`, `src/navigation/perfProbe.ts` (yalnızca `EXPO_PUBLIC_NAV_PERF=1`).

## Sonuçlar

- (+) Kamera konum gecikmelerinde durmaz: emülatörde (temiz durum) 110 sn'de >= 100 ms takılma 71 -> 5-10, 200-999 ms duraklama 16 -> 0-3, kare aralığı p99 118 -> 66-82 ms; yüksek kademede ortalama 23 -> 27-34 fps.
- (+) Yeniden rotadan sonra harita hemen yeni rota yönüne döner; rota puk'tan dümdüz yukarı çıkar (ürün sahibinin bildirdiği hata).
- (+) `NavMap` çizimi 1/sn -> ~0,1/sn (rota başı bileşeni hariç); uygulama penceresi hedef uzaktayken harita kare hızında yeniden çizilmez; JS iş parçacığı işi azaldı.
- (+) Yük altında (2 GB, takaslı emülatör) düşük kademe ve 30 fps tavanı tabandan iyi; sabit 60 fps tavanı GL doygunken kötüleştirdiği için yüksek kademede tavan uyarlanır.
- (-) **Emülatörde 60 fps'e ulaşılamadı:** darboğaz MapLibre GL iş parçacığıdır (emülatörün GL çeviri yolu, 2 GB RAM'de takas). Gerçek cihazda ölçülmedi; cihaz test protokolü eşikleri tanımlar.
- (-) Düşük kademede kamera düzdür (eğim yok): ürün sahibinin "orta eğim" tercihinden bilinçli sapma, yalnızca 2 GB sınıfı cihazlarda. Koridor etiketleri geride en çok ~280 m kalabilir.
- (-) Yeni yerel bağımlılık: `expo-device` (Expo SDK, ücretsiz, MIT).
- (-) Kamera hedefi en çok 0,5 aralık (40 km/sa'te ~5,5 m) ileri öngörülür; ani duruşta kamera bir an bu kadar ileri gidip geri gelebilir.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| MapLibre `trackUserLocation="course"` (yerel konum takibi) | Yön GPS seyrine bağlıdır (emülatör `geo fix` yön vermez, düşük hızda gürültülü); puk rotaya oturtulamaz ve ekranda sabit noktaya kilitlenemez (course-up kararı ADR-0015) |
| Reanimated paylaşılan değerleriyle kamera | MapLibre kamerası RN köprüsünde animasyonlu prop desteklemez; her karede komut göndermek yerel `easeTo`'dan pahalıdır |
| `line-gradient` + `line-progress` ile geçilen rotayı gizleme | Gradyan dokusu rota boyunca ~256 örnek: 3-5 km rotada 15-20 m adım, puk arkasında çizgi taşması; baş / kuyruk bölmesi tam hassas |
| Hedef işaretini simge katmanına çevirmek | `StopBadge` (kare / daire / elmas + numara) için sprite üretimi gerekir; görünüm değişir. Uzaklık eşiği aynı kazancı görünür değişiklik olmadan verir |
| Koridor etiketlerini kaldırmak | Ürün sahibi kararı (kapı numaraları yalnızca kurye çevresinde); seyrek yenileme yeterli |
| Ücretli SDK (Mapbox / Google Navigation) | ADR-0015'te reddedildi; ücretli servis yok |
