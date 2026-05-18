# Mobile-Responsive Layout Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the Strata CRM usable on phones by adding a bottom tab bar, hiding the sidebar, and making modals full-screen — all inside `@media (max-width: 767px)` so the desktop layout is completely untouched.

**Architecture:** Three additive changes to `index.html`: (1) a `@media (max-width: 767px)` CSS block added before `</style>`, (2) bottom tab bar + More menu HTML added before `</body>`, and (3) two JS functions (`mobileNav`, `toggleMobileMore`) added to the script section. The existing `showPage(id, btn)` navigation function is reused without modification.

**Tech Stack:** Vanilla CSS media queries, HTML, JavaScript — all changes in `C:\Users\zzthu\Downloads\Strata CRM\index.html`. No new libraries.

---

## File Map

**Single file modified:** `C:\Users\zzthu\Downloads\Strata CRM\index.html`

| Section | What changes |
|---|---|
| `<style>` block — before `</style>` (line ~382) | Add `@media (max-width: 767px)` CSS block |
| HTML body — before `</body>` (line ~3166) | Add `#mobile-tab-bar` + `#mobile-more-menu` HTML |
| `<script>` block — end of JS | Add `mobileNav()` + `toggleMobileMore()` functions |

---

## Task 1: Add Mobile CSS Block

**Files:**
- Modify: `index.html` — `<style>` block, immediately before `</style>` (line ~382)

Add the entire `@media (max-width: 767px)` block that controls all mobile layout changes.

- [ ] **Step 1: Find the insertion point**

Locate `</style>` in the file (line ~382 — it is the only `</style>` tag). The line just before it ends the `@media print` block with `}`.

- [ ] **Step 2: Insert the mobile CSS block immediately before `</style>`**

```css
/* ── MOBILE RESPONSIVE (max-width: 767px) ── */
@media (max-width: 767px) {
  /* 1. Hide sidebar, remove main offset */
  .sidebar { display: none !important; }
  .main { margin-left: 0 !important; }

  /* 2. Topbar — tighter on mobile */
  .topbar { padding: 0 14px; height: 48px; }
  .topbar-sub { display: none; }

  /* 3. Page padding — tighter */
  .page { padding: 12px 14px; }

  /* 4. Add bottom padding to main so content doesn't hide behind tab bar */
  .main { padding-bottom: 56px; }

  /* 5. Metric grid — 2 columns */
  .metric-grid { grid-template-columns: repeat(2, 1fr); }

  /* 6. Dashboard charts row — single column */
  .grid-2 { grid-template-columns: 1fr !important; }

  /* 7. Quick-peek drawer — full width (keep transform for animation) */
  .qp-drawer { width: 100%; }

  /* 8. Modals — full screen */
  .modal-overlay.open { align-items: flex-start; }
  .modal-overlay .modal {
    width: 100% !important;
    max-width: 100% !important;
    height: 100vh;
    max-height: 100vh;
    border-radius: 0;
    overflow-y: hidden;
    display: flex;
    flex-direction: column;
    box-shadow: none;
  }
  .modal-overlay .modal-body {
    flex: 1;
    overflow-y: auto;
    -webkit-overflow-scrolling: touch;
  }
  .modal-overlay .modal-head,
  .modal-overlay .modal-foot { flex-shrink: 0; }

  /* 9. Bottom tab bar — mobile only */
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
    transition: color 0.15s;
  }
  .mob-tab.active { color: var(--accent2); }
  .mob-tab-icon { font-size: 18px; line-height: 1; }
  .mob-tab-label { font-size: 10px; font-weight: 500; }

  /* 10. More menu */
  #mobile-more-menu {
    display: none;
    position: fixed;
    bottom: 56px; right: 0;
    width: 180px;
    background: var(--surface);
    border: 1px solid var(--border2);
    border-radius: var(--radius) var(--radius) 0 0;
    z-index: 201;
    padding: 6px 0;
    box-shadow: -4px -4px 16px rgba(0,0,0,0.1);
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
  #mobile-more-menu button:active { background: var(--surface2); }
}
```

