# ADR-0005: Konum takibi ve KVKK

- **Durum:** Kabul edildi (saklama süresi ve hukuki dayanak müşteri/hukuk görüşü bekliyor; İterasyon 1 varsayılanı 30 gün)
- **Tarih:** 2026-10-30

## Bağlam

Kurye konumu, KVKK kapsamında çalışan / yüklenici kişisel verisidir. Hukuki dayanak (meşru menfaat dengeleme testi veya açık rıza), aydınlatma yükümlülüğü, ölçülülük ve amaçla sınırlılık gerekir. Pazar araştırmasına göre KVKK Kurulu 2026/347 sayılı karara göre açık rıza metni ile aydınlatma metni ayrı düzenlenmelidir. Teknik tarafta Android arka plan konumu için foreground service gerekir ve OEM pil sonlandırıcıları (Huawei, Xiaomi, OnePlus, Samsung) güvenilirliği tehdit eder. Kuryelere özgü bir Kurul kararı bulunamadı.

## Karar

**Veri minimizasyonu ve şeffaflık:**

- Konum **yalnızca açık vardiya sırasında** alınır; vardiya kapalıyken takip yoktur.
- Kurye vardiyayı uygulama ekrandayken başlatır. Uygulama React Native + Expo ile yazılır ([ADR-0002](0002-kurye-uygulamasi-react-native-expo.md)); konum `expo-location` + `expo-task-manager` ile alınır.
- **Android:** konum, `foregroundServiceType="location"` ile **foreground service** olarak çalışır (kalıcı bildirim görünür). Servisin yeniden başlayabilmesi için `ACCESS_BACKGROUND_LOCATION` ayrıca istenir (Android 11+ iki adımlı izin).
- **iOS:** "Her zaman" (Always) konum izni istenir (önce "Uygulamayı kullanırken", sonra yükseltme). `UIBackgroundModes: location` tanımlanır. Vardiya açıkken `showsBackgroundLocationIndicator: true` (durum çubuğunda mavi gösterge) ve `pausesUpdatesAutomatically: false` kullanılır. Kullanıcı uygulamayı uygulama değiştiriciden kapatırsa iOS arka plan konumunu durdurur; bu durumda sunucudaki "konum kesildi" uyarısı devreye girer. iOS hareketsizken güncellemeleri seyreltebilir. iOS ince ayarı (significant-change yedeği vb.) iOS yayına alınırken (İterasyon 2+) cihazda doğrulanır.
- **Örnekleme:** hareket halinde 5-10 sn (dengeli güç önceliği, 10-20 m minimum mesafe), boşta 30-60 sn; gönderimler **toplu** yapılır (pil ve veri tasarrufu). Hareketsizken de en az 60 sn'de bir nokta ya da boş "kalp atışı" gönderilir; böylece "konum kesildi" uyarısı yanlış alarm vermez.
- **Ayrı belgeler:** aydınlatma metni ve açık rıza metni ayrı hazırlanır ve ilk girişte ayrı ayrı sunulur.
- **Sınırlı saklama:** konum geçmişi için kısa ve tanımlı saklama süresi; süre sonunda otomatik silme. İterasyon 1 varsayılanı `LOCATION_RETENTION_DAYS=30` (günlük silme işi); kesin süre müşteri/hukuk ile belirlenecek. Son konum Redis'te en fazla 24 saat tutulur.
- Firma ile yazılım sağlayıcı arasında veri işleme sözleşmesi; gerekirse VERBİS kaydı firma sorumluluğundadır.

**Güvenilirlik:**

- **Heartbeat:** sunucu son konum zamanını izler; süre aşılırsa yöneticiye "konum kesildi" uyarısı gider.
- **OEM onboarding (Android):** `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` isteği (`expo-intent-launcher` ile sistem ekranı açılır) ve Xiaomi / Samsung / Huawei için ayrı yönlendirme ekranları (Autostart, "kısıtlama yok", "uyumayan uygulamalar", son uygulamalarda kilitleme). Bu sorun çerçeveden bağımsızdır; native Kotlin de aynı önlemleri gerektirirdi.
- **Huawei / GMS yok:** FCM token alınamaz. `expo-location`, Google Play Services yoksa Android `LocationManager`'a düşer (cihazda doğrulanacak). İterasyon 1'de push yoktur; uygulama vardiya açıkken WebSocket bağlantısı tutar. İterasyon 2'de FCM (gerekirse Huawei Push Kit) değerlendirilir.
- `POST_NOTIFICATIONS` izni (Android 13+) istenir.

## Sonuçlar

- (+) Ölçülülük ve amaçla sınırlılık ilkelerine uyum; müşteri ve kuryeler için güven.
- (+) Heartbeat ve onboarding ile konum kesilmeleri hızlı fark edilir.
- (-) Vardiya dışı izleme yapılamaz; bu bilinçli bir sınırdır.
- (-) Rıza ve aydınlatma süreçleri onboarding'i uzatır.
- (-) Rıza geri alınırsa kurye sistemden çalışamaz hâle gelebilir; hukuki dayanak netleştirilmeli.
- **Açık sorular:** saklama süresi, dayanak (rıza mı meşru menfaat mi), VERBİS eşikleri, cihazlarda GMS durumu.

## Alternatifler

| Alternatif | Değerlendirme |
|---|---|
| Sürekli (vardiya dışı) konum takibi | Ölçülülük ilkesine aykırı, KVKK riski yüksek |
| Yalnızca paket alındığında konum | Canlı harita ve konum kesilme uyarısı mümkün olmaz |
| Ticari arka plan konum eklentisi | Lisans maliyeti; ADR-0002 ile native çözüm seçildi |
| Her konumu tek tek göndermek | Pil ve veri tüketimi yüksek; toplu gönderim seçildi |
