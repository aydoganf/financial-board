# Raporlar — Tüm Zamanlar — Design Spec
Date: 2026-10-12

## Overview

Topbar'a `📈 Raporlar` dropdown menüsü eklenir. İçindeki tek link **Tüm Zamanlar**, sistemde kayıtlı tüm aylık verilerin gelir, gider ve kalan değerlerini line chart olarak gösteren bir sayfa açar. Sayfanın sağ üstündeki yıl dropdown'ı ile grafik yalnızca seçili yıla göre yeniden çizilir.

Grafik, mevcut pasta grafikler gibi elle yazılmış inline SVG ile çizilir. Harici kütüphane eklenmez — "bağımlılık yok, build yok" ilkesi korunur.

## Topbar Değişikliği

- `📁 Veri` ile `🏦 Birikimler` arasına yeni bir `.data-menu-wrap` eklenir:
  - Buton: `#btn-reports`, `class="btn-secondary"`, içerik `📈 <span class="btn-label">Raporlar</span>`
  - Panel: `#reports-menu-panel`, `class="data-menu-panel"`
  - Panel öğesi: `<button class="data-menu-item">📈 Tüm Zamanlar</button>` → `openAllTimeReport()` + menüyü kapat
- Mobilde (≤640px) `.btn-label` gizlendiği için yalnızca ikon görünür (mevcut kural, ek CSS gerekmez).
- Menü davranışı:
  - `toggleReportsMenu()` paneli açar/kapatır; açılırken Veri menüsünü kapatır. `toggleDataMenu()` da açılırken Raporlar menüsünü kapatır.
  - Mevcut document click handler'ı (`!e.target.closest('.data-menu-wrap')`) her iki menüyü de kapatır. Bir menünün butonuna tıklamak diğer menüyü kapatmaz çünkü ikisi de `.data-menu-wrap` içinde — bu yüzden toggle fonksiyonlarındaki karşılıklı kapatma gereklidir.

## Navigasyon / State

Yeni global değişkenler:
- `let activeReportView = null;` — `null` veya `'all-time'`
- `let reportYear = 'all';` — `'all'` veya yıl sayısı (number). Sayfadan çıkıp dönünce korunur, sayfa yenilenince sıfırlanır.

Kurallar:
- `openAllTimeReport()`: `activeReportView = 'all-time'`; `activeMonth = null`, `activeYearView = null`, `activePackagesView = false`; `editingState = null`; `closeDrawer()`; `renderSidebar()` + `renderContent()`.
- `selectMonth`, `selectYearView`, `openPackages` → `activeReportView = null` yapar.
- `renderContent()` içinde öncelik sırası: `activePackagesView` → `activeReportView` → `activeYearView` → `activeMonth`.
- Rapor sayfasındayken sidebar'da hiçbir ay aktif görünmez (`activeMonth = null` olduğu için kendiliğinden sağlanır).
- Eğer `reportYear` olarak seçili yılın verisi artık yoksa (ör. yıl temizlendi), render sırasında `reportYear = 'all'`a düşülür.

## Sayfa Düzeni

