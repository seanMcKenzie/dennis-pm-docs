# Sprint 5 PRD — MedSales iOS App
**Author:** Dennis Reynolds, Project Manager  
**Sprint:** Sprint 5 (sprint_id=6)  
**Dates:** 2026-03-19 → 2026-04-01  
**Status:** Active

---

## Sprint Goal

Deliver advanced CMS-based physician search filters (Medicare Part B HCPCS, Part D drug prescribing, Open Payments payer data), real Apple Maps on the physician profile, and search-by-name/NPI on the physician list screen.

---

## Background & Motivation

Sprint 4 shipped CMS data ingestion and basic physician search. The data is there — now we use it. Sales reps need to filter physicians by *what procedures they actually bill*, *what drugs they actually prescribe*, and *who's paying them*. This turns our physician search into a targeting tool, not just a directory.

Secondary scope fixes two UX gaps that were embarrassing to ship with: an emoji placeholder where a map should be, and no way to find a specific doctor by name without scrolling forever.

---

## Features

### 1. CMS-Based Physician Search Filters (Backend)

**Endpoint:** `GET /api/v1/physicians/search`

Six new optional query parameters:

| Param | Type | Source Table | Behavior |
|---|---|---|---|
| `hcpcsCode` | string | `part_b_physicians` | Match `hcpcs_code` exactly |
| `minProcedureCount` | integer | `part_b_physicians` | `line_srvc_cnt >= N` |
| `drugName` | string | `part_d_prescribers` | `brand_name ILIKE` OR `generic_name ILIKE` |
| `minDrugClaims` | integer | `part_d_prescribers` | `total_claim_count >= N` |
| `payerName` | string | `open_payments` | `manufacturer_name ILIKE` |
| `minPaymentAmount` | decimal | `open_payments` | `total_amount_of_payment >= N` |

