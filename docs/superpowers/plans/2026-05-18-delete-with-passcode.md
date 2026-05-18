# Delete with Passcode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a passcode-protected delete button to every edit modal (Clients, Quotations, COA Tasks, Providers) that moves deleted records to a combined `Deleted` Google Sheet with cascade support for Clients.

**Architecture:** A single `deleteRecord(type, idx)` function handles all deletion logic — appending to the Deleted sheet, removing the row from its source sheet via `batchUpdate/deleteDimension`, updating in-memory arrays, and re-rendering. A new `#delete-confirm-modal` shows the full cascade list and passcode field before any action is taken. Delete buttons are added to all 4 modal footers, shown only when editing existing records.

**Tech Stack:** Vanilla JavaScript, HTML, CSS — all changes in `C:\Users\zzthu\Downloads\Strata CRM\index.html`. Google Sheets API v4 (`gapi.client.sheets`).

---

## File Map

**Single file modified:** `C:\Users\zzthu\Downloads\Strata CRM\index.html`

| Section | What changes |
|---|---|
| `let CFG = {...}` (~line 1032) | Add `deletedSheet: 'Deleted'` to defaults |
| Settings HTML (~line 865) | Add Deleted sheet name input field |
| `saveSheetNames()` (~line 1066) | Save `CFG.deletedSheet` from new input |
| Settings load (~line 1048) | Load `CFG.deletedSheet` into new input |
| After `updateSheetRow()` (~line 1196) | Add `getSheetGid()` + `deleteSheetRow()` functions |
| After `deleteSheetRow()` | Add `showDeleteConfirm()` + `confirmDelete()` + `deleteRecord()` |
| Modal HTML — 4 modal footers | Add Delete button (hidden by default) to each modal footer |
| `openClientModal()` (~line 2417) | Show/hide delete button based on `idx !== null` |
| `openQuoteModal()` (~line 2496) | Show/hide delete button |
| `openCOAModal()` (~line 2549) | Show/hide delete button |
| `openProviderModal()` (~line 2611) | Show/hide delete button |
| After Provider modal HTML (~line 1022) | Add `#delete-confirm-modal` HTML |

---

## Task 1: Add `deletedSheet` to CFG and Settings

**Files:**
- Modify: `index.html` — CFG defaults, Settings HTML, saveSheetNames(), Settings load

- [ ] **Step 1: Add `deletedSheet` to CFG defaults**

Find this line (~line 1032):
```javascript
let CFG = { sheetId:'', apiKey:'', clientId:'', clientsSheet:'Client', quotesSheet:'Quotations', coaSheet:'COA', providersSheet:'ProviderEmails', passcode:'' };
```

Replace with:
```javascript
let CFG = { sheetId:'', apiKey:'', clientId:'', clientsSheet:'Client', quotesSheet:'Quotations', coaSheet:'COA', providersSheet:'ProviderEmails', clientSummarySheet:'ClientSummary', coaHistorySheet:'COAHistory', deletedSheet:'Deleted', passcode:'' };
```

- [ ] **Step 2: Add Deleted sheet input to Settings HTML**

Find this line in the Settings HTML (~line 865):
```html
          <div class="form-group"><div class="form-label">COA History (COAHistory)</div><input class="form-input" id="cfg-coahistory-sheet" value="COAHistory"></div>
```

Add this line IMMEDIATELY AFTER it:
```html
          <div class="form-group"><div class="form-label">Deleted Records</div><input class="form-input" id="cfg-deleted-sheet" value="Deleted"></div>
```

- [ ] **Step 3: Load `deletedSheet` into Settings**

Find this block (~line 1048):
```javascript
  if(document.getElementById('cfg-clientsummary-sheet')) document.getElementById('cfg-clientsummary-sheet').value=CFG.clientSummarySheet||'ClientSummary';
  if(document.getElementById('cfg-coahistory-sheet')) document.getElementById('cfg-coahistory-sheet').value=CFG.coaHistorySheet||'COAHistory';
```

Add this line IMMEDIATELY AFTER:
```javascript
  if(document.getElementById('cfg-deleted-sheet')) document.getElementById('cfg-deleted-sheet').value=CFG.deletedSheet||'Deleted';
```

- [ ] **Step 4: Save `deletedSheet` in `saveSheetNames()`**

Find this block (~line 1070):
```javascript
  CFG.coaHistorySheet=document.getElementById('cfg-coahistory-sheet').value.trim()||'COAHistory';
```

