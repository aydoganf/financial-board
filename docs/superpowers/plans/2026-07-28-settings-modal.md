# Settings Modal Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `⚙ Tipler` butonunu `⚙ Ayarlar` modalına dönüştür; sol menü + sağ içerik layout'unda Tipler ve Remote DB ayar bölümlerini sun.

**Architecture:** Mevcut `#modal-overlay` / `#modal` (tipler için) dokunulmaz. Yeni `#settings-overlay` / `#settings-modal` HTML öğeleri eklenir. `openTypesModal()` ve ilgili render fonksiyonları settings modal'ının Tipler bölümünden çağrılır. Remote DB ayarları `localStorage['financial-board-settings']` anahtarında saklanır; `syncFromRemote` / `syncToRemote` bu değerleri okur.

**Tech Stack:** Vanilla HTML5 / CSS3 / JavaScript — bağımlılık yok, build adımı yok.

## Global Constraints

- Tek dosya: `index.html` — ayrı `.css` veya `.js` dosyası oluşturma
- Mevcut `#modal-overlay` / `#modal` ve tüm tip yönetimi fonksiyonları (`renderModal`, `openTypesModal`, `closeModal`, `setModalTab`, `startTypeEdit`, `cancelTypeEdit`, `saveType`, `deleteType`, `addType`) DEĞİŞTİRİLMEZ — sadece çağrıldıkları yer değişir
- Settings modal z-index: `300` (mevcut `#modal-overlay` z-index `100`'ün üstünde)
- Settings storage key: `financial-board-settings`
- Breakpoint: `640px` (mevcut CSS değişkeni ile tutarlı)

---

### Task 1: CSS — Settings modal stilleri

**Files:**
- Modify: `index.html` — `<style>` bloğuna settings modal CSS kuralları eklenir

**Interfaces:**
- Produces: `#settings-overlay`, `#settings-modal`, `.settings-nav`, `.settings-nav-item`, `.settings-nav-item.active`, `.settings-content` CSS kuralları (Task 2 HTML'de, Task 3 JS'de kullanır)

- [ ] **Step 1: Mevcut `@media (max-width: 640px)` bloğundan hemen önce şu CSS'i ekle**

`#modal-overlay { ... }` satırının hemen altına (yaklaşık satır 27'den sonra, `.btn-secondary` kuralından önce) ekle:

```css
    #settings-overlay { display: none; position: fixed; inset: 0; background: rgba(0,0,0,.4); z-index: 300; align-items: center; justify-content: center; }
    #settings-overlay.open { display: flex; }
    #settings-modal { background: var(--surface); border-radius: 10px; width: 640px; max-height: 80vh; display: flex; flex-direction: column; overflow: hidden; }
    .settings-modal-header { display: flex; align-items: center; justify-content: space-between; padding: 16px 20px; border-bottom: 1px solid var(--border); flex-shrink: 0; }
    .settings-modal-header h2 { font-size: 15px; font-weight: 700; }
    .settings-modal-body { display: flex; flex: 1; overflow: hidden; }
    .settings-nav { width: 160px; border-right: 1px solid var(--border); padding: 8px 0; flex-shrink: 0; overflow-y: auto; }
    .settings-nav-item { width: 100%; text-align: left; padding: 9px 16px; font-size: 13px; font-weight: 500; color: var(--text); border-radius: 0; }
    .settings-nav-item:hover { background: var(--bg); }
    .settings-nav-item.active { background: #eff6ff; color: var(--remaining); font-weight: 600; }
    .settings-content { flex: 1; overflow-y: auto; padding: 24px; }
```

- [ ] **Step 2: Mevcut `@media (max-width: 640px)` bloğunun içine (bloğun sonuna, kapanış `}` öncesine) mobil kuralları ekle**

```css
      #settings-modal {
        width: 100%;
        max-width: 100%;
        height: 100vh;
        height: 100dvh;
        max-height: 100dvh;
        border-radius: 0;
        flex-direction: column;
      }
      .settings-modal-body { flex-direction: column; overflow: visible; }
      .settings-nav { width: 100%; border-right: none; border-bottom: 1px solid var(--border); display: flex; flex-direction: row; padding: 0; overflow-x: auto; }
      .settings-nav-item { white-space: nowrap; padding: 10px 16px; border-bottom: 2px solid transparent; }
      .settings-nav-item.active { background: transparent; border-bottom-color: var(--remaining); }
      .settings-content { overflow-y: auto; flex: 1; }
```

- [ ] **Step 3: Tarayıcıda kontrol et** — CSS'in sözdizimi hatası olmadığını doğrula (sayfanın bozulmadığını kontrol et)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "style: add settings modal CSS"
```

---

### Task 2: HTML — Settings overlay + topbar değişikliği

**Files:**
- Modify: `index.html` — `#settings-overlay` HTML eklenir, topbar butonu değiştirilir

**Interfaces:**
- Consumes: Task 1'in `#settings-overlay`, `#settings-modal`, `.settings-nav` CSS kuralları
- Produces: `#settings-overlay`, `#settings-modal`, `#btn-settings` HTML öğeleri (Task 3 bunlara event listener ekler)

- [ ] **Step 1: `<!-- Types Modal -->` yorumunun hemen altına, `<div id="modal-overlay">` öğesinden önce settings overlay'i ekle**

```html
  <!-- Settings Modal -->
  <div id="settings-overlay">
    <div id="settings-modal">
      <div class="settings-modal-header">
        <h2>⚙ Ayarlar</h2>
        <button class="modal-close" onclick="closeSettings()">✕</button>
      </div>
      <div class="settings-modal-body">
        <nav class="settings-nav">
          <button class="settings-nav-item active" onclick="setSettingsSection('types')">Tipler</button>
          <button class="settings-nav-item" onclick="setSettingsSection('remote')">Remote DB</button>
        </nav>
        <div class="settings-content" id="settings-content"></div>
      </div>
    </div>
  </div>
```

- [ ] **Step 2: Topbar'da `#btn-types` butonunu `#btn-settings` ile değiştir**

Mevcut:
```html
    <button id="btn-types" class="btn-secondary">⚙ <span class="btn-label">Tipler</span></button>
```

Yeni:
```html
    <button id="btn-settings" class="btn-secondary">⚙ <span class="btn-label">Ayarlar</span></button>
```

- [ ] **Step 3: Tarayıcıda kontrol et** — Sayfa yükleniyor, topbar'da "Ayarlar" butonu görünüyor (henüz çalışmıyor)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add settings modal HTML and replace types button"
```

---

### Task 3: JS — Settings modal aç/kapa + bölüm render + Remote DB + sync güncelleme

**Files:**
- Modify: `index.html` — yeni JS fonksiyonları, `init()` listener güncellemesi, `syncFromRemote` / `syncToRemote` güncellenmesi, `REMOTE_URL` / `REMOTE_KEY` sabitlerinin kaldırılması

**Interfaces:**
- Consumes: Task 2'nin `#settings-overlay`, `#settings-content`, `#btn-settings`; mevcut `renderModal()` fonksiyonu
- Produces: `openSettings()`, `closeSettings()`, `setSettingsSection(section)`, `renderSettingsContent()`, `loadRemoteSettings()`, `saveRemoteSettings()` fonksiyonları

- [ ] **Step 1: Module-level değişkeni ekle**

`let typeEditingId = null;` satırının hemen altına:

```js
    let settingsSection = 'types'; // 'types' | 'remote'
```

- [ ] **Step 2: `REMOTE_URL` ve `REMOTE_KEY` sabitlerini kaldır, yerine `loadRemoteSettings` ekle**

Şu iki satırı sil:
```js
    const REMOTE_URL = 'https://api.jsonbin.io/v3/b/6a684be2f5f4af5e29cb7996';
    const REMOTE_KEY = '$2a$10$H9CF40b3AwUUodhS5Gw8Y.yLmGuKLyWKqbppVecFIDUIRwdU/gLkW';
```

Yerine (aynı yere) şunu ekle:

```js
    const SETTINGS_KEY = 'financial-board-settings';

    function loadRemoteSettings() {
      try {
        const raw = localStorage.getItem(SETTINGS_KEY);
        return raw ? JSON.parse(raw) : null;
      } catch { return null; }
    }
```

- [ ] **Step 3: `syncFromRemote` fonksiyonunu güncelle**

Mevcut `syncFromRemote` fonksiyonunun tamamını şununla değiştir:

```js
    async function syncFromRemote() {
      const cfg = loadRemoteSettings();
      if (!cfg || !cfg.binId || !cfg.accessKey) {
        alert('Lütfen Ayarlar > Remote DB bölümünden bağlantı bilgilerini girin.');
        return;
      }
      const btn = document.getElementById('btn-sync-pull');
      btn.disabled = true;
      btn.textContent = '⏳';
      try {
        const res = await fetch(`https://api.jsonbin.io/v3/b/${cfg.binId}`, {
          headers: { 'X-Access-Key': cfg.accessKey }
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        state = data.record;
        saveState();
        renderSidebar();
        renderContent();
      } catch (err) {
        alert('Uzaktan alma başarısız: ' + err.message);
      } finally {
        btn.disabled = false;
        btn.innerHTML = '☁ <span class="btn-label">Uzaktan Al</span>';
      }
    }
