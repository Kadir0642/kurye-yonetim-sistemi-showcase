# ADR-0002: Kurye uygulaması native Kotlin + Jetpack Compose

- **Durum:** Önerildi (geliştiricinin beceri durumuna bağlı)
- **Tarih:** 2026-10-30

## Bağlam

Kurye uygulaması yalnızca Android hedefler; düşük uç telefonlarda hafif ve akıcı çalışmalıdır. En riskli bileşen arka plan konum takibidir (foreground service, OEM pil sonlandırıcıları, Huawei GMS'siz cihazlar). Harita için Maps SDK for Android birinci sınıf destek sunar.

## Karar

Kurye uygulaması **native Kotlin + Jetpack Compose** ile geliştirilir.

- Arka plan konum mantığı en az katmanla, yerel API'lerle yazılır (`FusedLocationProviderClient`; GMS'siz cihazlar için `LocationManager` yedeği).
- Tek platform (Android) olduğundan çapraz platform kazancı küçüktür.
- Yönetim paneli ayrı web uygulamasıdır; Android ile kod paylaşımı hedeflenmez.

## Sonuçlar

- (+) Konum güvenilirliği ve düşük uç cihaz performansı en iyi bu yolla sağlanır.
- (+) Maps SDK, foreground service ve PackageInstaller ile doğrudan, aracısız çalışılır.
- (+) Ticari arka plan konum eklentisine (lisans) bağımlılık yoktur.
- (-) iOS gerekirse sıfırdan yeni uygulama gerekir; şu an iOS hedefi yok.
- (-) Geliştirici Kotlin / Compose'a hâkim değilse öğrenme eğrisi; bu durumda karar yeniden değerlendirilir.

## Alternatifler

| Alternatif | Değerlendirme |
|---|---|
| Flutter | Ekip biliyorsa kabul edilebilir; arka plan mantığı yine özel Kotlin platform kanalı gerektirir. `flutter_background_geolocation` Android release sürümleri için lisans ister (üçüncü taraf fiyat ~399-500 USD, doğrulanmadı); ücretsiz alternatifler genç |
| React Native (yeni mimari) | Benzer native arka plan modülü ve aynı lisans sorunu; zayıf telefonlarda JS köprüsü yükü |
| PWA / web | Güvenilir arka plan konum için uygun değil |
