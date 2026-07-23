# Mobile Responsive Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Finansal pano'yu 640px altındaki ekranlarda hamburger drawer + tam ekran modal ile kullanılabilir hale getirmek.

**Architecture:** Tek `index.html` dosyasında CSS `@media` bloğu eklenir; HTML'e hamburger butonu ve backdrop div eklenir; JS'e drawer aç/kapa fonksiyonları eklenir. Desktop layout değişmez.

**Tech Stack:** Vanilla HTML5 / CSS3 / JS — bağımlılık yok, build adımı yok.

## Global Constraints

- Tek dosya: `index.html` — ayrı `.css` veya `.js` dosyası oluşturma
- Breakpoint: `640px` (`max-width: 640px`)
- Desktop (> 640px) layout hiç değişmemeli
- Mevcut JS event listener'ları `init()` içinde kayıtlı; yeni listener'lar da `init()` içine eklenmeli

---

### Task 1: CSS — @media bloğu

**Files:**
- Modify: `index.html` — `<style>` bloğuna `@media (max-width: 640px)` kuralları eklenir

**Interfaces:**
- Produces: `.drawer-open` body class'ı (Task 3 bunu set edecek), `#btn-menu` (Task 2 ekleyecek), `#drawer-backdrop` (Task 2 ekleyecek) için stil kuralları

- [ ] **Step 1: `<style>` bloğunun kapanış `</style>` etiketinden hemen önce şu CSS'i ekle**

```css
    /* ── Mobile (≤ 640px) ──────────────────────────────────────── */
    @media (max-width: 640px) {
      #btn-menu { display: flex; }

      #topbar .btn-label { display: none; }

      #sidebar {
        position: fixed;
        top: 52px;
        left: calc(-1 * var(--sidebar-w));
        height: calc(100vh - 52px);
        z-index: 200;
        transition: left .25s ease;
        box-shadow: 2px 0 12px rgba(0,0,0,.15);
      }
      body.drawer-open #sidebar { left: 0; }
      body.drawer-open { overflow: hidden; }

      #drawer-backdrop {
        display: none;
        position: fixed;
        inset: 0;
        z-index: 199;
        background: rgba(0,0,0,.35);
      }
      body.drawer-open #drawer-backdrop { display: block; }

      #modal {
        width: 100%;
        max-width: 100%;
        height: 100vh;
        max-height: 100vh;
        border-radius: 0;
      }

      #summary-strip { flex-direction: column; }

      .item-edit-row input,
      .item-edit-row input[type=number],
      .item-edit-row select { width: 100%; }

      .pie-chart-wrap { width: 100%; }
    }
```

- [ ] **Step 2: Desktop'ta `#btn-menu` gizlemek için, media query'den ÖNCE (normal stillerin içine) şu kuralı ekle**

`</style>` öncesine değil, `.modal-close` kuralının hemen altına ekle:

```css
    #btn-menu { display: none; }
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "style: add mobile media query foundation"
```

---

### Task 2: HTML — Hamburger butonu ve backdrop

**Files:**
- Modify: `index.html` — `#topbar` ve `<body>` içine iki element eklenir

**Interfaces:**
- Consumes: Task 1'in `#btn-menu`, `#drawer-backdrop` CSS kuralları
- Produces: `#btn-menu` ve `#drawer-backdrop` elementleri (Task 3 bunlara addEventListener ekleyecek); topbar buton etiketleri `<span class="btn-label">` ile sarılır

- [ ] **Step 1: Topbar'a hamburger butonunu ekle ve mevcut buton etiketlerini `btn-label` span'ı ile sar**

`#topbar` içeriğini şu şekilde değiştir:

```html
  <div id="topbar">
    <button id="btn-menu" aria-label="Menüyü aç" style="font-size:20px;padding:4px 8px;">☰</button>
    <h1>💰 Finansal Pano</h1>
    <button id="btn-export" class="btn-secondary">⬇ <span class="btn-label">Dışa Aktar</span></button>
    <button id="btn-import" class="btn-secondary">⬆ <span class="btn-label">İçe Aktar</span></button>
    <input type="file" id="import-file-input" accept=".json" style="display:none">
    <button id="btn-types" class="btn-secondary">⚙ <span class="btn-label">Tipler</span></button>
  </div>
```

- [ ] **Step 2: `#main` div'inden hemen önce backdrop div'ini ekle**

`<div id="main">` satırından hemen önce:

```html
  <div id="drawer-backdrop"></div>
```

- [ ] **Step 3: Tarayıcıda 375px genişliğinde kontrol et**

DevTools'ta mobil görünüme geç (375px). Beklenen:
- ☰ butonu görünür, buton etiket metinleri gizli
- Sidebar gizli (sol dışarıda)
- Backdrop görünmüyor (henüz JS yok)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add hamburger button and drawer backdrop"
```

---

### Task 3: JS — Drawer aç/kapa mantığı

**Files:**
- Modify: `index.html` — `<script>` bloğuna `toggleDrawer` / `closeDrawer` eklenir; `init()` içine listener'lar; `selectMonth()` ve `selectYearView()` içine `closeDrawer()` çağrısı

**Interfaces:**
- Consumes: Task 1'in `body.drawer-open` class'ı, `#drawer-backdrop`; Task 2'nin `#btn-menu`
- Produces: global `toggleDrawer()`, `closeDrawer()` fonksiyonları

- [ ] **Step 1: `exportData` fonksiyonunun hemen üstüne iki fonksiyon ekle**

```js
    function toggleDrawer() {
      document.body.classList.toggle('drawer-open');
    }

    function closeDrawer() {
      document.body.classList.remove('drawer-open');
    }
```

- [ ] **Step 2: `init()` içinde mevcut listener'ların altına drawer listener'larını ekle**

`document.getElementById('import-file-input').addEventListener('change', importData);` satırının hemen altına:

```js
      document.getElementById('btn-menu').addEventListener('click', toggleDrawer);
      document.getElementById('drawer-backdrop').addEventListener('click', closeDrawer);
```

- [ ] **Step 3: `selectMonth()` fonksiyonuna `closeDrawer()` çağrısı ekle**

`selectMonth` içindeki `activeMonth = key;` satırından önce:

```js
    function selectMonth(key) {
      closeDrawer();
      activeMonth = key;
      activeYearView = null;
      ensureMonth(key);
      renderSidebar();
      renderContent();
    }
```

- [ ] **Step 4: `selectYearView()` fonksiyonuna da `closeDrawer()` ekle**

`selectYearView` içindeki `activeYearView = y;` satırından önce:

```js
    function selectYearView(y) {
      closeDrawer();
      activeYearView = y;
      activeMonth = null;
      renderSidebar();
      renderContent();
    }
```

- [ ] **Step 5: Tarayıcıda 375px genişliğinde uçtan uca test**

- ☰ butonuna tıkla → sidebar soldan kayarak açılır, backdrop görünür
- Backdrop'a tıkla → drawer kapanır
- Bir aya tıkla → drawer kapanır, içerik güncellenir
- 📊 yıl ikonuna tıkla → drawer kapanır
- 1024px'e geç → hamburger yok, sidebar her zaman görünür, hiç bir şey bozulmamış

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: mobile drawer toggle and auto-close on navigation"
```
