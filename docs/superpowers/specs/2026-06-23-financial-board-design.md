# Financial Board — Design Spec
Date: 2026-06-23

## Overview

A single `index.html` personal finance tracker. No build step, no dependencies, no server. Data persists in `localStorage`. Works offline in any modern browser.

## Layout

Three-column structure:
- **Left sidebar (~200px):** Year/month navigation tree. Years are collapsible. Active month is highlighted.
- **Right content area:** The selected month's full page.

## Month Page

### Summary Strip (sticky top)
Three stat cards always visible at the top:
- Total Income (green)
- Total Expenses (red)
- Remaining = Income − Expenses (blue, turns red if negative)

### Income Section
- Collapsible section header "GELİRLER" with section total
- Flat item list below
- "Yeni Gelir Ekle" button at the bottom of the list

### Expense Section
- Collapsible section header "GİDERLER" with section total
- Flat item list below
- "Yeni Gider Ekle" button at the bottom of the list

### Line Item
Each item displays:
```
[⠿ drag handle] [Tip etiketi] [Ad]   [Tutar]  [✏ düzenle] [🗑 sil]
```
Tip etiketi: colored pill badge, color comes from the type definition.

Clicking edit opens an inline edit form (same row expands) with fields: Ad, Tutar, Tip, Açıklama (optional).

## Drag & Drop

Native HTML5 Drag & Drop API. Incomes can be reordered within the income list; expenses within the expense list. Cross-section dragging (income ↔ expense) is not supported. Order is persisted to localStorage.

## Type Management

Accessible via a "⚙ Tipler" button in the top navigation bar. Opens a modal with two tabs: **Gelir Tipleri** and **Gider Tipleri**.

Each type has:
- Name (text)
- Color (color picker, used for the pill badge)
- Actions: edit, delete (deletion blocked if type is in use — shows warning)

### Default Income Types
| Name | Color |
|------|-------|
| Düzenli Maaş | #2563eb (blue) |
| Yan Gelir | #7c3aed (purple) |
| Kira Geliri | #059669 (green) |
| Yatırım | #d97706 (amber) |
| Diğer | #64748b (slate) |

### Default Expense Types
| Name | Color |
|------|-------|
| Konut | #dc2626 (red) |
| Market & Gıda | #ea580c (orange) |
| Faturalar | #ca8a04 (yellow) |
| Ulaşım | #0891b2 (cyan) |
| Sağlık | #db2777 (pink) |
| Eğitim | #4f46e5 (indigo) |
| Eğlence | #7c3aed (violet) |
| Diğer | #64748b (slate) |

## Data Model (localStorage)

```json
{
  "incomeTypes": [{ "id": "uuid", "name": "Maaş", "color": "#2563eb" }],
  "expenseTypes": [{ "id": "uuid", "name": "Konut", "color": "#dc2626" }],
  "months": {
    "2026-06": {
      "incomes": [
        { "id": "uuid", "name": "Maaş", "amount": 15000, "typeId": "uuid", "note": "", "order": 0 }
      ],
      "expenses": [
        { "id": "uuid", "name": "Kira", "amount": 8000, "typeId": "uuid", "note": "", "order": 0 }
      ]
    }
  }
}
```

## Theme

Light theme:
- Background: `#f8fafc`
- Cards/panels: `#ffffff` with `1px solid #e2e8f0` border
- Income amounts: `#16a34a`
- Expense amounts: `#dc2626`
- Remaining (positive): `#2563eb`, (negative): `#dc2626`
- Type badges: colored per type definition, with a light tinted background version for readability

## Navigation

On first load, the current month is auto-selected. Clicking any month in the sidebar loads it (creates an empty record if it doesn't exist yet).

Years listed: 2020 through (current year + 5), recalculated at page load — so in 2026 the list goes to 2031, in 2027 to 2032, etc. Current year is expanded by default; all years are navigable. December → January wraps to the next year correctly (no hardcoded upper bound).

## Error Handling

- Deleting a type that is in use: show inline warning "X kalem bu tipi kullanıyor. Önce kalemleri güncelleyin."
- Invalid amount (non-numeric or negative): inline field validation before save.
- localStorage unavailable: show a banner warning that data won't be saved.
