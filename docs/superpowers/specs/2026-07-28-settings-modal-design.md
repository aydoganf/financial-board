# Settings Modal — Design Spec
Date: 2026-07-28

## Overview

Mevcut `⚙ Tipler` topbar butonu kaldırılır; yerine `⚙ Ayarlar` butonu eklenir. Bu buton yeni bir settings modal'ı açar. Modal, sol menü + sağ içerik düzeninde iki bölüm sunar: **Tipler** (mevcut tip yönetimi buraya taşınır) ve **Remote DB** (uzak senkronizasyon ayarları).

## Modal Yapısı

### Desktop (>640px)
- Ayrı `#settings-overlay` / `#settings-modal` öğeleri — mevcut `#modal-overlay` / `#modal` (tipler için) dokunulmaz
- Modal genişliği: `640px`, yükseklik: `80vh`, `overflow: hidden`
- İç düzen: `display: flex; flex-direction: row`
  - Sol menü: `width: 160px`, `border-right: 1px solid var(--border)`, `padding: 8px 0`, dikey liste
  - Sağ içerik: `flex: 1`, `overflow-y: auto`, `padding: 24px`
- Aktif menü öğesi: `background: #eff6ff; color: var(--remaining); font-weight: 600`

### Mobil (≤640px)
- Modal tam ekran: `width: 100%; height: 100vh; border-radius: 0`
- Sol menü yatay sekme şeridine dönüşür: `display: flex; flex-direction: row; border-right: none; border-bottom: 1px solid var(--border); width: 100%`
- İç düzen dikey: `flex-direction: column`

## Topbar Değişikliği

- `#btn-types` kaldırılır
- Yerine `#btn-settings` eklenir: `⚙ <span class="btn-label">Ayarlar</span>`
- Mevcut `openTypesModal()` ve `closeModal()` fonksiyonları korunur (tipler içeriği için kullanılmaya devam eder, sadece artık settings modal içinden çağrılır)

## Menü Öğeleri

### Tipler
- Mevcut `renderModal()` içeriği buraya taşınır
- Gelir Tipleri / Gider Tipleri sekmeleri, tip ekleme/düzenleme/silme aynı kalır
- `modalTab`, `typeEditingId` state değişkenleri korunur

### Remote DB
Alanlar:
- **Bin ID** — `<input type="text" id="settings-bin-id">`
- **X-Access-Key** — `<input type="password" id="settings-access-key">` + yanında toggle butonu (`👁`) ile `type` password/text arasında geçiş
- **Kaydet** butonu — değerleri `localStorage['financial-board-settings']` anahtarına `{ binId, accessKey }` formatında yazar
- Kaydet sonrası inline onay mesajı: "Kaydedildi ✓" (1.5s sonra kaybolur)

## localStorage

Yeni anahtar: `financial-board-settings`
```json
{ "binId": "6a684be2f5f4af5e29cb7996", "accessKey": "$2a$10$..." }
```

Mevcut `financial-board-v1` anahtarı değişmez.

## Remote Sync Değişikliği

`REMOTE_URL` ve `REMOTE_KEY` sabit değerleri kaldırılır. `syncFromRemote()` ve `syncToRemote()` çağrısında:
1. `localStorage['financial-board-settings']` okunur
2. `binId` ve `accessKey` alınır
3. Eksikse kullanıcıya "Lütfen Ayarlar > Remote DB bölümünden bağlantı bilgilerini girin." uyarısı gösterilir (işlem iptal)

## State

Yeni module-level değişken: `let settingsSection = 'types'` — `'types'` | `'remote'`

## Kapsam Dışı

- Diğer ayar grupları (tema, dil, bildirimler) — ileride eklenecek
- Mevcut hardcoded `REMOTE_URL` / `REMOTE_KEY` fallback — kaldırılır, ayar zorunlu hale gelir
