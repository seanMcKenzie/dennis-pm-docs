# MedSales Sprint 4 — Product Requirements Document

**Author:** Dennis Reynolds, PM  
**Sprint:** 4 (ID: 5 in dashboard.db)  
**Dates:** 2026-03-17 → 2026-03-31  
**Status:** Active  
**Project ID:** 2

---

## Sprint Goal

Surface enrichment intelligence — Open Payments pharma payments, Medicare Part D prescribing patterns, and Part B procedure utilization — directly on the physician profile screen in the iOS app.

This is the intelligence layer. This is what turns MedSales from a fancy contacts app into a tool that actually gives sales reps leverage in the field.

---

## Background & Motivation

Sprint 3 delivered real Oklahoma NPPES physician data, geocoding, and all three CMS enrichment datasets loaded into the database:

- `open_payments` — pharma manufacturer payments to physicians
- `part_d_drugs` — Medicare Part D drug prescribing claims
- `part_b_utilization` — Medicare Part B procedure billing

The data is *there*. It's just sitting in Postgres doing nothing. That ends in Sprint 4.

A sales rep calling on a cardiologist should be able to open that physician's profile and immediately see: which drug companies are paying them, what drugs they're prescribing to Medicare patients, and what procedures they're billing. That context changes the entire sales conversation. We're building that.

---

## Sprint 4 Scope

### 1. Backend Enrichment API Endpoints

Three new REST endpoints + one enhancement to the existing profile endpoint.

#### S4-01 · Flyway V14 Migration

**Ticket ID:** 61  
**Assigned:** Charlie  

- Add B-tree indexes: `open_payments(npi)`, `part_d_drugs(npi)`, `part_b_utilization(physician_npi)`
- Populate or redefine `physician_cms_profiles` as an aggregated view/table per NPI
- Fields: `total_payment_amount`, `payment_count`, `top_manufacturer`, `top_drug_generic`, `top_hcpcs_code`, `is_opioid_prescriber`
- **Prerequisite for all S4-02 through S4-05 work**

#### S4-02 · GET /api/v1/physicians/{npi}/payments

**Ticket ID:** 62  
**Assigned:** Charlie  

```
GET /api/v1/physicians/{npi}/payments

Response:
{
  "npi": "string",
  "totalAmount": number,
  "paymentCount": number,
  "payments": [
    {
      "manufacturer": "string",
      "amount": number,
      "natureOfPayment": "string",
      "product": "string"
    }
  ]
}
```

Sorted by amount descending. Source: `open_payments` table.

#### S4-03 · GET /api/v1/physicians/{npi}/prescribing

**Ticket ID:** 63  
**Assigned:** Charlie  

```
GET /api/v1/physicians/{npi}/prescribing

Response:
{
  "npi": "string",
  "totalClaims": number,
  "totalDrugCost": number,
  "opioidPrescriber": boolean,
  "drugs": [
    {
      "brandName": "string",
      "genericName": "string",
      "totalClaims": number,
      "totalDrugCost": number,
      "opioidFlag": boolean
    }
  ]
}
```

Sorted by totalClaims descending. Source: `part_d_drugs` table.

#### S4-04 · GET /api/v1/physicians/{npi}/procedures

**Ticket ID:** 64  
**Assigned:** Charlie  

```
GET /api/v1/physicians/{npi}/procedures

Response:
{
  "npi": "string",
  "procedureCount": number,
  "procedures": [
    {
      "hcpcsCode": "string",
      "description": "string",
      "lineServiceCount": number,
      "avgMedicarePayment": number
    }
  ]
}
```

Sorted by lineServiceCount descending. Source: `part_b_utilization` table.

#### S4-05 · Enrich Existing Profile Endpoint

**Ticket ID:** 65  
**Assigned:** Charlie  

Update `GET /api/v1/physicians/{npi}` to include enrichment summary inline:

```json
"enrichmentSummary": {
  "totalOpenPayments": number,
  "openPaymentsCount": number,
  "isOpioidPrescriber": boolean,
  "topProcedure": {
    "hcpcsCode": "string",
    "description": "string"
  }
}
```

**Additive only — no breaking changes to existing response contract.**

---

### 2. iOS Profile UI Sections

Three new collapsible card sections on `PhysicianProfileScreen`, plus CRM history.

#### S4-06 · Open Payments Card

**Ticket ID:** 66  
**Assigned:** Charlie  

- Collapsible card section, collapsed by default
- Header: total payment amount (formatted), payment count, top manufacturer
- Expanded: scrollable list of individual payments
- Visual design: warning-tone accent (amber/orange) — payments signal potential conflict of interest, which is relevant context for reps
- Loading skeleton while API call in flight
- Data source: S4-02 endpoint

#### S4-07 · Part D Prescribing Card

**Ticket ID:** 67  
**Assigned:** Charlie  

