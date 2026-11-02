# ADR-0003: Dağıtım — Android APK + OTA, iOS TestFlight / özel uygulama

- **Durum:** Kabul edildi (güncellendi: kurye uygulaması React Native + Expo, iOS eklendi; bkz. [ADR-0002](0002-kurye-uygulamasi-react-native-expo.md))
- **Tarih:** 2026-10-30

## Bağlam

Kurye uygulaması Play Store dışında dağıtılacak. Bu, Play'in arka plan konum beyanı, öne çıkan açıklama videosu ve veri güvenliği formu gibi gereksinimlerini kapsam dışı bırakır; ancak güncelleme ve kurulum yönetimi uygulamaya düşer. KVKK rıza ekranı yine gereklidir. Android geliştirici doğrulaması 2026-09-30'da yalnızca Brezilya, Endonezya, Singapur ve Tayland'da başladı; küresel yayılım 2027 için planlanıyor, Türkiye ve kurumsal/MDM durumu belirsiz.

## Güncelleme (2026-10-30): iki platform

Kurye uygulaması artık React Native + Expo ile tek kod tabanından Android ve iOS için üretilir ([ADR-0002](0002-kurye-uygulamasi-react-native-expo.md)). Dağıtım buna göre şöyle değişir.

**Android**
- APK doğrudan dağıtım (aşağıdaki madde 1) ve isteğe bağlı Headwind MDM (madde 3) geçerlidir.
- **JS ve varlık güncellemeleri `expo-updates` ile OTA** yapılır: native kod değişmedikçe yeni APK kurulumu gerekmez. Demo için EAS Update (Expo'nun barındırdığı servis, ücretsiz katman) kullanılabilir. Sonraki seçenek, Expo Updates protokolüne uyan ve kendi VPS'imizde çalışan bir güncelleme sunucusudur (veri Türkiye'de kalır).
- Native değişiklik gerektiren sürümler için madde 2'deki uygulama içi APK güncellemesi (PackageInstaller) İterasyon 2+ işidir. O zamana kadar yeni APK elle veya MDM ile kurulur.

**iOS**
- Türkiye'de iOS'ta yan yükleme (sideload) yoktur; dağıtım Apple üzerinden yapılır.
- **Demo:** TestFlight (dahili veya harici test kullanıcıları).
- **Üretim:** (a) Apple Business Manager üzerinden **Custom App**: kurye firmasının D-U-N-S numarası ve ABM hesabı gerekir. (b) Giriş zorunlu, listelenmemiş (unlisted) App Store uygulaması. Seçim iOS yayına alınırken müşteriyle yapılır.
- Apple Developer Program (yıllık 99 USD) **yalnızca iOS yayına alınırken** gerekir. İterasyon 1'de iOS derlemesi yapılmaz; iOS derlemeleri Windows'tan EAS Build (bulut) ile alınır.
- OTA (`expo-updates`) iOS'ta da yalnızca JS ve varlık güncellemeleri için kullanılır. Uygulamanın temel amacını değiştiren güncellemeler, App Store kurallarına uygun kalmak için OTA ile yapılmaz.

## Karar (ilk sürüm; Android için geçerliliğini korur)

1. **APK doğrudan dağıtım:** Paket adı ve imzalama anahtarı geliştirici doğrulaması için kaydedilir; anahtar depoda tutulmaz.
2. **Uygulama içi güncelleme:** `REQUEST_INSTALL_PACKAGES` + `PackageInstaller`. Sunucu sürüm bilgisi ve APK indirme adresi verir; imza ve özet (hash) doğrulanır.
   - Device-owner yoksa kullanıcı kurulumu onaylar.
   - Android 12+ üzerinde `USER_ACTION_NOT_REQUIRED` ile sessiz güncelleme denenebilir (uygulama kendi installer'ı ise); AOSP kaynaklı bilgi olduğundan hedef cihazlarda test edilmelidir.
3. **Headwind MDM (Community Edition, ücretsiz, Apache lisanslı):** Telefonlar firma mülkiyetindeyse isteğe bağlı olarak kullanılır (sessiz kurulum, pil optimizasyonu beyaz listesi). Kuryelerin kendi telefonlarında zorunlu tutulmaz.

## Sonuçlar

- (+) Play inceleme süreci ve politika kısıtları yok; hızlı sürüm döngüsü.
- (+) MDM isteğe bağlı olduğundan kişisel telefonlarda da çalışır.
- (-) Sessiz kurulum için device-owner (QR ile ilk kurulum / fabrika ayarı) gerekir; kişisel telefonlarda zordur.
- (-) Kurye bilinmeyen kaynak kurulum izni vermelidir; onboarding ekranı gerekir.
- (-) Güncelleme sunucusu ve imzalama anahtarı yönetimi güvenlik sorumluluğu getirir.
- **Açık risk:** Geliştirici doğrulamasının Türkiye'ye gelişi ve kurumsal muafiyet bilgisi doğrulanamadı; izlenecek.

## Alternatifler

| Alternatif | Değerlendirme |
|---|---|
| Google Play (iç test / özel uygulama) | Güvenilir güncelleme; ancak Play politika süreçleri (arka plan konum beyanı vb.) ve hesap yükü getirir |
| Android Enterprise özel uygulama | Yönetilen Google Play üzerinden, teknik olarak Play tabanlı; cihaz yönetimi gerektirir |
| Ücretli MDM (Scalefusion, Hexnode vb., ~2-3 USD/cihaz/ay) | Kadro büyürse değerlendirilir; başlangıçta gereksiz maliyet |
| Headwind Cloud / Premium | Ücretli (19 USD/cihaz/yıl; 2.990 USD tek seferlik, 300 cihaza kadar); topluluk sürümü yeterliyse gerekmez |