Add this line IMMEDIATELY AFTER:
```javascript
  CFG.deletedSheet=document.getElementById('cfg-deleted-sheet').value.trim()||'Deleted';
```

- [ ] **Step 5: Verify**

Open `index.html`. Go to Settings. A new "Deleted Records" input should appear below "COA History" with default value `Deleted`. Change it and save — confirm the value persists after navigating away and back.

- [ ] **Step 6: Commit**
```bash
cd "C:/Users/zzthu/Downloads/Strata CRM"
git add index.html
git commit -m "feat: add deletedSheet config to CFG and Settings"
```

---

## Task 2: Add `getSheetGid()` and `deleteSheetRow()` utilities

**Files:**
- Modify: `index.html` — JavaScript, after `updateSheetRow()` (~line 1196)

- [ ] **Step 1: Find the insertion point**

Locate this line (~line 1196):
```javascript
}

// ── OAUTH & GAPI ──
```

Insert the following two functions between the closing `}` and the `// ── OAUTH & GAPI ──` comment:

- [ ] **Step 2: Insert `getSheetGid()` and `deleteSheetRow()`**

```javascript
async function getSheetGid(sheetName){
  if(!CFG.sheetId) throw new Error('No Sheet ID configured.');
  const resp = await gapi.client.sheets.spreadsheets.get({
    spreadsheetId: CFG.sheetId,
    fields: 'sheets.properties'
  });
  const sheet = resp.result.sheets.find(s => s.properties.title === sheetName);
  if(!sheet) throw new Error(`Sheet "${sheetName}" not found in spreadsheet.`);
  return sheet.properties.sheetId;
}

async function deleteSheetRow(sheetName, rowNumber){
  // rowNumber is 1-based (matches _row property on records)
  // API startIndex is 0-based
  if(!CFG.sheetId) throw new Error('No Sheet ID configured.');
  if(!window.gapi||!gapi.client) throw new Error('OAuth not initialized.');
  const token = gapi.client.getToken ? gapi.client.getToken() : null;
  if(!token||!token.access_token) throw new Error('NO_TOKEN');
  const sheetGid = await getSheetGid(sheetName);
  const startIndex = rowNumber - 1;
  await gapi.client.sheets.spreadsheets.batchUpdate({
    spreadsheetId: CFG.sheetId,
    resource: {
      requests: [{
        deleteDimension: {
          range: {
            sheetId: sheetGid,
            dimension: 'ROWS',
            startIndex: startIndex,
            endIndex: startIndex + 1
          }
        }
      }]
    }
  });
}

```

- [ ] **Step 3: Verify syntax**

Open `index.html` in Chrome. Open DevTools Console. Type:
```javascript
typeof getSheetGid
```
Expected: `"function"`

```javascript
typeof deleteSheetRow
```
Expected: `"function"`

- [ ] **Step 4: Commit**
```bash
git add index.html
git commit -m "feat: add getSheetGid and deleteSheetRow utilities"
```

---

## Task 3: Add Delete Confirm Modal HTML

**Files:**
- Modify: `index.html` — after the Provider modal closing `</div>` (~line 1022)

- [ ] **Step 1: Find the insertion point**

Locate the end of the Provider modal (~line 1022):
```html
</div>

<script>
```

Insert the following HTML BETWEEN the closing `</div>` of the provider modal and `<script>`:

- [ ] **Step 2: Insert the modal HTML**

```html
<!-- Delete Confirmation Modal -->
<div class="modal-overlay" id="delete-confirm-modal">
  <div class="modal" style="max-width:480px">
    <div class="modal-head">
      <h3 style="color:var(--red)">⚠ Confirm Delete</h3>
      <button class="modal-close" onclick="closeModal('delete-confirm-modal')">×</button>
    </div>
    <div class="modal-body">
      <div style="font-size:13px;font-weight:600;color:var(--red);margin-bottom:10px">This will permanently delete:</div>
      <div id="del-list" style="background:var(--red-bg);border-radius:var(--radius-sm);padding:10px 14px;margin-bottom:14px;font-size:13px;line-height:1.8"></div>
      <div style="font-size:12px;color:var(--text2);margin-bottom:10px">Records will be moved to the <b>Deleted</b> sheet in Google Sheets. Enter team passcode to confirm:</div>
      <input class="form-input" type="password" id="del-passcode" placeholder="Enter team passcode..." autocomplete="off">
      <div id="del-error" style="color:var(--red);font-size:12px;margin-top:6px;min-height:18px"></div>
    </div>
    <div class="modal-foot" style="justify-content:space-between">
      <button class="btn" onclick="closeModal('delete-confirm-modal')">Cancel</button>
      <button class="btn" id="del-confirm-btn" onclick="confirmDelete()" style="background:var(--red);color:#fff;border-color:transparent">Confirm Delete</button>
    </div>
  </div>
</div>

```

