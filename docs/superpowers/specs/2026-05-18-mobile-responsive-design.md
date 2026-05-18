# Strata CRM — Mobile-Responsive Layout Design Spec
**Date:** 2026-05-18
**Version:** 1.0
**Status:** Approved
**Scope:** Mobile-responsive layout for phones (< 768px). Desktop unchanged.

---

## Overview

Add mobile responsiveness to the Strata CRM using CSS media queries and a new bottom tab bar. All changes are contained within `@media (max-width: 767px)` rules — zero risk to the existing desktop layout. One new HTML element (`#mobile-tab-bar`) is added to the page. Existing JavaScript logic, data handling, and Google Sheets sync are untouched.

---

## Breakpoints

| Range | Layout |
|---|---|
| `< 768px` | Mobile layout (this spec) |
| `768px – 1024px` | Desktop layout (no changes) |
| `> 1024px` | Desktop layout (no changes) |

---

## 1. Sidebar

**Mobile behavior:** Hidden completely via `display: none`.

**Main content offset:** `.main { margin-left: 0 }` on mobile — content fills full screen width.

**CSS:**
```css
@media (max-width: 767px) {
  .sidebar { display: none; }
  .main { margin-left: 0; }
}
```

---

## 2. Bottom Tab Bar

### HTML Structure
A new `<div id="mobile-tab-bar">` element added to the HTML, as the last child of `<body>` before `</body>`. Visible only on mobile via CSS.

```html
<div id="mobile-tab-bar">
  <button class="mob-tab active" data-page="dashboard" onclick="mobileNav('dashboard')">
    <span class="mob-tab-icon">📊</span>
    <span class="mob-tab-label">Dashboard</span>
  </button>
  <button class="mob-tab" data-page="clients" onclick="mobileNav('clients')">
    <span class="mob-tab-icon">👥</span>
    <span class="mob-tab-label">Clients</span>
  </button>
  <button class="mob-tab" data-page="renewals" onclick="mobileNav('renewals')">
    <span class="mob-tab-icon">🔄</span>
    <span class="mob-tab-label">Renewals</span>
  </button>
  <button class="mob-tab" data-page="coa" onclick="mobileNav('coa')">
    <span class="mob-tab-icon">✅</span>
    <span class="mob-tab-label">COA</span>
  </button>
  <button class="mob-tab" data-page="more" onclick="toggleMobileMore(this)">
    <span class="mob-tab-icon">⋯</span>
    <span class="mob-tab-label">More</span>
  </button>
</div>
```

### CSS
```css
#mobile-tab-bar {
  display: none; /* hidden on desktop */
}
@media (max-width: 767px) {
  #mobile-tab-bar {
    display: flex;
    position: fixed;
    bottom: 0; left: 0; right: 0;
    height: 56px;
    background: var(--surface);
    border-top: 1px solid var(--border2);
    z-index: 200;
    align-items: stretch;
  }
  .mob-tab {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 2px;
    background: none;
    border: none;
    cursor: pointer;
    color: var(--text3);
    font-family: 'DM Sans', sans-serif;
    padding: 4px 0;
  }
  .mob-tab.active {
    color: var(--accent2);
  }
  .mob-tab-icon { font-size: 18px; line-height: 1; }
  .mob-tab-label { font-size: 10px; font-weight: 500; }
}
```

### "More" Menu
A small panel (`#mobile-more-menu`) appears above the tab bar when "More ⋯" is tapped. Contains: Quotations, Providers, Settings. Tapping any item navigates and closes the menu. Tapping outside also closes it.

```html
<div id="mobile-more-menu" style="display:none">
  <button onclick="mobileNav('quotations')">📋 Quotations</button>
  <button onclick="mobileNav('providers')">🏥 Providers</button>
  <button onclick="mobileNav('settings')">⚙️ Settings</button>
</div>
```

