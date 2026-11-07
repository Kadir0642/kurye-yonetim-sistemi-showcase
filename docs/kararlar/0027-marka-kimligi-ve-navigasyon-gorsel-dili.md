# ADR-0027: Marka kimliği ve navigasyon görsel dili: koyu krom + neon yeşil, gündüz açık harita, koyu krom üzerinde metin ≥ 7:1

- **Durum:** Kabul edildi (ürün sahibi, 2026-11-06; uygulama ayrı görev)
- **Tarih:** 2026-11-06
- **İlgili:** [ADR-0015](0015-uygulama-ici-navigasyon.md) (uygulama içi navigasyon), [ADR-0018](0018-harita-karo-ve-stil-sunumu.md) (stiller, glyph, sprite), [ADR-0004](0004-cok-kiracili-mimari.md) (çok kiracılı), [`docs/tasarim/navigasyon/secim.md`](../tasarim/navigasyon/secim.md), [`docs/tasarim/marka/tokens.json`](../tasarim/marka/tokens.json)

## Bağlam

Uygulama içi navigasyon ekranı (ADR-0015) çalışıyor, ama ürün sahibi görünüşü "kaba / keskin" buldu. Ölçülen nedenler: koyu kahverengi-gri yol kenarlıkları (`#9c958a`), tam doygun yeşil bant ve sert gölge, ikon yerine metin glifleri (`☰ ✕ ◎ ⤢`), her yerde 900 kalınlık ve gündüz stilinden formülle türetilen, ara sokakları parlak gri çizgilere çeviren gece haritası.

Ürünün henüz bir marka dili yok: kurye uygulaması teal `#0F766E`, panel Mantine `indigo` kullanıyor. Sistem başka kurye ekiplerine çok kiracılı hizmet (SaaS) olarak satılacak (kurye uygulaması + yönetici paneli); tanınır ve tutarlı bir kimlik gerekiyor.

Kurye kısıtları değişmez: telefon motosiklet tutucusunda, bir bakışta okunur; çok büyük mesafe ve ok, güneşte yüksek kontrast, otomatik gece, ≥ 56 dp eldiven hedefi, kasıtlı çıkış, ses öncelikli, puk alt üçte birde, yön-yukarı eğik kamera.

Uygulama kodu değiştirilmeden üç görsel varyant statik maket olarak hazırlandı ve karşılaştırıldı: [`docs/tasarim/navigasyon/index.html`](../tasarim/navigasyon/index.html) (README ve `tokens.json` ile).

## Karar

