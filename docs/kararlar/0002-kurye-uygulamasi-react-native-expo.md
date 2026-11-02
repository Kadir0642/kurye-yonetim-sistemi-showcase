# ADR-0002: Kurye uygulaması — React Native + Expo, tek kod Android + iOS

- **Durum:** Kabul edildi
- **Tarih:** 2026-10-30
- **Geçmiş:** Bu ADR'nin ilk sürümü (aynı gün, "Önerildi") native Kotlin + Jetpack Compose öneriyordu. Ürün sahibinin "kurye uygulaması Android **ve** iOS'ta tek kod tabanından çalışmalı" kararıyla yeniden yazıldı. Dosya adı da buna göre değişti (`0002-kurye-uygulamasi-native-kotlin.md` → bu dosya).

## Bağlam

- Kurye uygulaması artık **iki platformu** hedefler: Android (ağırlıklı, düşük ve orta segment telefonlar) ve iOS (kendi iPhone'unu kullanan kuryeler).
- Ürün sahibi kod yazmıyor; tüm geliştirme yapay zekâ ajanlarıyla yapılıyor. Bu yüzden tek dil ve tek tip sistemi bakım maliyetini doğrudan düşürür.
- Backend ve panel TypeScript ile yazılıyor ([ADR-0007](0007-backend-ve-web-yigini.md)); API sözleşmesi `packages/contracts` içinde OpenAPI'den üretilen TypeScript tipleridir.
- En riskli bileşen arka plan konum takibidir: Android'de foreground service ve OEM pil sonlandırıcıları, iOS'ta "Always" izni ve arka plan konum modu ([ADR-0005](0005-konum-takibi-ve-kvkk.md)).
- Geliştirme makinesi Windows'tur; yerelde iOS derlemesi yapılamaz.
- Harita gösterimi MapLibre + OpenFreeMap ile yapılır ([ADR-0008](0008-acik-kaynak-harita.md)).

## Karar

Kurye uygulaması **React Native + Expo (TypeScript)** ile, tek kod tabanından Android ve iOS için geliştirilir. Uygulama `apps/courier-mobile` klasöründedir ve npm workspaces monoreposunun parçasıdır.

- **Temel:** güncel kararlı Expo SDK (yazım tarihinde SDK 58, React Native 0.88). New Architecture ve Hermes açıktır. Native modüller gerektiği için **development build** kullanılır, Expo Go kullanılmaz. `android/` ve `ios/` klasörleri `npx expo prebuild` ile üretilir ve commit edilmez (Continuous Native Generation).
- **Kütüphaneler:**
  - `expo-router`: ekranlar ve gezinme.
  - TanStack Query: sunucu durumu.
  - `openapi-fetch` ve `@kurye/contracts` tipleri: sözleşmeye bağlı API istemcisi.
  - `expo-secure-store`: token saklama.
  - React Native'in yerleşik `WebSocket`'i: gerçek zamanlı kanal.
- **Harita:** `@maplibre/maplibre-react-native`, OpenFreeMap stil adresiyle.
- **Arka plan konum:** `expo-location` + `expo-task-manager`.
  - Android: kalıcı bildirimli foreground service ve `expo-intent-launcher` ile pil optimizasyonu onboarding ekranı.
  - iOS: "Always" izni, `UIBackgroundModes: location`, `showsBackgroundLocationIndicator: true`, vardiya açıkken `pausesUpdatesAutomatically: false`.
- **Navigasyona devir:**
  - Android: `google.navigation:q=lat,lng&mode=l`, açılmazsa `geo:` URI'si.
  - iOS: Google Maps kuruluysa `comgooglemaps://` (`LSApplicationQueriesSchemes` beyanı ile), değilse Apple Haritalar `maps://?daddr=lat,lng`.
- **Güncelleme:** JS ve varlık güncellemeleri `expo-updates` ile OTA yapılır. Demoda EAS Update kullanılabilir; kendi barındırdığımız güncelleme sunucusu sonraki seçenektir ([ADR-0003](0003-dagitim-apk-self-update.md)).
- **İterasyon 1 derleme ve doğrulama yalnızca Android'de yapılır:** `npx expo prebuild` + `gradlew assembleDebug` / `assembleRelease`, Android emülatöründe çalıştırma. iOS yapılandırması (izin metinleri, arka plan modları, URL şemaları) eksiksiz ve doğru tutulur. iOS derlemesi (EAS Build bulut, Apple Developer hesabı) İterasyon 2+ işidir.

## Sonuçlar

- (+) Tek kod tabanı, iki platform; özellikler bir kez yazılır.
- (+) Uçtan uca TypeScript: API sözleşmesinin tipleri (`@kurye/contracts`) API, panel ve mobil uygulama arasında paylaşılır; sözleşme değişikliği derleme zamanında yakalanır.
- (+) iOS derlemeleri Windows'tan EAS Build (bulut) ile alınabilir; Mac gerekmez.
- (+) `expo-updates` ile OTA güncelleme, yan yüklenen APK'ların güncelleme sorununun büyük kısmını çözer (native değişiklik yoksa yeni APK kurulumu gerekmez).
- (+) MapLibre ve arka plan konum modülleri olgun; ticari lisans gerektiren eklentiye bağımlılık yok.
- (-) Native Kotlin'e göre biraz daha büyük APK ve JS çalışma zamanı yükü. Hermes ve New Architecture ile düşük uç cihazlarda kabul edilebilir bekleniyor; pilot cihazlarda ölçülecek.
- (-) Expo SDK yükseltmeleri (yılda yaklaşık 3 kez) düzenli bakım işi getirir.
- (-) Monorepo içinde Metro ayarı dikkat ister: tek React kopyası (kök `package.json` `overrides`) ve workspace paketlerinin çözülmesi gerekir. Expo SDK 52+ monorepoları otomatik algılar; ayrıntı için `apps/courier-mobile` brief'ine bakın.
- **Risk (çerçeveden bağımsız):** Android OEM pil sonlandırıcıları (Xiaomi, Huawei, Samsung, OnePlus) her çerçevede aynıdır. Onboarding ekranı ve sunucu tarafı "konum kesildi" uyarısı korunur ([ADR-0005](0005-konum-takibi-ve-kvkk.md)).
- **Risk:** iOS'ta kullanıcı uygulamayı uygulama değiştiriciden kapatırsa arka plan konumu durur; heartbeat uyarısı bunu yönetici tarafında görünür kılar.
- **Risk:** Windows'ta React Native New Architecture derlemeleri uzun dosya yollarında (260 karakter sınırı, CMake) hata verebilir; çalışma dizini kısa bir yolda tutulmalı veya Windows uzun yol desteği açık olmalıdır.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Native Kotlin + Jetpack Compose (ilk öneri) | Yalnızca Android; iOS için ikinci bir uygulama gerekir |
| Kotlin Multiplatform + Compose Multiplatform | Harita kütüphanesi desteği zayıf; iOS derlemesi yine Mac veya CI gerektirir; TS yığınıyla tip paylaşımı yok |
| Flutter | Dart, TypeScript yığınımızla kod veya tip paylaşmaz. İyi arka plan konum eklentisi (`flutter_background_geolocation`) Android release sürümleri için ücretli lisans ister |
| PWA / web | Güvenilir arka plan konum için uygun değil (özellikle iOS) |