- [ ] **Step 3: Verify CSS was inserted correctly**

Open `index.html` in Chrome. Open DevTools. Switch to device emulation (Ctrl+Shift+M), set width to `375px` (iPhone). Expected:
- Sidebar is hidden
- Content spans full width
- Page has `12px 14px` padding (visible in DevTools box model)
- Metric cards appear in 2 columns

- [ ] **Step 4: Commit**
```bash
cd "C:/Users/zzthu/Downloads/Strata CRM"
git add index.html
git commit -m "feat: add mobile CSS — sidebar hidden, full-width layout, responsive modals"
```

---

## Task 2: Add Bottom Tab Bar and More Menu HTML

**Files:**
- Modify: `index.html` — before `</body>` (line ~3166)

- [ ] **Step 1: Find the insertion point**

Locate `</body>` in the file (line ~3166 — second to last line).

- [ ] **Step 2: Insert the tab bar and more menu HTML immediately before `</body>`**

```html

<!-- Mobile Bottom Tab Bar -->
<div id="mobile-tab-bar" style="display:none">
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

<!-- Mobile More Menu (appears above tab bar) -->
<div id="mobile-more-menu" style="display:none">
  <button onclick="mobileNav('quotations')">📋 Quotations</button>
  <button onclick="mobileNav('providers')">🏥 Providers</button>
  <button onclick="mobileNav('settings')">⚙️ Settings</button>
</div>

```

Note: `style="display:none"` on `#mobile-tab-bar` hides it on desktop. The `@media (max-width: 767px)` CSS overrides this with `display: flex` on mobile.

- [ ] **Step 3: Verify elements exist**

Open `index.html` in Chrome. Open DevTools Console. Type:
```javascript
document.getElementById('mobile-tab-bar') !== null
```
Expected: `true`

```javascript
document.getElementById('mobile-more-menu') !== null
```
Expected: `true`

- [ ] **Step 4: Verify tab bar is visible on mobile**

In DevTools, switch to device emulation at 375px. The tab bar should appear fixed at the bottom of the screen with 5 items. Dashboard should be the active (blue) tab.

- [ ] **Step 5: Commit**
```bash
git add index.html
git commit -m "feat: add mobile bottom tab bar and More menu HTML"
```

---

## Task 3: Add `mobileNav()` and `toggleMobileMore()` Functions

**Files:**
- Modify: `index.html` — end of `<script>` block, before `</script>`

- [ ] **Step 1: Find the insertion point**

Locate the closing `</script>` tag (the last one in the file, near line 3165). Insert the following block immediately before it.

- [ ] **Step 2: Insert the two JS functions**

```javascript

// ── MOBILE NAVIGATION ──
function mobileNav(pageId) {
  // Close More menu if open
  const moreMenu = document.getElementById('mobile-more-menu');
  if (moreMenu) moreMenu.style.display = 'none';
  // Navigate using existing showPage — pass null for btn (sidebar hidden on mobile)
  showPage(pageId, null);
  // Update active tab highlight
  document.querySelectorAll('.mob-tab').forEach(t => t.classList.remove('active'));
  const primaryTab = document.querySelector(`.mob-tab[data-page="${pageId}"]`);
  if (primaryTab) primaryTab.classList.add('active');
}

function toggleMobileMore(btn) {
  const menu = document.getElementById('mobile-more-menu');
  if (!menu) return;
  const isOpen = menu.style.display === 'block';
  menu.style.display = isOpen ? 'none' : 'block';
  // Toggle active state on the More tab
  document.querySelectorAll('.mob-tab').forEach(t => t.classList.remove('active'));
  if (!isOpen) btn.classList.add('active');
}

// Close More menu when tapping anywhere outside it
document.addEventListener('click', function(e) {
  const menu = document.getElementById('mobile-more-menu');
  const moreBtn = document.querySelector('.mob-tab[data-page="more"]');
  if (menu && menu.style.display === 'block') {
    if (!menu.contains(e.target) && e.target !== moreBtn && !moreBtn.contains(e.target)) {
      menu.style.display = 'none';
      document.querySelectorAll('.mob-tab').forEach(t => t.classList.remove('active'));
      // Re-highlight the currently active page tab
      const activePage = document.querySelector('.page.active');
      if (activePage) {
        const pageId = activePage.id.replace('page-', '');
        const tab = document.querySelector(`.mob-tab[data-page="${pageId}"]`);
        if (tab) tab.classList.add('active');
      }
    }
  }
});
```

