# Navigasyon tasarımı: seçim kaydı

**Tarih:** 2026-11-06 · **Karar veren:** ürün sahibi · **Durum:** kabul edildi (uygulama henüz yapılmadı)

## Karar

Kurye navigasyon ekranı için **B · Koyu / premium** varyantı seçildi: siyaha yakın krom, neon yeşil vurgu (`#C8FF3A`), beyaz metin, dar ve kalın rakamlar (Barlow Condensed), geometrik manevra okları. A · Sade (Google Maps benzeri) yakın ikinci oldu.

Aynı görsel dil yalnızca navigasyon ekranında kalmayacak, **ürünün kurumsal marka dili** olacak. Sistem başka kurye ekiplerine çok kiracılı bir hizmet (SaaS) olarak satılacak: kurye telefon uygulaması + yönetici paneli. Marka belirteçleri: [`../marka/`](../marka/).

## Neden B

- **Ayırt edici kimlik.** A, Google Haritalar'a çok benziyor; satılacak bir ürün için "bir başka harita uygulaması" izlenimi veriyor. B ilk bakışta tanınır ve "premium / profesyonel araç" algısı yaratıyor.
- **Okunurluk sayıları güçlü.** Koyu kartta beyaz metin 18,6:1, neon ok 15,8:1. Mesafe 84 px dar rakamla tek satırda kalıyor.
- **Gece en iyisi.** Kuryeler akşam yoğun çalışıyor; B gece en az parlamayı ve göz yorgunluğunu veriyor.
- **Tek vurgu rengi her yüzeye taşınabiliyor.** Neon dolgu + siyah metin (16,5:1) hem koyu hem açık yüzeyde çalışıyor; panel için koyu bir "on-light" tonu (`#3F6212`, beyazda 7,1:1) türetilebiliyor.

## Güneş ışığı riski ve cevabı

Koyu krom doğrudan güneşte camdaki yansımayı daha çok gösterir; koyu harita ise güneşte okunmaz. Seçilen çözüm (**B v1**, [`b-v1.html`](b-v1.html)): **açık harita + koyu krom (melez)**.

| Karar | Değer |
|---|---|
| Gündüz harita | Açık "gümüş" (zemin `#ECEEF1`, yollar beyaz, kenarlıklar gri) |
| Gündüz krom | %100 opak `#111318`, 1 px açık kenar, saydamlık yok |
| Krom üzerindeki her metin | ≥ 7:1 (beyaz 18,6 · neon 15,8 · ikincil gri `#B4BAC4` 9,5 · varış saati camgöbeği 10,3 · Çıkış kırmızısı `#FF7A7A` 7,4) |
| Gündüz rota | Koyu mavi `#006BB8` + lacivert kenar `#002E4F` + hafif parıltı; zeminle 4,6:1. Neon rota açık haritada görünmez (1,0:1), bu yüzden rota neon değil. |
| Gece | Koyu harita, krom %94 opak, neon bir ton kısık (`#B8F03A`), rota camgöbeği `#22D3EE` |

Elenen seçenekler: her şey koyu (gündüz harita okunmuyor), gündüz açık krom (marka gündüz kayboluyor). Sahada doğrulama gerekiyor: öğle güneşinde, gidon tutucuda, gerçek bir düşük segment Android telefonla.

## Değişmeyenler

Bilgi seti ve kurye kuralları aynı: çok büyük mesafe ve ok, ≥ 56 dp hedefler (alt düğmeler 68×64, Ortala 64), Çıkış onay sayfası, ses öncelikli, puk alt üçte birde, yön-yukarı eğik kamera, otomatik gece. Harita etiket yazı tipi (Noto Sans, ADR-0018) değişmez. Canlı bulanıklık (blur) kullanılmaz.

## Sonraki adımlar (ayrı görevler)

1. Kurye uygulaması: `NavTheme` + `NavBanner` / `NavBottomBar` / `ManeuverArrow` / `NavMap` stilleri, Inter + Barlow Condensed TTF (OFL, `expo-font`). Değerler: `../marka/tokens.json` → `navigation`. Tahmini iş: M.
2. Harita: `generate-styles.mjs` içinde `NAV_DAY` paletini gümüş palete çevirmek, gece için formül yerine elle `NAV_NIGHT` paleti (`navigation.maplibre.night`). Rota parıltısı için yeni `nav-route-glow` katmanı, kapı numarası çipi için sprite'a `chip-dark`.
3. Uygulamanın diğer ekranları ve yönetici paneli: marka belirteçleriyle ayrı bir iterasyonda (bkz. `../marka/README.md`).
4. Karar [ADR-0027](../../kararlar/0027-marka-kimligi-ve-navigasyon-gorsel-dili.md) olarak kaydedildi (gündüz rota koyu mavi ve nane yeşili başarı rengi ürün sahibince onaylandı).

## Kaynaklar

- Üç varyantın karşılaştırması: [`index.html`](index.html), [`README.md`](README.md), [`tokens.json`](tokens.json)
- Seçilen tasarımın son hali: [`b-v1.html`](b-v1.html)
- Marka belirteçleri: [`../marka/tokens.json`](../marka/tokens.json), [`../marka/README.md`](../marka/README.md)
