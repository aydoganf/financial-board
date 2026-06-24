# Financial Board Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single `index.html` personal finance tracker with month-based income/expense entry, type management, drag & drop reordering, and localStorage persistence.

**Architecture:** Everything lives in one `index.html` file — HTML structure, CSS styles in a `<style>` tag, and JavaScript in a `<script>` tag. State is managed as a plain JS object and synced to localStorage on every change. UI is re-rendered from state on each update (no virtual DOM, direct innerHTML).

**Tech Stack:** Vanilla HTML5, CSS3, JavaScript (ES6+). Native HTML5 Drag & Drop API. localStorage for persistence. No build tools, no dependencies.

---

## File Structure

```
index.html          — entire app: HTML skeleton + <style> + <script>
```

The `<script>` section is organized into these logical sections (in order):
1. **Constants** — default types, month names
2. **State** — load/save/init functions
3. **Render** — all render* functions
4. **Drag & Drop** — drag event handlers
5. **Event Handlers** — add/edit/delete/modal handlers
6. **Init** — bootstrap on DOMContentLoaded

---

## Task 1: HTML Skeleton + CSS Reset + Theme Variables

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create index.html with full HTML skeleton**

```html
<!DOCTYPE html>
<html lang="tr">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Finansal Pano</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    :root {
      --bg: #f8fafc;
      --surface: #ffffff;
      --border: #e2e8f0;
      --text: #0f172a;
      --text-muted: #64748b;
      --income: #16a34a;
      --expense: #dc2626;
      --remaining: #2563eb;
      --sidebar-w: 210px;
    }
    body { font-family: system-ui, sans-serif; background: var(--bg); color: var(--text); display: flex; flex-direction: column; height: 100vh; overflow: hidden; }
    #topbar { height: 52px; background: var(--surface); border-bottom: 1px solid var(--border); display: flex; align-items: center; padding: 0 16px; gap: 12px; flex-shrink: 0; }
    #topbar h1 { font-size: 16px; font-weight: 700; flex: 1; }
    #main { display: flex; flex: 1; overflow: hidden; }
    #sidebar { width: var(--sidebar-w); background: var(--surface); border-right: 1px solid var(--border); overflow-y: auto; flex-shrink: 0; padding: 8px 0; }
    #content { flex: 1; overflow-y: auto; padding: 20px; }
    button { cursor: pointer; border: none; background: none; font-family: inherit; }
  </style>
</head>
<body>
  <div id="topbar">
    <h1>💰 Finansal Pano</h1>
    <button id="btn-types" style="background:#f1f5f9;border:1px solid var(--border);border-radius:6px;padding:6px 12px;font-size:13px;">⚙ Tipler</button>
  </div>
  <div id="main">
    <nav id="sidebar"></nav>
    <main id="content"></main>
  </div>
  <!-- Types Modal -->
  <div id="modal-overlay" style="display:none;position:fixed;inset:0;background:rgba(0,0,0,.4);z-index:100;align-items:center;justify-content:center;">
    <div id="modal" style="background:#fff;border-radius:10px;width:480px;max-height:80vh;overflow-y:auto;padding:24px;"></div>
  </div>
  <script>
    // app code goes here
  </script>
</body>
</html>
```

- [ ] **Step 2: Open index.html in browser and verify layout renders**

