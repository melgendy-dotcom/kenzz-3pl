# Kenzz 3PL Logistics Analyzer — Master Documentation

## What This App Is

A single-file web app for auditing 3PL (third-party logistics) carrier invoices for Kenzz Electronics Egypt.
It parses uploaded Excel files, applies Kenzz's contracted rates, and flags overcharges.

**Carriers covered:** Bosta, Aramex, Aramex Mashreq, Raya
**Currency:** Egyptian Pounds (EGP)
**Language:** Arabic (RTL default) / English toggle

---

## Deployment

| Item | Value |
|------|-------|
| **Live URL** | https://melgendy-dotcom.github.io/kenzz-3pl/ |
| **Alt URL (CDN bypass)** | https://melgendy-dotcom.github.io/kenzz-3pl/app.html |
| **GitHub Repo** | https://github.com/melgendy-dotcom/kenzz-3pl |
| **Local path** | `C:\Users\DELL_PC\Desktop\Project\kenzz-3pl\index.html` |
| **Password** | *(stored in `index.html` → `AUTH.PASSWORD` constant — do not commit here)* |
| **Deploy** | `git add index.html app.html && git commit -m "..." && git push origin main` |

**IMPORTANT:** Always copy `index.html` to `app.html` before committing:
```
cp index.html app.html
```
GitHub Pages CDN (Fastly) caches for ~10 min. Use `app.html` URL to bypass cache.

---

## Architecture

- **Single file:** All HTML, CSS, and JS in `index.html` (mirrored to `app.html`)
- **No build step:** Plain HTML/JS, deployed directly via GitHub Pages
- **Libraries:**
  - SheetJS (xlsx-0.20.3) — parses Excel files in the browser
  - Firebase 10.12.2 (compat SDK) — Firestore cloud storage
  - Inter font (Google Fonts)
- **Storage:** Firestore (primary) + localStorage (fallback/cache)
- **Auth:** Simple password check (`Mohamed@@2026`), stored in `sessionStorage`

---

## Firebase / Firestore

| Item | Value |
|------|-------|
| **Project ID** | kenzz3pl |
| **API Key** | *(in `index.html` → `FIREBASE_CONFIG.apiKey`)* |
| **App ID** | *(in `index.html` → `FIREBASE_CONFIG.appId`)* |
| **Firestore collection** | `kenzz3pl` |
| **Console** | https://console.firebase.google.com/u/0/project/kenzz3pl |
| **Rules URL** | https://console.firebase.google.com/u/0/project/kenzz3pl/firestore/rules |

**Firestore Documents:**
- `bosta` / `bosta_0` … `bosta_N` — Bosta audit data (chunked, 2000 rows/doc)
- `aramex` / `aramex_0` … — Aramex audit data (chunked)
- `mashreq` / `mashreq_0` … — Mashreq data (chunked)
- `raya` / `raya_0` … — Raya data (chunked)
- `breakdown` — Data Breakdown tab store
- `aramex_cfg` — Aramex rate configuration

**Data is gzip-compressed + base64 encoded** before saving to Firestore (97% size reduction).
Chunk size = 2000 rows per doc to stay under Firestore's 1MB doc limit.