```
┌──────────────────────────────────────────────────────┐
│ Tüm Zamanlar                       [Tüm Yıllar ▾]    │  ← h2, flex space-between
├──────────────────────────────────────────────────────┤
│ section-card                                         │
│  ● Gelir  ● Gider  ● Kalan            (legend)       │
│  ┌────────────────────────────────────────────────┐  │
│  │  SVG line chart                                │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

- Başlık: `<h2>` — mevcut ay/yıl sayfası başlık stiliyle aynı (`font-size:18px;margin-bottom:16px;display:flex;align-items:center;justify-content:space-between;`).
- Yıl dropdown'ı: `<select id="report-year-select">`, klon dropdown'ıyla aynı inline stil. Seçenekler:
  - `Tüm Yıllar` (value `all`) — varsayılan
  - Verisi olan yıllar, en yeniden en eskiye
  - `onchange` → `reportYear` güncellenir, `renderContent()`.
- Grafik kartı: `class="section-card"`, `padding:16px`. Üstte legend (renkli kare + ad), altında SVG.
- Veri yoksa grafik yerine kart içinde: `Henüz kayıtlı aylık veri yok.` (`color:var(--text-muted)`). Bu durumda yıl dropdown'ı gösterilmez.

## Veri

`getReportSeries(year)` → `[{ key, label, income, expense, remaining }]`:
- `state.months` anahtarları alınır; yalnızca **dolu** aylar (`incomes.length || expenses.length`) dahil edilir.
- `year !== 'all'` ise `key.startsWith(\`${year}-\`)` filtresi uygulanır.
- Anahtara göre kronolojik sıralanır (`localeCompare`).
- `income` / `expense` hesabı yıl özetiyle aynıdır: `reduce((s, i) => s + (i.amount || 0), 0)`; `remaining = income - expense`.
- `label`: kısa ay adı + iki haneli yıl, ör. `Oca 25`. Kısa adlar: `MONTHS[m].slice(0, 3)` (Oca, Şub, Mar, Nis, May, Haz, Tem, Ağu, Eyl, Eki, Kas, Ara).

Verisi olan yıllar listesi: dolu ayların anahtarlarından yıl çıkarılır, tekilleştirilir, azalan sıralanır.

## SVG Çizimi

`renderLineChart(series)` → SVG string.

- Boyut: `viewBox="0 0 W H"` (W=800, H=320), `width="100%"`, `preserveAspectRatio="none"` **kullanılmaz** (metin bozulmasın diye); SVG `width:100%;height:auto` ile ölçeklenir.
- İç boşluklar: sol ~64px (Y etiketleri), sağ ~16px, üst ~12px, alt ~32px (X etiketleri).
- Y ölçeği:
  - `min = Math.min(0, ...tüm değerler)`, `max = Math.max(0, ...tüm değerler)`; `min === max` ise `max = min + 1`.
  - "Güzel" adım: ~5 aralık hedeflenir; adım 1/2/5 × 10ⁿ'e yuvarlanır, min/max adıma göre genişletilir.
  - Yatay kılavuz çizgileri her adımda (`var(--border)`), **0 çizgisi** daha koyu (`var(--text-muted)`).
  - Y etiketleri kısaltılmış: `fmtShort(n)` → `₺1,2M`, `₺12,5B`, `₺850` (tr-TR, `B`=bin, `M`=milyon); negatifler `-₺…`.
- X ölçeği:
  - Tek nokta varsa yatay ortada çizilir.
  - Birden fazla noktada eşit aralıklı: `x_i = left + i * (plotW / (n - 1))`.
  - Etiket seyreltme: en fazla ~12 etiket; `step = Math.ceil(n / 12)`, `i % step === 0` olan etiketler çizilir.
- Seriler: Gelir `var(--income)`, Gider `var(--expense)`, Kalan `var(--remaining)`. Her biri `<polyline fill="none" stroke-width="2">` + her noktada `r=3` daire. (SVG `stroke`/`fill` attribute'larında CSS değişkeni çalışmadığı için renkler `style="stroke:var(--income)"` ile verilir.)

### Etkileşim

- Her ay için tam yükseklikte, görünmez bir hit alanı (`<rect fill="transparent">`), genişliği komşu noktalar arası mesafe kadar; `data-key`, `data-idx` taşır.
- **Hover (masaüstü):** hit alanının üzerine gelince
  - o x konumunda dikey kılavuz çizgi gösterilir,
  - grafik kartının içinde, absolute konumlu bir HTML tooltip (`#report-tooltip`) gösterilir: ay etiketi (uzun ad, ör. `Mart 2025`) + Gelir / Gider / Kalan değerleri `fmt()` ile, seri rengiyle. Kalan negatifse `var(--expense)` rengi.
  - Tooltip, imlecin kart kenarından taşmayacak şekilde sola/sağa yerleştirilir.
  - `mouseleave` ile tooltip ve kılavuz gizlenir.
- **Tıklama / dokunma:** `selectMonth(key)` → o ayın sayfası açılır (mobilde de doğrudan aya gider).
- Hit alanlarında `cursor:pointer`.
- Event'ler render sonrası `#report-chart` içindeki hit alanlarına `addEventListener` ile bağlanır (inline handler yerine; koordinat hesabı için event nesnesi gerekiyor).

## CSS

Yeni sınıflar `/* Reports */` başlığı altında, mobil media query'den önce:
- `.report-chart-wrap { position: relative; }`
- `.report-legend { display:flex; gap:16px; font-size:12px; margin-bottom:8px; }`
- `.report-tooltip { position:absolute; pointer-events:none; background:var(--surface); border:1px solid var(--border); border-radius:6px; box-shadow:0 4px 16px rgba(0,0,0,.1); padding:8px 10px; font-size:12px; display:none; white-space:nowrap; z-index:10; }`
- Mobil: ek kural gerekmez; SVG genişliğe göre ölçeklenir. (Çok dar ekranda metinler küçülür — kabul edilen davranış.)

## Kapsam Dışı (YAGNI)

- Grafik altında özet kartları veya tablo yok.
- Seri gizle/göster (legend tıklama) yok.
- Animasyon yok.
- Raporlar menüsüne başka rapor eklenmez; menü yapısı ileride ek link almaya hazırdır.

## Test / Doğrulama

Projede test altyapısı yok; tarayıcıda manuel doğrulama:
1. Raporlar menüsü açılıyor/kapanıyor; Veri menüsüyle karşılıklı kapanma; dışarı tıklayınca kapanma.
2. Tüm Zamanlar: birden çok yıla yayılmış veriyle 3 çizgi doğru çiziliyor, X ekseninde boş aylar atlanıyor.
3. Yıl dropdown'ı: yıl seçince yalnızca o yılın ayları; Tüm Yıllar'a dönünce hepsi.
4. Negatif kalanlı ay: Y ekseni sıfırın altına iniyor, 0 çizgisi belirgin.
5. Tek aylık veri: nokta ortada, hata yok.
6. Hiç veri yok: boş mesajı, dropdown yok.
7. Hover tooltip doğru değerleri gösteriyor ve kenarlardan taşmıyor; tıklayınca doğru ay açılıyor ve sidebar'da o ay aktif.
8. Sayfadan ayrılıp (ay/yıl özeti/birikimler) geri dönünce seçili yıl korunuyor.
9. 375px mobil görünüm: yalnızca 📈 ikonu, grafik taşmadan ölçekleniyor.