- Collapsible card section
- Header: total claims, total drug cost, opioid prescriber badge (if applicable)
- **Opioid prescriber badge must be prominent** — bold red/orange pill badge in the header. This is the most important signal for formulary sales calls.
- Expanded: top 5 drugs by claim count, with opioid flag indicator per drug
- Loading skeleton while API call in flight
- Data source: S4-03 endpoint

#### S4-08 · Part B Procedures Card

**Ticket ID:** 68  
**Assigned:** Charlie  

- Collapsible card section
- Header: procedure count, top procedure description
- Expanded: top procedures by service count, HCPCS code + description, avg Medicare payment
- Helps reps understand clinical focus and billing patterns
- Loading skeleton while API call in flight
- Data source: S4-04 endpoint

#### S4-09 · CRM Interaction History Timeline

**Ticket ID:** 69  
**Assigned:** Charlie  

- Show last 5 CRM interactions on physician profile
- Fields: date, type (call/visit/email), notes preview (truncated to 80 chars)
- Tap to expand full note
- **No new backend endpoints required** — use existing CRM call logging API
- This is the "what have I done with this doctor" context that reps need before walking in

---

### 3. Bug Fixes (Carry-Forward)

Both bugs are P2 priority and must be resolved in Sprint 4.

#### BUG-TC26 · physician_cms_profiles Empty

**Ticket ID:** 70  
**Assigned:** Charlie  

`physician_cms_profiles` exists but contains zero rows. The Flyway V14 migration (S4-01) will either populate this table or replace it with a PostgreSQL VIEW aggregating enrichment data per NPI. This is blocking the S4-05 inline summary enrichment.

**Fix path:** Implement as a `CREATE MATERIALIZED VIEW` or populate via `INSERT INTO ... SELECT` in V14 migration.

#### BUG-TC27 · opioid_flag Not Surfaced in Search

**Ticket ID:** 71  
**Assigned:** Charlie  

The `opioid_flag` column on `part_d_drugs` is loaded but not exposed. Sales managers building rep territory call lists need to filter by opioid-prescribing physicians.

**Fix:**
1. Add `isOpioidPrescriber` to `PhysicianSearchResponse` DTO
2. Add `opioidPrescriber=true|false` optional query param to `GET /api/v1/physicians/search`
3. Implement via JOIN on `part_d_drugs` where `opioid_flag = true` for at least one record per NPI

---

### 4. Search Enhancements

#### S4-10 · Extended Search Filters

**Ticket ID:** 72  
**Assigned:** Charlie  

Extends the physician search endpoint with targeted filter parameters for sales intelligence:

| Parameter | Type | Description |
|-----------|------|-------------|
| `specialty` | string | Exact/partial specialty match (already partially implemented) |
| `hasOpenPayments` | boolean | Filter to only physicians with payment records |
| `opioidPrescriber` | boolean | Filter by opioid prescribing flag (from BUG-TC27 fix) |
| `minTotalPayments` | number | Minimum total Open Payments amount threshold |

iOS search/filter screen must expose these as toggle/input controls.

---

## Technical Constraints

| Item | Constraint |
|------|-----------|
| Flyway | Currently at V13 — Sprint 4 work starts at V14 |
| Auth | Spring Boot local profile, no JWT enforcement — no auth changes in this sprint |
| Existing endpoint | `GET /api/v1/physicians/{npi}` must remain backward-compatible |
| Database | PostgreSQL (medsales-postgres Docker container) |
| Mobile | React Native (Hermes engine) — avoid URLSearchParams.set (known Hermes bug, fixed in BUG-010) |

---

## Definition of Done

- [ ] All three enrichment endpoints return valid JSON with real data from Postgres
- [ ] `GET /api/v1/physicians/{npi}` includes enrichment summary (no 404/500)
- [ ] iOS physician profile screen shows all three enrichment cards (collapsed by default)
- [ ] CRM history timeline visible on profile screen with ≥1 interaction shown
- [ ] BUG-TC26: physician_cms_profiles queryable (not empty)
- [ ] BUG-TC27: `opioidPrescriber` filter works end-to-end in search
- [ ] Search enhancements available on iOS filter screen
- [ ] Flyway V14 migration runs clean on fresh DB + existing dev DB
- [ ] Unit tests: ≥80% coverage on new controller/service classes
- [ ] Integration tests: all new endpoints covered

---

## Out of Scope (Sprint 4)

- Web companion / manager dashboard
- Territory management
- Push notifications
- Route export
- Performance/load testing

---

## Success Metric

A sales rep opens the physician profile for Dr. Jane Smith (NPI: 1234567890) and within 2 seconds sees:
1. She received $4,200 from Pfizer for "Food and Beverage" in 2023
2. She prescribes Eliquis (apixaban) at 340 claims/year
3. Her top procedure is echocardiography (93306)
4. Your last call with her was March 3rd and she was interested in the formulary update

*That* is a prepared sales rep. That's what Sprint 4 delivers.

---

*— Dennis Reynolds, PM*  
*"I have a very particular set of skills. One of them is making backlogs that actually make sense."*
