# ADR-0036: Gecikme ve takılma uyarıları (menajer için canlı uyarı)

- **Durum:** Kabul edildi (ürün sahibi, MVP+ madde 7: "paket geç kalıyorsa ya da kurye takıldıysa menajer paneli canlı uyarsın")
- **Tarih:** 2026-11-08
- **İlgili:** [ADR-0009](0009-rota-uyumlu-atama.md) (rota durak tahminleri `route_stops.eta_at`), [ADR-0025](0025-isletme-hazir-baglantisi.md) (işletme sayfası), [ADR-0028](0028-havuz-gorunurlugu-ve-atama-kabulu.md) (kabul süresi aşımı ayrıca ele alınır), [ADR-0029](0029-konum-kademeleri-ve-rota-geri-bildirimi.md) (konum kademeleri, `locationStaleMinutes`), [ADR-0033](0033-yuk-testi-darbogazlari.md) (olay güdümlü panel veri katmanı)

## Bağlam

Menajer bugün gecikmeyi haritaya bakarak ya da telefonla fark ediyor: hazır bekleyen paket alınmıyor, kurye sessizce yerinde duruyor, konumu kesilen kurye yalnızca haritada kırmızı işaret oluyor. Pilot müşteri bunların "kendisine söylenmesini" istiyor. Kapsam yalnızca panel içi uyarıdır; SMS, e-posta ve anlık bildirim (FCM) ayrı karardır (ücretli servis kuralı, CLAUDE.md madde 10).

## Karar

**1. Dört uyarı türü** (`alerts.type`); eşikler kiracı ayarıdır (`settings.alerts`, yalnızca OWNER yazar, aralık dışı 400):

| Tür | Koşul | Eşik (varsayılan, aralık) |
|---|---|---|
| `LATE_PICKUP` | Paket hazır (`ready_at` geçmiş) ama durumu CREATED / IN_POOL / ASSIGNED, yani alınmadı | `latePickupMin` 15 (1-120) |
| `LATE_DELIVERY` | ASSIGNED / PICKED_UP paketin planlı teslim durağı tahmini (`route_stops.eta_at`) eşik kadar aşıldı; tahmin yoksa atanalı `lateDeliveryNoEtaMin` geçti. Kabul bekleyen (PENDING) atama sayılmaz (kabul süresi aşımı kapsar) | `lateDeliveryMin` 10 (1-120); `lateDeliveryNoEtaMin` 45 (15-240) |
| `COURIER_STALE` | Vardiyada kuryeden en az `max(locationStaleMinutes, 5)` dakikadır konum yok (Redis `lastseen`; hiç konum yoksa vardiya başlangıcından). Asgari 5 dk: asansör / otopark gibi kısa kesintiler uyarı üretmez. Açmak için koşul **2 ardışık değerlendirmede** sürmeli, kapatmak için de **2 ardışık değerlendirmede** taze konum görülmeli (aç-kapa titremesi önlenir; sayaç Redis hash `t:{tenant}:alerts:stale-streak`, 1 saat TTL). Molada (`shifts.break_ends_at` dolu, MOLA) sinyal beklenmez: uyarı açılmaz, açıksa kapanır | mevcut `settings.locationStaleMinutes` (**yinelenmez**), etkin eşik `max(değer, 5)` |
| `STUCK_COURIER` | Vardiyada ve aktif paketi olan kurye son `stuckMin` dakikada 100 m'den az yer değiştirdi | `stuckMin` 20 (5-120) |

