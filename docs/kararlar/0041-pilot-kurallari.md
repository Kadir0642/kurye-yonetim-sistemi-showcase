# ADR-0041: Pilot kuralları (ret kapalı kipi, bırakma talebi, "gördü" kaydı, kurye kapasitesi, "kutum dolu", dış navigasyon sayacı, restorana iade, sürüm kapısı, menajer uyarıları)

- **Durum:** Kabul edildi (ürün sahibi onaylı "üçüncü paket", 2026-11-13; sürüm kapısı alanları ve "kilit yalnız aktif paket yokken" kararı 2026-11-14)
- **Tarih:** 2026-11-14
- **İlgili:** [ADR-0009](0009-rota-uyumlu-atama.md) (ret, kapasite), [ADR-0010](0010-kuryeler-arasi-devir.md) (devir), [ADR-0020](0020-cevrimdisi-eylem-kuyrugu-ve-idempotency.md) (çevrimdışı kuyruk), [ADR-0028](0028-havuz-gorunurlugu-ve-atama-kabulu.md) (kabul süresi), [ADR-0034](0034-kapida-tahsilat-mutabakati.md) (tahsilat), [ADR-0036](0036-gecikme-uyarisi.md) (uyarılar), [ADR-0038](0038-atama-isabeti-ve-harita-verisi-tarihi.md) (pilot kapasitesi 5); sözleşme **0.7.2**, migrasyon **0015**

## Bağlam

Pilot müşteri (bir kurye firması) kuryelerin atamayı reddetmesini istemiyor: iş verilir, kurye bırakmak isterse menajer karar verir. Bugün ret serbestti, kabul süresi dolan atama havuza dönüyordu, "teslim edilemedi" son durumdu (paketin restorana geri dönüşü izlenmiyordu), eski uygulama sürümleri sisteme bağlanabiliyordu ve menajer "kimse atanamadı / kurye yok" durumunu haritaya bakarak fark ediyordu. Kapasite yalnızca firma genelindeydi.

Referans ürün araştırması (13 Kasım 2026, yalnızca herkese açık yardım / satıcı sayfaları):

| Konu | Kanıt | Bize uyarlama |
|---|---|---|
| Reddedemezlik + bırakma | Wolt teklif 60 sn'de düşer, ret serbest; Deliveroo ve Wolt **kabul sonrası** "bırakma / unassign" kanalı sunar; Tookan / Bringg ret düğmesini yapılandırılabilir yapar | Firma ayarı "kurye reddedebilir"; kapalıyken tek yol menajer onaylı **bırakma talebi**; sessizlikte (kabul yok) paket otomatik başkasına gitmez, menajere uyarı |
| Kapasite / "dolu" | Onfleet "kurye başına görev sınırı" dispatcher ayarıdır; kuryenin kendi "dolu" düğmesi hiçbir resmi sayfada yok (kanıt zayıf) | Menajer ayarlı **kurye başına kapasite**; "kutum dolu" **yalnızca sinyal** (atamayı değiştirmez) |
| Dış navigasyon | DoorDash, Wolt, Onfleet dahili navigasyon + tercih edilen dış uygulamaya devir | Dış uygulamaya geçişi yalnızca **sayar** (konum yok) |
| Müşteri yok / iade | DoorDash "müşteri yok" için ~5 dk bekleme, sonra belgeli bitirme; Uber zamanlayıcı | "Teslim edilemedi" + neden + **kapıda bekleme süresi**, paket **restorana iade** yoluna girer, menajer kapatır |
| Kapıda ödeme | Detrack üç toplam (toplanacak / toplandı / admin aldı) | İade edilen paket tahsil edilmemiş sayılır; kurye kartında **tahsil edilecek** |
| Sürüm | Play uygulama içi güncelleme "anında" ve "esnek" kip | **En düşük sürüm** kapısı + "güncelleme gerekli"; indirme akışı ayrı karar |
| Menajer uyarısı | Bringg "neden atanamadı", Onfleet "güncel uyarılar" | "Uzun süre atanamadı" ve "uygun kurye yok" uyarıları + **elle işlem gerekenler** listesi |

