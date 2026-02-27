# Medical Sales Intelligence & CRM Platform
## Product Requirements Document

**Prepared by:** Dennis Reynolds, Project Manager  
**Date:** February 26, 2026  
**Version:** 1.1  
**Status:** Updated — Route Planning Revision (Feb 26, 2026)

### Revision History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | Feb 26, 2026 | Dennis Reynolds | Initial PRD — 48 FRs, 32 NFRs, 8 risks |
| 1.1 | Feb 26, 2026 | Dennis Reynolds | Route planning approach revised: FR-022/023 deferred, FR-024/025 replaced by FR-NEW-001 through FR-NEW-005; RISK-008 retired; RISK-NEW-001 and RISK-NEW-002 added; NFR-004 updated; Open Question 9 added |

---

## 1. Executive Summary

This document defines the requirements for a **Medical Sales Intelligence & CRM Platform** — a mobile-first application designed for pharmaceutical and medical device sales representatives. The platform aggregates publicly available federal healthcare data to produce rich physician profiles, supports territory management and route planning, and provides CRM functionality to track calls, communications, samples, and pipeline activity.

The core value proposition: a sales rep opens the app, sees every physician within 10 miles, taps any one of them, and immediately knows what they prescribe, what procedures they perform, who's paying them, where they work, and which hospitals they're affiliated with — then logs the call and moves to the next stop.

The data foundation is entirely built on free, publicly available CMS datasets. No commercial data licensing is required for the core product. This is a significant competitive advantage over legacy pharma data platforms that charge $50,000–$500,000/year for similar data.

**Target Users:** Pharmaceutical sales representatives, medical device sales representatives, territory sales managers, national accounts teams.

**Platform:** Mobile-first (iOS + Android) with a web companion for managers and reporting.

---

## 2. User Personas

### 2.1 Pharma Field Rep — Primary User
- **Name:** Sarah, 32, pharmaceutical sales rep
- **Territory:** 80–150 physicians across 3–5 counties
- **Day:** Drives 100–200 miles/day making 6–10 calls
- **Needs:** Know who to call before walking in. Know what they prescribe and whether they're a good target. Log the call fast. Plan the next route.
- **Pain points:** Current tools are desktop-only, data is stale, routing is manual, profile data is shallow.

### 2.2 Device Sales Rep — Primary User
- **Name:** Marcus, 38, medical device rep for a surgical robot company
- **Territory:** 20–40 surgeons and OR purchasing managers
- **Day:** Hospital-heavy. Needs to know procedure volumes, hospital affiliations, who the OR decision-makers are.
- **Needs:** Procedure volume data (Part B) is critical. Which surgeons are doing the most laparoscopic cases? Who's already on a competitor's device?

### 2.3 Territory Sales Manager — Secondary User
- **Name:** Linda, 45, manages 8 field reps across a 3-state region
- **Day:** Mostly desktop. Reviews rep activity, prioritizes targets, analyzes territory coverage.
- **Needs:** See rep activity dashboards. Identify coverage gaps. Pull reports on high-value physician targets the team hasn't contacted.

### 2.4 Sales Operations / Admin — Tertiary User
- Sets up territories, manages user accounts, exports data for CRM integration, manages system configuration.

---

## 3. Functional Requirements

### 3.1 Physician Data & Profiles

**FR-001** — The system shall maintain a physician master record for every active individual provider in the NPPES NPI Registry (entity type NPI-1, status = Active), comprising approximately 4–5 million individual providers.

**FR-002** — Each physician profile shall display the following identity fields sourced from NPPES:
- NPI (10-digit)
- Full name (first, middle, last, suffix)
- Credential (MD, DO, NP, PA, etc.)
- Gender
- Primary practice address (street, city, state, ZIP, phone, fax)
- Mailing address (if different from practice)
- Primary specialty (NUCC taxonomy code decoded to plain-English specialty name)
- All secondary specialties (up to 14 additional taxonomy entries)
- State license number(s) and issuing state(s)
- NPI enumeration date and last updated date
- Sole proprietor indicator

**FR-003** — Each physician profile shall display the following fields sourced from the CMS Provider Data Catalog (National Downloadable File):
- Group practice name and PAC ID
- Number of providers in the group practice
- Medical school attended
- Medical school graduation year
- Primary and secondary specialties (CMS plain-English format)
- Telehealth acceptance (Y/N)
- Medicare individual assignment status (accepts assignment Y/N/M)
- Medicare group assignment status

**FR-004** — Each physician profile shall display all hospital affiliations sourced from the CMS Facility Affiliation Data file, including facility type (Hospital, FQHC, RHC, etc.) and CMS Certification Number (CCN).