**Implementation notes:**
- All params are optional. Unset params don't affect the query.
- Filters are AND-combined with existing filters (specialty, location radius, etc.)
- Joins are on `npi` (the physician's NPI number)
- Flyway migration V14 must add appropriate indexes before query changes land
- Return existing physician response shape — no new fields needed

**Database migration (V14):**
```sql
CREATE INDEX IF NOT EXISTS idx_part_b_hcpcs ON part_b_physicians(hcpcs_code);
CREATE INDEX IF NOT EXISTS idx_part_b_npi ON part_b_physicians(npi);
CREATE INDEX IF NOT EXISTS idx_part_d_npi ON part_d_prescribers(npi);
CREATE INDEX IF NOT EXISTS idx_part_d_drug ON part_d_prescribers(brand_name, generic_name);
CREATE INDEX IF NOT EXISTS idx_open_payments_npi ON open_payments(npi);
CREATE INDEX IF NOT EXISTS idx_open_payments_manufacturer ON open_payments(manufacturer_name);
```

---

### 2. CMS Filter UI (iOS)

Add filter inputs to the physician search/filter screen for all 6 new params:

**HCPCS / Procedure filters:**
- `hcpcsCode`: text field with label "HCPCS Code (e.g. 99213)"
- `minProcedureCount`: numeric input with label "Min. Procedure Count"

**Drug filters:**
- `drugName`: text field with label "Drug Name"
- `minDrugClaims`: numeric input with label "Min. Claim Count"

**Payer filters:**
- `payerName`: text field with label "Payer / Manufacturer"
- `minPaymentAmount`: currency input with label "Min. Payment Amount ($)"

**UX requirements:**
- Show an active filter badge/count on the filter button when any CMS filter is set
- Validate numeric fields (no negative values, no non-numeric input)
- Clear All button resets all filters including CMS params

---

### 3. Apple Maps on Physician Profile (iOS)

**Current state:** The physician profile screen shows a `🗺 Map view` text placeholder.

**Target state:** Replace with a real `react-native-maps` MapView:
- Provider: `PROVIDER_DEFAULT` (Apple Maps on iOS — no API key needed)
- Show a single pin at the physician's practice address coordinates
- Map is non-interactive (static view) but supports pinch-to-zoom
- Tapping the map opens Apple Maps app with the physician's address
- If lat/lng is missing, fall back to showing the address text with a "View in Maps" link

**Dependencies:**
- `react-native-maps` should already be installed; if not, add it
- Backend physician response must include `latitude` / `longitude` — verify this is present from Sprint 4 geocoding work

---

### 4. Search-by-Name / NPI on Physician List Screen (iOS)

**Current state:** Physician list shows all results from the current filter set. No free-text search.

**Target state:**
- Search bar at the top of the physician list screen
- User types a name (partial match OK) or a full NPI number
- Debounce: 300ms after last keystroke before firing API call
- Backend call: `GET /api/v1/physicians/search?name=<query>` or `?npi=<query>`
- Loading indicator while request is in flight
- Empty state: "No physicians found for '[query]'" with a clear button
- Clear (×) button resets to unfiltered list

**Note for backend:** If `name` and `npi` params aren't already supported on the search endpoint, add them as part of this ticket.

---

### 5. Sprint 4 Regression Test Suite (QA)

Mac updates the QA regression suite to cover:
- Specialty filter end-to-end
- Location radius filter end-to-end
- CMS data sections on physician profile (Part B, Part D, Open Payments)
- Any known flaky tests are flagged and documented

Full suite runs against staging before Sprint 5 acceptance testing begins.

---

## Acceptance Criteria

### CMS Filters — Backend
- [ ] `GET /api/v1/physicians/search?hcpcsCode=99213` returns only physicians with that HCPCS code in Part B data
- [ ] `minProcedureCount=100` excludes physicians with fewer than 100 services
- [ ] `drugName=lipitor` matches both brand_name and generic_name (case-insensitive)
- [ ] `payerName=pfizer` uses ILIKE — matches "Pfizer Inc", "Pfizer US Pharmaceuticals"
- [ ] Combining multiple CMS filters ANDs them together
- [ ] Existing filters (specialty, location) still work when CMS filters are also set
- [ ] V14 migration runs cleanly with no errors

### CMS Filters — iOS
- [ ] All 6 filter inputs visible and functional on filter screen
- [ ] Active filter badge shows correct count of active CMS filters
- [ ] Clear All resets CMS filter values
- [ ] Numeric validation prevents invalid input

### Apple Maps
- [ ] MapView renders on physician profile (not a placeholder)
- [ ] Pin is positioned at the correct address
- [ ] Tapping opens Apple Maps with the physician's address
- [ ] Graceful fallback if coordinates are missing

### Search-by-Name / NPI
- [ ] Typing a physician's last name filters the list correctly
- [ ] Typing a full NPI returns exact match
- [ ] Debounce prevents excessive API calls
- [ ] Empty state shows correct message
- [ ] Clear button resets the list

### QA
- [ ] Sprint 4 regression suite updated and all tests pass on staging
- [ ] Sprint 5 acceptance test results documented

---

## Tickets

| ID | Title | Assignee |
|---|---|---|
| #73 | [Backend] DB migration V14 — CMS search filter indexes | developer |
| #74 | [Backend] Extend physician search — hcpcsCode + minProcedureCount params | developer |
| #75 | [Backend] Extend physician search — drugName + minDrugClaims params | developer |
| #76 | [Backend] Extend physician search — payerName + minPaymentAmount params | developer |
| #77 | [Backend] Integration tests for all CMS search filter params | developer |
| #78 | [Backend] OpenAPI/Swagger docs update for new search params | developer |
| #79 | [iOS] CMS filter UI — HCPCS code + procedure count filter inputs | developer |
| #80 | [iOS] CMS filter UI — drug name, drug claims, payer, payment amount filter inputs | developer |
| #81 | [iOS] Apple Maps MapView on physician profile screen | developer |
| #82 | [iOS] Search-by-name / NPI on physician list screen | developer |
| #83 | [QA] Sprint 4 regression test suite update | qa |
| #84 | [QA] Sprint 5 acceptance testing — CMS filters, Apple Maps, search-by-name | qa |

---

## Out of Scope (Sprint 5)

- Android Maps (Google Maps) — iOS-only this sprint
- CMS filter presets / saved searches
- Physician comparison view
- Push notifications

---

## Dependencies & Risks

| Risk | Mitigation |
|---|---|
| `part_b_physicians` / `part_d_prescribers` queries are slow without indexes | V14 migration adds indexes before any query changes — must land first |
| Physician lat/lng may not be populated for all records | Apple Maps ticket includes fallback for missing coordinates |
| `react-native-maps` native module may need pod install | Developer to verify pods are linked; run `npx pod-install` if needed |
| Part D drug name matching (brand vs generic) needs clarification | Match both columns with ILIKE — confirmed in spec |

---

*Dennis Reynolds, PM — MedSales Sprint 5*  
*"I am a golden god of project management."*