```

- [ ] **Step 4: `syncToRemote` fonksiyonunu güncelle**

Mevcut `syncToRemote` fonksiyonunun tamamını şununla değiştir:

```js
    async function syncToRemote() {
      const cfg = loadRemoteSettings();
      if (!cfg || !cfg.binId || !cfg.accessKey) {
        alert('Lütfen Ayarlar > Remote DB bölümünden bağlantı bilgilerini girin.');
        return;
      }
      const btn = document.getElementById('btn-sync-push');
      btn.disabled = true;
      btn.textContent = '⏳';
      try {
        const res = await fetch(`https://api.jsonbin.io/v3/b/${cfg.binId}`, {
          method: 'PUT',
          headers: {
            'Content-Type': 'application/json',
            'X-Access-Key': cfg.accessKey
          },
          body: JSON.stringify(state)
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
      } catch (err) {
        alert('Uzağa gönderme başarısız: ' + err.message);
      } finally {
        btn.disabled = false;
        btn.innerHTML = '☁ <span class="btn-label">Uzağa Gönder</span>';
      }
    }
```

- [ ] **Step 5: `renderModal()` fonksiyonuna opsiyonel `containerId` parametresi ekle**

`renderModal` imzasını ve içindeki `getElementById('modal')` referansını güncelle. Mevcut:

```js
    function renderModal() {
      const types = modalTab === 'income' ? state.incomeTypes : state.expenseTypes;
      const modal = document.getElementById('modal');
      modal.innerHTML = `...`;
    }
```

Şu şekilde değiştir (yalnızca ilk iki satır değişir, geri kalan innerHTML içeriği aynı kalır):

```js
    function renderModal(containerId) {
      const types = modalTab === 'income' ? state.incomeTypes : state.expenseTypes;
      const modal = document.getElementById(containerId || 'modal');
      modal.innerHTML = `...`;   // içerik değişmeden kalır
    }
```

- [ ] **Step 6: Settings modal fonksiyonlarını ekle**

`syncToRemote` fonksiyonunun hemen altına şu fonksiyonları ekle:

```js
    function openSettings() {
      settingsSection = 'types';
      document.getElementById('settings-overlay').classList.add('open');
      renderSettingsContent();
    }

    function closeSettings() {
      document.getElementById('settings-overlay').classList.remove('open');
    }

    function setSettingsSection(section) {
      settingsSection = section;
      document.querySelectorAll('.settings-nav-item').forEach(el => {
        el.classList.toggle('active', el.getAttribute('onclick') === `setSettingsSection('${section}')`);
      });
      renderSettingsContent();
    }

    function renderSettingsContent() {
      if (settingsSection === 'types') {
        renderModal('settings-content');
      } else {
        const cfg = loadRemoteSettings() || {};
        const el = document.getElementById('settings-content');
        el.innerHTML = `
          <h3 style="font-size:14px;font-weight:700;margin-bottom:16px;">Remote DB Ayarları</h3>
          <div style="display:flex;flex-direction:column;gap:12px;max-width:400px;">
            <label style="font-size:13px;font-weight:600;">Bin ID
              <input id="settings-bin-id" type="text" value="${escHtml(cfg.binId || '')}"
                style="display:block;width:100%;margin-top:4px;font-family:inherit;font-size:13px;border:1px solid var(--border);border-radius:5px;padding:6px 10px;">
            </label>
            <label style="font-size:13px;font-weight:600;">X-Access-Key
              <div style="display:flex;gap:6px;margin-top:4px;">
                <input id="settings-access-key" type="password" value="${escHtml(cfg.accessKey || '')}"
                  style="flex:1;font-family:monospace;font-size:12px;border:1px solid var(--border);border-radius:5px;padding:6px 10px;">
                <button onclick="toggleKeyVisibility()" style="border:1px solid var(--border);border-radius:5px;padding:6px 10px;background:var(--surface);font-size:14px;" title="Göster/Gizle">👁</button>
              </div>
            </label>
            <div style="display:flex;align-items:center;gap:12px;">
              <button class="btn-save" onclick="saveRemoteSettings()">Kaydet</button>
              <span id="settings-save-msg" style="font-size:12px;color:var(--income);"></span>
            </div>
          </div>`;
      }
    }

    function toggleKeyVisibility() {
      const inp = document.getElementById('settings-access-key');
      inp.type = inp.type === 'password' ? 'text' : 'password';
    }

    function saveRemoteSettings() {
      const binId = document.getElementById('settings-bin-id').value.trim();
      const accessKey = document.getElementById('settings-access-key').value.trim();
      localStorage.setItem(SETTINGS_KEY, JSON.stringify({ binId, accessKey }));
      const msg = document.getElementById('settings-save-msg');
      msg.textContent = 'Kaydedildi ✓';
      setTimeout(() => { msg.textContent = ''; }, 1500);
    }
```

- [ ] **Step 7: `init()` içinde listener'ı güncelle**

`document.getElementById('btn-types').addEventListener('click', openTypesModal);` satırını şununla değiştir:

```js
      document.getElementById('btn-settings').addEventListener('click', openSettings);
```

`document.getElementById('modal-overlay').addEventListener('click', ...)` bloğunun hemen altına settings overlay listener'ını da ekle:

```js
      document.getElementById('settings-overlay').addEventListener('click', function(e) {
        if (e.target === this) closeSettings();
      });
```

- [ ] **Step 8: Uçtan uca test**

Tarayıcıda şunları doğrula:
- "⚙ Ayarlar" butonuna tıkla → modal açılır, sol menüde "Tipler" aktif, sağda tip yönetimi görünür
- Gelir/Gider tip sekmeleri çalışıyor, tip ekleme/silme/düzenleme çalışıyor
- Sol menüde "Remote DB" ye tıkla → sağda Bin ID ve X-Access-Key alanları görünür
- Değer gir, Kaydet'e tıkla → "Kaydedildi ✓" mesajı 1.5s görünür, localStorage'da `financial-board-settings` anahtarı yazılır
- Modal dışına tıkla → kapanır
- Mobilde (375px): modal tam ekran, menü üstte yatay bant
- "☁ Uzaktan Al" → Remote DB ayarı yoksa "Lütfen Ayarlar > Remote DB..." uyarısı çıkar

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat: settings modal with types and remote DB sections"
```