**FR-005** — Each hospital affiliation displayed shall be expandable to show the affiliated hospital's profile data from Hospital Compare General Information, including: hospital name, address, phone, hospital type, ownership type, emergency services availability, and overall CMS star rating (1–5).

**FR-006** — Each physician profile shall display a Medicare Part B Utilization section showing the top procedures billed to Medicare (by HCPCS code), including: procedure code, procedure description, total services billed, total unique beneficiaries, total submitted charges, and total Medicare payment amount. Data shall be sourced from the CMS Medicare Physician & Other Practitioners by Provider and Service dataset (most recent available year).

**FR-007** — Each physician profile shall display a Medicare Part D Prescribing section showing the top drugs prescribed to Medicare patients, including: brand name, generic name, total claims, total 30-day fills, total drug cost, and total beneficiaries. Data shall be sourced from the Medicare Part D Prescribers by Provider and Drug dataset (most recent available year).

**FR-008** — Each physician profile shall display an Industry Payments section sourced from Open Payments General Payment Data, showing: paying company name, nature of payment, total amount, date, and associated drug or device name. The system shall display individual transactions and a rolled-up annual total by company and by nature of payment.

**FR-009** — Each physician profile shall display an Industry Payments summary showing lifetime total payments received (all years 2013–present), broken down by General Payments, Research Payments, and Ownership Interests.

**FR-010** — The system shall display a data freshness indicator on each physician profile showing the source dataset and last update date for each data section (NPPES, Provider Data Catalog, Part B, Part D, Open Payments).

**FR-011** — The system shall support a "flag" or "bookmark" function allowing users to mark physicians for follow-up or custom list building.

---

### 3.2 Office Locations & Multi-Site Practices

**FR-012** — The system shall display all practice locations associated with a physician NPI, including cases where the National Downloadable File contains multiple rows for the same NPI (indicating multiple group/address combinations).

**FR-013** — Each practice location shall be displayed on an embedded map with a pin, showing the address, phone number, and driving distance from the user's current location.

**FR-014** — The system shall display the physician's primary practice group and allow navigation to a Group Practice profile showing all other physicians in the same group (matched by org PAC ID from the National Downloadable File).

---

### 3.3 Search & Filter

**FR-015** — The system shall provide a physician search function supporting the following filter criteria, usable individually or in combination:
- **Geography:** State, city, ZIP code, or radius from a given point (1–100 miles)
- **Specialty:** Single or multi-select from a normalized specialty list (decoded from NPPES taxonomy and/or CMS specialty names)
- **Credential:** MD, DO, NP, PA, etc.
- **Gender**
- **Medicare assignment:** Accepts assignment Y/N
- **Telehealth:** Accepts telehealth Y/N
- **Group practice:** By group name or org PAC ID
- **Hospital affiliation:** By hospital name or CCN
- **Part B procedure volume:** Filter by HCPCS code + minimum total services threshold (e.g., "surgeons billing CPT 27447 [total knee replacement] ≥ 50 times")
- **Part D drug prescribing:** Filter by drug name (brand or generic) + minimum claim count threshold (e.g., "prescribers of Eliquis with ≥ 25 claims")
- **Open Payments total:** Filter by minimum total payments received (e.g., "physicians receiving > $10,000/year from any manufacturer")
- **Open Payments by company:** Filter by specific paying manufacturer name
- **Open Payments by nature:** Filter by nature of payment (consulting fee, speaker fee, etc.)

**FR-016** — Search results shall be sortable by: name, specialty, distance from current location, Part B total billing volume, Part D total claim volume, and total Open Payments received.

**FR-017** — The system shall support saving named search filters as "segments" for reuse (e.g., "High-volume cardiologists in my territory").

**FR-018** — The system shall support exporting search results to CSV, including all profile fields visible in the result list.

---

### 3.4 Location-Aware Features

**FR-019** — The system shall request and use device GPS location to display a map view of physicians within a configurable radius (default 10 miles, adjustable to 1–100 miles).

**FR-020** — The map view shall display physician pins color-coded by specialty or by CRM status (e.g., green = called this week, yellow = due for follow-up, red = never contacted, grey = not a target).

**FR-021** — Tapping a map pin shall open a brief physician summary card with name, specialty, distance, and CRM last-contact date, with a button to open the full profile.

