# Karar Kayıtları (ADR)

Mimari ve teknoloji kararları, bağlamı ve alternatifleriyle birlikte burada tutulur. Karar değişirse ilgili ADR güncellenir (eski karar silinmez, "Yerine geçti" olarak işaretlenir).

## Dizin

| No | Başlık | Durum |
|---|---|---|
| [0001](0001-hibrit-harita-mimarisi.md) | Hibrit harita mimarisi (Google Places + OSRM/VROOM) | Önerildi |
| [0002](0002-kurye-uygulamasi-native-kotlin.md) | Kurye uygulaması: native Kotlin + Jetpack Compose | Önerildi |
| [0003](0003-dagitim-apk-self-update.md) | Dağıtım: APK + uygulama içi güncelleme | Önerildi |
| [0004](0004-cok-kiracili-mimari.md) | Çok kiracılı (multi-tenant) mimari | Önerildi |
| [0005](0005-konum-takibi-ve-kvkk.md) | Konum takibi ve KVKK | Önerildi |
| [0006](0006-hizmet-bolgesi-eskisehir.md) | Hizmet bölgesi: Eskişehir pilotu, çok şehre hazır tasarım | Kabul edildi |

Backend (NestJS, PostgreSQL + PostGIS, Redis, WebSocket, FCM) ve web paneli (React + Vite) için önerilen yığın [MIMARI.md](../MIMARI.md) içinde taslak olarak durur; ayrı ADR'ye pilot müşteri keşfinden sonra dönüştürülecek.

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
