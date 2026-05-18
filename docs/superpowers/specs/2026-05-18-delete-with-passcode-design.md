# Strata CRM — Delete with Passcode Design Spec
**Date:** 2026-05-18
**Version:** 1.0
**Status:** Approved
**Scope:** Delete functionality with passcode confirmation for all 4 data types

---

## Overview

Add a protected delete capability to every edit modal (Clients, Quotations, COA Tasks, Providers). Deleted records are moved to a combined `Deleted` sheet in Google Sheets rather than being permanently erased. Deletion requires the team passcode. Deleting a Client cascades to linked Quotations and COA Tasks. All changes are additive — no existing modal logic is modified.

---

## Delete Button

### Placement
A **🗑 Delete** button is added to the bottom-left of every edit modal footer, visually separated from the Save/Cancel buttons which remain on the right.

### Appearance
- Background: `var(--red-bg)` — subtle red tint
- Text color: `var(--red)`
- Border: `1px solid` red-toned border
- Label: `🗑 Delete`

### Visibility
- **Only shown when editing an existing record** — hidden when creating a new record
- Determined by whether the modal was opened with a valid index (existing record) vs `null` (new record)

### Affected modals
- `openClientModal(idx)` — shown when `idx !== null`
- `openQuoteModal(idx)` — shown when `idx !== null`
- `openCOAModal(idx)` — shown when `idx !== null`
- `openProviderModal(idx)` — shown when `idx !== null`

---

## Confirmation Modal

### Trigger
Clicking 🗑 Delete closes the edit modal and opens a dedicated confirmation modal.

### Content
```
⚠ This will permanently delete:

[list of records to be deleted]

Records will be moved to the Deleted sheet in Google Sheets.
Enter team passcode to confirm:

[passcode input field]

[Cancel]  [Confirm Delete]
```

### Record list format
Each item in the list shows:
- `👤 {name}` for Client
- `📋 {id} — {provider}` for Quotation
- `✅ {task} (truncated to 50 chars)` for COA Task
- `🏥 {provider}` for Provider

### Cascade display (Clients only)
When deleting a Client, the list shows:
1. The client record first
2. All linked Quotations (`clientId === client.id`)
3. All linked COA Tasks (`clientId === client.id`)

If a Client has no linked records, only the client row appears.

### Passcode validation
- Uses `CFG.passcode` from localStorage (same passcode as manual sign-in)
- Wrong passcode: inline error below the input — "Incorrect passcode" — no deletion
- Correct passcode: proceed with deletion
- Empty passcode: inline error — "Enter passcode to confirm"

### Buttons
- **Cancel** — closes confirmation modal, returns to normal view (does NOT reopen edit modal)
- **Confirm Delete** (red, `background: var(--red)`, white text) — executes deletion on correct passcode

---

## Deletion Execution

### Step-by-step on confirmed delete

1. **Append each record to the `Deleted` sheet** in Google Sheets
   - One row per deleted record: `[type, id, label, deletedAt, JSON.stringify(record)]`
   - `type` = `Client` / `Quotation` / `COA` / `Provider`
   - `deletedAt` = ISO timestamp (`new Date().toISOString()`)

2. **Delete each record's row from its source sheet** via Sheets API
   - Uses `batchUpdate` with `deleteDimension` request to remove the row by `_row` index
   - Order: delete from highest `_row` to lowest to avoid row number shifting

3. **Remove from in-memory arrays**
   - `CLIENTS.splice(idx, 1)` for Client
   - Filter out linked Quotations from `QUOTATIONS`
   - Filter out linked COA Tasks from `COA_DATA`

4. **Save updated arrays to localStorage**

5. **Re-render affected views**
   - `renderClients()` + `renderDashboard()` for Client deletion
   - `renderQuotations()` + `renderDashboard()` for Quotation deletion
   - `renderCOA()` + `renderDashboard()` for COA deletion
   - `renderProviders()` for Provider deletion

6. **Show toast:** `"Deleted — moved to Deleted sheet ✓"`

7. **Close confirmation modal**

### Cascade rules
| Deleted record | Also deletes |
|---|---|
| Client | All `QUOTATIONS` where `q.clientId === client.id` + All `COA_DATA` where `c.clientId === client.id` |
| Quotation | Nothing |
| COA Task | Nothing |
| Provider | Nothing |

---

## Deleted Sheet Structure

**Sheet name:** `Deleted` (configurable in Settings alongside other sheet names)

Since records of different types are mixed in one sheet, a Raw JSON column avoids complex multi-type column alignment and makes recovery simple — the full record can be reconstructed from the JSON string.

**Columns:**
`Type | ID | Label | Deleted At | Raw JSON`

- `Type` — `Client` / `Quotation` / `COA` / `Provider`
- `ID` — `c.id`, `q.id`, `coa.date + coa.client`, `p.provider`
- `Label` — `c.name`, `q.client`, `coa.task`, `p.provider`
- `Deleted At` — `new Date().toISOString()`
- `Raw JSON` — `JSON.stringify(record)` (full object, allows recovery)

---

## New Function: `deleteRecord(type, idx)`

A single reusable function handles deletion for all types.

```
deleteRecord(type, idx)
  type: 'client' | 'quotation' | 'coa' | 'provider'
  idx: index in the relevant in-memory array
```

Called by `confirmDelete()` after passcode validation passes.

---

## Settings

`CFG.deletedSheet` — name of the Deleted sheet, defaulting to `"Deleted"`. Configurable in the Settings page alongside other sheet names.

---

## What Is NOT Changing

- All existing modal open/save/close logic
- Google Sheets sync (`syncAll()`)
- Write-back queue and debounce
- History logging (ClientSummary, COAHistory)
- OAuth and passcode authentication flow
- All existing table rendering logic

---

## Out of Scope

- Restoring/recovering deleted records from the CRM UI (recovery is done directly in Google Sheets)
- Bulk delete (multi-select and delete)
- Deleting records from the table view without opening the edit modal
- Audit log of who deleted what (the `Deleted` sheet timestamp serves this purpose)