> **⚠️ Version 1.1 Update — Route Planning Approach Revised**
> FR-022 and FR-023 from v1.0 (native routing API engine) have been **deferred** to a future release pending scale validation. FR-024 and FR-025 have been **replaced** by FR-NEW-001 through FR-NEW-005 below. See RISK-008 (retired) and RISK-NEW-001 in Section 7.

**~~FR-022~~** *(DEFERRED — Native Route Calculation Engine)* — ~~The system shall provide a Route Planner feature. The user selects 2–15 physician locations and the system generates an optimized driving route minimizing total drive time, using a mapping/routing API (e.g., Google Maps Directions API or Mapbox).~~ Deferred pending scale. Retained in backlog for re-evaluation when platform reaches 500+ active reps.

**~~FR-023~~** *(DEFERRED — Routing API Integration)* — ~~The route planner shall allow the user to set a starting point (current location or custom address) and an optional ending point (home, office, or custom).~~ Deferred with FR-022.

**FR-NEW-001** — The system shall provide a Route Export feature. The user selects 2–15 physician stops, sets a starting location (current GPS location or custom address), and optionally specifies an available time window. The system generates a pre-filled AI planning prompt containing all stop details and exports it to the clipboard via a "Copy AI Prompt" button.

**FR-NEW-002** — The AI prompt generated by FR-NEW-001 shall include for each stop: physician full name, primary specialty, practice address, CRM priority level, last contact date, and any open follow-up task notes. The prompt shall include a natural-language instruction asking the AI to suggest an optimized driving order and estimated arrival times, and to flag any stops that are significantly out of the way.

**FR-NEW-003** — The system shall generate a Google Maps multi-stop deep-link URL (`https://www.google.com/maps/dir/[stop1]/[stop2]/...`) for the selected stops in the rep's current manual order. Tapping "Open in Google Maps" shall launch the URL in the Google Maps app or browser.

**FR-NEW-004** — The system shall generate an Apple Maps deep-link URL for the same stops. Tapping "Open in Apple Maps" shall launch the URL in the Apple Maps app (iOS only; this button shall be hidden on Android).

**FR-NEW-005** — Prior to export, the system shall display the selected stops in a drag-reorderable list, allowing the rep to manually set stop sequence before generating the AI prompt or map URLs. The system shall display each stop's CRM priority and driving distance from the previous stop to assist manual ordering.

---

### 3.5 CRM — Call & Activity Tracking

**FR-026** — The system shall allow sales reps to log a call/visit against any physician record, capturing: date/time, call type (in-person visit, phone call, email, virtual meeting), outcome (reached, not reached, left message, appointment set), duration, and free-text notes.

**FR-027** — The system shall display a call history log on each physician profile showing all logged interactions by date, with user attribution for team accounts.

**FR-028** — The system shall support attaching a product to a call log entry (from a configurable product catalog maintained by the organization).

**FR-029** — The system shall support logging samples distributed during a call, including product name, quantity, lot number, and expiration date.

**FR-030** — The system shall enforce sample accountability: total samples distributed per product shall be trackable per rep, per period, against a configurable sample budget/allotment.

**FR-031** — The system shall support creating follow-up tasks linked to a physician record, including: due date, task type (call, visit, send materials, etc.), priority, and notes. Tasks shall appear in a unified task list sortable by due date and priority.

**FR-032** — The system shall send push notifications for tasks that are due today or overdue.

**FR-033** — The system shall maintain a communication log supporting: email (manual log entry or via email integration), phone calls (manual log or via call integration), and in-person visit notes.

---

### 3.6 CRM — Pipeline & Opportunity Tracking

**FR-034** — The system shall support creating an Opportunity record linked to a physician, capturing: product of interest, estimated order value, stage (prospect, engaged, trial, committed, closed-won, closed-lost), expected close date, and notes.

**FR-035** — Opportunity stages shall be configurable by organization administrators.

**FR-036** — The system shall display a pipeline view (kanban or list) showing all open opportunities by stage, filterable by rep, territory, and product.

**FR-037** — The system shall support logging an Order against a physician record, capturing: product, quantity, order date, order value, and order number/reference.

**FR-038** — The system shall display a rep dashboard showing: calls logged this week/month, physicians contacted, samples distributed, open opportunities by stage, and orders closed.

**FR-039** — The system shall display a manager dashboard showing the same metrics rolled up across all reps in the manager's team, with drill-down by individual rep.

---

### 3.7 Territory Management

**FR-040** — The system shall support defining sales territories by geographic boundaries (state, county, ZIP code list, or drawn polygon on a map).

**FR-041** — Each rep account shall be assigned to one or more territories. Physician search and map views shall default to the rep's assigned territory.