**Firestore Security Rules** (must be updated annually):
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if request.time < timestamp.date(2027, 12, 31);
    }
  }
}
```
**Current expiry: 2027-12-31.** When rules expire, all reads/writes silently fail and data won't load.

---

## localStorage Keys

| Key | Contents |
|-----|----------|
| `kenzz3pl_bosta_v1` | Bosta data cache |
| `kenzz3pl_aramex_v1` | Aramex data cache |
| `kenzz3pl_mq_v1` | Mashreq data cache |
| `kenzz3pl_raya_v1` | Raya data cache |
| `kenzz3pl_lang` | UI language (`ar` / `en`) |
| `kenzz3pl_auth_v1` | Session auth key (sessionStorage) |

---

## Data Load / Save Flow

**On login → `showApp()` → `cloudLoadAll()`:**
1. Loads all 4 carriers + breakdown from Firestore in parallel
2. Decompresses gzip, parses JSON
3. Populates `bostaData`, `auditData`, `mqData`, `rayaData`, `bdStore`
4. Renders dirty tabs

**On "Save Results" → `saveBosta()` / `saveAramex()` etc.:**
1. Merges new month data into persisted data (keeps other months, replaces uploaded month)
2. `cloudSaveChunked(key, rows)` — chunks into 2000-row docs, gzips, saves to Firestore
3. Deletes stale extra chunks from previous saves

**Multi-month accumulation:**
- Global state: `_bostaPersisted`, `_aramexPersisted`, `_mqPersisted`, `_rayaPersisted`
- Save merges by `uploadMonth`: keeps rows for OTHER months, replaces rows for the CURRENT uploaded month
- Month tabs (All / Jul'26 / Jun'26 / May'26) appear automatically when 2+ months exist

**Backup/Restore:**
- 💾 Backup button: exports ALL localStorage data as `.json` file
- 📂 Restore button: re-imports backup JSON, restores all carriers

---

## Tab Structure

| Tab | Description |
|-----|-------------|
| Bosta Audit | Upload Bosta billing Excel + optional contract, audit per row |
| Aramex Audit | Upload Aramex report + Odoo export + zone map + status file |
| Aramex Mashreq | Upload Mashreq billing + optional rates file + second-attempt file |
| Raya Review | Upload Raya billing + optional contract file |
| Analytics | Auto-generated from all carrier data — ACPS bridge, size/zone tables |
| Data Breakdown | Manual entry / auto-population of monthly KPI table |

---

## Bosta Audit

### Files Required
| File | Description |
|------|-------------|
| Bosta Report (required) | Monthly billing Excel from Bosta portal |
| Contract (optional) | Kenzz rate card Excel — enables overbilling detection |

### Bosta Report File Formats
Two variants exist from the Bosta portal:

**Format 1 — Billing/Breakdown file** (e.g. `Bosta Breakdown-July'26.xlsx`):
- Sheet dimension: `A1:AN23219` — headers at row 1, data from row 2
- Contains real `Price Before VAT` values (52.5, 61.5, etc.)
- Located at `D:\3PL\Non-Trade\Bosta\2026\`

**Format 2 — Portal tracking download** (auto-named `Book1.xlsx`):
- Sheet dimension: `B2:AO23409` — 11 rows of invoice metadata, headers at row 12
- `Price Before VAT` = 0 for all rows (pre-invoice / tracking report)
- Downloaded directly from Bosta portal website

**Header detection logic** (`processBosta()`):
1. Try `sheet_to_json({defval:''})` — works for Format 1
2. If `rawRows[0]` lacks 'Tracking Number' key → fallback to `sheet_to_json({header:1})` array mode
3. Find header row by searching for 'Tracking Number' cell
4. Manually build row objects from that header row

### Price BAV Fallback
When `Price Before VAT = 0` (Format 2 / tracking file) and contract is loaded:
- `effectiveBAV = kShip1 + kShip2 + insurFee` (Kenzz contract rate)
- `diff = 0` for these rows (no overbilling measurable)
- Total Invoice shows estimated amount from contract

### Column Mapping
| Field | Excel Column | Fallback |
|-------|-------------|---------|
| Tracking Number | `Tracking Number` | `tracking number`, `Tracking` |
| Pkg Size | `Pkg Size` | `pkg size` |
| Type | `Type` | `type` |
| State | `State` | `state` |
| Zone | `Zone` | `zone` |
| Insurance Fees | `Insurance Fees` | `insurance fees` |
| Price Before VAT | `Price Before VAT` | `price before vat` |
| Comments | `Comments` | `comments`, falls back to `Type` |

### Row Type Labels (`BOSTA_LABEL`)
| Raw Type / Comment | Display Label |
|-------------------|--------------|
| SEND | Delivered |
| FXF_SEND | Delivered |
| CASH_COLLECTION | Delivered |
| EXCHANGE | Delivered |
| RTO | RTO |
| CUSTOMER_RETURN_PICKUP | RTN |
| SIGN_AND_RETURN | RTN |
| COD Charges | COD Fees |
| Lost & Damaged | Lost & Damaged |
| Free Orders | Last Mile CN |

### Special Rows (`BOSTA_SPECIAL`)
`Lost & Damaged`, `Free Orders`, `COD Charges` — these are financial adjustment rows:
- `kShip1 = priceBAV` (use Bosta's reported price as-is)
- `kShip2 = 0`
- `diff = 0`
- `COD Charges` rows are **excluded** from results (`return null`)

### Calculations Per Row
```
kShip1 = bostaShipFee(contract, type, zone)     ← from Contract shipping table
kShip2 = bostaPkgFee(contract, pkgSize, zone)   ← from Contract PKG table (0 for Normal)
diff   = effectiveBAV - kShip1 - kShip2 - insurFee
isOver = |diff| > threshold
```

### Contract Excel Format
The Contract file has two tables:
1. **Shipping table**: rows = Type (SEND/RTO/etc.), cols = Zone names, values = EGP rate
2. **PKG table**: rows = Pkg Size (Large/X-Large/etc.), cols = Zone names, values = extra EGP fee

Parser (`parseBostaContract`) auto-detects both tables by scanning for header rows containing 'Type' or zone names.

### KPI Cards
- **Total Shipments** — row count after filtering
- **Total Invoice** — `sum(effectiveBAV)`
- **Overcharged** — count of rows where `|diff| > threshold`

---

## Aramex Audit

### Files Required
| File | Description |
|------|-------------|
| Aramex Report (required) | Monthly Aramex billing Excel |
| Odoo Export (optional) | Odoo shipment data for COD matching |
| Zone Map (optional) | City → Zone mapping Excel |
| Status File (optional) | Delivery status per AWB |

### Rate Configuration (per session, saved to Firestore)
| Parameter | Default | Description |
|-----------|---------|-------------|
| COD Threshold | — | Min COD amount before COD fees apply |
| COD % | — | COD fee percentage above threshold |
| Annual % | — | Annual/surcharge percentage |
| Fuel % | — | Fuel surcharge percentage |
| Over Threshold | — | EGP diff threshold to flag as overcharge |

### COD Fee Formula
```
if (isCOD && isCODCollected && paymentValue >= COD_THRESH):
    codFees = (paymentValue - COD_THRESH) × COD_PCT
