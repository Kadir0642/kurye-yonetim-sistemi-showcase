# Navigasyon ekranı: tasarım varyantları

> **Karar (2026-11-06):** ürün sahibi **B · Koyu / premium** varyantını seçti ve bunun ürünün kurumsal görsel dili olmasını istedi. Ayrıntı: [`secim.md`](secim.md), son hali: [`b-v1.html`](b-v1.html), marka belirteçleri: [`../marka/`](../marka/). Bu sayfa ve `index.html` seçimin dayanağı olarak korunuyor.

Kurye uygulamasının adım adım navigasyon ekranı (ADR-0015) ürün sahibine "kaba / keskin" geldi ([`ref/mevcut-ekran-1.png`](ref/mevcut-ekran-1.png), [`ref/mevcut-ekran-2.png`](ref/mevcut-ekran-2.png)). Bu klasör, uygulama koduna dokunmadan önce seçilecek üç görsel dili gösterir.

| Dosya | İçerik |
|---|---|
| [`index.html`](index.html) | Üç varyant yan yana, telefon çerçevesinde (390×844). Gündüz / gece / yan yana, "harita kaydırıldı" (Ortala), bildirim ve kaba güneş parlaması benzetimi düğmeleri. Altında her varyantın renkleri, köşe, gölge, tip ölçeği, manevra ikon seti ve bileşenleri. |
| [`tokens.json`](tokens.json) | Üç varyantın belirteçleri. `ui.day` / `ui.night` alanları `NavTheme` (`navigation/engine/theme.ts`) ile aynı adlıdır; `map.*` alanları `infra/map/scripts/generate-styles.mjs` içindeki `NAV_DAY` paletiyle eşleşir. |

**Açmak için:** `index.html` dosyasını tarayıcıda çift tıklayın. İnternet yoksa yazı tipleri sistem yazı tiplerine düşer; sayfa yine okunur. Harita elle çizilmiş bir SVG'dir (gerçek karo değil), ama renkleri doğrudan MapLibre stil anahtarlarına karşılık gelir.

## Ne sabit, ne değişiyor

Her varyantta aynı an gösterilir: **50 m · Cengiz Topel Caddesi'ne sağa dönün**, *Ardından* Müderris Sokak'a sola, hız 40 km/sa, **1 dk · 750 m · 16:39**, "Genel bakış" hapı, Ses açık / Menü / Çıkış, "Yeni rota hesaplandı" bildirimi. Hiçbir bilgi kaldırılmadı.

Kurye kuralları her varyantta korunur: çok büyük mesafe (64-84 px) ve ok, güneşte yüksek kontrast, otomatik gece, ≥ 56 dp dokunma hedefi (alt düğmeler 68×64, Ortala 64), kasıtlı çıkış (Çıkış hâlâ onay sayfasını açar), ses öncelikli, puk ekranın alt üçte birinde sabit ve rota ondan yukarı gider (yön-yukarı eğik kamera).

Değişen: renk, köşe yarıçapı, gölge, yazı tipi, ikon dili ve harita paleti.

### Mevcut ekranı "kaba" yapan şeyler (ölçülen)

- **Yol kenarlıkları koyu kahverengi-gri** (`road.case.minor` `#9c958a`, beyaz yolla 3,0:1). Her ara sokak kalın bir çerçeveyle çiziliyor; harita "çizgi film" gibi. Varyantlarda kenarlık ya çok açık (A: 1,3:1) ya da zemine yakın.
- **Gece haritası formülle türetiliyor** (`nightColor`: yollar L = 0,36 + 0,4·L). Ara sokaklar %76 açıklıkta parlak gri çizgiler olarak gece gözü yoruyor. Varyantlar elle ayarlanmış, kısık bir gece paleti öneriyor.
- **Metin glifleri ikon yerine kullanılıyor** (`☰`, `✕`, `◎`, `⤢`); yazı tipine göre boyutları ve kalınlıkları değişiyor. Varyantlarda hepsi tek çizgi kalınlığında SVG.
- **Her şey 900 kalınlıkta** ve tam doygun yeşil bant `#177A33` + `shadowOpacity 0.3`. Hiyerarşi yok; varyantlarda yalnızca mesafe en kalın.