**FR-042** — The system shall support a target list function: administrators or managers can assign specific physician NPIs to a rep's target list, overriding or supplementing geographic territory assignment.

**FR-043** — The system shall provide a territory coverage report showing: total physicians in territory, physicians contacted at least once, physicians contacted in the last 30/60/90 days, and physicians never contacted.

---

### 3.8 Data Refresh & Administration

**FR-044** — The system shall support automated ingestion of updated CMS data files on the following schedule:
- NPPES NPI Bulk Download: monthly (full file) + weekly (incremental)
- CMS Provider Data Catalog: quarterly
- CMS Facility Affiliation Data: quarterly
- Medicare Part B Utilization: annually
- Medicare Part D Prescribers: annually
- Open Payments: annually

**FR-045** — The system shall maintain a data lineage log showing when each source dataset was last successfully ingested and processed.

**FR-046** — Organization administrators shall be able to manage user accounts, assign roles (rep, manager, admin), and configure territories.

**FR-047** — The system shall support a configurable product catalog: product name, description, therapeutic area, and associated HCPCS/NDC codes (to link to Part B/Part D data).

**FR-048** — The system shall support data export of CRM activity logs (calls, tasks, opportunities, orders) in CSV format for integration with external systems (Salesforce, Veeva, etc.).

---

## 4. Non-Functional Requirements

### 4.1 Performance

**NFR-001** — Physician profile pages shall load within 2 seconds on a 4G mobile connection under normal load conditions.

**NFR-002** — Search results for geography-based queries (radius search) shall return within 3 seconds for radii up to 50 miles.

**NFR-003** — Map view with physician pins shall render within 2 seconds for up to 500 visible pins in the viewport.

**NFR-004** — *(Updated v1.1)* The route export feature (AI prompt generation + Google Maps and Apple Maps URL generation) for up to 15 stops shall complete within 1 second of the user tapping export. No external API call is required for generation.

**NFR-005** — The system shall support at least 10,000 concurrent active users without degradation in response times.

**NFR-006** — Batch data ingestion jobs (monthly NPPES full file ~8GB uncompressed) shall complete within 4 hours of file availability.

### 4.2 Scalability

**NFR-007** — The physician database shall support storage and querying of at least 10 million physician records (covering current active NPI count plus historical records).

**NFR-008** — The system shall support horizontal scaling of the API layer to handle traffic spikes (e.g., Monday morning when all reps start their week simultaneously).

**NFR-009** — The CRM data model shall support at least 100 million call log entries without performance degradation in query response times.

### 4.3 Availability & Reliability

**NFR-010** — The system shall maintain 99.9% uptime (≤ 8.7 hours downtime/year) for the mobile API and web application, excluding scheduled maintenance windows.

**NFR-011** — Scheduled maintenance windows shall not occur during business hours (8am–6pm local time, Monday–Friday, across all US time zones).

**NFR-012** — The system shall implement automated backups of all CRM data with a Recovery Point Objective (RPO) of 1 hour and a Recovery Time Objective (RTO) of 4 hours.

### 4.4 Security

**NFR-013** — All data in transit shall be encrypted using TLS 1.2 or higher.

**NFR-014** — All data at rest shall be encrypted using AES-256.

**NFR-015** — User authentication shall support username/password with multi-factor authentication (MFA). SSO via SAML 2.0 or OAuth 2.0 shall be supported for enterprise customers.

**NFR-016** — The system shall implement role-based access control (RBAC) with at minimum three roles: Rep (own data only), Manager (team data), Admin (full organization).

**NFR-017** — All API endpoints shall require authenticated sessions. API keys shall be supported for system integrations.

**NFR-018** — The system shall log all administrative actions and data export events for audit purposes. Audit logs shall be retained for a minimum of 2 years.

**NFR-019** — The system shall support organization-level data isolation — one organization's CRM data shall never be accessible to another organization's users.

### 4.5 Compliance

**NFR-020** — **HIPAA:** The platform does not store or process Protected Health Information (PHI). The physician data sourced from CMS datasets is provider data, not patient data, and is publicly available. CRM data (call logs, notes, orders) is sales activity data about physicians as business contacts, not patients. The system does not require a HIPAA Business Associate Agreement for its core functionality. **However**, if the system stores any information that could be considered PHI in the future (e.g., patient-level outcomes data), a full HIPAA compliance review must be conducted before that feature is deployed.

**NFR-021** — **Data Use:** All CMS data used (NPPES, Provider Data Catalog, Part B, Part D, Open Payments) is publicly available and free to use without a data use agreement. The system shall not sublicense or resell the raw CMS data.