else:
    codFees = 0
```

---

## Aramex Mashreq

### Files Required
| File | Description |
|------|-------------|
| Main Report (required) | Mashreq monthly billing Excel |
| Rates File (optional) | Kenzz rate card — last mile + pickup by size/zone |
| Second Attempt File (optional) | Second delivery attempt charges |

### Configuration
| Parameter | Description |
|-----------|-------------|
| Insurance % | `calcIns = goodsValue × INS_PCT` |
| COD % | `calcCOD = codValue × COD_PCT` (delivered only) |
| Threshold | EGP diff to flag |

### Billing Rules
- **Delivered / Returned:** `calcLM = rate[size][destZone] × qty`
- **Cancel on Floor:** `calcLM = actualCharge × 0.5` (dispute 50% only), accept pickup/ins/COD as-is
- **Second attempt file:** all charges summed as a flat `mqSecondTotal` added to invoice

### Size detection
Reads `Large QTY`, `Medium QTY`, `Small QTY` columns (July+ format).
Falls back to single size column for older formats.

---

## Raya Review

### Files Required
| File | Description |
|------|-------------|
| Raya Report (required) | Raya monthly billing Excel (auto-detects best sheet) |
| Contract (optional) | Kenzz contract rates |

### Price Column Detection
Tries multiple column names in order:
`Price`, `DL Cost`, `DLCost`, `Delivery Cost`, `DeliveryCost`, `DL price`, `DLPrice`, `Raya Price`, `RayaPrice`

If `DL Cost = 0` but `Total Billed > 0`:
```
billedPrice = totalBilled - pickupCost - codFees - insFees - extraCost
```

### Contract Rate Formula
- **With size QTY columns:** `contractPrice = (Large rate × largeQty) + (Medium rate × mediumQty) + (Small rate × smallQty)`
- **Without size QTY:** `contractPrice = rate[category] × totalQty`
- **Returns/Cancels:** uses `contract.returns` table instead of `contract.delivery`

---

## Multi-Month Feature (All Carriers)

1. Upload file for a specific month
2. Set **Data Month** picker (e.g. July 2026)
3. Click **Process & Analyze** (month filter resets to 'ALL')
4. Click **Save Results** — merges this month into cloud storage
5. Repeat for each month

Month tabs appear above KPI cards when 2+ months are in the dataset.
Month key format: `"Jul'26"`, `"Jun'26"`, etc.

---

## Data Recovery Procedure

If data disappears from the website:

1. **Check Firestore rules** — if expired, update at https://console.firebase.google.com/u/0/project/kenzz3pl/firestore/rules (extend date to next year)
2. **Log out and log back in** — triggers fresh `cloudLoadAll()` from Firestore
3. **Check Firestore database** — https://console.firebase.google.com/u/0/project/kenzz3pl/firestore/data — confirm docs exist
4. **If Firestore has no data** — re-upload each month's Excel files and save
5. **Backup file** — if you have a `.json` backup from the 💾 Backup button, use 📂 Restore

**Source files location:** `D:\3PL\Non-Trade\Bosta\2026\`
| File | Month |
|------|-------|
| `Bosta Breakdown- May'26.xlsx` | May 2026 |
| `Bosta Breakdown-July'26.xlsx` | July 2026 |
| `Contract.xlsx` | Bosta rate contract |

---

## Common Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| All data missing on login | Firestore rules expired | Update rules date in Firebase console |
| Bosta Total Invoice = 0 | Uploaded tracking file (not billing file) — Price BAV = 0 | App auto-uses contract rate; or upload `Bosta Breakdown-*.xlsx` |
| Blank table after upload | Month filter was set to a different month | Fixed: month resets to 'ALL' on every new upload |
| Data not saving | Firestore rules expired during save | Fix rules first, then re-save |
| Website shows old version | CDN cache (Fastly ~10 min TTL) | Use `/app.html` URL or wait 10 min |
| Month tabs not appearing | Only 1 month in data | Upload and save 2+ months |

---

## Analytics Tab

Auto-generated from all saved carrier data. Sections:
- **Bosta:** shipment value breakdown by label, zone/size table, ACPS bridge
- **Aramex:** similar breakdown
- **Mashreq:** similar breakdown  
- **Raya:** size/zone value dashboard, ACPS bridge
- **ACPS Bridge:** month-over-month average cost per shipment decomposition (3 factors: size mix, zone mix, residual)

Requires navigating to Analytics tab to trigger render (lazy rendering via `_tabDirty` flags).

---

## Key Constants (in code)

```js
AUTH.PASSWORD = '...'           // see index.html line ~743
AUTH.SESSION_KEY = 'kenzz3pl_auth_v1'
STORAGE_KEY = 'kenzz3pl_bosta_v1'
CHUNK_SIZE = 2000               // rows per Firestore document
```