```css
@media (max-width: 767px) {
  #mobile-more-menu {
    position: fixed;
    bottom: 56px; right: 0;
    width: 180px;
    background: var(--surface);
    border: 1px solid var(--border2);
    border-radius: var(--radius) var(--radius) 0 0;
    z-index: 201;
    padding: 8px 0;
  }
  #mobile-more-menu button {
    display: block;
    width: 100%;
    padding: 12px 16px;
    background: none;
    border: none;
    text-align: left;
    font-family: 'DM Sans', sans-serif;
    font-size: 14px;
    color: var(--text);
    cursor: pointer;
  }
  #mobile-more-menu button:hover { background: var(--surface2); }
}
```

### JavaScript
```javascript
function mobileNav(pageId) {
  // Close more menu if open
  document.getElementById('mobile-more-menu').style.display = 'none';
  // Use existing showPage(id, btn) — pass null for btn (sidebar is hidden on mobile)
  showPage(pageId, null);
  // Update active tab highlight
  document.querySelectorAll('.mob-tab').forEach(t => t.classList.remove('active'));
  const primaryTab = document.querySelector(`.mob-tab[data-page="${pageId}"]`);
  if (primaryTab) primaryTab.classList.add('active');
}

function toggleMobileMore(btn) {
  const menu = document.getElementById('mobile-more-menu');
  const isOpen = menu.style.display !== 'none';
  menu.style.display = isOpen ? 'none' : 'block';
  document.querySelectorAll('.mob-tab').forEach(t => t.classList.remove('active'));
  if (!isOpen) btn.classList.add('active');
}
```

### Sync with existing `showPage(id, btn)`
The existing `showPage(id, btn)` function handles page switching and already guards against `btn` being null (`if(btn) btn.classList.add('active')`). `mobileNav()` calls it with `null` as the btn — no changes to `showPage()` needed.

---

## 3. Topbar

**Mobile behavior:** Stays visible. Height reduces slightly. Sync status text hidden (icon only) on very small screens.

```css
@media (max-width: 767px) {
  .topbar { padding: 0 14px; height: 48px; }
  .topbar-sub { display: none; } /* hide subtitle text */
  .main { padding-bottom: 56px; } /* prevent content hiding behind tab bar */
}
```

---

## 4. Modals — Full Screen on Mobile

All modals (`.modal-overlay .modal`) go full-screen on mobile.

```css
@media (max-width: 767px) {
  .modal-overlay .modal {
    width: 100% !important;
    max-width: 100% !important;
    height: 100%;
    max-height: 100%;
    border-radius: 0;
    margin: 0;
    display: flex;
    flex-direction: column;
  }
  .modal-overlay {
    align-items: flex-start;
    padding: 0;
  }
  .modal-body {
    flex: 1;
    overflow-y: auto;
  }
  .modal-foot {
    flex-shrink: 0;
  }
  .modal-head {
    flex-shrink: 0;
  }
}
```

---

## 5. Page Padding

```css
@media (max-width: 767px) {
  .page { padding: 12px 14px; }
}
```

---

## 6. Metric Grid — 2 Columns

```css
@media (max-width: 767px) {
  .metric-grid { grid-template-columns: repeat(2, 1fr); }
}
```

---

## 7. Quick-Peek Drawer — Full Width

```css
@media (max-width: 767px) {
  .qp-drawer { width: 100%; right: -100%; }
  .qp-drawer.open { right: 0; }
}
```

---

## 8. Grid-2 (Dashboard charts row) — Single Column

```css
@media (max-width: 767px) {
  .grid-2 { grid-template-columns: 1fr; }
}
```

---

## What Does NOT Change

- All desktop CSS — completely untouched
- All JavaScript business logic
- Google Sheets sync, write-back, history logging
- Authentication flow
- All form/modal functionality
- Dark mode
- Print styles

---

## Out of Scope

- Tablet-specific layout (768px–1024px uses desktop layout)
- Touch gestures (swipe to dismiss modals)
- Push notifications
- Offline PWA functionality