**NFR-022** — **AMA Opt-Out (PDRP):** The Medicare Part D prescribing data is published by CMS and is public. The AMA Physician Data Restriction Program (PDRP) applies to commercial prescription data vendors (IQVIA, Symphony), not to CMS's own public publications. No PDRP compliance obligation applies to the Part D public data used here.

**NFR-023** — **CCPA / State Privacy:** The platform handles publicly available professional data about physicians in their professional capacity. This is generally not considered personal data subject to CCPA for commercial purposes. Legal review is recommended before launch in California.

**NFR-024** — **SOC 2 Type II:** The platform shall achieve SOC 2 Type II certification within 18 months of launch, covering Security, Availability, and Confidentiality trust service categories.

### 4.6 Mobile Requirements

**NFR-025** — The mobile application shall be available natively on iOS (minimum iOS 16) and Android (minimum Android 12).

**NFR-026** — Core features shall function in offline mode (no network connection), including: viewing recently accessed physician profiles, logging call notes, and viewing the day's planned route. Data shall sync when connectivity is restored.

**NFR-027** — The mobile app shall support GPS location access for nearby physician search and route planning.

**NFR-028** — The mobile app shall support push notifications for task reminders and manager alerts.

**NFR-029** — The mobile app shall pass Apple App Store and Google Play Store review requirements without policy exceptions.

### 4.7 Data Quality

**NFR-030** — The system shall deduplicate physician records across source datasets using NPI as the canonical key. Conflicting field values across sources shall be resolved using a defined precedence order (documented in data architecture).

**NFR-031** — The system shall validate NPI format (10 digits, passing Luhn checksum) on all imported records. Invalid NPIs shall be flagged and quarantined, not imported.

**NFR-032** — The system shall display data vintage prominently — specifically, Part B and Part D data is 18–24 months old at time of publication. Users must be informed of the data year displayed.

---

## 5. Data Architecture Overview

### 5.1 Source Datasets & Ingestion

| Dataset | Source | Format | Frequency | Size (approx) | Join Key |
|---|---|---|---|---|---|
| NPPES NPI Registry | download.cms.gov/nppes | CSV | Monthly (full) + Weekly (delta) | 8–10 GB | NPI |
| Provider Data Catalog — National Downloadable File | data.cms.gov/provider-data | CSV | Quarterly | ~500 MB |  NPI, org_pac_id |
| Facility Affiliation Data | data.cms.gov/provider-data | CSV | Quarterly | ~200 MB | NPI, CCN |
| Hospital Compare — General Information | data.cms.gov/provider-data | CSV | Quarterly | ~5 MB | CCN |
| Medicare Part B — By Provider & Service | data.cms.gov | CSV | Annual | ~2–3 GB | NPI |
| Medicare Part D — By Provider | data.cms.gov | CSV | Annual | ~500 MB | NPI |
| Medicare Part D — By Provider & Drug | data.cms.gov | CSV | Annual | ~3 GB | NPI |
| Open Payments — General | download.cms.gov/openpayments | CSV | Annual | ~2 GB | NPI |
| Open Payments — Research | download.cms.gov/openpayments | CSV | Annual | ~500 MB | NPI (as PI) |
| Open Payments — Ownership | download.cms.gov/openpayments | CSV | Annual | ~50 MB | NPI |

### 5.2 Core Data Model