- [ ] **Step 3: Verify**

Open `index.html`. Open DevTools Console. Type:
```javascript
document.getElementById('delete-confirm-modal') !== null
```
Expected: `true`

- [ ] **Step 4: Commit**
```bash
git add index.html
git commit -m "feat: add delete confirmation modal HTML"
```

---

## Task 4: Add `showDeleteConfirm()`, `confirmDelete()`, and `deleteRecord()`

**Files:**
- Modify: `index.html` — JavaScript, before the `// ── QUICK-PEEK DRAWER ──` comment (~line 2643)

- [ ] **Step 1: Find the insertion point**

Locate this comment (~line 2643):
```javascript
// ── QUICK-PEEK DRAWER ──
let _qpActiveRow = null;
```

Insert the following block IMMEDIATELY BEFORE it:

- [ ] **Step 2: Insert the three functions**

```javascript
// ── DELETE WITH PASSCODE ──
let _pendingDeleteType = null;
let _pendingDeleteIdx  = null;

function showDeleteConfirm(type, idx) {
  _pendingDeleteType = type;
  _pendingDeleteIdx  = idx;

  let lines = [];
  if(type === 'client') {
    const c = CLIENTS[idx];
    lines.push(`👤 <b>${c.name}</b> (Client)`);
    QUOTATIONS.filter(q => q.clientId === c.id).forEach(q =>
      lines.push(`📋 ${q.id} — ${q.provider||'—'} (Quotation)`)
    );
    COA_DATA.filter(t => t.clientId === c.id).forEach(t =>
      lines.push(`✅ ${(t.task||'').substring(0,50)}${(t.task||'').length>50?'…':''} (COA Task)`)
    );
  } else if(type === 'quotation') {
    const q = QUOTATIONS[idx];
    lines.push(`📋 <b>${q.id} — ${q.client}</b> (Quotation)`);
  } else if(type === 'coa') {
    const t = COA_DATA[idx];
    lines.push(`✅ <b>${(t.task||'').substring(0,50)}${(t.task||'').length>50?'…':''}</b> (COA Task)`);
  } else if(type === 'provider') {
    const p = PROVIDERS[idx];
    lines.push(`🏥 <b>${p.provider}</b> (Provider)`);
  }

  document.getElementById('del-list').innerHTML = lines.join('<br>');
  document.getElementById('del-passcode').value = '';
  document.getElementById('del-error').textContent = '';
  openModal('delete-confirm-modal');
}

async function confirmDelete() {
  const passcode = document.getElementById('del-passcode').value.trim();
  const errEl = document.getElementById('del-error');
  if(!passcode){ errEl.textContent = 'Enter passcode to confirm.'; return; }
  if(passcode !== CFG.passcode){ errEl.textContent = 'Incorrect passcode.'; return; }
  errEl.textContent = '';
  const btn = document.getElementById('del-confirm-btn');
  btn.disabled = true;
  btn.textContent = 'Deleting…';
  try {
    await deleteRecord(_pendingDeleteType, _pendingDeleteIdx);
    closeModal('delete-confirm-modal');
    toast('Deleted — moved to Deleted sheet ✓', 'success');
  } catch(e) {
    errEl.textContent = 'Error: ' + e.message;
  } finally {
    btn.disabled = false;
    btn.textContent = 'Confirm Delete';
  }
}

async function deleteRecord(type, idx) {
  const deletedAt = new Date().toISOString();
  const deletedSheetName = CFG.deletedSheet || 'Deleted';

  // Build list of records to delete (main record first, then cascade)
  const toDelete = []; // [{sheetName, record, rowFn, array, arrayIdx}]

  if(type === 'client') {
    const c = CLIENTS[idx];
    toDelete.push({ sheetName: CFG.clientsSheet, record: c, type: 'Client',
                    label: c.name });
    // Cascade: linked quotations
    QUOTATIONS.forEach((q, qi) => {
      if(q.clientId === c.id)
        toDelete.push({ sheetName: CFG.quotesSheet, record: q, type: 'Quotation',
                        label: q.client });
    });
    // Cascade: linked COA tasks
    COA_DATA.forEach((t, ti) => {
      if(t.clientId === c.id)
        toDelete.push({ sheetName: CFG.coaSheet, record: t, type: 'COA',
                        label: (t.task||'').substring(0,60) });
    });
  } else if(type === 'quotation') {
    const q = QUOTATIONS[idx];
    toDelete.push({ sheetName: CFG.quotesSheet, record: q, type: 'Quotation',
                    label: q.client });
  } else if(type === 'coa') {
    const t = COA_DATA[idx];
    toDelete.push({ sheetName: CFG.coaSheet, record: t, type: 'COA',
                    label: (t.task||'').substring(0,60) });
  } else if(type === 'provider') {
    const p = PROVIDERS[idx];
    toDelete.push({ sheetName: CFG.providersSheet, record: p, type: 'Provider',
                    label: p.provider });
  }

  // 1. Append each record to the Deleted sheet
  for(const item of toDelete) {
    const id = item.record.id || item.record.provider || (item.record.date + '_' + item.record.client);
    await appendToSheet(deletedSheetName, [
      item.type, id, item.label, deletedAt, JSON.stringify(item.record)
    ]);
  }

  // 2. Delete rows from source sheets — sort by _row descending within each sheet
  //    to avoid row-index shifting when multiple rows are deleted from the same sheet
  const bySheet = {};
  toDelete.forEach(item => {
    if(item.record._row) {
      if(!bySheet[item.sheetName]) bySheet[item.sheetName] = [];
      bySheet[item.sheetName].push(item.record._row);
    }
  });
  for(const [sheetName, rows] of Object.entries(bySheet)) {
    const sorted = [...rows].sort((a,b) => b - a); // highest first
    for(const rowNumber of sorted) {
      await deleteSheetRow(sheetName, rowNumber);
    }
  }

  // 3. Remove from in-memory arrays
  if(type === 'client') {
    const clientId = CLIENTS[idx].id;
    CLIENTS.splice(idx, 1);
    const quoteIndicesToRemove = QUOTATIONS.map((q,i) => q.clientId===clientId ? i : -1)
      .filter(i => i !== -1).sort((a,b) => b-a);
    quoteIndicesToRemove.forEach(i => QUOTATIONS.splice(i, 1));
    const coaIndicesToRemove = COA_DATA.map((t,i) => t.clientId===clientId ? i : -1)
      .filter(i => i !== -1).sort((a,b) => b-a);
    coaIndicesToRemove.forEach(i => COA_DATA.splice(i, 1));
  } else if(type === 'quotation') {
    QUOTATIONS.splice(idx, 1);
  } else if(type === 'coa') {
    COA_DATA.splice(idx, 1);
  } else if(type === 'provider') {
    PROVIDERS.splice(idx, 1);
  }

  // 4. Persist to localStorage
  saveLocalData();

  // 5. Re-render
  if(type === 'client')    { renderClients(); renderDashboard(); }
  if(type === 'quotation') { renderQuotations(); renderDashboard(); }
  if(type === 'coa')       { renderCOA(); renderDashboard(); }
  if(type === 'provider')  { renderProviders(); }
}

```