- [ ] **Step 3: Verify functions exist**

Open `index.html`. Open DevTools Console. Type:
```javascript
typeof mobileNav
```
Expected: `"function"`

```javascript
typeof toggleMobileMore
```
Expected: `"function"`

- [ ] **Step 4: Test navigation on mobile**

In DevTools device emulation (375px):
1. Tap **Clients** tab → Clients page loads, Clients tab turns blue
2. Tap **Renewals** tab → Renewals page loads, Renewals tab turns blue
3. Tap **COA** tab → COA page loads, COA tab turns blue
4. Tap **⋯ More** → More menu appears above tab bar with Quotations, Providers, Settings
5. Tap **Quotations** in More menu → Quotations page loads, More menu closes
6. Tap anywhere outside More menu → menu closes

- [ ] **Step 5: Commit**
```bash
git add index.html
git commit -m "feat: add mobileNav and toggleMobileMore JS functions"
```

---

## Task 4: Verify Full Mobile Experience and Push

**Files:**
- No new changes — verification and push only

- [ ] **Step 1: Full mobile layout check (375px — iPhone size)**

Open `index.html` in Chrome DevTools device emulation at **375px width**.

| Check | Expected |
|---|---|
| Sidebar | Hidden — not visible at all |
| Content | Full-width, 12px/14px padding |
| Bottom tab bar | Visible, fixed at bottom, 5 tabs |
| Dashboard tab | Active (blue) by default |
| Metric cards | 2 columns (not 4) |
| Dashboard charts | Stacked single column |

- [ ] **Step 2: Modal check on mobile**

Click Edit on any existing client record:
- Modal opens full-screen (covers entire viewport)
- Header stuck at top
- Fields scroll in the middle
- Footer with Save/Cancel/Delete stuck at bottom
- No horizontal overflow

- [ ] **Step 3: More menu check**

Tap ⋯ More:
- Panel appears above tab bar (right side)
- Shows: 📋 Quotations, 🏥 Providers, ⚙️ Settings
- Tap Quotations → navigates to Quotations page, menu closes
- Tap outside menu → menu closes

- [ ] **Step 4: Quick-peek drawer check**

Tap any client name to open the quick-peek drawer:
- Drawer slides in from the right, full-width on mobile
- Tap the overlay to close

- [ ] **Step 5: Dark mode check on mobile**

Toggle dark mode — all mobile elements (tab bar, More menu, modals) should switch correctly since they use CSS variables.

- [ ] **Step 6: Desktop regression check**

Switch DevTools back to full desktop width (1280px+):
- Sidebar is visible
- Tab bar is hidden
- Modals are normal size (580px centered)
- Metric grid is 4 columns
- Everything looks exactly as before

- [ ] **Step 7: Push to GitHub**
```bash
git push origin main
```

---

## Summary of All Changes

| What | Where | Risk |
|---|---|---|
| `@media (max-width: 767px)` CSS block | Before `</style>` | Zero — scoped to mobile only |
| `#mobile-tab-bar` HTML | Before `</body>` | Zero — new element, hidden on desktop |
| `#mobile-more-menu` HTML | Before `</body>` | Zero — new element, hidden on desktop |
| `mobileNav()` function | End of `<script>` | Zero — new function, calls existing `showPage()` |
| `toggleMobileMore()` function | End of `<script>` | Zero — new function |
| Outside-click listener | End of `<script>` | Minimal — only fires when menu is open |