```
Physician (master record)
├── npi                          VARCHAR(10) PK
├── last_name                    VARCHAR
├── first_name                   VARCHAR
├── middle_name                  VARCHAR
├── credential                   VARCHAR
├── gender                       CHAR(1)
├── status                       CHAR(1)          -- NPPES: A/D
├── enumeration_date             DATE
├── last_updated_nppes           DATE
├── source_nppes_updated_at      TIMESTAMP        -- when we last ingested

PhysicianAddress (1:many → Physician)
├── id                           UUID PK
├── npi                          FK → Physician
├── address_purpose              VARCHAR          -- LOCATION / MAILING
├── address_line_1               VARCHAR
├── address_line_2               VARCHAR
├── city                         VARCHAR
├── state                        CHAR(2)
├── zip5                         VARCHAR(5)
├── phone                        VARCHAR
├── fax                          VARCHAR
├── geo_lat                      DECIMAL          -- geocoded
├── geo_lng                      DECIMAL          -- geocoded

PhysicianTaxonomy (1:many → Physician)
├── id                           UUID PK
├── npi                          FK → Physician
├── taxonomy_code                VARCHAR          -- NUCC code
├── taxonomy_desc                VARCHAR          -- decoded plain English
├── is_primary                   BOOLEAN
├── license_number               VARCHAR
├── license_state                CHAR(2)

PhysicianCMSProfile (1:1 → Physician)
├── npi                          FK → Physician PK
├── ind_pac_id                   VARCHAR          -- Medicare PAC ID
├── ind_enrl_id                  VARCHAR
├── group_name                   VARCHAR
├── org_pac_id                   VARCHAR
├── num_org_members              INT
├── medical_school               VARCHAR
├── grad_year                    INT
├── primary_specialty_cms        VARCHAR
├── accepts_telehealth           BOOLEAN
├── ind_assignment               CHAR(1)
├── grp_assignment               CHAR(1)
├── source_updated_at            TIMESTAMP

HospitalAffiliation (1:many → Physician)
├── id                           UUID PK
├── npi                          FK → Physician
├── ccn                          VARCHAR(6)       -- FK → Hospital
├── facility_type                VARCHAR
├── source_updated_at            TIMESTAMP

Hospital (master record)
├── ccn                          VARCHAR(6) PK
├── name                         VARCHAR
├── address                      VARCHAR
├── city                         VARCHAR
├── state                        CHAR(2)
├── zip                          VARCHAR
├── phone                        VARCHAR
├── hospital_type                VARCHAR
├── ownership_type               VARCHAR
├── emergency_services           BOOLEAN
├── overall_star_rating          INT
├── geo_lat                      DECIMAL
├── geo_lng                      DECIMAL
├── source_updated_at            TIMESTAMP

PartBService (1:many → Physician)
├── id                           UUID PK
├── npi                          FK → Physician
├── data_year                    INT
├── hcpcs_code                   VARCHAR
├── hcpcs_desc                   VARCHAR
├── total_beneficiaries          INT
├── total_services               INT
├── total_submitted_charges      DECIMAL
├── total_medicare_payment       DECIMAL
├── avg_medicare_payment         DECIMAL

PartDDrug (1:many → Physician)
├── id                           UUID PK
├── npi                          FK → Physician
├── data_year                    INT
├── brand_name                   VARCHAR
├── generic_name                 VARCHAR
├── total_claims                 INT
├── total_30day_fills            DECIMAL
├── total_day_supply             INT
├── total_drug_cost              DECIMAL
├── total_beneficiaries          INT
├── opioid_flag                  BOOLEAN

OpenPaymentsGeneral (1:many → Physician)
├── record_id                    VARCHAR PK
├── npi                          FK → Physician
├── program_year                 INT
├── manufacturer_name            VARCHAR
├── manufacturer_id              VARCHAR
├── payment_amount               DECIMAL
├── payment_date                 DATE
├── nature_of_payment            VARCHAR
├── form_of_payment              VARCHAR
├── product_name_1               VARCHAR
├── product_type_1               VARCHAR
├── product_ndc_1                VARCHAR
├── change_type                  VARCHAR

-- CRM Tables (per-organization, isolated)

CRMCall
├── id                           UUID PK
├── org_id                       UUID FK
├── user_id                      UUID FK → CRMUser
├── npi                          FK → Physician
├── call_date                    TIMESTAMP
├── call_type                    VARCHAR          -- in-person, phone, email, virtual
├── outcome                      VARCHAR          -- reached, not-reached, appt-set, etc.
├── duration_minutes             INT
├── notes                        TEXT
├── product_id                   UUID FK → Product

CRMTask
├── id                           UUID PK
├── org_id                       UUID FK
├── user_id                      UUID FK
├── npi                          FK → Physician
├── task_type                    VARCHAR
├── due_date                     DATE
├── priority                     VARCHAR
├── completed                    BOOLEAN
├── notes                        TEXT

CRMOpportunity
├── id                           UUID PK
├── org_id                       UUID FK
├── user_id                      UUID FK
├── npi                          FK → Physician
├── product_id                   UUID FK → Product
├── stage                        VARCHAR
├── estimated_value              DECIMAL
├── expected_close_date          DATE
├── notes                        TEXT

CRMOrder
├── id                           UUID PK
├── org_id                       UUID FK
├── user_id                      UUID FK
├── npi                          FK → Physician
├── product_id                   UUID FK → Product
├── quantity                     INT
├── order_value                  DECIMAL
├── order_date                   DATE
├── order_reference              VARCHAR

CRMSample
├── id                           UUID PK
├── org_id                       UUID FK
├── call_id                      UUID FK → CRMCall
├── npi                          FK → Physician
├── product_id                   UUID FK → Product
├── quantity                     INT
├── lot_number                   VARCHAR
├── expiration_date              DATE
```

