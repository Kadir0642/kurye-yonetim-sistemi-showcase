# Yönetim paneli marka seçimi

- **Tarih:** 2026-11-07
- **Karar veren:** ürün sahibi
- **Seçilen:** **P2 · Koyu kenar çubuğu + açık içerik** (maket: [`index.html#harita/2`](index.html), açıklama: [`README.md`](README.md))
- **Kayıt:** [ADR-0027](../../kararlar/0027-marka-kimligi-ve-navigasyon-gorsel-dili.md) → "Uygulama notları: Panel" (yeni ADR açılmadı)
- **Uygulama:** `feature/panel-marka-maket`, ayrıntı: `../panel-uygulama-notu.md`

## Neden P2

1. Neon `#C8FF3A` açık yüzeyde kullanılamıyor (beyazda 1,0:1); panelde neonun yaşayabildiği tek kalıcı yer koyu kenar çubuğu. İçerik kurye uygulamasının onaylı A2 açık dilinde kalır.
2. Gün boyu ekrana bakan menajer için geniş, sabit koyu alan toplam parlaklığı düşürür; okunan tablo ve formlar ≥ 7:1.
3. Koyu harita kromu ile kurye navigasyonu ve panel haritası aynı bileşen dilini konuşur.
4. Efor orta (M); P1'e dönüş tek tema değişikliği.

## Kapsam kuralları (seçimle birlikte)

- Kenar çubuğu `#111318`; etkin öğe `#22262E` zemin + 4 px neon gösterge + neon ikon. Neon başka yalnızca logo karosunda, koyu yüzeydeki odak halkasında ve koyu yüzeyli düğmede.
- Üst çubuk beyaz; içerik zemini `#F3F4F1`, beyaz kartlar, birincil `#3F6212` + beyaz yazı.
- Durumlar %10 ton + koyu yazı + ikon (yalnızca renge dayanmaz).
- Harita karo stilleri değişmez; değişen harita kontrolleri (koyu krom) ve işaret renkleri.
- P3'ün yoğun görünümü şimdilik yok; ileride "Yoğunluk: Rahat / Sıkı" kullanıcı seçeneği olarak ayrı iş.

## Reddedilenler

- **P1 · Açık A2 birebir:** en ucuz ve güvenli, ama marka panelde neredeyse görünmüyor.
- **P3 · Yoğun operasyon:** bilgi yoğunluğu en yüksek, ama ayrı bir kimlik değil; efor L. Yoğunluk sonradan seçenek olarak eklenebilir.
