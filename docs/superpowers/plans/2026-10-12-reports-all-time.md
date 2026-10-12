# Raporlar — Tüm Zamanlar Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Topbar'a `📈 Raporlar` menüsü ve içinde "Tüm Zamanlar" sayfası eklemek; sayfada tüm dolu ayların gelir/gider/kalan değerlerini yıl filtresiyle line chart olarak çizmek.

**Architecture:** Her şey `index.html` içinde. Mevcut görünüm kalıbı (`activePackagesView` gibi global bayrak + `renderContent()` dallanması) izlenir. Grafik, `renderPieChart` gibi elle yazılmış inline SVG string'i olarak üretilir; genişlik render anında kapsayıcıdan ölçülür ve pencere yeniden boyutlanınca yeniden çizilir. Etkileşim (tooltip, kılavuz çizgi, tıklayınca aya git) render sonrası `addEventListener` ile bağlanır.

**Tech Stack:** Vanilla HTML/CSS/JS, inline SVG. Bağımlılık yok, build yok.

**Spec:** `docs/superpowers/specs/2026-10-12-reports-all-time-design.md`

## Global Constraints

- Harici kütüphane/CDN eklenmez; tek dosya `index.html`.
- UI metinleri Türkçe: `Raporlar`, `Tüm Zamanlar`, `Tüm Yıllar`, `Gelir`, `Gider`, `Kalan`, `Henüz kayıtlı aylık veri yok.`
- Renkler mevcut CSS değişkenlerinden: `var(--income)`, `var(--expense)`, `var(--remaining)`, `var(--border)`, `var(--text-muted)`. SVG'de renkler `style="stroke:…"`/`style="fill:…"` ile verilir.
- X ekseninde yalnızca dolu aylar (en az bir gelir veya gider kalemi).
- Tutar biçimi: tooltip'te `fmt()`, Y ekseninde `fmtShort()` (`B` = bin, `M` = milyon).
- Commit mesajları conventional (`feat:`, `fix:` …) ve şu satırla biter: `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`

### Spec'ten bilinçli sapma

Spec, mobilde sabit `viewBox` (800×320) ölçeklenmesini ve metinlerin küçülmesini kabul ediyordu. 375px ekranda bu ~4px metin demek. Bunun yerine SVG genişliği render anında kapsayıcıdan ölçülür (`viewBox` genişliği = gerçek piksel genişliği), dar ekranda yükseklik 240px'e iner, X etiket sayısı genişliğe göre seyreltilir ve `resize` olayında yeniden çizilir. Davranış spec'le aynı, yalnızca okunabilirlik iyileşir.

## Doğrulama Yöntemi

Projede test altyapısı yok. Her task, uygulamanın tarayıcıda (`file://…/index.html`) açılıp konsolda çalıştırılan doğrulama ifadeleri ve görsel kontrol ile doğrulanır. Örnek veri seed'i (konsolda çalıştırılır, sonra sayfa yenilenir):

```js
localStorage.setItem('financial-board-v1', JSON.stringify({
  incomeTypes: [{ id: 'it1', name: 'Düzenli Maaş', color: '#2563eb' }],
  expenseTypes: [{ id: 'et1', name: 'Konut', color: '#dc2626' }],
  savingsPackages: [],
  months: {
    '2025-01': { incomes: [{ id: 'a', name: 'Maaş', amount: 50000, typeId: 'it1', note: '' }], expenses: [{ id: 'b', name: 'Kira', amount: 20000, typeId: 'et1', note: '' }] },
    '2025-02': { incomes: [{ id: 'c', name: 'Maaş', amount: 50000, typeId: 'it1', note: '' }], expenses: [{ id: 'd', name: 'Kira', amount: 65000, typeId: 'et1', note: '' }] },
    '2025-03': { incomes: [], expenses: [] },
    '2025-06': { incomes: [{ id: 'e', name: 'Maaş', amount: 55000, typeId: 'it1', note: '' }], expenses: [{ id: 'f', name: 'Kira', amount: 22000, typeId: 'et1', note: '' }] },
    '2026-01': { incomes: [{ id: 'g', name: 'Maaş', amount: 70000, typeId: 'it1', note: '' }], expenses: [{ id: 'h', name: 'Kira', amount: 30000, typeId: 'et1', note: '' }] },
  },
}));
```

Bu veride: 4 dolu ay (2025-03 boş → atlanır), 2025-02 kalanı negatif (−15.000), iki yıl (2026, 2025).