### 5.3 Geocoding

All physician practice addresses and hospital addresses must be geocoded (lat/lng) to support radius search and map display. Use a geocoding API (Google Maps Geocoding API or Mapbox Geocoding API). Geocode on initial import; re-geocode on address change. Budget for ~5 million geocoding calls on initial load, then incremental on updates.

### 5.4 Search Infrastructure

Geography-based radius search requires geospatial indexing. Options:
- **PostgreSQL + PostGIS** — `ST_DWithin()` with a geography index. Handles millions of points with sub-second radius queries. Recommended for v1.
- **Elasticsearch with geo_point** — Better for full-text + geo hybrid queries. Consider for v2 if multi-attribute search performance becomes a bottleneck.

Full-text search on physician name and organization name requires a text search index. PostgreSQL `pg_trgm` or Elasticsearch.

---

## 6. Feature Prioritization

### MVP (Phase 1) — Core Intelligence + Basic CRM

| Feature | Priority | Notes |
|---|---|---|
| Physician profile (NPPES + Provider Data Catalog) | P0 | Foundation — nothing works without this |
| Hospital affiliations display | P0 | |
| Specialty + location search & filter | P0 | |
| Nearby physicians map view (GPS) | P0 | Core differentiator |
| Part B billing summary per physician | P1 | Annual data only |
| Part D prescribing summary per physician | P1 | Annual data only |
| Open Payments summary per physician | P1 | |
| Call logging | P0 | |
| Task / follow-up tracking | P0 | |
| Push notifications for tasks | P1 | |
| Route export — AI prompt + Google/Apple Maps URLs (FR-NEW-001 to FR-NEW-005) | P1 | Replaces native routing for v1 |
| Native route optimization engine (FR-022, FR-023) | DEFERRED | Re-evaluate at 500+ reps |
| Rep dashboard | P1 | |
| Basic territory assignment | P1 | |

### Phase 2 — Enhanced CRM + Analytics

| Feature | Priority |
|---|---|
| Opportunity / pipeline tracking | P2 |
| Sample accountability tracking | P2 |
| Order logging | P2 |
| Manager dashboard + team roll-up | P2 |
| Saved search segments | P2 |
| CSV export (search results + CRM data) | P2 |
| Offline mode (mobile) | P2 |
| Advanced Part B/D filtering (by specific procedure/drug) | P2 |
| Open Payments filter by company / nature | P2 |
| Group practice profiles | P2 |

### Phase 3 — Enterprise + Integrations

| Feature |
|---|
| SSO / SAML 2.0 |
| Salesforce / Veeva CRM export integration |
| Email integration (log emails automatically) |
| Web companion app (manager-focused) |
| SOC 2 certification |
| Multi-product catalog support |
| Territory polygon drawing on map |
| Custom physician scoring / weighting engine |
| DEA number lookup (via paid service like DEA Lookup.com) |

---

## 7. Key Technical Risks & Considerations

**RISK-001 — Data Staleness**
Part B and Part D data is published 18–24 months after the reporting year. A rep looking at Part D data today is seeing 2023 prescribing patterns. This is a known limitation of the free data. Mitigation: display the data year prominently everywhere it appears. Consider supplementing with IQVIA or Symphony data if budget allows (Phase 3 option).

**RISK-002 — NPPES Address Quality**
NPPES addresses are self-reported by providers and are not always current. A physician who moved practices 2 years ago may still show the old address. Mitigation: cross-reference NPPES address with Provider Data Catalog address and surface discrepancies. Flag records where NPPES and Provider Data Catalog addresses differ.

**RISK-003 — Geocoding Cost and Accuracy**
Initial geocoding of ~5 million addresses will cost approximately $25,000 at standard Google Maps API rates ($5/1,000 calls). Ongoing monthly geocoding of ~200,000 updated records costs ~$1,000/month. Mitigation: negotiate volume pricing with Google or evaluate Mapbox (lower per-call pricing). Cache aggressively — only re-geocode when address changes.

**RISK-004 — DEA Number Gap**
DEA numbers are not available in any free public dataset. NPPES does not include DEA numbers. For pharmaceutical reps whose workflow requires DEA verification, this is a gap. Mitigation: note the gap in v1, add DEA Lookup.com API integration in Phase 3. Estimated cost ~$200–500/month.

**RISK-005 — Specialty Normalization**
Three datasets (NPPES, Provider Data Catalog, Open Payments) use three different specialty naming conventions. NPPES uses NUCC taxonomy codes, Provider Data Catalog uses CMS plain-English specialty names, Open Payments uses a pipe-delimited "Category|Specialty" format. A specialty normalization mapping table is required to enable consistent filtering across datasets.