- [ ] **Step 3: Verify in browser**

Open `index.html`. Open DevTools Console. Type:
```javascript
typeof showDeleteConfirm
```
Expected: `"function"`

```javascript
typeof deleteRecord
```
Expected: `"function"`

- [ ] **Step 4: Commit**
```bash
git add index.html
git commit -m "feat: add showDeleteConfirm, confirmDelete, and deleteRecord functions"
```

---

## Task 5: Add Delete Buttons to all 4 Modal Footers

**Files:**
- Modify: `index.html` — 4 modal footer sections in HTML

- [ ] **Step 1: Client modal footer**

Find this exact block (~line 922):
```html
    <div class="modal-foot">
      <span class="modal-saving" id="cm-saving"></span>
      <button class="btn" onclick="closeModal('client-modal')">Cancel</button>
      <button class="btn btn-primary" id="cm-save-btn" onclick="saveClient()">Save client</button>
    </div>
```

Replace with:
```html
    <div class="modal-foot" style="justify-content:space-between">
      <button class="btn" id="cm-delete-btn" onclick="closeModal('client-modal');showDeleteConfirm('client',editClientIdx)" style="background:var(--red-bg);color:var(--red);border-color:var(--red);display:none">🗑 Delete</button>
      <div style="display:flex;gap:8px;align-items:center">
        <span class="modal-saving" id="cm-saving"></span>
        <button class="btn" onclick="closeModal('client-modal')">Cancel</button>
        <button class="btn btn-primary" id="cm-save-btn" onclick="saveClient()">Save client</button>
      </div>
    </div>
```