## Karşılaştırma

| Ölçüt | A · Sade | B · Koyu / premium | C · Marka / canlı |
|---|---|---|---|
| Güneşte okunurluk | **Çok iyi.** Beyaz kart / `#202124` metin 16,1:1; mavi karoda beyaz ok 4,5:1 (büyük şekil) | **İyi.** Panel metni 18,6:1, lime ok 15,8:1; ama koyu yüzey ekran camında yansımayı daha çok gösterir | **Çok iyi.** Teal bantta beyaz 5,5:1 (mevcut yeşil 5,4:1 ile aynı), beyaz alt panel |
| Gece | İyi. Koyu gri yüzeyler, metin 11,4:1 | **Çok iyi.** En az parlama, harita neredeyse siyah | İyi. Koyu teal bant 9,1:1, sıcak kısık harita |
| Eldiven hedef boyutu | Düğme 68×64, hap 56, Ortala 64 | Düğme 68×64, hap 56, Ortala 64 | Düğme 70×64 (hap), hap 56, Ortala 64 |
| Uygulama maliyeti | M | M | S–M |
| MapLibre stil değişikliği gerekiyor mu | Evet: kenarlıklar açılır, ana yol sarısı kısılır, elle gece paleti | Evet: iki palet de yeni (gümüş gündüz, koyu gece) | Küçük: sıcak tonlar + elle gece paleti |
| Yeni yazı tipi | Inter (isteğe bağlı) | Barlow Condensed (rakam) + Inter | Nunito |
| Rota rengi / harita | Mavi `#1A73E8`, koyu mavi kenar; zeminle 3,9:1 | Camgöbeği-mavi `#0091E6`, lacivert kenar; zeminle 2,8:1 (kenarlık taşıyor) | Teal `#0E9384`, koyu teal kenar; zeminle 3,3:1 |

Kontrast oranları WCAG formülüyle `tokens.json` değerlerinden hesaplandı.

## Uygulama notları (seçilen varyantı uygulayacak agent için)

Ortak: `NavTheme` (`apps/courier-mobile/src/navigation/engine/theme.ts`) şu alanları kazanır: `surface2`, `time`, `dangerSoft`, `toastBg`, `toastText`, `toastIcon`, `chipBg`, `chipText`, `chipBorder`, `meStroke`, `meHalo`, `meCone` (+ varyanta özel `maneuverTile`, `accent2`, `glassBorder`, `nextChip`, `accentWarm`). Değerler `tokens.json` → `variants.X.ui` içinden birebir kopyalanır. Uyarı şeritleri (GPS, çevrimdışı, yeniden rota hesaplanıyor) her varyantta kehribar `#FBBF24` kalır; yalnızca bilgi tonundaki `notice` ("Yeni rota hesaplandı") bandın altında ayrı bir bildirim (toast) olur (`NavBanner.statusStrip` → `tone: 'info'`).

| Dosya | A | B | C |
|---|---|---|---|
| `components/nav/NavBanner.tsx` | Bant → kenarlardan 10 dp içeride yüzen beyaz kart (radius 24); ok 76 dp mavi karo içinde; talimat 21/600 | Yüzen koyu kart (radius 20, %94 opak + 1 dp açık kenar); ok 92 dp lime; mesafe Barlow Condensed 84; "ARDINDAN" ayrı koyu şerit | Tam genişlik teal bant, alt köşeler 32; "Ardından" bandın kenarına asılı hap |
| `components/nav/NavBottomBar.tsx` | Beyaz alt sayfa (radius 24, tutamaç); süre yeşil; Çıkış açık kırmızı zeminde kırmızı | Kenarlardan 8 dp içeride yüzen koyu "dock" (radius 26); etiketler büyük harf; varış saati camgöbeği | Beyaz sayfa (radius 32); düğmeler hap; Çıkış dolu kırmızı |
| `components/nav/ManeuverArrow.tsx` | Yuvarlak uçlu çizgi (11/100), açık ok başı (çizgi) | Kalın köşeli çizgi (14/100), dolu sivri ok başı, kavşağın diğer kolları %22 "hayalet" | Tombul çizgi (15/100), yuvarlatılmış dolu ok başı |
| `components/nav/NavMap.tsx` | Hız rozeti daire 66; hap/Ortala gölge `elevation 4`; `Puck` + yön konisi; `nav-route-casing` koyu mavi | Hız rozeti köşeli kare 72; puk etrafında hafif parıltı; rota altında %22 geniş parıltı çizgisi | Hız rozeti daire + 4 dp teal halka; Ortala turuncu |
| Kapı numarası çipleri (`nav-housenumbers`) | Beyaz çip, gri kenar | Koyu çip | Beyaz hap, turuncu kenar |
| Yazı tipi | Inter TTF (`expo-font`, OFL) ya da Roboto | Barlow Condensed + Inter TTF | Nunito TTF |
| Tahmini iş | **M** | **M** | **S–M** |

