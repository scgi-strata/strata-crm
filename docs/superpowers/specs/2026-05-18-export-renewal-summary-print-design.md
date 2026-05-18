# Strata CRM — Features 3–5 Design Spec
**Date:** 2026-05-18  
**Version:** 1.0  
**Status:** Approved  
**Scope:** CSV Export, Renewal Alerts Summary, Dashboard + Renewals Print

---

## Overview

Three additive features for `working_v2.8_1.html`. All changes are purely additive — no existing logic is modified. Implementation is in-file JavaScript and CSS only, with no new dependencies.

---

## Feature 3: Filter-Aware CSV Export

### Purpose
Allow team members to export the currently visible (filtered) records from any data page to a CSV file for use in Excel, reports, or client presentations.

### Placement
A **⬇ Export CSV** button is added to the toolbar of each of the four data pages, positioned to the left of the existing `+ Add` button. Consistent placement across all pages.

**Affected pages:** Clients, Quotations, Renewals, COA Tasks

### Behavior
- Reads the same filtered/rendered array currently used to build the table on screen
- If a search or filter is active, only the visible rows are exported
- If no filter is active, all rows are exported
- Triggers a browser file download immediately on click — no modal, no confirmation
- File download uses `Blob` + `URL.createObjectURL()` — no external libraries

### Exported Columns

**Clients**
`ID, Name, Contact Person, Mobile, Email, Business Type, Insurance Type, Current Provider, Renewal Date, Pipeline Status, Last Contact, Next Follow-Up, Assigned, Notes`

**Quotations**
`ID, Client, Provider, Franchise Date, Franchise End, Initial Email Sent, TAT Date, Status, Date Released, Rates, TOR/SOB, Special Requests, Follow-Up Date`

**Renewals**
`Client, Insurance Type, Current Provider, Renewal Date, Days Remaining, Urgency, Assigned`

**COA Tasks**
`Date, Client, Contact Type, Task Description, Assigned, Completed Date`

### File Naming
Format: `strata-{page}-{YYYY-MM-DD}.csv`  
Examples: `strata-clients-2026-05-18.csv`, `strata-quotations-2026-05-18.csv`

### Implementation Notes
- Create a single reusable `exportToCSV(headers, rows, filename)` utility function
- Each page calls this function with its own filtered array and column mapping
- Columns with commas or quotes must be wrapped in double-quotes (standard CSV escaping)
- Internal fields (`_row`, `id` internal keys) are excluded from export

### Risk Assessment
- **No risk to existing data** — reads from the same in-memory array, never writes
- **No risk to Google Sheets sync** — export is entirely client-side
- Existing filter/search logic is untouched

---

## Feature 4: Bulk Renewal Alerts Summary

### Purpose
A persistent summary panel at the top of the Renewals page showing a count of renewals by urgency tier and a grouped client list. Printable as a clean one-page report.

### Placement
Inserted as a block element **above the existing renewals table** on the Renewals page. The existing table and its filters are unchanged.

### Layout

**Stat Cards Row**
Four cards side by side, color-coded by urgency:
| Card | Color | Condition |
|---|---|---|
| Critical | Red | ≤ 14 days |
| Soon | Amber | 15–30 days |
| Upcoming | Blue | 31–90 days |
| Future | Gray | 91+ days |

Each card shows: count (large) + label (small).

**Grouped Client List**
Below the stat cards, clients are listed in urgency groups:
- **Critical:** Each client shown individually — name, renewal date, insurance type, assigned team member
- **Soon:** Each client shown individually — same columns
- **Upcoming:** Collapsed by default — "9 clients renewing [date range]" with an expand toggle to show individual clients. Collapsed state is not persisted — resets each time the Renewals page is rendered.
- **Future:** Collapsed by default — count only, no expand toggle needed

### Data Source
Reads from `DATA.CLIENTS` in memory — the same array used by the Renewals page. No new API calls or sync operations.

### Expiry handling
Expired clients (renewal date in the past) are excluded from the summary panel (they already appear in their own section in the existing table).

### Print behavior
When the user prints the Renewals page, the summary panel prints in full. The existing renewals table is hidden on print. See Feature 5 for print CSS details.

### Risk Assessment
- Panel is inserted above the table — does not modify table HTML or its rendering logic
- Reads `DATA.CLIENTS` without modifying it
- No impact on sync, write-back, or history logging

---

## Feature 5: Dashboard + Renewals Print

### Purpose
A **🖨 Print** button on the Dashboard and Renewals pages triggers `window.print()` with `@media print` CSS that produces a clean, professional printed output.

### Placement
- **Dashboard:** Print button in the page toolbar, right side
- **Renewals:** Print button in the page toolbar, right side — alongside the Export CSV button

### Print Output — Dashboard
Header: `Strata Consultancy Group · Dashboard Report · {date}`

Printed elements:
- KPI metric cards (active clients, overdue quotations, critical renewals, pipeline count)
- Pipeline stage counts (text-based, one line per stage)
- Renewal alert summary (Critical / Soon counts)
- Overdue quotations list

Hidden on print:
- Sidebar, topbar, toolbar buttons
- Chart.js canvas elements (replaced with a `<div class="print-only">` text summary showing the same KPI values: e.g. "Active Clients: 28 · Closed Won: 12 · For Quotation: 5 · Overdue Quotations: 3")
- Collapsible section toggles

### Print Output — Renewals
Header: `Strata Consultancy Group · Renewal Alerts · {date}`

Printed elements:
- The 4 stat cards (Feature 4)
- Full grouped client list — all tiers expanded, no pagination

Hidden on print:
- Sidebar, topbar, search bar, filter controls, Export CSV button, Print button

### Implementation
- A single `@media print { }` block added to the existing `<style>` section
- CSS classes used: `.no-print` (applied to sidebar, topbar, buttons), `.print-only` (print-only elements like headers and chart text fallbacks)
- `window.print()` called on button click
- No new libraries — browser native print

### Cross-browser notes
- `@media print` is supported in all modern browsers
- Chart.js canvases do not render in print — hidden and replaced with text summaries
- Page breaks controlled with `page-break-before` / `page-break-after` where needed

### Risk Assessment
- `@media print` CSS has zero effect on screen rendering
- `.no-print` and `.print-only` classes are new — no collision with existing class names
- `window.print()` is a native browser API — no side effects on app state

---

## What Is NOT Changing

The following existing behaviors are preserved unchanged:

- Google Sheets sync (`syncAll()`)
- Write-back queue and debounce logic
- COA task completion workflow (3-sheet update)
- Next follow-up auto-calculation (`computeNextFollowUp()`)
- Quotation status auto-computation
- OAuth and passcode authentication
- History logging (ClientSummary, COAHistory)
- All existing modals, drawers, and form logic
- Dark mode behavior

---

## Implementation Order

1. **Feature 3 — CSV Export** (lowest risk, purely additive, no UI restructuring)
2. **Feature 4 — Renewal Summary Panel** (new HTML block above existing table)
3. **Feature 5 — Print CSS + Buttons** (CSS-only addition + two button wires)

---

## Out of Scope

- Email/share functionality for exports
- Scheduled or automated exports
- PDF generation via library (browser print is sufficient)
- Mobile print optimization (app is desktop-only)
- Pagination of export results (full export of filtered set)
