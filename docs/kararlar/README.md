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
| [0008](0008-acik-kaynak-harita.md) | Açık kaynak harita (MapLibre + OpenFreeMap), sağlayıcı arkasında adres arama | Kabul edildi |
| [0009](0009-rota-uyumlu-atama.md) | Rota uyumlu atama: kurye başına canlı rota, sapma süresine dayalı uygunluk, üç atama modu | Kabul edildi |
| [0010](0010-kuryeler-arasi-devir.md) | Kuryeler arası devir: yakındaki meslektaşa teklif, alıştan sonra fiziksel buluşma ve kanıt (kod + konum) | Kabul edildi |
| [0011](0011-osrm-motosiklet-profili-ve-rota-altyapisi.md) | OSRM motosiklet profili ve rota altyapısı (OSRM MLD + VROOM, Docker, Eskişehir çıkarımı) | Önerildi |
| [0012](0012-rls-iterasyon-2b-ye-ertelendi.md) | PostgreSQL RLS İterasyon 2b'ye ertelendi; İterasyon 2 tabloları RLS'ye hazır | Önerildi |
| [0013](0013-rota-motoru-siniri-ve-yaklasik-mod.md) | Rota motoru sınırı, yaklaşık (düşüş) modu ve süreç içi kesin sıralayıcı | Önerildi |
| [0014](0014-toplu-atama-regret-ekleme.md) | Toplu atama: mevcut rotalara regret-2 ekleme, gecikme sınırı, aynı işletme gruplaması | Önerildi |

İterasyon 1 teknik spesifikasyonu: docs/iterasyon-1/SPEC.md. İterasyon 2 ana hedefi (rota uyumlu atama) tasarımı: docs/iterasyon-2/ROTA-UYUMLU-ATAMA.md. Kuryeler arası devir tasarımı: docs/iterasyon-2/KURYE-DEVIR.md. İterasyon 2 teknik spesifikasyonu: docs/iterasyon-2/SPEC.md.

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