---

### Task 1: Raporlar menüsü, navigasyon ve grafik sayfası

**Files:**
- Modify: `index.html` — CSS (`/* Reports */` bloğu, mobil media query'den önce), topbar HTML (Veri menüsünden sonra), state değişkenleri (~satır 283), `selectMonth`/`selectYearView`/`openPackages`/`renderContent`, `toggleDataMenu` + document click handler, yeni `/* ── Reports ── */` JS bölümü (Savings Packages bölümünden önce).

**Interfaces:**
- Produces:
  - `let activeReportView` (`null | 'all-time'`), `let reportYear` (`'all' | number`)
  - `openAllTimeReport(): void`, `setReportYear(value: string): void`
  - `toggleReportsMenu(): void`, `closeReportsMenu(): void`
  - `getReportSeries(year: 'all'|number): Array<{ key, label, longLabel, income, expense, remaining }>`
  - `getReportYears(): number[]` (azalan)
  - `fmtShort(n: number): string`, `niceScale(min, max): { min, max, step }`
  - `REPORT_SERIES: Array<{ field, label, color }>`
  - `renderLineChart(series, width: number): string` — SVG; içinde `#report-chart`, `#report-guide` (gizli dikey çizgi) ve her ay için `rect.report-hit[data-idx][data-cx]`
  - `renderAllTimeReport(): void` — `#report-chart-wrap` içine SVG ve boş `#report-tooltip` koyar, sonra `bindReportChartEvents(series)` çağırır (Task 2'de tanımlanır; Task 1'de boş gövdeli olarak eklenir)

- [ ] **Step 1: CSS ekle** — `.data-menu-sep { … }` satırından sonra:

```css
    /* Reports */
    .report-legend { display: flex; gap: 16px; font-size: 12px; margin-bottom: 8px; color: var(--text); }
    .report-chart-wrap { position: relative; }
    .report-tooltip { position: absolute; pointer-events: none; background: var(--surface); border: 1px solid var(--border); border-radius: 6px; box-shadow: 0 4px 16px rgba(0,0,0,.1); padding: 8px 10px; font-size: 12px; display: none; white-space: nowrap; z-index: 10; }
```

- [ ] **Step 2: Topbar HTML** — `<input type="file" id="import-file-input" …>` satırından sonra, `#btn-packages` öncesine:

```html
    <div class="data-menu-wrap">
      <button id="btn-reports" class="btn-secondary" onclick="toggleReportsMenu()">📈 <span class="btn-label">Raporlar</span></button>
      <div id="reports-menu-panel" class="data-menu-panel">
        <button class="data-menu-item" onclick="openAllTimeReport();closeReportsMenu()">📈 Tüm Zamanlar</button>
      </div>
    </div>
```

- [ ] **Step 3: State** — `let activePackagesView = false; …` satırından sonra:

```js
    let activeReportView = null; // null | 'all-time' — when set, show reports page
    let reportYear = 'all'; // 'all' | number — year filter on reports page
```

- [ ] **Step 4: Navigasyon sıfırlama** — `selectMonth`, `selectYearView` ve `openPackages` içinde `activePackagesView`/`activeYearView` atamalarının yanına `activeReportView = null;` ekle. `renderContent` başında:

```js
      if (activePackagesView) { renderPackagesPage(); return; }
      if (activeReportView) { renderAllTimeReport(); return; }
```

- [ ] **Step 5: Menü fonksiyonları** — `toggleDataMenu`'yü değiştir ve yanına ekle:

```js
    function toggleDataMenu() {
      closeReportsMenu();
      document.getElementById('data-menu-panel').classList.toggle('open');
    }

    function toggleReportsMenu() {
      closeDataMenu();
      document.getElementById('reports-menu-panel').classList.toggle('open');
    }

    function closeReportsMenu() {
      document.getElementById('reports-menu-panel').classList.remove('open');
    }
```

`init()` içindeki document click handler:

```js
      document.addEventListener('click', function(e) {
        if (!e.target.closest('.data-menu-wrap')) { closeDataMenu(); closeReportsMenu(); }
      });
```

`init()` içine (event listener'ların yanına) yeniden boyutlanma:

```js
      let resizeTimer = null;
      window.addEventListener('resize', function() {
        clearTimeout(resizeTimer);
        resizeTimer = setTimeout(() => { if (activeReportView) renderContent(); }, 150);
      });
```

- [ ] **Step 6: Reports JS bölümü** — `/* ── Savings Packages ── */` yorumundan önce:

```js
    /* ── Reports ────────────────────────────────────────────── */
    const REPORT_SERIES = [
      { field: 'income',    label: 'Gelir', color: 'var(--income)' },
      { field: 'expense',   label: 'Gider', color: 'var(--expense)' },
      { field: 'remaining', label: 'Kalan', color: 'var(--remaining)' },
    ];

    function openAllTimeReport() {
      closeDrawer();
      if (editingState && activeMonth) cancelEdit(editingState.kind, editingState.idx);
      activeReportView = 'all-time';
      activeMonth = null;
      activeYearView = null;
      activePackagesView = false;
      renderSidebar();
      renderContent();
    }

    function setReportYear(value) {
      reportYear = value === 'all' ? 'all' : parseInt(value, 10);
      renderContent();
    }

    function isMonthFilled(data) {
      return !!(data && (data.incomes.length || data.expenses.length));
    }

    function getReportSeries(year) {
      return Object.keys(state.months)
        .filter(k => isMonthFilled(state.months[k]))
        .filter(k => year === 'all' || k.startsWith(`${year}-`))
        .sort((a, b) => a.localeCompare(b))
        .map(key => {
          const data = state.months[key];
          const [y, m] = key.split('-');
          const mi = parseInt(m, 10) - 1;
          const income = data.incomes.reduce((s, i) => s + (i.amount || 0), 0);
          const expense = data.expenses.reduce((s, i) => s + (i.amount || 0), 0);
          return {
            key,
            label: `${MONTHS[mi].slice(0, 3)} ${y.slice(2)}`,
            longLabel: `${MONTHS[mi]} ${y}`,
            income, expense, remaining: income - expense,
          };
        });
    }

    function getReportYears() {
      const years = new Set(Object.keys(state.months)
        .filter(k => isMonthFilled(state.months[k]))
        .map(k => parseInt(k.slice(0, 4), 10)));
      return [...years].sort((a, b) => b - a);
    }

    function fmtShort(n) {
      const abs = Math.abs(n);
      const sign = n < 0 ? '-' : '';
      let val = abs, suffix = '';
      if (abs >= 1e6) { val = abs / 1e6; suffix = 'M'; }
      else if (abs >= 1e3) { val = abs / 1e3; suffix = 'B'; }
      return `${sign}₺${val.toLocaleString('tr-TR', { maximumFractionDigits: 1 })}${suffix}`;
    }

    function niceScale(min, max) {
      if (min === max) max = min + 1;
      const rawStep = (max - min) / 5;
      const mag = Math.pow(10, Math.floor(Math.log10(rawStep)));
      const norm = rawStep / mag;
      const step = (norm <= 1 ? 1 : norm <= 2 ? 2 : norm <= 5 ? 5 : 10) * mag;
      return { min: Math.floor(min / step) * step, max: Math.ceil(max / step) * step, step };
    }

    function renderLineChart(series, width) {
      const W = Math.max(300, Math.round(width));
      const H = W < 500 ? 240 : 320;
      const pad = { top: 12, right: 16, bottom: 32, left: 64 };
      const plotW = W - pad.left - pad.right;
      const plotH = H - pad.top - pad.bottom;
      const n = series.length;
      const values = series.flatMap(p => [p.income, p.expense, p.remaining]);
      const scale = niceScale(Math.min(0, ...values), Math.max(0, ...values));
      const x = i => n === 1 ? pad.left + plotW / 2 : pad.left + i * (plotW / (n - 1));
      const y = v => pad.top + (scale.max - v) / (scale.max - scale.min) * plotH;

      let grid = '';
      const tickCount = Math.round((scale.max - scale.min) / scale.step);
      for (let t = 0; t <= tickCount; t++) {
        const v = scale.min + t * scale.step;
        const isZero = Math.abs(v) < scale.step / 1e6;
        const yy = y(v).toFixed(1);
        grid += `<line x1="${pad.left}" x2="${W - pad.right}" y1="${yy}" y2="${yy}" stroke-width="1" style="stroke:${isZero ? 'var(--text-muted)' : 'var(--border)'}"/>
          <text x="${pad.left - 8}" y="${yy}" text-anchor="end" dominant-baseline="middle" font-size="11" style="fill:var(--text-muted)">${fmtShort(isZero ? 0 : v)}</text>`;
      }

      const maxLabels = Math.max(2, Math.floor(plotW / 56));
      const labelStep = Math.ceil(n / maxLabels);
      const xLabels = series.map((p, i) => i % labelStep === 0
        ? `<text x="${x(i).toFixed(1)}" y="${H - 10}" text-anchor="middle" font-size="11" style="fill:var(--text-muted)">${p.label}</text>`
        : '').join('');

      const lines = REPORT_SERIES.map(s => {
        const pts = series.map((p, i) => `${x(i).toFixed(1)},${y(p[s.field]).toFixed(1)}`).join(' ');
        const dots = series.map((p, i) => `<circle cx="${x(i).toFixed(1)}" cy="${y(p[s.field]).toFixed(1)}" r="3" style="fill:${s.color}"/>`).join('');
        return `<polyline points="${pts}" fill="none" stroke-width="2" stroke-linejoin="round" style="stroke:${s.color}"/>${dots}`;
      }).join('');

      const slotW = n === 1 ? plotW : plotW / (n - 1);
      const hits = series.map((p, i) => {
        const x0 = Math.max(pad.left, x(i) - slotW / 2);
        const x1 = Math.min(W - pad.right, x(i) + slotW / 2);
        return `<rect class="report-hit" data-idx="${i}" data-cx="${x(i).toFixed(1)}" x="${x0.toFixed(1)}" y="${pad.top}" width="${(x1 - x0).toFixed(1)}" height="${plotH}" fill="transparent" style="cursor:pointer"/>`;
      }).join('');

      return `<svg id="report-chart" viewBox="0 0 ${W} ${H}" width="100%" style="display:block;height:auto;">
        ${grid}${xLabels}
        <line id="report-guide" x1="0" x2="0" y1="${pad.top}" y2="${pad.top + plotH}" stroke-width="1" stroke-dasharray="3,3" style="stroke:var(--text-muted);display:none"/>
        ${lines}${hits}
      </svg>`;
    }

    function renderAllTimeReport() {
      const content = document.getElementById('content');
      const years = getReportYears();
      if (reportYear !== 'all' && !years.includes(reportYear)) reportYear = 'all';
      const series = getReportSeries(reportYear);

      const yearSelect = years.length ? `
        <select id="report-year-select" onchange="setReportYear(this.value)" style="font-family:inherit;font-size:13px;border:1px solid var(--border);border-radius:5px;padding:4px 8px;background:#fff;color:var(--text);">
          <option value="all"${reportYear === 'all' ? ' selected' : ''}>Tüm Yıllar</option>
          ${years.map(y => `<option value="${y}"${reportYear === y ? ' selected' : ''}>${y}</option>`).join('')}
        </select>` : '';

      const legend = REPORT_SERIES.map(s => `
        <span style="display:flex;align-items:center;gap:6px;">
          <span style="width:10px;height:10px;border-radius:2px;background:${s.color};"></span>${s.label}
        </span>`).join('');

      content.innerHTML = `
        <h2 style="font-size:18px;margin-bottom:16px;display:flex;align-items:center;justify-content:space-between;">
          <span>Tüm Zamanlar</span>
          ${yearSelect}
        </h2>
        <div class="section-card" style="padding:16px;">
          ${series.length
            ? `<div class="report-legend">${legend}</div>
               <div class="report-chart-wrap" id="report-chart-wrap"><div class="report-tooltip" id="report-tooltip"></div></div>`
            : '<p style="font-size:13px;color:var(--text-muted);">Henüz kayıtlı aylık veri yok.</p>'}
        </div>`;

      if (!series.length) return;
      const wrap = document.getElementById('report-chart-wrap');
      wrap.insertAdjacentHTML('afterbegin', renderLineChart(series, wrap.clientWidth));
      bindReportChartEvents(series);
    }

    function bindReportChartEvents(series) {}
```

- [ ] **Step 7: Doğrula** — seed verisini yükle, sayfayı yenile, Raporlar → Tüm Zamanlar. Konsolda:

```js
[getReportSeries('all').map(p => p.key).join(','),        // "2025-01,2025-02,2025-06,2026-01"
 getReportYears().join(','),                               // "2026,2025"
 getReportSeries(2025).length,                             // 3
 fmtShort(12500), fmtShort(-1200000), fmtShort(850),       // "₺12,5B" "-₺1,2M" "₺850"
 JSON.stringify(niceScale(-15000, 70000)),                 // {"min":-20000,"max":80000,"step":20000}
 document.querySelectorAll('#report-chart polyline').length, // 3
 document.querySelectorAll('.report-hit').length]          // 4
```

Görsel: 3 çizgi, 0 çizgisi koyu, 2025-02 kalanı sıfırın altında. Yıl dropdown'ında 2025 seç → 3 nokta; Tüm Yıllar → 4. Veri menüsü açıkken Raporlar'a tıkla → Veri kapanır. Ay seç → rapor kapanır; Raporlar'a dön → seçili yıl korunmuş. `localStorage.removeItem('financial-board-v1')` + yenile → Raporlar'da boş mesajı, dropdown yok. Tek ay (yalnızca 2026-01 dolu) → nokta ortada, hata yok.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat: add reports menu with all-time income/expense line chart"
```

---

### Task 2: Grafik etkileşimi — tooltip, kılavuz çizgi, tıklayınca aya git

**Files:**
- Modify: `index.html` — Task 1'deki boş `bindReportChartEvents` gövdesi.

**Interfaces:**
- Consumes: `REPORT_SERIES`, `series[i].{ key, longLabel, income, expense, remaining }`, DOM: `#report-chart-wrap`, `#report-chart`, `#report-guide`, `#report-tooltip`, `rect.report-hit[data-idx][data-cx]`; `fmt()`, `selectMonth(key)`.
- Produces: `bindReportChartEvents(series): void` (tam gövde).

- [ ] **Step 1: Uygula** — boş fonksiyonu şununla değiştir:

```js
    function bindReportChartEvents(series) {
      const wrap = document.getElementById('report-chart-wrap');
      const guide = document.getElementById('report-guide');
      const tip = document.getElementById('report-tooltip');

      function positionTip(e) {
        const r = wrap.getBoundingClientRect();
        let left = e.clientX - r.left + 12;
        if (left + tip.offsetWidth > r.width) left = e.clientX - r.left - tip.offsetWidth - 12;
        let top = e.clientY - r.top + 12;
        if (top + tip.offsetHeight > r.height) top = r.height - tip.offsetHeight;
        tip.style.left = Math.max(0, left) + 'px';
        tip.style.top = Math.max(0, top) + 'px';
      }

      wrap.querySelectorAll('.report-hit').forEach(hit => {
        const p = series[parseInt(hit.dataset.idx, 10)];
        hit.addEventListener('mouseenter', e => {
          guide.setAttribute('x1', hit.dataset.cx);
          guide.setAttribute('x2', hit.dataset.cx);
          guide.style.display = '';
          tip.innerHTML = `<div style="font-weight:700;margin-bottom:4px;">${p.longLabel}</div>` +
            REPORT_SERIES.map(s => {
              const v = p[s.field];
              const color = s.field === 'remaining' && v < 0 ? 'var(--expense)' : s.color;
              return `<div style="display:flex;justify-content:space-between;gap:16px;">
                <span style="color:var(--text-muted)">${s.label}</span>
                <span style="font-weight:600;color:${color}">${fmt(v)}</span>
              </div>`;
            }).join('');
          tip.style.display = 'block';
          positionTip(e);
        });
        hit.addEventListener('mousemove', positionTip);
        hit.addEventListener('mouseleave', () => {
          guide.style.display = 'none';
          tip.style.display = 'none';
        });
        hit.addEventListener('click', () => selectMonth(p.key));
      });
    }
```

- [ ] **Step 2: Doğrula** — seed verisiyle Tüm Zamanlar:
  - 2025-02 noktasının üzerine gel → kesikli dikey çizgi + tooltip "Şubat 2025 / Gelir ₺50.000,00 / Gider ₺65.000,00 / Kalan -₺15.000,00 (kırmızı)".
  - En sağdaki noktada tooltip sola açılır, kart dışına taşmaz.
  - Fareyi çek → tooltip ve çizgi kaybolur.
  - Noktaya tıkla → o ayın sayfası açılır, sidebar'da ay aktif.
  - Pencereyi daralt → grafik yeniden çizilir; 375px'te yalnızca 📈 ikonu, etiketler okunur.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add hover tooltip and click-to-month on report chart"
```

---

### Task 3: README

- [ ] **Step 1:** `README.md` Özellikler listesine, "Pasta grafikler" maddesinden sonra ekle:

```markdown
- **Raporlar — Tüm Zamanlar** — tüm aylık gelir, gider ve kalanın line chart'ı; yıl filtresi, tooltip ve tıklayınca aya gitme
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: mention all-time report in README"
```
