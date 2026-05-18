# Export, Renewal Summary & Print Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add filter-aware CSV export to all 4 data pages, a renewal alerts summary panel on the Renewals page, and print buttons on the Dashboard and Renewals pages — all as purely additive changes to `working_v2.8_1.html`.

**Architecture:** All changes are made inside the single HTML file. A shared `_exportData` object caches the last filtered array from each render function so export buttons always reflect what's on screen. The renewal summary panel is a new `<div>` inserted above the existing renewals table, populated by a new `renderRenewalSummary()` function called from the existing `renderRenewals()`. Print uses browser-native `window.print()` with a `@media print` CSS block.

**Tech Stack:** Vanilla JavaScript, HTML, CSS — no new libraries. All changes are in `working_v2.8_1.html`.

---

## File Map

**Single file modified:** `working_v2.8_1.html`

Changes by section:

| Section | What changes |
|---|---|
| `<style>` block (ends line ~341) | Add `@media print` block + `.print-header` class |
| Dashboard HTML (~line 447) | Add print button div + two print-only fallback divs for charts |
| Clients filter-bar (line 556) | Add Export CSV button |
| Quotations filter-bar (line 624) | Add Export CSV button |
| COA filter-bar (line 647) | Add Export CSV button |
| Renewals page (line 658) | Add renewal summary div + Export CSV + Print buttons to filter-bar |
| `<script>` block — near top globals | Add `_exportData` object |
| `<script>` block — after `renderRenewals` | Add `exportToCSV()` utility + 4 named export functions + `renderRenewalSummary()` |
| `renderClients()` (~line 1754) | Save filtered array to `_exportData.clients` |
| `renderQuotations()` (~line 1829) | Save filtered array to `_exportData.quotations` |
| `renderCOA()` (~line 1870) | Save filtered array to `_exportData.coa` |
| `renderRenewals()` (~line 2045) | Save filtered array to `_exportData.renewals` + call `renderRenewalSummary()` |

---

## Task 1: Add `_exportData` cache variable

**Files:**
- Modify: `working_v2.8_1.html` — JavaScript globals section

This adds the shared object that render functions write to and export functions read from.

- [ ] **Step 1: Find the globals section**