`settings.alerts.enabled` (varsayılan true) tüm değerlendirmeyi kapatır; kapatınca açık uyarılar kapanır. `STUCK_COURIER` yanlış alarmı azaltan kurallar: vardiya ve en eski aktif paket pencereden eski; pencere ping ile en az %70 kapsanmış (en az 2 ping); son ping taze (konum kesilmişse `COURIER_STALE`'in işidir); kurye planlı bir durağın 100 m'si içinde değil (durakta beklemek takılmak değildir, gecikmesini `LATE_PICKUP` / `LATE_DELIVERY` yakalar); sahte (mock) konumlar sayılmaz; molada kurye takılmış sayılmaz (mola bayrağı `shifts.break_ends_at` **varsa** okunur, sütun yoksa atlanır; MOLA ADR-0035 ile gelir).

**2. Veri modeli** (`alerts`, migrasyon 0013): `tenantId, type, severity, packageId?, courierId?, data (jsonb, açılışta ölçülen aşım dakikası), openedAt, resolvedAt?, ackedBy?, ackedAt?, ackNote?`. Konu (paket ya da kurye) + tür başına **en çok bir açık satır** (kısmi benzersiz dizin `COALESCE(package_id, courier_id)`, `resolved_at IS NULL`); koşul kalkıp yeniden doğarsa yeni satır açılır. Satır kişisel veri taşımaz: müşteri adı / telefonu / adresi yoktur; kurye adı ve paket numarası okuma anında birleştirilir. `severity`: açıldığı anda eşiğin iki katı aşılmışsa `CRITICAL`, aksi `WARNING` (sonradan yükseltilmez). Çözülmüş kayıtlar 30 gün sonra silinir.

**3. Değerlendirme işi.** Her 60 sn, aktif her kiracı için (`SET NX PX 50s` kilidi `lock:job:alerts:{tenantId}`: çok örnekli kurulumda tek örnek) dört koşul hesaplanır, açık satırlarla karşılaştırılır: yeni olan açılır (`alert.opened`), koşulu kalkan kapanır (`alert.resolved`; teslim, iptal, hareket, konum geldi, vardiya bitti, ETA tazelendi, ayar değişti). İdempotent: tekrar çalıştırmak değişiklik üretmez. Bir koşul türü hesaplanamazsa (ör. Redis erişilemez) o türün açık uyarılarına dokunulmaz. Gecikme en çok bir dakikadır; teslim anında olay güdümlü kapatma yapılmaz (paket modülüne dokunmamak için bilinçli sade tutuldu).

**4. Uçlar** (OWNER, DISPATCHER; kiracıya bağlı, başka kiracı 404): `GET /alerts?status=open|resolved&type=&limit=&offset=`, `GET /alerts/summary` (`openByType`, `openTotal`, `unackedTotal`), `POST /alerts/{id}/ack` ("Gördüm": doğal idempotent, ilk not korunur, uyarıyı kapatmaz, yalnızca rozetten düşer). WS (yalnızca panel kitlesi, kurye uygulamasına gitmez): `alert.opened`, `alert.resolved`, `alert.acked`, hepsi güncel `Alert` taşır (websocket-events.md §4.7).

**5. Panel (P2).** Üst çubukta zil: okunmamış (Gördüm denmemiş) açık uyarı sayısı. Çekmece: açık uyarılar, simge + etiket çipi, "N dk önce", "Pakete git" (paket çekmecesi) / "Kuryeye git" (canlı harita, kurye seçili), "Gördüm"; yeni uyarıda bildirim, ses varsayılan kapalı (çekmecede anahtar, `localStorage`). `/uyarilar` sayfası: açık + geçmiş, türe göre süzgeç, sayfalama. Ayarlar › Uyarılar (OWNER). `alert.*` olayları yalnızca uyarı sorgularını yamar (sayaçlar tekrar saymaz); paket / kurye sorguları yeniden çekilmez (ADR-0033 deseni). İşletme "hazır" sayfasında açık `LATE_DELIVERY` olan pakete "Gecikiyor" rozeti (`BusinessLinkPackage.delayed`; kurye bilgisi yok).

**COURIER_STALE kararlılığı (PR #61 incelemesi):** gecikme en fazla ~2 dk büyür (2 tur), karşılığında kısa kesinti ve zayıf sinyal titremesi menajeri yormaz. `STUCK_COURIER`'in "son ping taze" penceresi de aynı etkin eşiği (`max(locationStaleMinutes, 5)`) kullanır, böylece iki tür arasında boşluk kalmaz. Mola koşulu `shifts.break_ends_at` (MOLA, ADR-0035, migrasyon 0012) doğrudan okunur; hem `STUCK_COURIER` hem `COURIER_STALE` için e2e testi zorunludur.

## Sonuçlar

- Menajer gecikmeyi haritaya bakmadan, bir dakika içinde görür; uyarı koşul kalkınca kendiliğinden kapanır, "Gördüm" yalnızca gürültüyü azaltır.
- ETA'sı yeniden hesaplanan rota, tahmini ileri taşıdığı için gecikmeyi bir süre gizleyebilir; bu durumda `lateDeliveryNoEtaMin` ve `STUCK_COURIER` yedek ağdır. Pilotta eşikler gerçek verilerle ayarlanacak.
- Yeni tablo ve kiracı sütunları (varsayılanlar mevcut kiracılar için de açık: uyarılar varsayılan **açık**); kapatmak için `settings.alerts.enabled = false`.
- Ek yük: kiracı başına dakikada birkaç hafif sorgu (aktif paketler, açık vardiyalar, yalnızca aday kuryeler için ping penceresi).

## Alternatifler

- **Olay güdümlü hesap (paket / konum olaylarında):** daha az gecikme, ama paket, konum ve dağıtım modüllerine dokunmak gerekir; dakikalık iş yeterli bulundu.
- **Tek "gecikme" eşiği:** farklı olaylar (alınmama, teslim aşımı, takılma) farklı süre ister; ayrı eşikler seçildi.
- **Anlık bildirim (FCM) / SMS / e-posta:** panel içi uyarı yeterli; harici sağlayıcılar ayrı karar ve onay gerektirir.

## Ek (2026-11-12, MVP+ 12): kurye ret oranı uyarısı `COURIER_HIGH_REJECTION`

**Gerekçe.** Mevcut `courier.rejection_alert` olayı yalnızca vardiya başına ret **sayısına** bakar ve bir kez bildirir. Menajer "bu kurye tekliflerin çoğunu reddediyor" bilgisini panelde kalıcı, kapanan / açılan bir uyarı olarak ister; oran, çok teklif alan kuryeyi (10 tekliften 2 ret) az teklif alan kuryeden (3 tekliften 2 ret) ayırır.

**Beşinci uyarı türü:**

| Tür | Koşul | Ayar (varsayılan, aralık) |
|---|---|---|
| `COURIER_HIGH_REJECTION` | Açık vardiyası olan kuryenin son `rejectionWindowHours` saatteki ret oranı >= `rejectionRatePct` ve en az `rejectionMinOffers` teklif | `rejectionEnabled` true; `rejectionRatePct` 40 (10-100); `rejectionWindowHours` 24 (1-168); `rejectionMinOffers` 5 (3-100) |

**Tanımlar (mevcut veriden):**
- **Teklif** = `package_events` içinde kuryeye yazılan `ASSIGNED`, `REASSIGNED`, `ASSIGNED_AUTO` olayı (yönetici ya da sistem ataması). Kuryenin havuzdan kendi aldığı paket (`data.assignment.source` = `CLAIM` / `SMART_POOL_CLAIM`) ve devirle gelen paket (`HANDOVER`, devri kurye zaten kabul etmiştir) teklif **sayılmaz**.
- **Ret** = aynı kuryenin `REJECTED` olayı (kurye uygulamasında gerekçeli ret) **ve** o paket için kuryeye yapılan son atamanın teklif olması (kaynağı `CLAIM` / `SMART_POOL_CLAIM` / `HANDOVER` değil). Böylece pay ve payda aynı kümedendir: havuzdan kendi aldığı ya da devirle gelen paketi "Reddet" ile geri vermek ret oranını şişirmez. Eşleşme, retten önceki en son atama olayına bakılarak sorguda yapılır (şema değişikliği yok). Serbest bırakma (`RELEASED`) ret değildir.
- **Kabul süresi aşımı** (`ASSIGNMENT_EXPIRED`, ADR-0028) orana **girmez**; ayrı sayaç olarak uyarı ayrıntısında görünür (`rejection.timeouts`). Gerekçe: aşım "reddetme" kararı değildir (uygulama kapalı, telefon cebinde olabilir); yine de teklif sayısında yer alır. Ayrı eşik gerekirse ayar eklenir (ADR-0028 sonuçlarındaki not).
- Pencere olay kayıt zamanına (`created_at`) göre **kayan** saat penceresidir (vardiya penceresi değil): bir önceki günün ret yoğunluğu da görünür, vardiya sınırında oran sıfırlanıp uyarıyı gizlemez. Oran `ret / teklif`, en çok %100 (pencere dışı teklifin pencere içi reti oranı şişirebilir; kırpılır).

**Yaşam döngüsü (diğer türlerle aynı kalıp):** konu = kurye, kurye + tür başına en çok bir açık satır (mevcut kısmi benzersiz dizin); dakikalık değerlendirme, `alert.opened` / `alert.resolved` / `alert.acked` olayları (yeni olay yok), yalnızca panel rolleri, her sorgu `tenant_id` ile süzülür. **Histerezis:** açmak için oran >= eşik, açıkken oran >= eşik - 10 puan kaldığı sürece açık kalır; örneklem `rejectionMinOffers`'ın altına inerse (kayan pencere) kapanır. Yalnızca açık vardiyası olan kuryeler değerlendirilir: vardiya bitince uyarı kapanır. **Mola** oranı değiştirmez; uyarıyı açmaz da kapatmaz da (`COURIER_STALE`'in tersine mola burada bir beklenen durum değil, geçmiş veri). `severity`: oran eşiğin iki katına (en çok %100) ulaşmışsa `CRITICAL`, aksi `WARNING` (açılışta sabitlenir). Açılış anındaki sayılar (`offers`, `rejections`, `timeouts`, `ratePct`, `windowHours`) `alerts.data.rejection` içinde saklanır ve `Alert.rejection` olarak döner; kişisel veri yok, günlüğe yalnızca sayılar / kimlikler yazılır. Anahtar kapalıysa (`rejectionEnabled = false` ya da ana `enabled = false`) açık ret uyarıları kapanır.

**Veri modeli (migrasyon 0014):** `tenants.alerts_rejection_enabled / _rate_pct / _window_h / _min_offers` (CHECK aralıkları), `alerts_type_chk` yeni türle genişler, `package_events(tenant_id, courier_id, created_at)` üzerinde kısmi dizin (`type IN ASSIGNED, REASSIGNED, ASSIGNED_AUTO, REJECTED, ASSIGNMENT_EXPIRED`) kurye başına sayımı hızlı yapar. Sözleşme 0.6.8: `AlertType`, `Alert.rejection`, `AlertSummary.openByType`, `AlertSettings` / `AlertSettingsUpdate` eklemedir.

**Panel:** zil / çekmece / `/uyarilar` listesinde "Yüksek ret" çipi (simge + etiket), ayrıntı satırı "Açılışta: son 24 saat, 10 tekliften 5 ret (%50) · 1 kabul süresi aşımı (orana dahil değil)" (açılış anındaki ölçümdür, canlı değildir), "Kuryeye git", türe göre süzgeç; Ayarlar › Uyarılar'da anahtar + üç eşik.

### Ürün sahibi kararı bekleyen (tek satırda değiştirilebilir varsayılanlar)

| Karar | Önerilen / uygulanan varsayılan | Nerede değişir |
|---|---|---|
| Uyarı varsayılan açık mı | Açık (`rejectionEnabled = true`) | `tenants.alerts_rejection_enabled` varsayılanı (`db/schema/tenants.ts`) ya da Ayarlar › Uyarılar |
| Ret oranı eşiği | %40 | `alerts_rejection_rate_pct` varsayılanı (10-100) |
| Pencere | Son 24 saat, kayan (vardiya değil) | `alerts_rejection_window_h` varsayılanı (1-168) |
| Asgari teklif sayısı | 5 | `alerts_rejection_min_offers` varsayılanı (3-100) |
| Kapanış payı (histerezis) | 10 puan | `REJECTION_CLOSE_MARGIN_PCT` (`alerts/alert-rules.ts`) |
| Süre aşımı orana girsin mi | Hayır (ayrı sayaç olarak gösterilir) | `highRejection` sorgusundaki `rejections` sayacı (`alerts-evaluator.service.ts`) |
| Havuzdan kendi aldığı paket teklif sayılsın mı | Hayır (geri verilmesi de ret sayılmaz) | `highRejection` sorgusundaki `source` süzgeci |
| Pencere türü: kayan saat mi, yalnızca geçerli vardiya mı | Kayan 24 saat. Sonucu: uyarı vardiya bitince kapanır, ertesi vardiyanın ilk dakikasında önceki günün verisiyle yeniden açılır (menajere yeni zil bildirimi gider). Alternatif: "yalnızca geçerli vardiyanın teklifleri" (her vardiya sıfırdan başlar, gün sonu yoğun ret ertesi güne taşınmaz; ama vardiyanın ilk teklifleri asgari örneklemi doldurana dek uyarı çıkmaz) | `highRejection` sorgusundaki pencere koşulu (`created_at > now - window`) yerine vardiya başlangıcı |