**RISK-006 — NPI Matching Across Datasets**
Most records join cleanly on NPI. However, some Open Payments records have blank or incorrect NPIs (name-matching was used historically). Approximately 3–5% of payment records may not join cleanly to an NPI. Mitigation: implement a name+state fuzzy matching fallback for unmatched Open Payments records, with a low-confidence flag.

**RISK-007 — CMS Dataset Format Changes**
CMS has broken downstream integrations before (Socrata → ODA migration in 2021, NPPES V1 → V2 in 2026). The ingestion pipeline must be monitored and must include automated schema validation that alerts on unexpected field changes before data is committed to production.

**~~RISK-008~~** *(RETIRED — v1.1)* — ~~Route Planning API Costs. Google Maps Directions API charges per request. Optimizing a 10-stop route requires multiple API calls. At scale (10,000 reps, 5 routes/week each), this could reach $50,000+/month at standard rates.~~ **Risk eliminated.** The v1.1 route export approach (FR-NEW-001 through FR-NEW-005) requires no routing API. AI prompt generation and map deep-link URL construction are performed entirely client-side at zero marginal cost. Risk is re-opened if/when the native routing engine (FR-022, FR-023) is undeferred.

**RISK-NEW-001 — AI Hallucination & Liability in Route Recommendations**
The AI prompt export feature (FR-NEW-001) delegates route optimization logic to a third-party AI tool (Claude, ChatGPT, etc.) that the rep uses independently. If an AI assistant returns an incorrect route, bad address, or poor sequencing advice, the rep may waste significant time or miss calls. The app has no control over the quality of the AI's output. Mitigation: (1) the AI prompt template shall include a disclaimer: *"Verify all addresses before navigating. This data is sourced from federal records and may not reflect recent changes."* (2) App UI shall display a note that the AI prompt is provided as a planning aid and that the app does not endorse or guarantee AI-generated routing advice. (3) Legal should review whether the prompt export workflow creates any implied warranty of routing accuracy.

**RISK-NEW-002 — Data Privacy in AI Prompt Export**
When a rep pastes the generated AI prompt into a consumer AI tool (ChatGPT, Claude.ai, etc.), physician names, addresses, specialties, and CRM activity data leave the organization's controlled environment. This data is publicly available (CMS sources), but CRM notes added by the rep may contain sensitive business information. Mitigation: (1) add a visible warning in the Route Export UI: *"The copied prompt contains physician data and your CRM notes. Do not paste into AI tools on networks subject to your organization's data handling policies."* (2) Document this data flow in the Terms of Service. (3) Organizations with strict data policies may wish to disable the AI prompt export feature via an admin setting — this should be a configurable toggle.

---

## 8. Open Questions for Sean

1. **Commercial data supplement:** The free CMS data is 18–24 months stale for Part B/D. Is there budget to add a commercial data layer (IQVIA, Symphony, or Definitive Healthcare) for more current prescribing data? This significantly changes the cost structure.

2. **Target market:** Is this being built as an internal tool for one sales organization, or as a SaaS product sold to multiple pharma/device companies? This changes the multi-tenancy requirements, pricing model, and compliance obligations substantially.

3. **Mobile platform priority:** iOS first, or iOS and Android simultaneously? Native apps or React Native / Flutter cross-platform?

4. **Email integration:** Do reps want emails they send to physicians auto-logged in the CRM? This requires integration with Gmail/Outlook and adds significant scope.

5. ~~**Mapping provider:** Google Maps vs. Mapbox vs. Apple Maps?~~ *(Resolved in v1.1 — no routing API required for v1. Geocoding still uses Google Maps or Mapbox; see RISK-003.)*

6. **Existing CRM:** Is there an existing CRM (Salesforce, Veeva, HubSpot) that this needs to integrate with, or is this replacing one?

7. **Sample management compliance:** Some regulated sample distribution requires FDA-compliant e-signature and chain-of-custody tracking. Is pharmaceutical sample management in scope, or just internal tracking?

8. **International scope:** US only for now, or will international markets (Canada, EU) be required? This changes data sourcing completely.

9. *(New — v1.1)* **AI prompt export admin toggle:** Should organizations be able to disable the AI prompt export feature for reps on networks with strict data handling policies? If yes, this requires an admin configuration setting.

---

*Prepared by Dennis Reynolds, Project Manager*  
*This is the plan. It's a good plan. Follow it.*
