# PETRAS GROUP — SYSTEM STATUS
_Last updated: 2026-03-12 (Session 13)_

---

## Credentials
- **AT Token:** `[AT_TOKEN]`
- **Anthropic Key:** `[ANTH_KEY]`
- **Base ID:** `appElT5CQV6JQvym8`
- **GitHub Token:** `[GH_TOKEN]`
- **GitHub Pages:** `https://dimitrispetras21-del.github.io/petras-assign/`
- **GitHub Repo:** `https://github.com/dimitrispetras21-del/petras-assign`
- **Dispatcher email:** `d.petras@petrasgroup.com`

---

## Airtable Table IDs
| Table | ID |
|---|---|
| ORDERS | tblgHlNmLBH3JTdIM |
| TRIPS | tblgoyV26PBc6L9uE |
| TRIP COSTS | tblWUus6uSpqE1LMW |
| TRUCKS | tblEAPExIAjiA3asD |
| TRAILERS | tblDcrqRJXzPrtYLm |
| DRIVERS | tbl7UGmYhc2Y82pPs |
| DRIVER LEDGER | tblZVr4BCr9sGFf8n |
| FUEL RECEIPTS | tblxRFsMeVhlLrBjF |
| RAMP PLAN | tblT8W5WcuToBQNiY |
| PALLET LEDGER | tblAAH3N1bIcBRPXi |
| CLIENTS | tblFWKAQVUzAM8mCE |
| LOCATIONS | tblxu8DRfTQOFRCzS |

---

## GitHub Pages Apps
| App | URL | Status |
|---|---|---|
| Trip Assignment v2 | `.../petras_assign-2.html` | ✅ Live |
| Order Intake | `.../order_upload.html` | ✅ Live |
| Pallet Upload v2 | `.../pallet_upload_v2.html` | ✅ Live |
| Trip P&L Entry | `.../trip_costs.html` | ✅ Live |
| Fuel & Tolls Import | `.../fuel_import.html` | ✅ Live |

---

## fuel_import.html — Architecture (Session 13)

### DADI Tab (Испратница receipts)
- Upload PDF → Claude AI extracts per receipt: `receipt_no`, `plate`, `date`, `liters`, `amount_mkd`, `currency`, `km`, `driver`
- **Matching:** `normPlate()` with Greek→Latin transliteration (Ι→I, Α→A, Β→B, Ρ→P etc.)
- **Trip range:** Start = Export Loading DateTime, End = Import Delivery DateTime OR Export Delivery +1 day
- **Truck plate match** → Fuel Type: `🏠 DADI Departure`
- **Trailer plate match** → Fuel Type: `❄️ Reefer`
- **Save:** 1 FUEL RECEIPTS record per Испратница + TRIP COSTS upsert (dates only)
- **Duplicate check:** `Invoice Number` = `DADI-XXXXXX` — skips if already exists

### DKV Tab (E-Invoice)
- Upload PDF → Claude AI extracts per line: `plate`, `date`, `product`, `amount_eur`, `currency`, `liters`, `country`
- **Products:** `TOLL_AT/HU/DE/SK/CZ/PL/BE/NL/HR/SL/RO/BG/RS/MK` → `Tolls XX` fields
- **FUEL** → FUEL RECEIPTS (⛽ External)
- **Everything else** (AdBlue, system fee etc.) → `Other Expenses` in TRIP COSTS
- **Save:** FUEL RECEIPTS for diesel + TRIP COSTS patch (dates + tolls + other)

### Currency Conversion
- Live rates from `frankfurter.app` (ECB) on page load
- Fallback static rates if offline (MKD=61.5, HUF=390, CZK=25.3, PLN=4.25 etc.)
- `toEur(amount, currency)` used for all conversions

### Match Statuses
| Status | Meaning |
|---|---|
| ✅ OK | plate + date within trip range |
| ⚠️ Approx | plate found, date outside range → dropdown |
| ⚡ Conflict | 2 trips overlap → dropdown |
| ❌ No plate | not found in trucks or trailers |

- **All records save** (matched or not) — unmatched = `Assignment Status: ⚠️ Pending`
- **Red rows** for unmatched — dropdown shows all trips for manual assignment

---

## TRIP COSTS Table — Design

### Core Model
**1 TRIPS record → 1 TRIP COSTS record → full P&L**

