# Gider "Ödendi" İşareti Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Gider kalemlerine ✓ butonu ile "ödendi" işareti eklemek; ödenmişler üstü çizili ve listenin sonunda, geri alınan ödenmemişlerin sonunda.

**Architecture:** Kaleme `paid` alanı eklenir; sıra `expenses` dizisinde tutulur (önce ödenmemiş, sonra ödenmiş). Diziye kalem ekleyen her yol (`addItem`, periyot kopyaları, geri alma) ortak `insertUnpaid()` yardımcısıyla ödenmemişlerin sonuna ekler. Sürükle-bırak gruplar arası engellenir.

**Tech Stack:** Vanilla HTML/CSS/JS, tek dosya `index.html`.

**Spec:** `docs/superpowers/specs/2026-10-30-expense-paid-design.md`

## Global Constraints

- Toplamlar, yıl özeti, raporlar, pasta grafikler, birikimler `paid`'den etkilenmez.
- `paid` alanı yoksa ödenmemiş sayılır (migration yok).
- Commit mesajları conventional ve `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` ile biter.

---

### Task 1: Ödendi işareti

**Files:** Modify `index.html` — CSS (`.item-actions button:hover` sonrası), `renderItemList`, `cloneMonth`, `addItem`, `saveItem`, drag & drop fonksiyonları; yeni `insertUnpaid`, `togglePaid`.

**Interfaces:**
- `insertUnpaid(items: Item[], item: Item): number` — ödenmemişlerin sonuna ekler, eklendiği index'i döndürür.
- `togglePaid(id: string): void` — aktif aydaki gider kalemini id ile bulur, işareti çevirir ve taşır.

- [ ] **Step 1: CSS** — `.item-actions button:hover { … }` satırından sonra:

```css
    .item-row.paid .type-badge, .item-row.paid .item-name, .item-row.paid .item-amount { text-decoration: line-through; opacity: .55; }
    .item-note.paid { text-decoration: line-through; opacity: .55; }
    .item-actions button.paid-btn.active { background: var(--income); border-color: var(--income); color: #fff; }
```

- [ ] **Step 2: Satır render** — `renderItemList` içinde:

```js
        const paidCls = item.paid ? ' paid' : '';
        const noteHtml = item.note ? `<div class="item-note${paidCls}">${escHtml(item.note)}</div>` : '';
        const paidBtn = kind === 'expense'
          ? `<button class="paid-btn${item.paid ? ' active' : ''}" onclick="togglePaid('${item.id}')" title="${item.paid ? 'Ödenmedi olarak işaretle' : 'Ödendi olarak işaretle'}" aria-pressed="${item.paid ? 'true' : 'false'}">✓</button>`
          : '';
```

Satır açılışı `class="item-row${paidCls}"` ve `data-paid="${item.paid ? 1 : 0}"` alır; `.item-actions` içinde `${paidBtn}` ✏'dan önce gelir.

- [ ] **Step 3: Yardımcılar** — `cloneMonth` öncesine:

```js
    function insertUnpaid(items, item) {
      const at = items.filter(i => !i.paid).length;
      items.splice(at, 0, item);
      return at;
    }

    function togglePaid(id) {
      if (editingState) cancelEdit(editingState.kind, editingState.idx);
      const items = state.months[activeMonth].expenses;
      const idx = items.findIndex(i => i.id === id);
      if (idx === -1) return;
      const [item] = items.splice(idx, 1);
      item.paid = !item.paid;
      if (item.paid) items.push(item);
      else insertUnpaid(items, item);
      saveState();
      renderContent();
    }
```

- [ ] **Step 4: Ekleme yolları**
  - `cloneMonth`: `expenses: src.expenses.map(i => ({ ...i, id: uuid(), _isNew: false, paid: false }))`
  - `addItem`: `items.push({...}); editItem(kind, items.length - 1);` → `const idx = insertUnpaid(items, {...}); editItem(kind, idx);`
  - `saveItem` periyot kopyası: `state.months[nextKey][key].push({ ...baseItem, id: uuid() });` → `insertUnpaid(state.months[nextKey][key], { ...baseItem, id: uuid(), paid: false });`

- [ ] **Step 5: Sürükle-bırak** — `let dragSrcPaid = null;` eklenir; `onDragStart` `dragSrcPaid = e.currentTarget.dataset.paid;` atar; `onDragOver` ve `onDrop`'taki kind kontrolleri `|| target.dataset.paid !== dragSrcPaid` ile genişler; `onDrop` ve `onDragEnd` `dragSrcPaid = null;` sıfırlar.

- [ ] **Step 6: Doğrula** — spec'teki 9 madde (tarayıcıda, konsol + ekran görüntüsü).

- [ ] **Step 7: Commit** — `feat: mark expense items as paid`