**Harita stili.** Gündüz paleti `infra/map/scripts/generate-styles.mjs` içindeki `NAV_DAY` nesnesinde değişir: `bg`, `land.residential`, `land.park` / `parkLine`, `water`, `building` / `buildingLine`, `road.fill.{primary,secondary,tertiary,minor,service}`, `road.case.{…}`, `rail`, `text.{road,halo,house,poi}`. `tokens.json`'daki `map.day.roadFill.minor` hem `minor` hem `tertiary` / `service` için kullanılır. Gece için öneri: `courier-nav-night` stilini formülden (`lib/style-colors.mjs` → `nightLayers`) değil, `map.night` değerlerinden kurulan elle bir `NAV_NIGHT` paletinden üretmek. Uygulamadaki eski yol (`navigation/mapStyle.ts` → `nightColor`) yalnızca `tenant.mapStyles` olmayan eski kiracılar içindir; o da aynı paletle değiştirilebilir ya da olduğu gibi bırakılabilir. Harita etiket yazı tipi (Noto Sans glyph, ADR-0018) hiçbir varyantta değişmez. Kapı numarası çipi için sprite'a 9 dilimli bir `chip` görseli eklenir ve sembol katmanında `icon-text-fit: both` kullanılır (sprite bizim, ücretsiz).

**Performans (düşük segment Android).**
- Gölgeler yalnızca `elevation` (2-12) ile; Android'de ucuz. iOS gölgesi (`shadow*`) sadece iOS'ta etkili.
- **Canlı bulanıklık (blur) yok.** B'nin "cam" görünümü %94 opak renk + 1 dp kenarla elde edilir. `@react-native-community/blur` ya da `expo-blur` Android'de her karede harita görüntüsünü yeniden örnekler ve 30 fps harita akışını düşürür; kullanılmamalı.
- B'nin rota parıltısı ikinci bir `line` katmanıdır (aynı GeoJSON kaynağı, `line-opacity 0.22`); ihmal edilebilir. Puk parıltısı ve yön konisi ekran uzayında tek bir SVG'dir (zaten `Puck`).
- Yazı tipleri uygulamaya gömülür (her ağırlık ~80-300 KB). Yalnızca kullanılan 2-3 ağırlık eklenmeli.

## Riskler ve açık noktalar

- **B gündüz haritası koyu değil.** Brief "koyu harita" diyordu; ama koyu harita güneşte okunmuyor, bu yüzden B'de gündüz harita açık "gümüş", koyu olan yalnızca paneller. Gece harita koyu. Seçilen B v1 bu yaklaşımı resmileştirir ([`secim.md`](secim.md)).
- **B'de koyu panel camda yansır.** Kontrast yüksek, ama doğrudan güneşte koyu yüzeyde parmak izi ve yansıma daha çok görünür. B v1, gündüz panelleri tam opak yapar ve tüm metni ≥ 7:1'e çeker.
- **C rotası teal**, park yeşiliyle karışabilir; koyu kenarlık bunu azaltır, ama sahada denenmeli.
- Ekrandaki harita bir benzetimdir: perspektif CSS 3D ile yaklaşık, yol düzeni kurgusaldır (sokak adları referans ekran görüntüsünden).
