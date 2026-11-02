# ADR-0010: Kuryeler arası devir (yakındaki meslektaşa teklif, alıştan sonra fiziksel buluşma ve kanıt)

- **Durum:** Kabul edildi (varsayılan eşikler pilot müşteri ile doğrulanacak)
- **Tarih:** 2026-10-30

## Bağlam

Pilot müşteri (bir kurye firması) yeni bir istek iletti: bir kurye, kendisine atanmış paketi, **haritadaki konumunun belirli bir yarıçapı içindeki vardiyada olan herhangi bir meslektaşına** devredebilsin. Sahada bu iş bugün telefonla ve sistem dışında yapılıyor; paket el değiştiriyor ama teslimat kaydı, nakit sorumluluğu ve hakediş yanlış kuryede kalıyor.

[ADR-0009](0009-rota-uyumlu-atama.md) kontrollü ret ve devri zaten tanımlıyordu, ancak yalnızca **yönetici onaylı devir isteği** olarak: alıştan sonra kurye yalnızca istek gönderir, yönetici yeni kuryeyi seçip onaylar. Yeni istek, kuryelerin birbirleriyle doğrudan devir yapmasını ister. Tasarım gerilimi şudur:
- Kuryeye esneklik tanımak (yakınındaki arkadaşı paketi zaten rotasında taşıyabilir) ile kontrolsüz bırakmayı geri getirmemek (ADR-0009'un çözdüğü sorun) arasındaki denge.
- Alınmış paketin **fiziksel olarak** el değiştirmesi gerekir; sistem bunun gerçekten olduğunu doğrulamalıdır.
- Taze kalma süresi ve kapıda nakit sorumluluğu devirle kaybolmamalıdır.

Ayrıntılı tasarım: iterasyon-2/KURYE-DEVIR.md.

## Karar

**1. Aday listesi.** Kurye bir pakette "Devret"e dokunur ve yakındaki meslektaşları görür. Bir meslektaşın listede görünmesi için:
- vardiyada olmalı ve **STALE olmamalı** (konumu taze),
- kapasitesi olmalı (`max_active_packages_per_courier`),
- devredenin son konumuna **`handover_radius_m`** (kiracı ayarı, varsayılan **1500 m**) içinde olmalı. Kontrol son konumlar üzerinde **PostGIS** (`ST_DWithin`) ile yapılır.

Sıralama **önce mesafe**dir. [ADR-0009](0009-rota-uyumlu-atama.md) skoru hazırsa her satıra paketin devralanın rotasına eklediği sapma da yazılır (ör. "350 m · rotasına +2 dk"). Listede koordinat dönmez, yalnızca yuvarlanmış mesafe döner.

**2. Alıştan önce (`ASSIGNED`): teklif.**
- Devir, devralana giden bir **istektir**; devralan `handover_offer_timeout_s` (varsayılan **60 sn**) içinde kabul eder veya reddeder.
- Kabulde paket, havuzdan alma gibi **koşullu UPDATE** ile atomik olarak yeniden atanır.
- Ret veya zaman aşımında paket devredende kalır.
- Kiracı ayarı `handover_requires_dispatcher_approval` (varsayılan **false**): `true` ise yönetici de onaylamalıdır.
- Yönetici devri her durumda **WebSocket olayı** ve **denetim kaydı** ile canlı görür.

**3. Alıştan sonra (`PICKED_UP`): fiziksel devir.** Paket devredenin üzerindedir, bu yüzden kuryeler buluşmalıdır.
- Onay anında iki kurye birbirinden en çok **`handover_meet_radius_m`** (varsayılan **150 m**) uzakta olmalıdır.
- **Kanıt:** devralan, devredenin telefonunda gösterilen kısa ömürlü bir **QR'ı tarar veya 4 haneli kodu girer**.
- **Kural 1:** "alıştan teslime azami dakika" sınırı ([ADR-0009](0009-rota-uyumlu-atama.md)) **orijinal alış zamanından** işlemeye devam eder; devir `picked_up_at`'i sıfırlamaz.
- **Kural 2:** `CASH` pakette kapıda tahsilat sorumluluğu devralana geçer; bu, kasa mutabakatı için ayrı bir olayla kaydedilir.
- Yönetici bilgilendirilir. Onay isteğe bağlıdır ve aynı `handover_requires_dispatcher_approval` ayarına bağlıdır.

**4. Kötüye kullanım sınırları.** `max_handovers_per_package` (varsayılan **2**) ve `max_handovers_per_courier_per_day` (varsayılan **10**). Her adım `package_events`'e yazılır ve metriklerde görünür (ör. "en çok devreden / devralan").

**5. Mevcut akışlarla ilişki.**
- "Bırak / gerekçeli ret ile havuza dön" ([ADR-0009](0009-rota-uyumlu-atama.md), alıştan önce) aynen kalır.
- ADR-0009'un yönetici onaylı **devir isteği** artık alıştan sonra iki devir yolundan biridir: **A) kurye-kurye devir** (bu ADR), **B) yönetici onaylı devir isteği** (kurye devralanı seçemiyor veya devam edemiyor; yönetici yeni kuryeyi seçer). İki yol aynı buluşma ve kanıt mekaniğini kullanır; yönetici onayı sonrası B yolu da `handover_offers` kaydı üretir. ADR-0009 ve tasarım dokümanı bu doğrultuda güncellenmiştir.
- Paket durum makinesi (SPEC §5) **değişmez**; devir `courier_id`'yi değiştirir, yeni durum eklenmez.

**6. Taslak (sözleşme çalışması için; bu ADR `packages/contracts`'a dokunmaz).**
- *Veri:* `handover_offers` (id, package_id, from_courier, to_courier, stage `ASSIGNED | PICKED_UP`, status `PENDING | ACCEPTED | DECLINED | EXPIRED | CANCELLED`, code_hash, expires_at, decided_at ve ek alanlar), kiracı ayarları (aşağıdaki tablo), yeni `package_events` tipleri (`HANDOVER_OFFERED`, `HANDOVER_OFFER_ACCEPTED`, `HANDOVER_OFFER_DECLINED`, `HANDOVER_OFFER_EXPIRED`, `HANDOVER_OFFER_CANCELLED`, `HANDOVER_COMPLETED`, `CASH_RESPONSIBILITY_TRANSFERRED`; ADR-0009'un `HANDOVER_APPROVED / DENIED` tipleri yeniden kullanılır).
- *Uç noktalar:* `GET /courier/packages/{id}/handover-candidates`, `POST /courier/packages/{id}/handover-offers`, `POST /courier/handover-offers/{id}/accept | decline`, `POST /courier/handover-offers/{id}/confirm` (kod + konum), yardımcı uçlar tasarım dokümanında.
- *WebSocket:* `handover.offered`, `handover.updated`.

**Kiracı ayarları** (varsayılanlar pilotla doğrulanacak varsayımlardır):

| Ayar | Varsayılan |
|---|---|
| `handover_radius_m` | 1500 m |
| `handover_offer_timeout_s` | 60 sn |
| `handover_requires_dispatcher_approval` | false |
| `handover_meet_radius_m` | 150 m |
| `max_handovers_per_package` | 2 |
| `max_handovers_per_courier_per_day` | 10 |

**Ek tasarım kararları** (kısa ömürlü değerler ve güvenlik ayrıntıları, ayrıntı tasarım dokümanında):
- Her uç durum için tek bir doğruluk kuralı vardır: **veritabanı koşullu UPDATE'i** karar verir; süre ve yarış kontrolü her çağrıda yapılır, arka plan işi yalnızca temizler.
- Pakette en çok **bir** `PENDING` teklif (kısmi benzersiz indeks); açık yönetici devir isteği varken de teklif açılamaz.
- 4 haneli kodun güvenliği kodun kendisine değil; kısa ömre (`handover_code_ttl_s`, 120 sn), 5 deneme sınırına, konum doğrulamasına ve yalnızca devralanın onaylayabilmesine dayanır. `code_hash` HMAC ile saklanır.
- Nakit pakette devralan onay ekranında tutarı görüp onaylamadan devir tamamlanmaz (`CASH_ACK_REQUIRED`).
- Alıştan sonra yönetici **elle onay** (`confirm-manual`, gerekçeli) verebilir: kaza gibi durumlarda devreden kodu gösteremeyebilir.
- Yönetici kaynaklı devir (yol B) sınırlara takılmaz ama sayılır.
- Sınırlar yalnızca tamamlanmış devirleri sayar; yanlış kod veya reddedilen teklif kuryenin hakkını tüketmez.
- Gizlilik: aday listesinde koordinat yoktur; karşı kuryenin canlı konumu yalnızca buluşma aşamasında ve yalnızca iki tarafa gösterilir. Aydınlatma metni buna göre güncellenir ([ADR-0005](0005-konum-takibi-ve-kvkk.md)).
- Kurye-kurye devir "kötü atama oranı"na dahil edilmez, ayrı raporlanır.

## Sonuçlar

- (+) Sahada telefonla yapılan, sistemin göremediği devirler kayıt altına girer; teslimat kaydı, nakit sorumluluğu ve (İterasyon 3) hakediş doğru kuryeye yazılır.
- (+) Kurye yakınındaki meslektaşla hızla çözüm bulur; yönetici her devri canlı görür ve istenirse onay şartı koyar. Varsayılan "bilgilendir, engelleme" operasyonu yavaşlatmaz.
- (+) Alınmış paketin devri fiziksel buluşmaya ve iki yönlü kanıta (kod + konum) bağlıdır; "kâğıt üstünde devir" ve sahte devir zorlaşır.
- (+) Taze kalma süresi sayacı ve nakit sorumluluğu devirde korunur; kasa mutabakatı (İterasyon 3) için hazır olay kaydı oluşur.
- (+) Mevcut altyapıyı kullanır: PostGIS `ST_DWithin`, koşullu UPDATE deseni, `package_events`, WebSocket; paket durum makinesine yeni durum eklenmez.
- (-) Kurye konumları meslektaşlara kısıtlı da olsa görünür hâle gelir (buluşma aşaması); KVKK aydınlatma metni, açık rıza ve hukuk onayı gerekir.
- (-) Şehir içinde GPS hatası 150 m buluşma yarıçapını güvenilmez kılabilir; pilotta ölçülecek, yedek olarak yönetici elle onayı vardır.
- (-) Kullanıcı arayüzü yükü: yeni kurye ekranları (aday listesi, teklif kartı, kod / QR, QR tarama) ve panelde onay akışı.
- (-) Kuryeler arası devir, rota uyumlu atamanın iyi çalıştığı durumda gereksiz olabilir; kötü atamanın yönetici yerine kuryeler arasında çözülmesi sinyali gizleyebilir. Bu yüzden "en çok devreden / devralan" metriği yöneticiye gösterilir.
- (-) 4 haneli kodun entropisi düşüktür; güvenlik katmanlı kontrollerle (konum, süre, deneme sınırı) sağlanır, tek başına kod yeterli sayılmaz.
- (-) Kalıcı veri modeline yeni tablo (`handover_offers`) ve kiracı ayarları eklenir; zaman aşımı için yeni bir arka plan işi gerekir.

## Alternatifler

| Alternatif | Neden seçilmedi |
|---|---|
| Devri yalnızca yönetici onaylı yapmak (ADR-0009'un tek yolu) | Sahadaki talebi karşılamaz ve yöneticiyi her devirde darboğaz yapar; yönetici yolu B olarak zaten kalır. |
| Devre kısıtlama koymamak (alıştan sonra da tek dokunuşla serbest devir) | Fiziksel olarak el değiştirmeden sistem kaydı değişirse nakit ve taze kalma takibi yanlış kuryeye yazılır; kötüye kullanım denetlenemez. |
| Alıştan sonra kodsuz, yalnızca konum yakınlığıyla otomatik devir | Yakında duran iki kurye ayrı işler yapıyor olabilir; devredenin bilinçli onayı olmadan paket sahibi değişirdi. |
| QR yerine kalıcı uzun bir bağlantı veya Bluetooth/NFC yakınlık kanıtı | Ek izin, cihaz bağımlılığı (iOS/Android farkı) ve karmaşıklık; kod + GPS pilot için yeterli ve her telefonda çalışır. |
| Devir için yönetici onayını varsayılan zorunlu yapmak | Hızlı sahada yavaşlatır; ayar açık olduğunda zaten mümkündür (`handover_requires_dispatcher_approval`). |
| Aday listesini yalnızca rota sapmasına göre sıralamak | Sapma hesabı OSRM'e bağımlıdır; mesafe her koşulda çalışır ve kuryenin sezgisine uyar. Sapma, varsa ek bilgidir (`sort=detour` ile de kullanılabilir). |
| Teklifleri yalnızca istemci tarafında (kuryeler arasında) takip etmek | Zaman aşımı, yarış ve denetim kaydı sunucu doğruluğu gerektirir. |