## Karar

Hepsi kiracı ayarının arkasındadır; **varsayılanlar bugünkü davranıştır**, "Pilot ayarlarını uygula" (Ayarlar › Dağıtım) pilot değerlerine çeker.

**1. Ret kapalı kipi** (`settings.dispatch.couriersCanReject`, varsayılan `true`, pilot `false`). Kapalıyken `POST /courier/packages/{id}/reject` ve eski `release` 409 `REJECT_DISABLED`. Kabul süresi (`acceptTimeoutS`) dolan atama **havuza dönmez**: 5 sn'lik süre aşımı işi `packages.accept_overdue_at`'i bir kez yazar (olay `ASSIGNMENT_OVERDUE`, aktör SYSTEM), panel ve kuryeye `ACCEPT_CHANGED`, menajere hemen `ACCEPT_OVERDUE` uyarısı (dakikalık iş de aynı koşulu tutar; kabul, alış, yeniden atama ya da iptal kapatır). Kabul süresi dolduktan sonra da geçerlidir.

**2. Bırakma talebi** (`package_release_requests`, paket başına en çok bir bekleyen). Kurye yalnızca alıştan önce (ASSIGNED) gerekçeyle (`RejectReason`) talep eder: `POST /courier/packages/{id}/release-request` (idempotent). Menajer `POST /release-requests/{id}/approve` (ret yolu: ASSIGNED -> IN_POOL, `package_rejections` ile aynı kuryeye bir daha önerilmez, olay `RELEASED` aktör menajer + `data.releaseRequestId`, atama moduna göre yeniden dağıtım) ya da `deny` (olay `RELEASE_DENIED`, paket kuryede) der; kurye geri çekebilir. Paket bu arada alınır / yeniden atanır / iptal edilirse talep okuma ve karar anında tembelce `CANCELLED (PACKAGE_CHANGED)` olur (olay `RELEASE_REQUEST_CANCELLED`). Bırakma talebi ret açıkken de kullanılabilir. WS `release_request.updated` (panel + kurye).

**3. "Gördü" kaydı** (bildirim güvencesi): `POST /courier/packages/{id}/seen` yalnızca ilk çağrıda `packages.assignment_seen_at` yazar (olay `ASSIGNMENT_SEEN`); kabulden ayrıdır; yeniden atamada sıfırlanır. Panel atama kartında "Kurye gördü SS:DD" / "Kurye henüz görmedi".

