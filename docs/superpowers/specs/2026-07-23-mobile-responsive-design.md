# Mobile Responsive Design — Finansal Pano
Date: 2026-07-23

## Overview

Single `index.html` uygulamasını mobil cihazlarda (≤ 640px) kullanılabilir hale getirme. Mevcut sabit-genişlik sidebar + içerik layout'u, hamburger drawer + responsive bileşenlerle değiştirilecek.

## Breakpoint

`640px` — bu değerin altındaki ekranlar mobil layout'a geçer.

## Topbar (mobil)

- Sol köşeye `☰` hamburger butonu eklenir (`#btn-menu`, yalnızca mobilde görünür)
- "⬇ Dışa Aktar", "⬆ İçe Aktar", "⚙ Tipler" butonlarının **metin etiketleri** gizlenir; yalnızca ikonlar (`⬇`, `⬆`, `⚙`) kalır
- Butonlar topbar'da kalır (sidebar'a taşınmaz)

## Sidebar Drawer (mobil)

- Desktop'ta değişiklik yok (`position: static`, her zaman görünür)
- Mobilde:
  - `position: fixed; top: 52px; left: -210px; height: calc(100vh - 52px); z-index: 200`
  - `transition: left .25s ease`
  - `.drawer-open` class'ı ile `left: 0` (kayarak açılır)
- **Backdrop:** `#drawer-backdrop` — `position: fixed; inset: 0; z-index: 199; background: rgba(0,0,0,.35)`; yalnızca `.drawer-open` aktifken görünür
- Backdrop'a tıklayınca drawer kapanır
- `body.drawer-open` → `overflow: hidden` (arka plan scroll'u engellenir)
- Bir aya tıklandığında drawer otomatik kapanır

## Modal (mobil)

- `width: 100%; max-width: 100%; height: 100vh; max-height: 100vh; border-radius: 0`
- Tam ekran açılır

## Summary Strip (mobil)

- `flex-direction: column` → üç stat kart dikey sıralanır

## Pie Chart'lar (mobil)

- Her chart `width: 100%` alır → alt alta dizilir
- Zaten `flex-wrap: wrap` var, min-width sıfırlanır

## Edit Form (mobil)

- Mevcut `flex-wrap: wrap` yeterli
- Input genişlikleri `width: 100%` → tam satır kaplar

## JS Değişiklikleri

- `toggleDrawer()` fonksiyonu: `body.classList.toggle('drawer-open')`, backdrop göster/gizle
- `closeDrawer()`: `selectMonth()` çağrısında otomatik tetiklenir
- Event listener: backdrop click → `closeDrawer()`
- Event listener: `#btn-menu` click → `toggleDrawer()`

## Kapsam Dışı

- Tablet (641–1024px) için ayrı layout (mevcut desktop layout yeterli)
- Dark mode
- Touch gesture (swipe to close)