### Field Structure
```
REVENUE
  Export Revenue        currency (manual)
  Import Revenue        currency (manual)
  Total Revenue         formula = Export + Import

COSTS — Owned Fleet
  DADI Cost             multipleLookupValues ← TRIPS rollup ← FUEL RECEIPTS (Fuel Type = 🏠 DADI)
  External Cost         multipleLookupValues ← TRIPS rollup ← FUEL RECEIPTS (Fuel Type = ⛽ External)
  Reefer Cost           multipleLookupValues ← TRIPS rollup ← FUEL RECEIPTS (Fuel Type = ❄️ Reefer)
  Tolls MK/RS/HU/AT/SK/CZ/BE/PL/HR/SL/DE/NL/IT/RO/BG   currency (from DKV import)
  Total Tolls           formula = SUM all toll fields
  Driver Pay            lookup ← TRIPS rollup ← DRIVER LEDGER (Trip Value) [PENDING — see below]
  Accommodation         currency (manual)
  Fixed Costs           lookup ← TRUCKS fixed cost profile
  Other Expenses        currency (from DKV import + manual)
  Fines                 currency (manual)
  Tires Service         currency (manual)
  Total Costs           formula = SUM all above [NEEDS UPDATE — see pending]

COSTS — Partner
  Partner Rate Export   currency (manual)
  Partner Rate Import   currency (manual)
  Total Partner Cost    formula

P&L
  Net Profit            formula = Total Revenue - IF(Partner, Partner Cost, Total Costs)
  Profit Margin %       formula
  PNL Status            formula: ✅ PROFIT / ⚪ BREAK EVEN / 🔴 LOSS
  Revenue Per KM        formula (Owned Fleet only)
  Cost per KM           formula
```

### ⚠️ PENDING — Manual Airtable changes required

**1. TRIPS rollup filters (6 fields) — must set via UI:**
In TRIPS table, each rollup field → toggle ON "Only include linked records from FUEL RECEIPTS that meet conditions":
| Rollup field | Condition |
|---|---|
| `DADI Liters` | Fuel Type = 🏠 DADI Departure |
| `DADI Cost` | Fuel Type = 🏠 DADI Departure |
| `External Liters` | Fuel Type = ⛽ External |
| `External Cost` | Fuel Type = ⛽ External |
| `Reefer Liters` | Fuel Type = ❄️ Reefer |
| `Reefer Cost` | Fuel Type = ❄️ Reefer |

**2. Driver Pay from DRIVER LEDGER:**
- Step 1: Add rollup field `Driver Pay Total` in TRIPS table → linked DRIVER LEDGER → `Trip Value` → SUM
- Step 2: Change `Driver Pay` in TRIP COSTS → Lookup → Trip Link → `Driver Pay Total`

**3. Total Costs formula update:**
```
SUM(
  {Total Tolls},
  {DADI Cost},
  {External Cost},
  {Reefer Cost},
  {Driver Pay},
  {Accommodation},
  {Fixed Costs},
  {Other Expenses},
  {Fines},
  {Tires Service}
)
```

---

## normPlate() — Greek→Latin transliteration
```javascript
const greek2latin = {
  'Ι':'I','Α':'A','Β':'B','Ε':'E','Ζ':'Z','Η':'H',
  'Κ':'K','Μ':'M','Ν':'N','Ο':'O','Ρ':'P','Τ':'T','Υ':'Y','Χ':'X'
};
// Then strip spaces/dashes/dots and uppercase
```

---

## FUEL RECEIPTS Table — Fields
| Field | Type | Notes |
|---|---|---|
| Receipt ID | autoNumber | |
| Date | date | |
| Truck | multipleRecordLinks → TRUCKS | |
| Trip | multipleRecordLinks → TRIPS | |
| Odometer KM | number | |
| Liters | number | |
| Total Cost | currency | EUR (converted) |
| Station | text | |
| Country | singleSelect | GR/MK/RS/HU/AT/SK/CZ/PL/DE/RO/BG/HR/SI/NL/BE/IT |
| Invoice Number | text | DADI-XXXXXX or DKV |
| Notes | multilineText | |
| Assignment Status | singleSelect | ✅ Assigned / ⚠️ Pending / ❌ Unassigned |
| Fuel Type | singleSelect | 🏠 DADI Departure / 🏠 DADI Return / ⛽ External / ❄️ Reefer |

---

