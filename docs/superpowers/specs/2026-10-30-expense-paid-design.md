# Gider "Ödendi" İşareti — Design Spec
Date: 2026-10-30

## Overview

Gider kalemlerinde ✏ / 🗑 aksiyonlarının soluna bir **✓** butonu eklenir. Kalem ödendi olarak işaretlenince üstü çizili ve soluk görünür, listenin en altına taşınır. İşaret geri alınınca kalem ödenmemişlerin en sonuna taşınır.

İşaret **yalnızca görseldir**: Toplam Gider, Kalan, yıl özeti, raporlar, pasta grafikler ve birikim paketleri değişmez.

## Veri Modeli

- Gider kalemine opsiyonel `paid: boolean` alanı eklenir. Alan yoksa/false ise kalem ödenmemiştir — eski veri ve JSON yedekleri migration gerektirmez.
- Gelir kalemlerinde `paid` kullanılmaz.

## Sıralama Kuralı (invariant)

`expenses` dizisinde her zaman **önce ödenmemişler, sonra ödenmişler** yer alır. Sıra dizinin kendisinde tutulur (sürükle-bırak sırası gibi); render'da ayrıca sıralama yapılmaz.

- İşaretle: kalem diziden çıkarılır, sonuna eklenir, `paid = true`.
- Geri al: kalem diziden çıkarılır, `paid = false`, ödenmemiş kalem sayısı kadar index'e (ilk ödenmiş kalemin önüne) eklenir.

Invariant'ı koruyan ek kurallar:
- **Yeni gider ekleme:** yeni kalem dizinin sonuna değil, ödenmemişlerin sonuna eklenir.
- **Sürükle-bırak:** yalnızca aynı ödeme grubu içinde (ödenmemiş↔ödenmemiş, ödenmiş↔ödenmiş). Gruplar arası bırakma engellenir (`dropEffect = 'none'`, drop yok sayılır).
- **Ay klonlama** ve **periyot ile çoklu ay ekleme:** kopyalanan kalemler `paid: false` ile oluşturulur.
- **Düzenleme (kaydet):** `paid` değeri korunur.

## UI

- Buton: `.item-actions` içinde, ✏'nın solunda, yalnızca `kind === 'expense'` için.
  - Ödenmemiş: mevcut aksiyon butonu stili, `title="Ödendi olarak işaretle"`.
  - Ödenmiş: `.paid-btn.active` — yeşil arka plan (`var(--income)`), beyaz ✓, `title="Ödenmedi olarak işaretle"`.
- Ödenmiş satır: `.item-row.paid` → `.type-badge`, `.item-name`, `.item-amount` için `text-decoration: line-through`, satır `opacity: .55`. Not satırı (`.item-note.paid`) da soluk ve üstü çizili.
- Mobil: üç buton 320px ekrana sığsın diye satır sıkılaştırılır — `.item-row` gap 6px / padding 8px 12px, `.item-amount` sabit genişlik yerine auto, aksiyon butonu padding 3px 6px, uzun tip etiketi `…` ile kısalır.

## Kapsam Dışı

- "Ödenecek" tutar özeti yok.
- Gelir kalemleri için "tahsil edildi" işareti yok.
- Ödeme tarihi tutulmaz.

## Doğrulama (manuel, tarayıcıda)

1. ✓ ile işaretle → kalem en alta iner, üstü çizili; toplamlar değişmez.
2. Birden çok kalem işaretle → ödenmişler işaretlenme sırasıyla altta.
3. Geri al → kalem ödenmemişlerin sonuna, ilk ödenmişin üstüne gider.
4. Ödenmişler varken "+ Yeni Gider Ekle" → yeni satır ödenmemişlerin sonunda; İptal edince kaybolur, kaydedince orada kalır.
5. Sürükle-bırak: grup içinde çalışır, gruplar arası çalışmaz.
6. Ödenmiş kalemi düzenle → `paid` korunur.
7. Ödenmiş kalemli ayı boş bir aya klonla → kopyalar ödenmemiş.
8. Sayfa yenile → durum korunur (localStorage).
9. 375px mobil: üç buton sığar, taşma yok.