- [ ] **Step 2: Quotation modal footer**

Find this exact block (~line 955):
```html
    <div class="modal-foot">
      <span class="modal-saving" id="qm-saving"></span>
      <button class="btn" onclick="closeModal('quote-modal')">Cancel</button>
      <button class="btn btn-primary" id="qm-save-btn" onclick="saveQuotation()">Save quotation</button>
    </div>
```

Replace with:
```html
    <div class="modal-foot" style="justify-content:space-between">
      <button class="btn" id="qm-delete-btn" onclick="closeModal('quote-modal');showDeleteConfirm('quotation',editQuoteIdx)" style="background:var(--red-bg);color:var(--red);border-color:var(--red);display:none">🗑 Delete</button>
      <div style="display:flex;gap:8px;align-items:center">
        <span class="modal-saving" id="qm-saving"></span>
        <button class="btn" onclick="closeModal('quote-modal')">Cancel</button>
        <button class="btn btn-primary" id="qm-save-btn" onclick="saveQuotation()">Save quotation</button>
      </div>
    </div>
```

- [ ] **Step 3: COA modal footer**

Find this exact block (~line 995):
```html
    <div class="modal-foot">
      <span class="modal-saving" id="coam-saving"></span>
      <button class="btn" onclick="closeModal('coa-modal')">Cancel</button>
      <button class="btn btn-primary" id="coam-save-btn" onclick="saveCOA()">Save task</button>
    </div>
```

Replace with:
```html
    <div class="modal-foot" style="justify-content:space-between">
      <button class="btn" id="coam-delete-btn" onclick="closeModal('coa-modal');showDeleteConfirm('coa',editCOAIdx)" style="background:var(--red-bg);color:var(--red);border-color:var(--red);display:none">🗑 Delete</button>
      <div style="display:flex;gap:8px;align-items:center">
        <span class="modal-saving" id="coam-saving"></span>
        <button class="btn" onclick="closeModal('coa-modal')">Cancel</button>
        <button class="btn btn-primary" id="coam-save-btn" onclick="saveCOA()">Save task</button>
      </div>
    </div>
```

- [ ] **Step 4: Provider modal footer**

Find this exact block (~line 1016):
```html
    <div class="modal-foot">
      <span class="modal-saving" id="pm-saving"></span>
      <button class="btn" onclick="closeModal('provider-modal')">Cancel</button>
      <button class="btn btn-primary" id="pm-save-btn" onclick="saveProvider()">Save provider</button>
    </div>
```

Replace with:
```html
    <div class="modal-foot" style="justify-content:space-between">
      <button class="btn" id="pm-delete-btn" onclick="closeModal('provider-modal');showDeleteConfirm('provider',editProviderIdx)" style="background:var(--red-bg);color:var(--red);border-color:var(--red);display:none">🗑 Delete</button>
      <div style="display:flex;gap:8px;align-items:center">
        <span class="modal-saving" id="pm-saving"></span>
        <button class="btn" onclick="closeModal('provider-modal')">Cancel</button>
        <button class="btn btn-primary" id="pm-save-btn" onclick="saveProvider()">Save provider</button>
      </div>
    </div>
```

- [ ] **Step 5: Verify all 4 delete buttons exist (hidden)**

Open `index.html`. Open DevTools Console. Type:
```javascript
['cm-delete-btn','qm-delete-btn','coam-delete-btn','pm-delete-btn'].map(id => document.getElementById(id) !== null)
```
Expected: `[true, true, true, true]`

All buttons should be hidden (style `display:none`) when opening a modal for a new record.

- [ ] **Step 6: Commit**
```bash
git add index.html
git commit -m "feat: add delete buttons to all 4 modal footers (hidden by default)"
```

---

## Task 6: Show/Hide Delete Button in each openModal function

