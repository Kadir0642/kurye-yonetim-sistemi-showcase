# ADR-0027: Marka kimliği ve navigasyon görsel dili: koyu krom + neon yeşil, gündüz açık harita, koyu krom üzerinde metin ≥ 7:1

- **Durum:** Kabul edildi (ürün sahibi, 2026-11-06; uygulama ayrı görev)
- **Tarih:** 2026-11-06
- **İlgili:** [ADR-0015](0015-uygulama-ici-navigasyon.md) (uygulama içi navigasyon), [ADR-0018](0018-harita-karo-ve-stil-sunumu.md) (stiller, glyph, sprite), [ADR-0004](0004-cok-kiracili-mimari.md) (çok kiracılı), `docs/tasarim/navigasyon/secim.md`, `docs/tasarim/marka/tokens.json`

## Bağlam

Uygulama içi navigasyon ekranı (ADR-0015) çalışıyor, ama ürün sahibi görünüşü "kaba / keskin" buldu. Ölçülen nedenler: koyu kahverengi-gri yol kenarlıkları (`#9c958a`), tam doygun yeşil bant ve sert gölge, ikon yerine metin glifleri (`☰ ✕ ◎ ⤢`), her yerde 900 kalınlık ve gündüz stilinden formülle türetilen, ara sokakları parlak gri çizgilere çeviren gece haritası.

Ürünün henüz bir marka dili yok: kurye uygulaması teal `#0F766E`, panel Mantine `indigo` kullanıyor. Sistem başka kurye ekiplerine çok kiracılı hizmet (SaaS) olarak satılacak (kurye uygulaması + yönetici paneli); tanınır ve tutarlı bir kimlik gerekiyor.

Kurye kısıtları değişmez: telefon motosiklet tutucusunda, bir bakışta okunur; çok büyük mesafe ve ok, güneşte yüksek kontrast, otomatik gece, ≥ 56 dp eldiven hedefi, kasıtlı çıkış, ses öncelikli, puk alt üçte birde, yön-yukarı eğik kamera.

Uygulama kodu değiştirilmeden üç görsel varyant statik maket olarak hazırlandı ve karşılaştırıldı: `docs/tasarim/navigasyon/index.html` (README ve `tokens.json` ile).

## Karar

1. **Varyant B (Koyu / premium) seçildi ve ürünün kurumsal marka dili olur.** Siyaha yakın krom (`#111318` / `#0A0C0F`), beyaz metin, tek vurgu **neon yeşil `#C8FF3A`** (gece `#B8F03A`), açık zeminde metin/bağlantı için `#3F6212` (7,1:1), ikon/kenar için `#4D7C0F`. Yazı tipleri Inter (arayüz) + Barlow Condensed (büyük rakamlar), ikisi de OFL ve uygulamaya gömülür. Arayüz ikonları Lucide tarzı çizgi (2,25); manevra okları geometrik kalın çizgi, dolu sivri ok başı, kavşağın diğer kolları %22 "hayalet".
2. **Güneş ışığı kuralı: koyu krom üzerinde her metin ≥ 7:1; gündüz harita açıktır.** Gündüz harita açık "gümüş" palet (zemin `#ECEEF1`, beyaz yollar, gri kenarlıklar), krom %100 opak (saydamlık yok) + 1 px açık kenar. Gece harita koyu, krom %94 opak. Ölçülen değerler: beyaz 18,6:1, neon 15,8:1, ikincil gri `#B4BAC4` 9,5:1, Çıkış kırmızısı `#FF7A7A` 7,4:1.
3. **Gündüz rota koyu mavi** `#006BB8` + lacivert kenar `#002E4F` + hafif parıltı (açık haritada 4,6:1); neon rota açık haritada görünmez (1,0:1). Gece rota camgöbeği `#22D3EE`.
4. **Başarı rengi nane yeşili `#3DDC97`** (açık zeminde `#047857`): "tamamlandı" anlamı marka neonuyla karışmaz. Uyarı `#FBBF24`, tehlike `#FF7A7A` / dolu `#FF4D4F` + siyah metin, bilgi `#22D3EE`.
5. **Gece haritası formülle değil, elle ayarlı palet** (`NAV_NIGHT`) ile üretilir; değerler MapLibre katman kimliğiyle tanımlıdır.
6. **Canlı bulanıklık (blur) kullanılmaz**; "cam" görünümü opak yüzey + ince kenarla. Gölgeler yalnızca Android `elevation`.
7. **Tek kaynak:** `docs/tasarim/marka/tokens.json`. `navigation.ui` alanları `NavTheme` ile aynı adlı, `navigation.maplibre` alanları `courier-nav-*.json` / `NavMap.tsx` katman kimlikleriyle aynı. Son maket: `docs/tasarim/navigasyon/b-v1.html`.
8. **Kapsam:** önce kurye uygulaması (navigasyon ekranı, sonra diğer ekranlar). Yönetici paneli **ayrı bir görevde** aynı belirteçlerle geçer (Mantine'de özel 10 tonlu `neon` paleti, koyu kenar çubuğu, açık içerik alanı; bkz. `docs/tasarim/marka/README.md`). Kiracıya özel logo / renk (white-label) bu kararın kapsamı dışında.

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