1. **Varyant B (Koyu / premium) seçildi ve ürünün kurumsal marka dili olur.** Siyaha yakın krom (`#111318` / `#0A0C0F`), beyaz metin, tek vurgu **neon yeşil `#C8FF3A`** (gece `#B8F03A`), açık zeminde metin/bağlantı için `#3F6212` (7,1:1), ikon/kenar için `#4D7C0F`. Yazı tipleri Inter (arayüz) + Barlow Condensed (büyük rakamlar), ikisi de OFL ve uygulamaya gömülür. Arayüz ikonları Lucide tarzı çizgi (2,25); manevra okları geometrik kalın çizgi, dolu sivri ok başı, kavşağın diğer kolları %22 "hayalet".
2. **Güneş ışığı kuralı: koyu krom üzerinde her metin ≥ 7:1; gündüz harita açıktır.** Gündüz harita açık "gümüş" palet (zemin `#ECEEF1`, beyaz yollar, gri kenarlıklar), krom %100 opak (saydamlık yok) + 1 px açık kenar. Gece harita koyu, krom %94 opak. Ölçülen değerler: beyaz 18,6:1, neon 15,8:1, ikincil gri `#B4BAC4` 9,5:1, Çıkış kırmızısı `#FF7A7A` 7,4:1.
3. **Gündüz rota koyu mavi** `#006BB8` + lacivert kenar `#002E4F` + hafif parıltı (açık haritada 4,6:1); neon rota açık haritada görünmez (1,0:1). Gece rota camgöbeği `#22D3EE`.
4. **Başarı rengi nane yeşili `#3DDC97`** (açık zeminde `#047857`): "tamamlandı" anlamı marka neonuyla karışmaz. Uyarı `#FBBF24`, tehlike `#FF7A7A` / dolu `#FF4D4F` + siyah metin, bilgi `#22D3EE`.
5. **Gece haritası formülle değil, elle ayarlı palet** (`NAV_NIGHT`) ile üretilir; değerler MapLibre katman kimliğiyle tanımlıdır.
6. **Canlı bulanıklık (blur) kullanılmaz**; "cam" görünümü opak yüzey + ince kenarla. Gölgeler yalnızca Android `elevation`.
7. **Tek kaynak:** [`docs/tasarim/marka/tokens.json`](../tasarim/marka/tokens.json). `navigation.ui` alanları `NavTheme` ile aynı adlı, `navigation.maplibre` alanları `courier-nav-*.json` / `NavMap.tsx` katman kimlikleriyle aynı. Son maket: [`docs/tasarim/navigasyon/b-v1.html`](../tasarim/navigasyon/b-v1.html).
8. **Kapsam:** önce kurye uygulaması (navigasyon ekranı, sonra diğer ekranlar). Yönetici paneli **ayrı bir görevde** aynı belirteçlerle geçer (Mantine'de özel 10 tonlu `neon` paleti, koyu kenar çubuğu, açık içerik alanı; bkz. [`docs/tasarim/marka/README.md`](../tasarim/marka/README.md)). Kiracıya özel logo / renk (white-label) bu kararın kapsamı dışında.

## Sonuçlar

- (+) Ayırt edici, satılabilir bir kimlik; kurye uygulaması ve panel aynı dili konuşur.
- (+) Okunurluk sayısal bir kurala bağlanır (≥ 7:1); yeni ekranlar ve kiracı özelleştirmeleri bu kurala göre denetlenebilir.
- (+) Gece en az parlama; elle gece paleti ara sokakların göz alıcı parlaklığını giderir.
- (+) Uygulama maliyeti sınırlı (M): `theme.ts`, `NavBanner`, `NavBottomBar`, `ManeuverArrow`, `NavMap` stilleri, iki TTF, `generate-styles.mjs` paletleri; yeni bir `nav-route-glow` katmanı ve sprite'a koyu çip görseli.
- (−) Koyu krom doğrudan güneşte camda yansımayı daha çok gösterir; tam opak krom ve ≥ 7:1 kural riski azaltır ama sahada (öğle güneşi, gidon tutucu, düşük segment Android) doğrulanmalı.
- (−) İki yazı tipi uygulama boyutunu artırır (kullanılan 2-3 ağırlık, birkaç yüz KB).
- (−) Mevcut teal / yeşil kullanılan ekranlar geçiş süresince tutarsız görünür; geçiş ekran ekran yapılır.
- (−) Harita etiket yazı tipi Noto Sans kalır (ADR-0018 glyph seti); arayüz yazı tipiyle birebir aynı değildir.

## Alternatifler

- **A · Sade (Google Maps benzeri):** beyaz yüzen kart, mavi ok karosu, açık yol kenarlıkları. Güneşte en güvenli (16,1:1) ve ucuz; yakın ikinci oldu. Reddedildi: Google Haritalar'a fazla benziyor, satılacak ürüne ayırt edici bir kimlik vermiyor; gece B kadar iyi değil.
- **C · Marka / canlı (teal + turuncu, Nunito, hap biçimli kontroller):** mevcut ekrana en yakın, en ucuz geçiş (S–M). Reddedildi: mevcut teal'ı sürdürür ama "premium araç" algısı vermiyor; teal rota park yeşiliyle karışma riski taşıyor.
- **B'nin her şey koyu hali (gündüz de koyu harita):** güneşte harita okunmuyor; yalnızca gece modu olarak kaldı.
- **B'nin gündüz açık krom hali:** güneşte güvenli ama neon beyazda kullanılamıyor (1,0:1); marka gündüz kaybolur.

## Ek (2026-11-07): uygulama içi ekranlar açık

Ürün sahibi, kurye uygulamasının navigasyon dışındaki ekranları için iki tur maketi (`docs/tasarim/kurye-app/`) değerlendirdi ve şunu kararlaştırdı:

1. **Koyu krom yalnızca navigasyon ekranı ve harita kromu içindir** (bu ADR'nin 1-7. maddeleri onlar için aynen geçerli).
2. Giriş, onboarding, listeler, paket ayrıntısı, profil ve devir ekranları **açık** tema kullanır: maket **A2 "Nötr gri + beyaz kartlar"** (zemin `#F3F4F1`, beyaz kart, hafif gölge). Birincil eylem marka yeşili `#3F6212` + beyaz yazı (7,1:1); neon açık yüzeyde dolgu ya da metin olmaz, yalnızca koyu öğede ya da `#4D7C0F` kenarlı küçük sinyal olarak görünür. Durumlar %10 renk tonu + koyu yazı + ikonla gösterilir. Açık zeminde gövde metni de ≥ 7:1.
3. Belirteçler `tokens.json → courierApp` (sürüm 1.1.0); uygulama: `docs/tasarim/kurye-app-uygulama-notu.md`.
4. Uygulama ikonu "K + ok" işareti (koyu arka plan + neon ön plan, uyarlanabilir); açılış ekranı açık zemin + koyu daire içinde neon işaret.
5. Koyu mod şimdilik yok; sistem koyu temasını izleme ayrı görev.

## Uygulama notları: Panel (2026-11-07)

Ürün sahibi yönetici paneli için üç maket ([`docs/tasarim/panel/`](../tasarim/panel/README.md)) arasından **P2 "Koyu kenar çubuğu + açık içerik"** seçti ([seçim kaydı](../tasarim/panel/secim.md)). Bu ek, Karar 8'deki panel geçişini tamamlar; yeni ADR açılmadı.

1. **Kenar çubuğu koyu, içerik açık:** kenar çubuğu navigasyon kromu `#111318`; içerik kurye uygulamasıyla aynı A2 (zemin `#F3F4F1`, beyaz kart, birincil `#3F6212` + beyaz yazı). Üst çubuk beyaz.
2. **Neon yalnızca koyu yüzeyde:** etkin menü öğesinde 4 px gösterge + ikon, logo karosu, koyu yüzeydeki odak halkası. Marka README'sindeki ilk taslağın aksine birincil düğme neon dolgu **değildir** (A2 kararıyla uyum).
3. **Mantine eşlemesi:** 10 tonlu `brand` paleti (7 = `#3F6212`), `primaryShade 7`; Mantine'in yerleşik renk adları marka anlam paletlerine eşlenir (green/teal → başarı, orange/yellow → uyarı, red → tehlike, blue/cyan/violet/grape → bilgi, gray → nötr, indigo/lime → marka). Her palette 9 / 1 (ton metni / ton zemini) ve beyaz / 7 (dolgu) ≥ 4,5:1.
4. **Paket durum renkleri** SPEC §5 tablosu yerine kurye uygulamasıyla aynı tonları kullanır (`courierApp.packageStatus`: Beklemede / Havuzda / İptal nötr, Atandı uyarı, Yolda bilgi, Teslim edildi başarı, Teslim edilemedi tehlike) ve her durum ayrı ikonla ayrılır.
5. **Harita:** karo stilleri değişmez. Harita kontrolleri ve harita üstü kartlar navigasyondaki koyu krom; kurye rota paleti aynı kalır; alış / teslim işaretleri kehribar / camgöbeği.
6. **Belirteçler** `tokens.json → adminPanel` (sürüm 1.2.0); uygulama `apps/admin-web/src/theme/` (tek kaynak, birim testle eşitlenir; `src/theme/` dışında ham hex yasak, testle taranır). Ayrıntı: `docs/tasarim/panel-uygulama-notu.md`.