**4. Kurye başına kapasite** (`couriers.max_active_packages`, NULL = firma ayarı, 1-20; `PATCH /couriers/{id}` `maxActivePackages`). Kapasiteyi okuyan tüm otomatik yollar (öneri ön süzmesi, rota motoru kurye anlık görüntüsü / v2 penceresi ve pencere uzatma sezgisi, sistem atamasının kilit altında yeniden denetimi, havuzdan alma, devir alıcısı, "uygun kurye yok" hesabı) `COALESCE(kurye, firma)` kullanır. **Kurye yükü** = aktif (ASSIGNED, PICKED_UP) + restorana iade yolundaki (FAILED, `RETURNING`) paketler: iade edilen paket kutuda yer kaplar. Rota motorunda iade paketinin durağı olmadığı için kapasiteden düşülür (PR #79 incelemesi). Aday seçme / puanlama mantığı değişmedi. Yönetici elle atamada sınır uygulanmaz (bugünkü gibi).

**5. "Kutum dolu" yalnızca sinyaldir** (ürün sahibi kararı): `POST /courier/box-full {full}` `couriers.box_full_at` yazar; `settings.dispatch.boxFullTtlMin` (varsayılan 30, 5-240) dakika sonra ya da vardiya bitince geçersizdir (iş gerekmez, okuma anında hesaplanır); menajer `DELETE /couriers/{id}/box-full`. **Atamayı, kapasiteyi, önerileri değiştirmez.** WS `courier.box_full`.

**6. Dış navigasyon sayacı:** `POST /courier/packages/{id}/external-nav {app: GOOGLE|YANDEX|OTHER, stopKind: PICKUP|DROPOFF|RETURN}` yalnızca bir satır yazar (`external_nav_opens`: kurye, paket, durak türü, uygulama, an; **konum yok**, 90 gün sonra saatlik işle silinir). Panel `GET /couriers/external-nav?date=` kurye başına günlük toplam / uygulama / farklı durak.

**7. Teslim edilemedi -> restorana iade** (`settings.delivery.returnToBusinessOnFail`, varsayılan `false`, pilot `true`). **Durum makinesi değişmedi** (ADR-0028'deki "durum değil özellik" ilkesi): paket FAILED olur, ayrıca `packages.return_state` = `RETURNING` (kurye işletmeye götürüyor) -> `RETURNED` (`POST /courier/packages/{id}/returned`, olay `RETURNED_TO_BUSINESS`) -> `CLOSED` (menajer `POST /packages/{id}/return/close`, olay `RETURN_CLOSED`; RETURNING'den de kapatılabilir). `FailRequest.waitedS` kapıda bekleme süresidir (cihaz ölçümü, `packages.fail_waited_s`; ayar kapalıyken de yazılır). Kapıda ödeme tahsil edilmez: tahsilat toplamları yalnızca DELIVERED paketlerden hesaplandığı için iade edilen paket hiçbir toplama girmez. Kurye iadeleri `GET /courier/packages?scope=returns` ile görür (müşteri bilgisi yok, işletme adresi / telefonu var). **"Müşteri yok" için kapıda en az bekleme** (ürün sahibi kararı, 2026-11-14): `settings.delivery.minDoorWaitS` (0-1800 sn, şema varsayılanı 0 = bugünkü davranış, pilot ön ayarı 300). `reason = CUSTOMER_UNREACHABLE` iken `waitedS` (yoksa 0) bu değerden küçükse 422 `MIN_DOOR_WAIT_NOT_MET` (`remainingS`, `requiredS`); diğer gerekçeler ve menajerin elle işaretlemesi serbest; ayar COURIER rolüne de açık (uygulama geri sayım gösterir). Referans: DoorDash "müşteri yok" için ~5 dk bekleme. Geçişler:

```
PICKED_UP --FAIL (ayar açık)--> FAILED + return_state RETURNING
RETURNING --kurye "işletmeye teslim ettim"--> RETURNED
RETURNING | RETURNED --menajer "iadeyi kapat"--> CLOSED
```

**8. Sürüm kapısı** (`settings.courierApp`: `minVersion`, `latestVersion`, `downloadUrl` (https), `apkSha256`; hepsi boş = kapalı). Kurye uygulaması her istekte `X-App-Version` gönderir. `minVersion` doluyken başlığı olmayan ya da daha eski COURIER isteği **yalnızca kuryenin aktif (ASSIGNED / PICKED_UP) paketi yoksa** 426 `APP_VERSION_UNSUPPORTED` alır (gövdede `appVersion`); aktif paketi varsa istek işlenir ve yanıta `X-App-Version-Warning: unsupported` eklenir (ürün sahibi kararı: yoldaki iş yarıda kalmasın). Kiracının sürüm ayarı süreç içinde 30 sn önbelleklenir (her kurye isteğinde veritabanına gidilmez; aynı süreçteki ayar değişikliği önbelleği hemen boşaltır, çok örnekli kurulumda en geç 30 sn). Muaf: `GET /courier/app-version` (uygulama açılışta çağırır; `supported`, `blocking`, `updateAvailable` döner ve sürümü kuryenin kaydına yazar), `/me`, `/me/password`, `/auth/*`. Panel rolleri etkilenmez. WebSocket kesilmez. **APK indirme / kurma akışı bu kararın dışındadır** (ürün sahibi seçenekleri değerlendiriyor); alanlar uygulamanın indirip SHA-256 ile doğrulaması için hazırdır.

**9. Menajer uyarıları** (ADR-0036 kalıbı, üç yeni tür): `ACCEPT_OVERDUE` (madde 1), `UNASSIGNED_TOO_LONG` (paket `alerts.unassignedMin` dk, varsayılan 10, kuryesiz; `unassignedEnabled` varsayılan kapalı) ve `NO_COURIER_AVAILABLE` (kuryesiz bekleyen paket `alerts.noCourierMin` dk, varsayılan 3, ve vardiyada uygun — molada değil, kapasitesi dolmamış — kurye yok; kiracı geneli tek uyarı, konu yok, `Alert.noCourier` sayıları; hiç vardiya yoksa CRITICAL). `alerts` tekillik dizini `COALESCE(package_id, courier_id, tenant_id)` oldu.

**10. Elle işlem gerekenler:** `GET /manual-actions` tek çağrıda uzun süre atanamayan paketleri (uyarı kapalı olsa da eşikle), kabulü gecikenleri, bekleyen bırakma taleplerini, açık iadeleri, "kutum dolu" diyenleri, sürümü eski kuryeleri ve "uygun kurye yok" durumunu döner; panelde `/elle-islem` sayfası. Kurye kartında (canlı harita) **tahsil edilecek** = kuryedeki aktif kapıda ödemeli paketlerin toplamı (`LiveCourier.toCollectKurus`).

Tüm sorgular `tenant_id` ile süzülür; başka kiracının talebi / paketi / kuryesi 404. Yeni tablolar ve uç çıktıları müşteri adı / telefonu / adresi içermez.

## Sonuçlar

- (+) Pilot kuralları tek ön ayarla açılır; mevcut kiracıların davranışı değişmez (e2e: varsayılanlar ve geriye uyumluluk).
- (+) Yeni paket durumu yok: rota, navigasyon, metrik, devir ve iki istemcinin durum mantığı dokunulmadan kalır; eski APK iade edilen paketi FAILED olarak görür.
- (+) Menajer "sessiz" atamayı, bırakma isteklerini ve iadeleri tek sayfada görür; kurye kartında kapıdaki para görünür.
- (−) Rotaya "işletmeye dönüş" durağı eklenmez (kurye dış navigasyonla gider); takip işi (TAKIP-ISLERI).
- (−) Bırakma talebi yalnızca alıştan önce; alıştan sonra kurye devir ya da menajeri arama yolunu kullanır (kaza / arıza için "işi menajere geri ver" ayrı iş).
- (−) Sürüm kapısı başlığı göndermeyen bugünkü APK'yı da engeller (aktif paketi yokken): kapı, yeni APK kuryelere dağıtılmadan açılmamalıdır (çalışma kitabı §12).
- (−) Ret kapalı kipi, restorana iade ve kapıda bekleme süresi de **yeni APK** gerektirir: bugünkü APK'da Reddet 409 verir, bırakma talebi / iade / geri sayım ekranı yoktur, `waitedS` gönderilmediği için "müşteri yok" 422 alır. Panel "Pilot ayarlarını uygula" düğmesi ve Ayarlar > Pilot kuralları bunu uyarır.
- (−) "Kutum dolu" tamamen bilgi amaçlıdır; menajer gerekirse kuryeye özel kapasiteyi düşürür.
- (−) `ACCEPT_OVERDUE` uyarısı ana uyarı anahtarı (`settings.alerts.enabled`) kapalıysa açılmaz.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Ret kapalıyken süre aşımında paketi otomatik başka kuryeye vermek | Referans önerisi "sessizlikte otomatik başkasına gitmesin ya da yalnızca menajer kararıyla"; pilot müşteri işi kuryede tutmak istiyor |
| Yeni paket durumları `RETURNING` / `RETURNED` | Durum makinesi, iki istemci, metrik ve rota kodunda geniş değişiklik; eski APK bilinmeyen durumu gösteremez |
| "Kutum dolu" kuryeyi otomatik atamadan çıkarsın | Ürün sahibi kararı: sinyal; referanslarda kuryenin kendi "dolu" düğmesi yok (kanıt zayıf) |
| Sürüm kapısını her durumda uygulamak | Yoldaki kuryenin işi yarıda kalır (ürün sahibi kararı) |
| Dış navigasyon açılışını paket olayı olarak yazmak | Paket geçmişini kalabalıklaştırır; sayaç için ayrı, 90 günlük tablo yeterli |