Search for this line in the file (it's near the top of the `<script>` block, around line 1370):
```javascript
const TODAY = new Date();
```

- [ ] **Step 2: Add `_exportData` immediately after `TODAY`**

Add this line directly after `const TODAY = new Date();`:
```javascript
const _exportData = { clients: [], quotations: [], coa: [], renewals: [] };
```

- [ ] **Step 3: Verify in browser**

Open `working_v2.8_1.html` in Chrome. Open DevTools Console (F12). Type:
```javascript
_exportData
```
Expected output: `{clients: Array(0), quotations: Array(0), coa: Array(0), renewals: Array(0)}`

- [ ] **Step 4: Commit**
```bash
git add working_v2.8_1.html
git commit -m "feat: add _exportData cache variable for filter-aware CSV export"
```

---

## Task 2: Add `exportToCSV()` utility function

**Files:**
- Modify: `working_v2.8_1.html` — JavaScript, after the `renderRenewals()` function (~line 2046)

- [ ] **Step 1: Find the insertion point**

Locate this comment in the file (just after `renderRenewals()` ends):
```javascript
// ── PROVIDERS ──
function renderProviders(){
```

- [ ] **Step 2: Insert `exportToCSV()` before that comment**

```javascript
// ── CSV EXPORT ──
function exportToCSV(headers, rows, filename) {
  const escape = val => {
    if (val === null || val === undefined) return '';
    const s = String(val);
    return (s.includes(',') || s.includes('"') || s.includes('\n'))
      ? '"' + s.replace(/"/g, '""') + '"'
      : s;
  };
  const lines = [
    headers.map(escape).join(','),
    ...rows.map(r => r.map(escape).join(','))
  ];
  const blob = new Blob([lines.join('\r\n')], { type: 'text/csv;charset=utf-8;' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}

function todayStr() {
  const d = new Date();
  return d.getFullYear() + '-' +
    String(d.getMonth() + 1).padStart(2, '0') + '-' +
    String(d.getDate()).padStart(2, '0');
}

function exportClients() {
  const headers = ['ID','Name','Contact Person','Mobile','Email','Business Type',
    'Insurance Type','Current Provider','Renewal Date','Pipeline Status',
    'Last Contact','Next Follow-Up','Assigned','Notes'];
  const rows = _exportData.clients.map(c => [
    c.id, c.name, c.contact, c.mobile, c.email, c.bizType,
    c.insType, c.currentProvider, c.renewal, c.status,
    c.lastContact, c.nextFollowUp, c.assigned, c.notes
  ]);
  exportToCSV(headers, rows, 'strata-clients-' + todayStr() + '.csv');
}

function exportQuotations() {
  const headers = ['ID','Client','Provider','Franchise Date','Franchise End',
    'Initial Email Sent','TAT Date','Status','Date Released','Rates',
    'TOR/SOB','Special Requests','Follow-Up Date'];
  const rows = _exportData.quotations.map(q => [
    q.id, q.client, q.provider, q.franchiseDate, q.franchiseEnd,
    q.initialEmail, q.tat, q.status, q.dateReleased, q.rates,
    q.torSob, q.requests, q.followUp
  ]);
  exportToCSV(headers, rows, 'strata-quotations-' + todayStr() + '.csv');
}

function exportCOA() {
  const headers = ['Date','Client','Contact Type','Task Description','Assigned','Completed Date'];
  const rows = _exportData.coa.map(c => [
    c.date, c.client, c.contactType, c.task, c.assigned, c.completed
  ]);
  exportToCSV(headers, rows, 'strata-coa-' + todayStr() + '.csv');
}

function exportRenewals() {
  const headers = ['Client','Insurance Type','Current Provider','Renewal Date',
    'Days Remaining','Urgency','Assigned'];
  const rows = _exportData.renewals.map(r => [
    r.name, r.insType, r.currentProvider, r.renewal,
    r.days !== null ? r.days : '', r.u ? r.u.label : '', r.assigned
  ]);
  exportToCSV(headers, rows, 'strata-renewals-' + todayStr() + '.csv');
}
```

- [ ] **Step 3: Verify syntax in browser**

Open the file in Chrome. Open DevTools Console. Type:
```javascript
typeof exportToCSV
```
Expected: `"function"`

Also type:
```javascript
typeof exportClients
```
Expected: `"function"`

- [ ] **Step 4: Commit**
```bash
git add working_v2.8_1.html
git commit -m "feat: add exportToCSV utility and named export functions"
```

---

## Task 3: Wire `_exportData` into render functions

**Files:**
- Modify: `working_v2.8_1.html` — inside `renderClients()`, `renderQuotations()`, `renderCOA()`, `renderRenewals()`

- [ ] **Step 1: Update `renderClients()`**

Find this line inside `renderClients()` (around line 1727):
```javascript
  document.getElementById('c-count').textContent=`${f.length} of ${CLIENTS.length}`;
```

Add this line **immediately before** it:
```javascript
  _exportData.clients = f;
```

- [ ] **Step 2: Update `renderQuotations()`**

Find this line inside `renderQuotations()` (around line 1824):
```javascript
  document.getElementById('q-count').textContent=`${f.length} quotations`;
```

Add this line **immediately before** it:
```javascript
  _exportData.quotations = f;
```

- [ ] **Step 3: Update `renderCOA()`**

Find this line inside `renderCOA()` (around line 1854):
```javascript
  document.getElementById('coa-count').textContent=`${f.length} tasks`;
```

Add this line **immediately before** it:
```javascript
  _exportData.coa = f;
```

- [ ] **Step 4: Update `renderRenewals()`**

Find this line inside `renderRenewals()` (around line 2044):
```javascript
  document.getElementById('r-count').textContent=`${f.length} clients`;
```

Add this line **immediately before** it:
```javascript
  _exportData.renewals = f;
```

- [ ] **Step 5: Verify in browser**

Open `working_v2.8_1.html`. Navigate to the Clients page. Open DevTools Console. Type:
```javascript
_exportData.clients.length
```
Expected: a number equal to the number of rows visible in the table (e.g. `5`).

Apply a filter (e.g. select "HMO" from the insurance type dropdown). Type again:
```javascript
_exportData.clients.length
```
Expected: a smaller number matching the filtered rows.

- [ ] **Step 6: Commit**
```bash
git add working_v2.8_1.html
git commit -m "feat: cache filtered arrays in _exportData on each render"
```

---

## Task 4: Add Export CSV buttons to all 4 pages

**Files:**
- Modify: `working_v2.8_1.html` — 4 filter-bar sections in the HTML body

- [ ] **Step 1: Clients page — add Export CSV button**

Find this exact line in the HTML (around line 556):
```html
        <button class="btn btn-primary" onclick="openClientModal(null)">+ Add client</button>
```

Add the Export button **immediately before** it:
```html
        <button class="btn no-print" onclick="exportClients()" title="Export visible rows to CSV">⬇ Export CSV</button>
```

Result after change:
```html
        <button class="btn no-print" onclick="exportClients()" title="Export visible rows to CSV">⬇ Export CSV</button>
        <button class="btn btn-primary" onclick="openClientModal(null)">+ Add client</button>
```

- [ ] **Step 2: Quotations page — add Export CSV button**

Find this exact line in the HTML (around line 624):
```html
        <button class="btn btn-primary" onclick="openQuoteModal(null)">+ Add quotation</button>
```

Add the Export button **immediately before** it:
```html
        <button class="btn no-print" onclick="exportQuotations()" title="Export visible rows to CSV">⬇ Export CSV</button>
```

- [ ] **Step 3: COA page — add Export CSV button**

Find this exact line in the HTML (around line 647):
```html
        <button class="btn btn-primary" onclick="openCOAModal(null)">+ Add COA task</button>
```

Add the Export button **immediately before** it:
```html
        <button class="btn no-print" onclick="exportCOA()" title="Export visible rows to CSV">⬇ Export CSV</button>
```

- [ ] **Step 4: Renewals page — add Export CSV + Print buttons**

Find this exact section in the HTML (around line 661–673). The filter-bar currently ends with:
```html
        <select id="r-assigned" onchange="renderRenewals()"><option value="">All assignees</option></select>
      </div>
```

Add both buttons after the `r-assigned` select and before the closing `</div>`:
```html
        <select id="r-assigned" onchange="renderRenewals()"><option value="">All assignees</option></select>
        <button class="btn no-print" onclick="exportRenewals()" title="Export visible rows to CSV">⬇ Export CSV</button>
        <button class="btn no-print" onclick="window.print()" title="Print renewal alerts report">🖨 Print</button>
      </div>
```

- [ ] **Step 5: Verify all 4 export buttons in browser**

Open `working_v2.8_1.html`. Check each page (Clients, Quotations, COA Tasks, Renewals) — each should show an "⬇ Export CSV" button in the toolbar.

- [ ] **Step 6: Test a download**

Go to the Clients page. Click "⬇ Export CSV". Expected: a file named `strata-clients-YYYY-MM-DD.csv` downloads. Open it in Excel or a text editor — verify it has the correct headers and one row per visible client.

Apply the "HMO" filter. Click Export again. Expected: only HMO clients appear in the downloaded file.

- [ ] **Step 7: Commit**
```bash
git add working_v2.8_1.html
git commit -m "feat: add filter-aware Export CSV buttons to Clients, Quotations, COA, and Renewals pages"
```

---

## Task 5: Add Renewal Alerts Summary Panel

**Files:**
- Modify: `working_v2.8_1.html` — Renewals page HTML + `renderRenewals()` JavaScript

- [ ] **Step 1: Add summary panel div to Renewals page HTML**

Find this section in the HTML (around line 658–660):
```html
    <!-- RENEWALS -->
    <div class="page" id="page-renewals">
      <div class="metric-grid" id="ren-metrics"></div>
      <div class="filter-bar">
```

Insert a new `<div id="ren-summary">` **between** `ren-metrics` and the `filter-bar`:
```html
    <!-- RENEWALS -->
    <div class="page" id="page-renewals">
      <div class="metric-grid" id="ren-metrics"></div>
      <div id="ren-summary"></div>
      <div class="filter-bar">
```

- [ ] **Step 2: Add `renderRenewalSummary()` function**

After the `exportRenewals()` function you added in Task 2, add:

```javascript
function renderRenewalSummary() {
  const el = document.getElementById('ren-summary');
  if (!el) return;
  const allRows = CLIENTS
    .map(c => { const d = pd(c.renewal); const days = dd(d); return {...c, days, u: urg(days)}; })
    .filter(r => r.days !== null && r.days >= 0)
    .sort((a, b) => a.days - b.days);

  const critical = allRows.filter(r => r.u.cat === 'critical');
  const soon     = allRows.filter(r => r.u.cat === 'soon');
  const upcoming = allRows.filter(r => r.u.cat === 'upcoming');
  const future   = allRows.filter(r => r.u.cat === 'future');

  const dateRange = arr => {
    if (!arr.length) return '';
    const first = fmt(pd(arr[0].renewal));
    const last  = fmt(pd(arr[arr.length - 1].renewal));
    return first === last ? first : first + ' – ' + last;
  };

  const clientRow = c =>
    `<div style="display:flex;justify-content:space-between;align-items:center;padding:5px 0;border-bottom:1px solid var(--border)">
      <span style="font-weight:500">${c.name}</span>
      <span style="font-size:12px;color:var(--text2)">${fmt(pd(c.renewal))} · ${c.insType||'—'} · ${c.assigned||'<span style="color:var(--text3)">unassigned</span>'}</span>
    </div>`;

  const upcomingToggleId = 'ren-summary-upcoming-detail';
  const futureCount = future.length;

  el.innerHTML = `
    <div class="card" style="margin-bottom:14px" id="ren-summary-card">
      <div class="card-head no-print" style="padding-bottom:8px">
        <h3>Renewal Alerts Summary</h3>
        <span style="font-size:11px;color:var(--text3)">Excludes expired · ${allRows.length} active</span>
      </div>
      <div class="print-header" style="display:none;padding:0 16px 8px">
        <div style="font-size:13px;font-weight:600">Strata Consultancy Group · Renewal Alerts · ${new Date().toLocaleDateString('en-PH',{month:'long',day:'numeric',year:'numeric'})}</div>
      </div>
      <div style="padding:0 16px 12px">

        <!-- STAT CARDS -->
        <div style="display:flex;gap:10px;margin-bottom:14px;flex-wrap:wrap">
          <div style="flex:1;min-width:80px;background:var(--red-bg);border-left:3px solid var(--red);border-radius:var(--radius-sm);padding:10px;text-align:center">
            <div style="font-size:22px;font-weight:700;color:var(--red)">${critical.length}</div>
            <div style="font-size:11px;color:var(--red);font-weight:500">Critical</div>
            <div style="font-size:10px;color:var(--text3)">≤14 days</div>
          </div>
          <div style="flex:1;min-width:80px;background:var(--amber-bg);border-left:3px solid var(--amber);border-radius:var(--radius-sm);padding:10px;text-align:center">
            <div style="font-size:22px;font-weight:700;color:var(--amber)">${soon.length}</div>
            <div style="font-size:11px;color:var(--amber);font-weight:500">Soon</div>
            <div style="font-size:10px;color:var(--text3)">15–30 days</div>
          </div>
          <div style="flex:1;min-width:80px;background:var(--blue-bg);border-left:3px solid var(--blue);border-radius:var(--radius-sm);padding:10px;text-align:center">
            <div style="font-size:22px;font-weight:700;color:var(--blue)">${upcoming.length}</div>
            <div style="font-size:11px;color:var(--blue);font-weight:500">Upcoming</div>
            <div style="font-size:10px;color:var(--text3)">31–90 days</div>
          </div>
          <div style="flex:1;min-width:80px;background:var(--gray-bg);border-left:3px solid var(--gray);border-radius:var(--radius-sm);padding:10px;text-align:center">
            <div style="font-size:22px;font-weight:700;color:var(--gray)">${futureCount}</div>
            <div style="font-size:11px;color:var(--gray);font-weight:500">Future</div>
            <div style="font-size:10px;color:var(--text3)">91+ days</div>
          </div>
        </div>

        <!-- CRITICAL GROUP -->
        ${critical.length ? `
          <div style="margin-bottom:10px">
            <div style="font-size:11px;font-weight:600;color:var(--red);text-transform:uppercase;letter-spacing:0.5px;margin-bottom:6px">🔴 Critical (≤14 days)</div>
            ${critical.map(clientRow).join('')}
          </div>` : ''}

        <!-- SOON GROUP -->
        ${soon.length ? `
          <div style="margin-bottom:10px">
            <div style="font-size:11px;font-weight:600;color:var(--amber);text-transform:uppercase;letter-spacing:0.5px;margin-bottom:6px">🟠 Soon (15–30 days)</div>
            ${soon.map(clientRow).join('')}
          </div>` : ''}

        <!-- UPCOMING GROUP (collapsible on screen via toggle; CSS expands on print) -->
        ${upcoming.length ? `
          <div style="margin-bottom:10px">
            <div style="font-size:11px;font-weight:600;color:var(--blue);text-transform:uppercase;letter-spacing:0.5px;margin-bottom:6px">🔵 Upcoming (31–90 days)</div>
            <button class="btn no-print" style="font-size:11px;height:24px;padding:0 10px;margin-bottom:6px" onclick="
              var d=document.getElementById('${upcomingToggleId}');
              var b=this;
              if(d.style.display==='none'){d.style.display='block';b.textContent='▾ Collapse';}
              else{d.style.display='none';b.textContent='▸ Show ${upcoming.length} clients';}
            ">▸ Show ${upcoming.length} clients</button>
            <div id="${upcomingToggleId}" style="display:none">
              ${upcoming.map(clientRow).join('')}
            </div>
          </div>` : ''}

        <!-- FUTURE GROUP -->
        ${futureCount ? `
          <div style="color:var(--text3);font-size:12px;padding:4px 0">
            <span style="font-weight:500">Future (91+ days):</span> ${futureCount} client${futureCount===1?'':'s'} renewing after ${dateRange(upcoming.concat(future).slice(-1))||'90 days'}
          </div>` : ''}

      </div>
    </div>`;
}
```

- [ ] **Step 3: Call `renderRenewalSummary()` from `renderRenewals()`**

Find the end of `renderRenewals()`. It ends with the line that sets `r-tbody` innerHTML (around line 2045):
```javascript
  document.getElementById('r-tbody').innerHTML=f.length?f.map(r=>`...`).join(''):noData(9);
}
```

Add a call to `renderRenewalSummary()` **on the line before the closing `}`**:
```javascript
  document.getElementById('r-tbody').innerHTML=f.length?f.map(r=>`...`).join(''):noData(9);
  renderRenewalSummary();
}
```

- [ ] **Step 4: Verify in browser**

Open `working_v2.8_1.html`. Navigate to the Renewals page. Expected: A card appears above the filter bar showing 4 colored stat tiles (Critical/Soon/Upcoming/Future) and grouped client lists below them.

Check: If there are Upcoming clients, a "▸ Show N clients" button appears. Click it — the list expands. Click "▾ Collapse" — it collapses again.

Check dark mode: Toggle dark mode — stat card colors should follow the CSS variables and look correct.

- [ ] **Step 5: Commit**
```bash
git add working_v2.8_1.html
git commit -m "feat: add renewal alerts summary panel above renewals table"
```

---

## Task 6: Add Print CSS

**Files:**
- Modify: `working_v2.8_1.html` — `<style>` block, just before `</style>` (line ~341)

- [ ] **Step 1: Find the end of the style block**

Locate `</style>` in the file (around line 341). It should be the only one.

- [ ] **Step 2: Insert `@media print` block immediately before `</style>`**

```css
/* ── PRINT STYLES ── */

/* Screen: hide print-only elements */
@media screen {
  .print-only,
  #dash-print-header,
  #dash-print-charts { display: none !important; }
}

/* Print: hide screen chrome, show print elements */
@media print {
  /* Hide navigation and interactive controls */
  .sidebar, .topbar, .no-print,
  .filter-bar, .stab-row, #ren-metrics { display: none !important; }

  /* Remove sidebar offset */
  .main { margin-left: 0 !important; padding: 0 !important; }
  .page { padding: 20px !important; }

  /* Show print-only elements */
  .print-only { display: block !important; }
  #dash-print-header { display: block !important; margin-bottom: 16px; }
  #dash-print-charts { display: block !important; margin-bottom: 16px; }
  .print-header { display: block !important; }

  /* Clean card styling */
  .card { box-shadow: none !important; border: 1px solid #ccc !important; margin-bottom: 12px !important; }
  body { background: #fff !important; color: #000 !important; font-size: 12px !important; }
  canvas { display: none !important; }

  /* Dashboard: expand all collapsibles */
  .dash-coll-body { display: block !important; max-height: none !important; overflow: visible !important; }

  /* Renewals: hide the renewals table card on print (summary panel is enough) */
  #page-renewals .card:not(#ren-summary-card) { display: none !important; }

  /* Renewals: expand the upcoming clients toggle on print */
  [id^="ren-summary-upcoming-detail"] { display: block !important; }
}
```

- [ ] **Step 3: Verify screen appearance is unchanged**

Open `working_v2.8_1.html` in the browser. Navigate through all pages. Nothing should look different on screen — the `@media screen` block hides all the print-only elements on screen.

- [ ] **Step 4: Commit**
```bash
git add working_v2.8_1.html
git commit -m "feat: add @media print CSS for dashboard and renewals pages"
```

---

## Task 7: Add Dashboard Print Button and Print-Only Elements

**Files:**
- Modify: `working_v2.8_1.html` — Dashboard HTML section (~line 447)

- [ ] **Step 1: Add print header, print button, and chart text fallback to dashboard HTML**

Find this section in the HTML (around line 447–455):
```html
    <div class="page active" id="page-dashboard">

      <!-- TOP METRICS -->
      <div class="metric-grid" id="dash-metrics"></div>

      <!-- CHARTS ROW -->
      <div class="grid-2" style="margin-bottom:16px">
```

Replace it with:
```html
    <div class="page active" id="page-dashboard">

      <!-- PRINT HEADER (screen: hidden, print: visible) -->
      <div id="dash-print-header" style="display:none;margin-bottom:16px">
        <div style="font-size:16px;font-weight:600">Strata Consultancy Group · Dashboard Report</div>
        <div style="font-size:12px;color:#666" id="dash-print-date"></div>
      </div>

      <!-- PRINT BUTTON (screen: visible, print: hidden) -->
      <div class="no-print" style="display:flex;justify-content:flex-end;margin-bottom:10px">
        <button class="btn" onclick="document.getElementById('dash-print-date').textContent=new Date().toLocaleDateString('en-PH',{month:'long',day:'numeric',year:'numeric',hour:'2-digit',minute:'2-digit'});window.print();" title="Print dashboard report">🖨 Print Dashboard</button>
      </div>

      <!-- TOP METRICS -->
      <div class="metric-grid" id="dash-metrics"></div>

      <!-- CHART TEXT FALLBACK (screen: hidden, print: visible) -->
      <div id="dash-print-charts" style="display:none">
        <div class="card" style="padding:12px 16px;margin-bottom:12px">
          <div style="font-size:12px;font-weight:600;margin-bottom:6px">Pipeline Summary</div>
          <div id="dash-print-pipeline" style="font-size:12px;line-height:2"></div>
        </div>
        <div class="card" style="padding:12px 16px;margin-bottom:12px">
          <div style="font-size:12px;font-weight:600;margin-bottom:6px">Quotation Status</div>
          <div id="dash-print-quotes" style="font-size:12px;line-height:2"></div>
        </div>
      </div>

      <!-- CHARTS ROW -->
      <div class="grid-2" style="margin-bottom:16px">
```

- [ ] **Step 2: Populate print chart fallbacks from `renderDashboard()`**

Find the end of `renderDashboard()` (around line 1705). It ends just before `// ── CLIENTS ──`. Add these lines at the very end of the function body, right before the closing `}`:

```javascript
  // ── PRINT FALLBACKS ──
  const printPipeline = document.getElementById('dash-print-pipeline');
  if (printPipeline) {
    printPipeline.innerHTML = STATUSES.map((s, i) =>
      `<span style="display:inline-block;min-width:180px"><b>${plLabels[i]}:</b> ${plData[i]} client${plData[i]===1?'':'s'}</span>`
    ).join('<br>');
  }
  const printQuotes = document.getElementById('dash-print-quotes');
  if (printQuotes) {
    printQuotes.innerHTML = qLabels.map((l, i) =>
      `<span style="display:inline-block;min-width:200px"><b>${l}:</b> ${qData[i]}</span>`
    ).join('<br>');
  }
```

- [ ] **Step 3: Verify Print button appears on Dashboard**

Open `working_v2.8_1.html`. Navigate to Dashboard. A "🖨 Print Dashboard" button should appear at the top right of the Dashboard page content area.

- [ ] **Step 4: Test print preview**

Click "🖨 Print Dashboard". Expected behavior:
- Browser print dialog opens
- Sidebar and topbar are hidden
- Print header shows: "Strata Consultancy Group · Dashboard Report" + date/time
- KPI metric cards are visible
- Chart canvases are replaced by the text summary (Pipeline Summary and Quotation Status sections)
- Collapsible sections (Pipeline breakdown, Upcoming renewals, etc.) are expanded
- No "Export CSV" or "Print" buttons appear on the printed page

- [ ] **Step 5: Commit**
```bash
git add working_v2.8_1.html
git commit -m "feat: add dashboard print button and print-only chart text fallbacks"
```

---

## Task 8: Verify Renewals Print

**Files:**
- No new changes — this task is verification only. The Print button was added in Task 4, the summary panel in Task 5, and the print CSS in Task 6.

- [ ] **Step 1: Verify Print button on Renewals page**

Open `working_v2.8_1.html`. Navigate to Renewals. The toolbar should show: search input · urgency dropdown · assigned dropdown · "⬇ Export CSV" · "🖨 Print"

- [ ] **Step 2: Test print preview**

Click "🖨 Print" on the Renewals page. Expected behavior:
- Sidebar and topbar are hidden
- The filter bar (search, dropdowns, buttons) is hidden
- The metric tiles row (`ren-metrics`) is hidden
- The Renewal Alerts Summary card is fully visible with all 4 stat tiles
- Critical and Soon clients are listed individually
- Upcoming clients are expanded (all shown, not collapsed)
- The main renewals table is hidden
- Print header "Strata Consultancy Group · Renewal Alerts · [date]" is visible inside the summary card

- [ ] **Step 3: Verify dark mode does not affect print**

Toggle dark mode. Click "🖨 Print". Expected: Print preview shows white background and black text regardless of dark mode setting.

- [ ] **Step 4: Final regression check**

Run through this checklist to confirm nothing is broken:

| Check | Expected |
|---|---|
| Sync from Google Sheets | Works, data loads |
| Add a new client | Modal opens, saves, appears in table |
| Edit a client | Modal opens with data, saves correctly |
| Complete a COA task | Prompts for date, updates client, logs history |
| Export Clients (no filter) | Downloads all clients |
| Export Clients (with filter) | Downloads only filtered clients |
| Export Quotations | Downloads correctly |
| Export COA | Downloads correctly |
| Export Renewals | Downloads correctly |
| Renewal summary on Renewals page | Shows 4 stat cards + grouped list |
| Upcoming expand toggle | Toggles correctly, resets on page re-render |
| Dashboard print | Print preview looks clean |
| Renewals print | Summary visible, table hidden |
| Dark mode toggle | No visual regressions |
| OAuth sign-in | Still works |

- [ ] **Step 5: Final commit**
```bash
git add working_v2.8_1.html
git commit -m "feat: features 3-5 complete — CSV export, renewal summary, print"
```

---

## Summary of All Changes

| What | Where in file | Risk |
|---|---|---|
| `_exportData` variable | Near `const TODAY` | Zero — new variable |
| `exportToCSV()` + helpers | After `renderRenewals()` | Zero — new functions |
| `_exportData.clients = f` | Inside `renderClients()` | Minimal — 1 line addition |
| `_exportData.quotations = f` | Inside `renderQuotations()` | Minimal — 1 line addition |
| `_exportData.coa = f` | Inside `renderCOA()` | Minimal — 1 line addition |
| `_exportData.renewals = f` | Inside `renderRenewals()` | Minimal — 1 line addition |
| Export buttons × 4 | In filter-bar HTML | Zero — new HTML elements |
| `<div id="ren-summary">` | Renewals page HTML | Zero — new div |
| `renderRenewalSummary()` | After export functions | Zero — new function |
| Call `renderRenewalSummary()` | End of `renderRenewals()` | Minimal — 1 line addition |
| `@media print` CSS block | Before `</style>` | Zero — print-only, no screen effect |
| Dashboard print button + print-only divs | Dashboard HTML | Zero — new HTML |
| Print fallback population | End of `renderDashboard()` | Minimal — 2 if-guarded additions |