## DRIVER LEDGER — Fields
| Field | Type |
|---|---|
| Entry Number | Auto Number |
| Driver | Link → DRIVERS |
| Trip | Link → TRIPS |
| Entry Type | Single Select: International Trip / Payment / National Trip |
| Date | Date |
| Advance Payment | Currency |
| Driver Expenses | Currency |
| Trip Value | Currency |
| Net Payable | Formula |
| Payment Amount | Currency |
| Payment Method | Single Select: Bank Transfer / Cash |
| Running Balance | Formula |
| Notes | Long text |
| + lookup fields from TRIPS/NATIONAL TRIPS | — |

---

## Automations — Status
| # | Name | Trigger | Status |
|---|---|---|---|
| 1 | Veroia Switch → National Order | Order updated, Veroia Switch checked | ✅ Active |
| 2 | Empty record delete | Record created | ✅ Active |
| 3 | Duplicate detection | Record created | ✅ Active |
| 4 | Invoiced blocker | Pallet sheets check | ✅ Active |
| 5 | NATIONAL ORDER Invoiced sync | — | ✅ Active |
| 6 | Status Assigned/Pending sync | TRIPS updated | ✅ Script ready |
| 7 | Daily Briefing + Overdue Flag | Scheduled 08:00 | ✅ Script ready |
| 8 | Maintenance Alert | Scheduled 08:00 | ✅ Script ready |
| 9 | Invoice Reminder (7 days) | Scheduled 09:00 | ✅ Script ready |
| 10 | RAMP PLAN auto-create from ORDER | Order Veroia Switch checked | ⏳ Script ready, NOT added |

### Script Pattern (critical)
- `sendEmail()` does NOT exist in Airtable scripts
- Pattern: script → `output.set("key", value)` → separate Send Email action
- Must run **Test step** on script before output variables appear in Send Email
- `input.config()` must be called at top even with no inputs
- Use `console.log()` not `output.text()` in scripts

---

## Key Technical Notes
- **Linked record format API:** `["recXXX"]` plain string arrays — NOT `[{id:"recXXX"}]`
- **normPlate():** Greek→Latin transliteration before any plate comparison
- **getExistingCostRecord():** uses `FIND("recXXX", ARRAYJOIN({Trip Link}, ","))` formula
- **Airtable button formula:** use `RECORD_ID()` — NOT `{Record ID}`
- **Rollup filters:** must be set via UI (API does not support filter conditions on rollups)
- **Formula fields:** cannot be created via API (UNSUPPORTED_FIELD_TYPE_FOR_CREATE)
- Veroia Switch is INTERNAL — never notify clients
- PARTNERS field: `Adress` (single 'd') — exact spelling in API
- `anthropic-dangerous-direct-browser-access: true` required for browser→Anthropic API calls
- Trip date range: Start = Export Loading DateTime, End = Import Delivery DateTime OR Export Delivery +1

---

## Pending Items (Priority Order)

### IMMEDIATE — Manual Airtable (do before next session)
1. ⏳ **TRIPS rollup filters** — set Fuel Type condition on 6 rollup fields via UI
2. ⏳ **Driver Pay from DRIVER LEDGER** — add `Driver Pay Total` rollup in TRIPS, change `Driver Pay` in TRIP COSTS to lookup
3. ⏳ **Total Costs formula** — add DADI Cost + External Cost + Reefer Cost + Driver Pay

### HIGH PRIORITY
4. ⏳ **P&L Interface** — 1 row per trip, clean columns (no grouping): Trip Link, Plate, Driver, Export Revenue, Total Costs, Net Profit, Margin %, PNL Status
5. ⏳ **trip_costs.html** — revenue entry form (Export Revenue, Import Revenue, Trip Type)
6. ⏳ **DKV duplicate detection** — same as DADI (check Invoice Number before save)

### MEDIUM PRIORITY
7. ⏳ **RAMP PLAN automation** — script written, needs adding
8. ⏳ **Driver Payroll historical import** — waiting for Cowork CSV
9. ⏳ **Weekly P&L report view**
10. ⏳ **Roles/Permissions system**

### BACKLOG
11. ⏳ Cleanup — 1090 test records
12. ⏳ Daily Dispatcher Briefing via WhatsApp (Make.com)
13. ⏳ MyGeotab GPS integration
14. ⏳ SoftOne GO — myDATA invoicing

---

## Strategic Notes
- DPS Logistics = active operating company, dual-brand model in progress
- Wednesday Cutoff = all export orders until Wednesday for weekend delivery
- Proactive Pulse Protocol: Mission Start (30min) → Pre-Alert (Tue AM) → Fresh-Check Close (1h after CMR)
- Red Light Rule: CALL immediately for delays, update every 2h
- SoftOne GO + Airtable = recommended hybrid stack for myDATA compliance