**Files:**
- Modify: `index.html` — inside `openClientModal()`, `openQuoteModal()`, `openCOAModal()`, `openProviderModal()`

- [ ] **Step 1: Update `openClientModal()`**

Find this line inside `openClientModal()` (~line 2430):
```javascript
  document.getElementById('cm-saving').textContent='';
  openModal('client-modal');
```

Add one line between them:
```javascript
  document.getElementById('cm-saving').textContent='';
  document.getElementById('cm-delete-btn').style.display = idx !== null ? 'inline-flex' : 'none';
  openModal('client-modal');
```

- [ ] **Step 2: Update `openQuoteModal()`**

Find this inside `openQuoteModal()` (~line 2510 — it's near the end of the function before `openModal`):
```javascript
  document.getElementById('qm-saving').textContent='';
  openModal('quote-modal');
```

Add one line between them:
```javascript
  document.getElementById('qm-saving').textContent='';
  document.getElementById('qm-delete-btn').style.display = idx !== null ? 'inline-flex' : 'none';
  openModal('quote-modal');
```

- [ ] **Step 3: Update `openCOAModal()`**

Find this inside `openCOAModal()` (~line 2573):
```javascript
  document.getElementById('coam-saving').textContent='';
  openModal('coa-modal');
```

Add one line between them:
```javascript
  document.getElementById('coam-saving').textContent='';
  document.getElementById('coam-delete-btn').style.display = idx !== null ? 'inline-flex' : 'none';
  openModal('coa-modal');
```

- [ ] **Step 4: Update `openProviderModal()`**

Find this inside `openProviderModal()` (~line 2618):
```javascript
  document.getElementById('pm-saving').textContent='';
  openModal('provider-modal');
```

Add one line between them:
```javascript
  document.getElementById('pm-saving').textContent='';
  document.getElementById('pm-delete-btn').style.display = idx !== null ? 'inline-flex' : 'none';
  openModal('provider-modal');
```

- [ ] **Step 5: End-to-end browser test**

Open `index.html`. Test this flow:

**Test A — Delete button visibility:**
- Click `+ Add client` → Delete button should be HIDDEN
- Click Edit on any existing client → Delete button should be VISIBLE (red, bottom-left)

**Test B — Confirmation modal:**
- Open Edit on an existing client → click 🗑 Delete
- The edit modal closes, the confirmation modal opens
- The list shows the client name + any linked quotations/COA tasks
- Enter wrong passcode → "Incorrect passcode." error, no deletion
- Enter correct team passcode → deletion proceeds

**Test C — Record removed:**
- After confirming, the client disappears from the Clients table
- Toast shows "Deleted — moved to Deleted sheet ✓"
- Open Google Sheets → verify the `Deleted` tab has a new row with Type=Client, the client name, and Raw JSON

**Test D — Other data types:**
- Edit a Quotation → Delete button visible → confirm → quotation removed from table
- Edit a COA Task → Delete button visible → confirm → task removed
- Edit a Provider → Delete button visible → confirm → provider removed

**Test E — Cascade:**
- Open a client that has linked quotations → click Delete
- Confirm list shows client + linked quotations + linked COA tasks
- After confirm, all cascade records are gone from their tables

- [ ] **Step 6: Final commit**
```bash
git add index.html
git commit -m "feat: show delete button on existing records in all 4 edit modals"
git push origin main
```

---

## Summary of All Changes

| What | Where | Risk |
|---|---|---|
| `CFG.deletedSheet` default | CFG object | Zero — new property |
| Settings input + save/load | Settings HTML + JS | Minimal — new field |
| `getSheetGid()` | After `updateSheetRow()` | Zero — new function |
| `deleteSheetRow()` | After `getSheetGid()` | Zero — new function |
| Delete confirm modal HTML | After provider modal | Zero — new HTML |
| `showDeleteConfirm()` | Before quick-peek section | Zero — new function |
| `confirmDelete()` | Before quick-peek section | Zero — new function |
| `deleteRecord()` | Before quick-peek section | Zero — new function |
| Delete buttons × 4 | Modal footer HTML | Minimal — new buttons, hidden by default |
| Show/hide logic × 4 | In each openModal function | Minimal — 1 line per function |

**Post-deletion note:** After deleting records, `_row` values of remaining records in Google Sheets may shift. A sync from Google Sheets (`Sync from Sheets` button) is recommended after bulk deletions to refresh `_row` values for accurate future edits.
