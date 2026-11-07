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
