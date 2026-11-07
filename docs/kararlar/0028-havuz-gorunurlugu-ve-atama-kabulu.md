# ADR-0028: Havuz görünürlüğü (kiracı ayarı) ve atama kabulü (paket durumu değil, atama özelliği; veritabanı son tarihi)

- **Durum:** Kabul edildi
- **Tarih:** 2026-11-07
- **İlgili:** [ADR-0009](0009-rota-uyumlu-atama.md) (modlar, ret), [ADR-0010](0010-kuryeler-arasi-devir.md) (devir), [ADR-0020](0020-cevrimdisi-eylem-kuyrugu-ve-idempotency.md) (çevrimdışı kuyruk), [ADR-0023](0023-dagitim-algoritmasi-v2.md) (v2 penceresi); spesifikasyon: SPEC-4a

## Bağlam

Pilot ekipte kuryeler havuzu görmüyor; sistem paketleri rota uyumlu olarak otomatik atıyor, yönetici paneldeki havuzdan yanlış atamaları düzeltiyor, kuryeler devrediyor ve reddediyor (ürün sahibi, 2026-11-07). OTOMATİK mod, ret ve devir vardı; eksikler havuzu kuryelerden gizleme, atamanın kurye tarafından kabulü ve süre aşımı, atama gerekçesinin panelde görünmesi ve havuz bekleme verisiydi.

## Karar

**1. Havuz görünürlüğü kiracı ayarıdır:** `settings.dispatch.poolVisibleToCouriers` (varsayılan `true`). `false` iken:
- `GET /courier/pool` boş liste ve `poolVisible = false` döner (vardiya denetimi yok); `claim` 409 `POOL_CLOSED`.
- Hiçbir paket kuryelere açık havuza girmez: `POOL` ile oluşturma ve `publish` AUTO gibi işlenir (bekleyenler + dağıtım), yönetici `open-to-pool` 409 `POOL_CLOSED`, OTOMATİK yaklaşık modda paket havuza açılmaz bekleyenlerde kalır, ayar kapanırken açık havuzdaki paketler bekleyenlere alınıp yeniden dağıtılır. Böylece "kimsenin görmediği açık havuz paketi" durumu oluşmaz.
- `AKILLI_HAVUZ` (kuryenin havuzdan seçmesine dayanır) gizli havuzla birleştirilemez: 400 `VALIDATION_FAILED`. Kuryeye bilgi amaçlı öneri yoktur (ONERI önerileri yalnızca yöneticiye gider).
- Kurye uygulaması ayarı var olan `GET /tenant` yanıtından okur (yeni uç yok).

**2. Kabul, paket durumu değil atamanın özelliğidir:** `settings.dispatch.acceptTimeoutS` (0 veya 30-600 sn, varsayılan 0 = adım yok). `> 0` iken `packages.assign` yolundan geçen her atama ve oluştururken atama `accept_state = PENDING`, `accept_deadline_at = atama + süre` ile başlar; havuzdan alma ve devir doğrudan `ACCEPTED`. Paket `ASSIGNED` kalır, durum makinesi (MIMARI §4) değişmez; rota, navigasyon ve alış PENDING iken de çalışır (alış örtük kabuldür).

**3. Kabul ucu:** `POST /courier/packages/{id}/accept` (COURIER), doğası gereği idempotent; süresi dolmuş atama için ayrı hata `ASSIGNMENT_EXPIRED` (kurye uygulaması "süre doldu" ile "başkasına verildi"yi ayırt eder).

**4. Süre aşımı = sistem reddi:** veritabanındaki son tarih 5 sn'lik dağıtım işiyle (çok örnekli Redis kilidi) taranır; koşullu UPDATE ile ret yolu çalışır (`IN_POOL`, `poolPriority + 1`, `package_rejections.reason = TIMEOUT` ile o kuryenin `REJECTED_BEFORE` dışlanması, rota ve devir temizliği, olay `ASSIGNMENT_EXPIRED`), ardından ret sonrasıyla aynı yeniden dağıtım. Panel ve kuryeye `package.assignment_expired`. ONERI'de kuyruk nedeni `ACCEPT_TIMEOUT`.

**5. Atama gerekçesi** `packages.assignment_info` (jsonb) sütununda atama anında saklanır (olaydaki `data.assignment` + `reassigned` + atayan kullanıcı kimliği) ve panel `Package.assignment` olarak döner; isim / telefon yok, yalnızca kimlikler. Havuz bekleme verisi `dispatch.poolEnteredAt` / `poolWaitS` (`pooled_at`'ten).

## Sonuçlar

- (+) Pilot akışı tek ayar setiyle açılır (OTOMATIK + gizli havuz + 90 sn); varsayılanlar bugünkü davranışı korur, eski APK ve panel bozulmaz.
- (+) Yeni paket durumu yok: rota, navigasyon, devir, geofence ve metrik kodu dokunulmadan çalışır.
- (+) Süre aşımı yeniden başlatmaya dayanıklıdır (bellek içi zamanlayıcı yok); ret yolunu yeniden kullandığı için dışlama ve yeniden dağıtım ret ile aynı test edilmiş davranıştır.
- (+) Atama gerekçesi olay taramadan, liste sorgusunda tek sütundan okunur.
- (−) Süre aşımı kesinliği ~5 sn (iş aralığı); son tarihten hemen sonra gelen kabul tarama öncesiyse geçerli sayılır.
- (−) Push yokken arka plandaki uygulama kabul kartını geç görebilir (vardiyada WS bağlı kalır; FCM İterasyon 2b).
- (−) Süre aşımları ret sayısına ve ret uyarısına dahildir; pilotta ayrı eşik gerekirse ayar eklenir.
- (−) `assignment_info` olaydaki verinin kopyasıdır (tek yazım noktası `packages.service`, migration etkin paketleri doldurur).

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Yeni paket durumu `OFFERED` / `PENDING_ACCEPT` | Durum makinesi, rota, navigasyon, devir, metrik ve iki istemcide geniş değişiklik; kabul atamanın niteliğidir, paketin yaşam döngüsü değil |
| Bellek içi `setTimeout` ile süre aşımı | Yeniden başlatmada ve çok örnekte kaybolur; veritabanı son tarihi + mevcut iş düzeni zaten var |
| Gizli havuzda `pool` ucunu 403 / 409 yapmak | Eski APK hata ekranı gösterirdi; boş liste + `poolVisible` bayrağı geriye uyumludur |
| Gizli havuzda AKILLI_HAVUZ'u kuryeye "bilgi amaçlı öneri" olarak sürdürmek | Kurye seçemiyorsa öneri anlamsızdır; OTOMATİK zaten aynı uygunluk motoruyla atar |
| Atama gerekçesini her istekte son olaydan türetmek (LATERAL sorgu) | Liste sorgularında her satır için olay taraması; tek sütun daha basit ve hızlı |