Open `index.html` directly in a browser (file:// URL).
Expected: White topbar with "💰 Finansal Pano" and "⚙ Tipler" button, empty sidebar on left, empty content area on right.

- [ ] **Step 3: Commit**

```bash
git init
git add index.html
git commit -m "feat: html skeleton and css theme variables"
```

---

## Task 2: Constants + State (localStorage)

**Files:**
- Modify: `index.html` — replace `// app code goes here` comment with the code below

- [ ] **Step 1: Add constants and state functions inside `<script>`**

```js
// ── Constants ──────────────────────────────────────────────
const MONTHS = ['Ocak','Şubat','Mart','Nisan','Mayıs','Haziran','Temmuz','Ağustos','Eylül','Ekim','Kasım','Aralık'];

const DEFAULT_INCOME_TYPES = [
  { id: 'it1', name: 'Düzenli Maaş', color: '#2563eb' },
  { id: 'it2', name: 'Yan Gelir',    color: '#7c3aed' },
  { id: 'it3', name: 'Kira Geliri',  color: '#059669' },
  { id: 'it4', name: 'Yatırım',      color: '#d97706' },
  { id: 'it5', name: 'Diğer',        color: '#64748b' },
];

const DEFAULT_EXPENSE_TYPES = [
  { id: 'et1', name: 'Konut',        color: '#dc2626' },
  { id: 'et2', name: 'Market & Gıda',color: '#ea580c' },
  { id: 'et3', name: 'Faturalar',    color: '#ca8a04' },
  { id: 'et4', name: 'Ulaşım',       color: '#0891b2' },
  { id: 'et5', name: 'Sağlık',       color: '#db2777' },
  { id: 'et6', name: 'Eğitim',       color: '#4f46e5' },
  { id: 'et7', name: 'Eğlence',      color: '#7c3aed' },
  { id: 'et8', name: 'Diğer',        color: '#64748b' },
];

// ── State ──────────────────────────────────────────────────
const STORAGE_KEY = 'financial-board-v1';
let state = null; // loaded in init()
let activeMonth = null; // "YYYY-MM"

function loadState() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (raw) return JSON.parse(raw);
  } catch (e) {
    showStorageWarning();
  }
  return null;
}

function saveState() {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
  } catch (e) {
    showStorageWarning();
  }
}

function initState() {
  const saved = loadState();
  state = saved || {
    incomeTypes: DEFAULT_INCOME_TYPES.map(t => ({ ...t })),
    expenseTypes: DEFAULT_EXPENSE_TYPES.map(t => ({ ...t })),
    months: {},
  };
}

function ensureMonth(key) {
  if (!state.months[key]) {
    state.months[key] = { incomes: [], expenses: [] };
    saveState();
  }
}

function uuid() {
  return Math.random().toString(36).slice(2) + Date.now().toString(36);
}

function showStorageWarning() {
  let banner = document.getElementById('storage-warning');
  if (!banner) {
    banner = document.createElement('div');
    banner.id = 'storage-warning';
    banner.style = 'background:#fef2f2;border:1px solid #fecaca;color:#dc2626;padding:8px 16px;font-size:13px;text-align:center;';
    banner.textContent = '⚠️ localStorage kullanılamıyor — veriler kaydedilmeyecek.';
    document.body.prepend(banner);
  }
}
```

- [ ] **Step 2: Verify in browser console**

Open DevTools console, reload page. Run:
```js
initState(); console.log(state);
```
Expected: Object with `incomeTypes` (5 items), `expenseTypes` (8 items), `months: {}`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: state management with localStorage"
```

---

## Task 3: Sidebar Navigation

**Files:**
- Modify: `index.html` — add sidebar CSS and `renderSidebar()` function

- [ ] **Step 1: Add sidebar CSS inside `<style>`**

```css
.year-group { }
.year-btn { width: 100%; text-align: left; padding: 8px 14px; font-size: 13px; font-weight: 600; color: var(--text-muted); display: flex; justify-content: space-between; align-items: center; }
.year-btn:hover { background: var(--bg); }
.year-months { padding: 0 0 4px 0; }
.month-btn { width: 100%; text-align: left; padding: 6px 14px 6px 28px; font-size: 13px; color: var(--text); border-radius: 0; transition: background .1s; }
.month-btn:hover { background: #f1f5f9; }
.month-btn.active { background: #eff6ff; color: var(--remaining); font-weight: 600; }
```

- [ ] **Step 2: Add `renderSidebar()` function in `<script>`**

```js
// ── Render: Sidebar ────────────────────────────────────────
function renderSidebar() {
  const sidebar = document.getElementById('sidebar');
  const currentYear = new Date().getFullYear();
  const maxYear = currentYear + 5;
  let html = '';
  for (let y = maxYear; y >= 2020; y--) {
    const isCurrentYear = y === currentYear;
    html += `<div class="year-group">
      <button class="year-btn" onclick="toggleYear(${y})" id="year-btn-${y}">
        <span>${y}</span><span id="year-arrow-${y}">${isCurrentYear ? '▾' : '▸'}</span>
      </button>
      <div class="year-months" id="year-months-${y}" style="display:${isCurrentYear ? 'block' : 'none'}">`;
    for (let m = 0; m < 12; m++) {
      const key = `${y}-${String(m + 1).padStart(2, '0')}`;
      const isActive = key === activeMonth;
      html += `<button class="month-btn${isActive ? ' active' : ''}" onclick="selectMonth('${key}')">${MONTHS[m]}</button>`;
    }
    html += `</div></div>`;
  }
  sidebar.innerHTML = html;
}

function toggleYear(y) {
  const months = document.getElementById(`year-months-${y}`);
  const arrow = document.getElementById(`year-arrow-${y}`);
  const open = months.style.display !== 'none';
  months.style.display = open ? 'none' : 'block';
  arrow.textContent = open ? '▸' : '▾';
}

function selectMonth(key) {
  activeMonth = key;
  ensureMonth(key);
  renderSidebar();
  renderContent();
}
```

- [ ] **Step 3: Add stub `renderContent()` so page doesn't crash**

```js
function renderContent() {
  document.getElementById('content').innerHTML = `<p style="color:var(--text-muted)">Ay: ${activeMonth}</p>`;
}
```

- [ ] **Step 4: Add `init()` and wire DOMContentLoaded**

```js
// ── Init ───────────────────────────────────────────────────
function init() {
  initState();
  const now = new Date();
  activeMonth = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`;
  ensureMonth(activeMonth);
  renderSidebar();
  renderContent();
}

document.addEventListener('DOMContentLoaded', init);
```

- [ ] **Step 5: Verify in browser**

Reload. Expected: sidebar shows years 2020–2031, current year expanded, current month highlighted in blue, clicking other months/years updates the active highlight.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: sidebar navigation with year/month tree"
```

---

## Task 4: Summary Strip + Section Headers

**Files:**
- Modify: `index.html` — add content CSS and `renderContent()` full implementation

- [ ] **Step 1: Add content area CSS inside `<style>`**

```css
#summary-strip { display: flex; gap: 12px; margin-bottom: 20px; }
.stat-card { flex: 1; background: var(--surface); border: 1px solid var(--border); border-radius: 8px; padding: 14px 16px; }
.stat-card .label { font-size: 11px; color: var(--text-muted); text-transform: uppercase; letter-spacing: .04em; margin-bottom: 4px; }
.stat-card .value { font-size: 22px; font-weight: 700; }
.section-card { background: var(--surface); border: 1px solid var(--border); border-radius: 8px; margin-bottom: 16px; overflow: hidden; }
.section-header { display: flex; align-items: center; gap: 8px; padding: 12px 16px; cursor: pointer; user-select: none; }
.section-header .section-title { font-size: 14px; font-weight: 700; flex: 1; }
.section-header .section-total { font-size: 14px; font-weight: 600; }
.section-body { border-top: 1px solid var(--border); }
```

- [ ] **Step 2: Replace stub `renderContent()` with full implementation**

```js
function renderContent() {
  const content = document.getElementById('content');
  if (!activeMonth) { content.innerHTML = ''; return; }
  const [y, m] = activeMonth.split('-');
  const monthData = state.months[activeMonth] || { incomes: [], expenses: [] };
  const totalIncome = monthData.incomes.reduce((s, i) => s + (i.amount || 0), 0);
  const totalExpense = monthData.expenses.reduce((s, i) => s + (i.amount || 0), 0);
  const remaining = totalIncome - totalExpense;
  const remainColor = remaining >= 0 ? 'var(--remaining)' : 'var(--expense)';
  const monthLabel = `${MONTHS[parseInt(m, 10) - 1]} ${y}`;

  content.innerHTML = `
    <h2 style="font-size:18px;margin-bottom:16px;">${monthLabel}</h2>
    <div id="summary-strip">
      <div class="stat-card">
        <div class="label">Toplam Gelir</div>
        <div class="value" style="color:var(--income)">${fmt(totalIncome)}</div>
      </div>
      <div class="stat-card">
        <div class="label">Toplam Gider</div>
        <div class="value" style="color:var(--expense)">${fmt(totalExpense)}</div>
      </div>
      <div class="stat-card">
        <div class="label">Kalan</div>
        <div class="value" style="color:${remainColor}">${fmt(remaining)}</div>
      </div>
    </div>
    <div class="section-card">
      <div class="section-header" onclick="toggleSection('incomes')">
        <span>▾</span>
        <span class="section-title" style="color:var(--income)">GELİRLER</span>
        <span class="section-total" style="color:var(--income)">${fmt(totalIncome)}</span>
      </div>
      <div class="section-body" id="section-incomes">
        ${renderItemList(monthData.incomes, 'income')}
        <div style="padding:10px 16px;">
          <button onclick="addItem('income')" style="font-size:13px;color:var(--income);border:1px dashed var(--income);border-radius:6px;padding:6px 12px;">+ Yeni Gelir Ekle</button>
        </div>
      </div>
    </div>
    <div class="section-card">
      <div class="section-header" onclick="toggleSection('expenses')">
        <span>▾</span>
        <span class="section-title" style="color:var(--expense)">GİDERLER</span>
        <span class="section-total" style="color:var(--expense)">${fmt(totalExpense)}</span>
      </div>
      <div class="section-body" id="section-expenses">
        ${renderItemList(monthData.expenses, 'expense')}
        <div style="padding:10px 16px;">
          <button onclick="addItem('expense')" style="font-size:13px;color:var(--expense);border:1px dashed var(--expense);border-radius:6px;padding:6px 12px;">+ Yeni Gider Ekle</button>
        </div>
      </div>
    </div>
  `;
}

function fmt(n) {
  return '₺' + Number(n).toLocaleString('tr-TR', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
}

function toggleSection(id) {
  const body = document.getElementById(`section-${id}`);
  const header = body.previousElementSibling;
  const arrow = header.querySelector('span');
  const open = body.style.display !== 'none';
  body.style.display = open ? 'none' : 'block';
  arrow.textContent = open ? '▸' : '▾';
}
```

- [ ] **Step 3: Add stub `renderItemList()` so renderContent doesn't crash**

```js
function renderItemList(items, kind) {
  return `<p style="padding:12px 16px;color:var(--text-muted);font-size:13px;">Henüz kalem yok.</p>`;
}
```

- [ ] **Step 4: Verify in browser**

Reload. Expected: current month shown with 3 stat cards (₺0.00 each), collapsible GELİRLER and GİDERLER sections, "Yeni Gelir/Gider Ekle" buttons visible.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: summary strip and section headers"
```

---

## Task 5: Line Items — Render + Add + Delete

**Files:**
- Modify: `index.html` — replace stub `renderItemList()`, add item CSS, add `addItem()` and `deleteItem()`

- [ ] **Step 1: Add item CSS inside `<style>`**

```css
.item-row { display: flex; align-items: center; gap: 10px; padding: 8px 16px; border-top: 1px solid var(--border); font-size: 13px; transition: background .1s; }
.item-row:hover { background: #f8fafc; }
.drag-handle { color: #cbd5e1; cursor: grab; font-size: 16px; flex-shrink: 0; user-select: none; }
.type-badge { border-radius: 99px; padding: 2px 8px; font-size: 11px; font-weight: 600; flex-shrink: 0; }
.item-name { flex: 1; font-size: 13px; }
.item-amount { font-weight: 600; font-size: 13px; flex-shrink: 0; width: 100px; text-align: right; }
.item-actions { display: flex; gap: 4px; flex-shrink: 0; }
.item-actions button { padding: 3px 7px; border-radius: 4px; font-size: 12px; border: 1px solid var(--border); background: var(--surface); color: var(--text-muted); }
.item-actions button:hover { background: var(--bg); }
.item-note { font-size: 11px; color: var(--text-muted); padding: 0 16px 8px 52px; border-top: none; }
```

- [ ] **Step 2: Replace stub `renderItemList()` with full implementation**

```js
function renderItemList(items, kind) {
  if (!items.length) {
    return `<p style="padding:12px 16px;color:var(--text-muted);font-size:13px;">Henüz kalem yok.</p>`;
  }
  const types = kind === 'income' ? state.incomeTypes : state.expenseTypes;
  const amtColor = kind === 'income' ? 'var(--income)' : 'var(--expense)';
  return items.map((item, idx) => {
    const type = types.find(t => t.id === item.typeId);
    const badgeColor = type ? type.color : '#64748b';
    const badgeBg = badgeColor + '20';
    const badgeName = type ? type.name : '—';
    const noteHtml = item.note ? `<div class="item-note">${escHtml(item.note)}</div>` : '';
    return `
      <div class="item-row" draggable="true" data-kind="${kind}" data-idx="${idx}"
           ondragstart="onDragStart(event)" ondragover="onDragOver(event)" ondrop="onDrop(event)" ondragend="onDragEnd(event)">
        <span class="drag-handle">⠿</span>
        <span class="type-badge" style="color:${badgeColor};background:${badgeBg}">${escHtml(badgeName)}</span>
        <span class="item-name">${escHtml(item.name)}</span>
        <span class="item-amount" style="color:${amtColor}">${fmt(item.amount)}</span>
        <div class="item-actions">
          <button onclick="editItem('${kind}', ${idx})">✏</button>
          <button onclick="deleteItem('${kind}', ${idx})" style="color:var(--expense)">🗑</button>
        </div>
      </div>${noteHtml}`;
  }).join('');
}

function escHtml(s) {
  return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
}
```

- [ ] **Step 3: Add `addItem()` and `deleteItem()`**

```js
function addItem(kind) {
  const types = kind === 'income' ? state.incomeTypes : state.expenseTypes;
  const defaultType = types[0] || null;
  const items = state.months[activeMonth][kind === 'income' ? 'incomes' : 'expenses'];
  items.push({
    id: uuid(),
    name: kind === 'income' ? 'Yeni Gelir' : 'Yeni Gider',
    amount: 0,
    typeId: defaultType ? defaultType.id : null,
    note: '',
    order: items.length,
  });
  saveState();
  renderContent();
  // open edit immediately for the new item
  const newIdx = state.months[activeMonth][kind === 'income' ? 'incomes' : 'expenses'].length - 1;
  editItem(kind, newIdx);
}

function deleteItem(kind, idx) {
  const key = kind === 'income' ? 'incomes' : 'expenses';
  state.months[activeMonth][key].splice(idx, 1);
  saveState();
  renderContent();
}
```

- [ ] **Step 4: Verify in browser**

Click "Yeni Gelir Ekle". Expected: a new row appears in the list. Click 🗑 on it. Expected: row removed. Reload page — if an item was saved before deletion it should persist.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: item list render, add and delete"
```

---

## Task 6: Inline Edit Form

**Files:**
- Modify: `index.html` — add edit row CSS, add `editItem()` and `saveItem()`

- [ ] **Step 1: Add edit form CSS inside `<style>`**

```css
.item-edit-row { padding: 10px 16px; border-top: 1px solid var(--border); background: #f8fafc; display: flex; flex-direction: column; gap: 8px; }
.item-edit-row .edit-fields { display: flex; gap: 8px; align-items: center; flex-wrap: wrap; }
.item-edit-row input, .item-edit-row select, .item-edit-row textarea { font-family: inherit; font-size: 13px; border: 1px solid var(--border); border-radius: 5px; padding: 5px 8px; background: #fff; }
.item-edit-row input { width: 160px; }
.item-edit-row input[type=number] { width: 110px; }
.item-edit-row select { width: 140px; }
.item-edit-row textarea { width: 100%; height: 48px; resize: none; }
.edit-error { color: var(--expense); font-size: 12px; }
.edit-actions { display: flex; gap: 8px; }
.btn-save { background: #2563eb; color: #fff; border-radius: 5px; padding: 5px 14px; font-size: 13px; }
.btn-cancel { background: var(--surface); border: 1px solid var(--border); border-radius: 5px; padding: 5px 14px; font-size: 13px; }
```

- [ ] **Step 2: Modify `renderItemList()` to support an `editingIdx` parameter**

Replace the `renderItemList` function signature and items.map section:

```js
function renderItemList(items, kind, editingIdx = -1) {
  if (!items.length && editingIdx === -1) {
    return `<p style="padding:12px 16px;color:var(--text-muted);font-size:13px;">Henüz kalem yok.</p>`;
  }
  const types = kind === 'income' ? state.incomeTypes : state.expenseTypes;
  const amtColor = kind === 'income' ? 'var(--income)' : 'var(--expense)';
  return items.map((item, idx) => {
    if (idx === editingIdx) return renderEditRow(item, kind, idx, types);
    const type = types.find(t => t.id === item.typeId);
    const badgeColor = type ? type.color : '#64748b';
    const badgeBg = badgeColor + '20';
    const badgeName = type ? type.name : '—';
    const noteHtml = item.note ? `<div class="item-note">${escHtml(item.note)}</div>` : '';
    return `
      <div class="item-row" draggable="true" data-kind="${kind}" data-idx="${idx}"
           ondragstart="onDragStart(event)" ondragover="onDragOver(event)" ondrop="onDrop(event)" ondragend="onDragEnd(event)">
        <span class="drag-handle">⠿</span>
        <span class="type-badge" style="color:${badgeColor};background:${badgeBg}">${escHtml(badgeName)}</span>
        <span class="item-name">${escHtml(item.name)}</span>
        <span class="item-amount" style="color:${amtColor}">${fmt(item.amount)}</span>
        <div class="item-actions">
          <button onclick="editItem('${kind}', ${idx})">✏</button>
          <button onclick="deleteItem('${kind}', ${idx})" style="color:var(--expense)">🗑</button>
        </div>
      </div>${noteHtml}`;
  }).join('');
}

function renderEditRow(item, kind, idx, types) {
  const opts = types.map(t => `<option value="${t.id}"${t.id === item.typeId ? ' selected' : ''}>${escHtml(t.name)}</option>`).join('');
  return `
    <div class="item-edit-row" id="edit-row-${kind}-${idx}">
      <div class="edit-fields">
        <input id="ef-name" type="text" value="${escHtml(item.name)}" placeholder="Ad" />
        <input id="ef-amount" type="number" min="0" step="0.01" value="${item.amount}" placeholder="Tutar" />
        <select id="ef-type">${opts}</select>
      </div>
      <textarea id="ef-note" placeholder="Açıklama (opsiyonel)">${escHtml(item.note || '')}</textarea>
      <div class="edit-actions">
        <button class="btn-save" onclick="saveItem('${kind}', ${idx})">Kaydet</button>
        <button class="btn-cancel" onclick="cancelEdit('${kind}', ${idx})">İptal</button>
        <span class="edit-error" id="ef-error"></span>
      </div>
    </div>`;
}
```

- [ ] **Step 3: Update `renderContent()` to accept an optional editing state**

Add a module-level variable and wire it in:

```js
let editingState = null; // { kind: 'income'|'expense', idx: number } | null
```

In `renderContent()`, change the two `renderItemList(...)` calls to:

```js
${renderItemList(monthData.incomes, 'income', editingState && editingState.kind === 'income' ? editingState.idx : -1)}
// ...
${renderItemList(monthData.expenses, 'expense', editingState && editingState.kind === 'expense' ? editingState.idx : -1)}
```

- [ ] **Step 4: Add `editItem()`, `saveItem()`, `cancelEdit()`**

```js
function editItem(kind, idx) {
  editingState = { kind, idx };
  renderContent();
  document.getElementById('ef-name').focus();
}

function saveItem(kind, idx) {
  const name = document.getElementById('ef-name').value.trim();
  const amountRaw = document.getElementById('ef-amount').value;
  const amount = parseFloat(amountRaw);
  const typeId = document.getElementById('ef-type').value;
  const note = document.getElementById('ef-note').value.trim();
  const errEl = document.getElementById('ef-error');

  if (!name) { errEl.textContent = 'Ad boş bırakılamaz.'; return; }
  if (isNaN(amount) || amount < 0) { errEl.textContent = 'Geçerli bir tutar girin (≥ 0).'; return; }

  const key = kind === 'income' ? 'incomes' : 'expenses';
  state.months[activeMonth][key][idx] = { ...state.months[activeMonth][key][idx], name, amount, typeId, note };
  saveState();
  editingState = null;
  renderContent();
}

function cancelEdit(kind, idx) {
  // If item was newly added with amount=0 and name is still default, remove it
  const key = kind === 'income' ? 'incomes' : 'expenses';
  const item = state.months[activeMonth][key][idx];
  if (item.amount === 0 && (item.name === 'Yeni Gelir' || item.name === 'Yeni Gider')) {
    state.months[activeMonth][key].splice(idx, 1);
    saveState();
  }
  editingState = null;
  renderContent();
}
```

- [ ] **Step 5: Verify in browser**

Click "Yeni Gelir Ekle" — inline edit form opens. Fill in name/amount/type, click Kaydet. Expected: row appears with correct values, correct type badge color. Click ✏ on existing row — form opens prefilled. Click İptal — form closes without changes.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: inline edit form for items"
```

---

## Task 7: Drag & Drop Reordering

**Files:**
- Modify: `index.html` — add drag & drop event handlers

- [ ] **Step 1: Add drag & drop state variables**

```js
let dragSrcKind = null;
let dragSrcIdx = null;
```

- [ ] **Step 2: Add drag event handlers**

```js
function onDragStart(e) {
  dragSrcKind = e.currentTarget.dataset.kind;
  dragSrcIdx = parseInt(e.currentTarget.dataset.idx, 10);
  e.currentTarget.style.opacity = '0.4';
  e.dataTransfer.effectAllowed = 'move';
}

function onDragOver(e) {
  e.preventDefault();
  const target = e.currentTarget;
  if (target.dataset.kind !== dragSrcKind) {
    e.dataTransfer.dropEffect = 'none';
    return;
  }
  e.dataTransfer.dropEffect = 'move';
  target.style.borderTop = '2px solid var(--remaining)';
}

function onDrop(e) {
  e.preventDefault();
  const target = e.currentTarget;
  target.style.borderTop = '';
  if (target.dataset.kind !== dragSrcKind) return;
  const destIdx = parseInt(target.dataset.idx, 10);
  if (destIdx === dragSrcIdx) return;
  const key = dragSrcKind === 'income' ? 'incomes' : 'expenses';
  const items = state.months[activeMonth][key];
  const [moved] = items.splice(dragSrcIdx, 1);
  items.splice(destIdx, 0, moved);
  saveState();
  renderContent();
}

function onDragEnd(e) {
  e.currentTarget.style.opacity = '';
  e.currentTarget.style.borderTop = '';
  dragSrcKind = null;
  dragSrcIdx = null;
}
```

- [ ] **Step 3: Verify in browser**

Add 3+ income items. Drag one by its ⠿ handle and drop it above another. Expected: order changes and persists on reload. Drag an income over expenses — drop should have no effect.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: drag and drop reordering within sections"
```

---

## Task 8: Type Management Modal

**Files:**
- Modify: `index.html` — add modal CSS, `openTypesModal()`, `renderModal()`, `addType()`, `saveType()`, `deleteType()`

- [ ] **Step 1: Add modal CSS inside `<style>`**

```css
#modal-overlay { display: none; position: fixed; inset: 0; background: rgba(0,0,0,.4); z-index: 100; align-items: center; justify-content: center; }
#modal-overlay.open { display: flex; }
#modal { background: #fff; border-radius: 10px; width: 500px; max-height: 80vh; overflow-y: auto; padding: 24px; }
.modal-tabs { display: flex; gap: 4px; margin-bottom: 16px; }
.modal-tab { padding: 6px 16px; border-radius: 6px; font-size: 13px; font-weight: 600; border: 1px solid var(--border); }
.modal-tab.active { background: #eff6ff; color: var(--remaining); border-color: #bfdbfe; }
.type-row { display: flex; align-items: center; gap: 8px; padding: 6px 0; border-bottom: 1px solid var(--border); font-size: 13px; }
.type-row input[type=text] { flex: 1; font-family: inherit; font-size: 13px; border: 1px solid var(--border); border-radius: 5px; padding: 4px 8px; }
.type-row input[type=color] { width: 32px; height: 28px; border: 1px solid var(--border); border-radius: 5px; padding: 1px; cursor: pointer; }
.type-add-row { display: flex; gap: 8px; margin-top: 12px; align-items: center; }
.type-add-row input[type=text] { flex:1; font-family: inherit; font-size:13px; border:1px solid var(--border); border-radius:5px; padding:5px 8px; }
.type-add-row input[type=color] { width:32px; height:32px; border:1px solid var(--border); border-radius:5px; padding:1px; cursor:pointer; }
.btn-add-type { background: #2563eb; color: #fff; border-radius: 5px; padding: 5px 14px; font-size: 13px; }
.type-del-btn { color: var(--expense); font-size: 13px; border: 1px solid var(--border); border-radius: 4px; padding: 2px 7px; }
.type-warn { color: var(--expense); font-size: 11px; margin-top: 2px; }
.modal-close { float: right; font-size: 18px; color: var(--text-muted); background: none; border: none; cursor: pointer; }
```

- [ ] **Step 2: Add modal state and open/close functions**

```js
let modalTab = 'income'; // 'income' | 'expense'
let typeEditingId = null;

function openTypesModal() {
  document.getElementById('modal-overlay').classList.add('open');
  renderModal();
}

function closeModal() {
  document.getElementById('modal-overlay').classList.remove('open');
  typeEditingId = null;
}

document.getElementById('modal-overlay').addEventListener('click', function(e) {
  if (e.target === this) closeModal();
});
```

- [ ] **Step 3: Add `renderModal()`**

```js
function renderModal() {
  const types = modalTab === 'income' ? state.incomeTypes : state.expenseTypes;
  const modal = document.getElementById('modal');
  modal.innerHTML = `
    <button class="modal-close" onclick="closeModal()">✕</button>
    <h3 style="font-size:16px;margin-bottom:14px;">Tip Yönetimi</h3>
    <div class="modal-tabs">
      <button class="modal-tab${modalTab === 'income' ? ' active' : ''}" onclick="setModalTab('income')">Gelir Tipleri</button>
      <button class="modal-tab${modalTab === 'expense' ? ' active' : ''}" onclick="setModalTab('expense')">Gider Tipleri</button>
    </div>
    <div id="type-list">
      ${types.map(t => `
        <div class="type-row" id="tr-${t.id}">
          ${typeEditingId === t.id
            ? `<input type="color" id="ec-${t.id}" value="${t.color}" />
               <input type="text" id="en-${t.id}" value="${escHtml(t.name)}" />
               <button class="btn-save" style="font-size:12px;padding:3px 10px;" onclick="saveType('${t.id}')">Kaydet</button>
               <button class="btn-cancel" style="font-size:12px;padding:3px 10px;" onclick="cancelTypeEdit()">İptal</button>`
            : `<span style="width:18px;height:18px;border-radius:50%;background:${t.color};display:inline-block;flex-shrink:0"></span>
               <span style="flex:1">${escHtml(t.name)}</span>
               <button style="font-size:12px;border:1px solid var(--border);border-radius:4px;padding:2px 7px;" onclick="startTypeEdit('${t.id}')">✏</button>
               <button class="type-del-btn" onclick="deleteType('${t.id}')">🗑</button>
               <span class="type-warn" id="tw-${t.id}"></span>`
          }
        </div>`).join('')}
    </div>
    <div class="type-add-row">
      <input type="color" id="new-type-color" value="#64748b" />
      <input type="text" id="new-type-name" placeholder="Yeni tip adı..." />
      <button class="btn-add-type" onclick="addType()">+ Ekle</button>
    </div>
    <span class="type-warn" id="add-type-error"></span>
  `;
}

function setModalTab(tab) { modalTab = tab; typeEditingId = null; renderModal(); }
function startTypeEdit(id) { typeEditingId = id; renderModal(); }
function cancelTypeEdit() { typeEditingId = null; renderModal(); }
```

- [ ] **Step 4: Add `addType()`, `saveType()`, `deleteType()`**

```js
function addType() {
  const name = document.getElementById('new-type-name').value.trim();
  const color = document.getElementById('new-type-color').value;
  const errEl = document.getElementById('add-type-error');
  if (!name) { errEl.textContent = 'Tip adı boş olamaz.'; return; }
  const types = modalTab === 'income' ? state.incomeTypes : state.expenseTypes;
  if (types.find(t => t.name.toLowerCase() === name.toLowerCase())) {
    errEl.textContent = 'Bu isimde bir tip zaten var.'; return;
  }
  types.push({ id: uuid(), name, color });
  saveState();
  renderModal();
  renderContent();
}

function saveType(id) {
  const name = document.getElementById(`en-${id}`).value.trim();
  const color = document.getElementById(`ec-${id}`).value;
  const types = modalTab === 'income' ? state.incomeTypes : state.expenseTypes;
  const errEl = document.getElementById(`tw-${id}`);
  if (!name) { if (errEl) errEl.textContent = 'Ad boş olamaz.'; return; }
  const type = types.find(t => t.id === id);
  if (type) { type.name = name; type.color = color; }
  saveState();
  typeEditingId = null;
  renderModal();
  renderContent();
}

function deleteType(id) {
  const types = modalTab === 'income' ? state.incomeTypes : state.expenseTypes;
  const allItems = Object.values(state.months).flatMap(m =>
    modalTab === 'income' ? m.incomes : m.expenses
  );
  const inUse = allItems.filter(i => i.typeId === id).length;
  const errEl = document.getElementById(`tw-${id}`);
  if (inUse > 0) {
    if (errEl) errEl.textContent = `${inUse} kalem bu tipi kullanıyor. Önce kalemleri güncelleyin.`;
    return;
  }
  const idx = types.findIndex(t => t.id === id);
  if (idx !== -1) types.splice(idx, 1);
  saveState();
  renderModal();
  renderContent();
}
```

- [ ] **Step 5: Wire "⚙ Tipler" button**

```js
document.getElementById('btn-types').addEventListener('click', openTypesModal);
```

- [ ] **Step 6: Verify in browser**

Click ⚙ Tipler. Expected: modal with Gelir/Gider tabs, type list with color dots. Add a new type — appears in list. Edit — name/color update. Try to delete a type that has items in use — error message shows. Delete unused type — disappears. Type badge colors update in the month view.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: type management modal with add/edit/delete"
```

---

## Task 9: Polish + Edge Cases

**Files:**
- Modify: `index.html` — minor visual polish and edge case handling

- [ ] **Step 1: Scroll the edit row into view after opening**

In `editItem()`, after `renderContent()`:

```js
function editItem(kind, idx) {
  editingState = { kind, idx };
  renderContent();
  const row = document.getElementById(`edit-row-${kind}-${idx}`);
  if (row) row.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
  document.getElementById('ef-name').focus();
}
```

- [ ] **Step 2: Add a "no data" empty state for fresh months**

Already handled by `renderItemList` returning the "Henüz kalem yok." message — verify it shows correctly on a new month with no items.

- [ ] **Step 3: Format amount input to not allow negative via HTML**

Already set with `min="0"` in `renderEditRow` — verify browser enforces it.

- [ ] **Step 4: Add keyboard shortcut — Enter to save in edit row**

In `renderEditRow`, add `onkeydown` to the name and amount inputs:

```js
// In renderEditRow, replace the name input with:
<input id="ef-name" type="text" value="${escHtml(item.name)}" placeholder="Ad" onkeydown="if(event.key==='Enter')saveItem('${kind}',${idx})" />
// Replace the amount input with:
<input id="ef-amount" type="number" min="0" step="0.01" value="${item.amount}" placeholder="Tutar" onkeydown="if(event.key==='Enter')saveItem('${kind}',${idx})" />
```

- [ ] **Step 5: Final verify in browser**

Full golden path test:
1. Open app — current month selected, stats show ₺0.00
2. Add 2 income items with different types, save each
3. Add 2 expense items, save each
4. Verify summary strip updates correctly
5. Drag income items to reorder — reload — order persists
6. Navigate to a future month (e.g. 2027-03) — empty state, add an item, go back to current month — data intact
7. Open Tipler, add a new expense type, assign it to an item
8. Try to delete that type — blocked with warning
9. Delete the item, retry type deletion — succeeds

- [ ] **Step 6: Final commit**

```bash
git add index.html
git commit -m "feat: keyboard shortcut, scroll-to-edit, final polish"
```
