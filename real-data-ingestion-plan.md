# Real Data Ingestion Plan — Sprint 3
**Project:** Medical Sales Rep Platform  
**Author:** Dennis Reynolds, PM  
**Date:** 2026-03-16  
**Status:** DRAFT — Pre-Sprint 3  
**Version:** 1.0

---

## Executive Summary

The current database contains 4,500 synthetic physician records with sequential NPIs, recycled names, null taxonomy codes, null gender, null enumeration dates, and zero enrichment data. Every enrichment table is empty. This is not acceptable. This document defines the complete plan to replace all synthetic data with real, publicly available CMS datasets — NPPES, Open Payments, Medicare Part B, and Medicare Part D — and to bring the full schema to a production-ready state before Sprint 3 closes.

This plan is written for Charlie (implementation) and Frank (infrastructure/DevOps) to execute from directly. No guesswork required.

---

## Current State Assessment

| Component | Status |
|---|---|
| `physicians` table | 4,500 rows — synthetic data, sequential NPIs, null taxonomy/gender/dates |
| `open_payments` | Empty |
| `part_b_services` | Empty |
| `part_d_drugs` | Empty |
| `hospital_affiliations` | Empty |
| `physician_specialties` | Empty |
| `physician_addresses` | Empty |
| `physician_cms_profiles` | Empty |
| PostGIS `geom` column | City-centroid only — not real address geocoding |
| Spring Batch job (S2-10) | Exists but generates mock data instead of importing NPPES |
| AWS Location Service (S2-01) | Integrated but not used at scale |
| Flyway version | V7 (current) |

**Bottom line:** The data layer is a skeleton. Sprint 3 puts the meat on it.

---

## 1. Data Sources

All sources are free, public domain (no licensing restrictions), and updated regularly by CMS.

### 1.1 NPPES — National Plan and Provider Enumeration System

| Attribute | Detail |
|---|---|
| URL | `https://download.cms.gov/nppes/NPI_Files.html` |
| Format | ZIP → CSV (~8GB uncompressed) |
| Update frequency | Monthly full replacement file |
| Target filter | `Entity_Type_Code = 1` (Individual), `Provider_Business_Mailing_Address_State_Name = OK` |
| Estimated Oklahoma physicians | ~25,000 active records |

**Fields we need from NPPES:**

| NPPES Field | Target Column |
|---|---|
| `NPI` | `physicians.npi` |
| `Provider_First_Name` | `physicians.first_name` |
| `Provider_Last_Name_(Legal_Name)` | `physicians.last_name` |
| `Provider_Credential_Text` | `physicians.credential` |
| `Provider_Gender_Code` | `physicians.gender` |
| `Provider_Enumeration_Date` | `physicians.enumeration_date` |
| `Healthcare_Provider_Taxonomy_Code_1` | `physician_specialties` + `physicians.taxonomy_code` |
| `Provider_First_Line_Business_Practice_Location_Address` | `physician_addresses.address_line1` |
| `Provider_Second_Line_Business_Practice_Location_Address` | `physician_addresses.address_line2` |
| `Provider_Business_Practice_Location_Address_City_Name` | `physician_addresses.city` |
| `Provider_Business_Practice_Location_Address_State_Name` | `physician_addresses.state` |
| `Provider_Business_Practice_Location_Address_Postal_Code` | `physician_addresses.zip` |
| `Provider_Business_Practice_Location_Address_Telephone_Number` | `physician_addresses.phone` |
| `NPI_Deactivation_Date` | Filter: exclude deactivated NPIs |
| `NPI_Reactivation_Date` | Used to determine active status |

**Notes:**
- NPPES taxonomy codes map to specialties via the NUCC Health Care Provider Taxonomy code set (also free, available at `nucc.org`)
- Up to 15 taxonomy codes per provider in NPPES; use `Healthcare_Provider_Taxonomy_Code_1` as primary
- Filter out `NPI_Deactivation_Date` is not null (unless `NPI_Reactivation_Date` is also set and later)

---

### 1.2 CMS Open Payments

| Attribute | Detail |
|---|---|
| URL | `https://openpaymentsdata.cms.gov` |
| Format | CSV download by year |
| Update frequency | Annual (published ~June for prior year) |
| Join key | `Covered_Recipient_NPI` → `physicians.npi` |
| Latest available | 2023 dataset |

**Fields we need:**

| Open Payments Field | Target Column |
|---|---|
| `Covered_Recipient_NPI` | Join key |
| `Applicable_Manufacturer_or_Applicable_GPO_Making_Payment_Name` | `open_payments.company_name` |
| `Total_Amount_of_Payment_USDollars` | `open_payments.payment_amount` |
| `Date_of_Payment` | `open_payments.payment_date` |
| `Nature_of_Payment_or_Transfer_of_Value` | `open_payments.payment_nature` |
| `Covered_Recipient_Type` | Filter: `Covered Recipient Physician` only |

**Note:** Open Payments covers "General Payments," "Research Payments," and "Ownership/Investment." Start with General Payments (largest dataset). Research and Ownership can be added later.

---

### 1.3 Medicare Part B — Provider Utilization

| Attribute | Detail |
|---|---|
| URL | `https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service` |
| Format | CSV download |
| Update frequency | Annual |
| Join key | `Rndrng_NPI` → `physicians.npi` |

**Fields we need:**

| Part B Field | Target Column |
|---|---|
| `Rndrng_NPI` | Join key |
| `HCPCS_Cd` | `part_b_services.hcpcs_code` |
| `HCPCS_Desc` | `part_b_services.hcpcs_description` |
| `Tot_Benes` | `part_b_services.total_beneficiaries` |
| `Tot_Srvcs` | `part_b_services.total_services` |
| `Tot_Mdcr_Pymt_Amt` | `part_b_services.total_medicare_payment` |
| `Avg_Mdcr_Pymt_Amt` | `part_b_services.avg_medicare_payment` |

---

### 1.4 Medicare Part D — Drug Prescribing

| Attribute | Detail |
|---|---|
| URL | `https://data.cms.gov/provider-summary-by-type-of-service/medicare-part-d-prescribers/medicare-part-d-prescribers-by-provider-and-drug` |
| Format | CSV download |
| Update frequency | Annual |
| Join key | `Prscrbr_NPI` → `physicians.npi` |

**Fields we need:**

| Part D Field | Target Column |
|---|---|
| `Prscrbr_NPI` | Join key |
| `Brnd_Name` | `part_d_drugs.brand_name` |
| `Gnrc_Name` | `part_d_drugs.generic_name` |
| `Tot_Clms` | `part_d_drugs.total_claims` |
| `Tot_30day_Fills` | `part_d_drugs.total_30day_fills` |
| `Tot_Day_Suply` | `part_d_drugs.total_day_supply` |
| `Tot_Drug_Cst` | `part_d_drugs.total_drug_cost` |
| `Tot_Benes` | `part_d_drugs.total_beneficiaries` |

---

## 2. AWS Resources & Cost Estimates

### 2.1 S3 — Raw Data Staging

**Purpose:** Store downloaded CMS files before processing. Do not process directly from download URLs in production.

| Bucket | Contents | Estimated Size |
|---|---|---|
| `medical-sales-raw-data/nppes/` | NPPES monthly ZIPs | ~8 GB/month |
| `medical-sales-raw-data/open-payments/` | Open Payments annual CSVs | ~3 GB/year |
| `medical-sales-raw-data/part-b/` | Part B annual CSVs | ~2 GB/year |
| `medical-sales-raw-data/part-d/` | Part D annual CSVs | ~1.5 GB/year |

**S3 Cost Estimate:**

| Item | Calculation | Cost |
|---|---|---|
| Storage (20 GB total raw) | $0.023/GB/month × 20 GB | ~$0.46/month |
| PUT/COPY requests (ingestion) | ~1,000 requests × $0.005/1K | ~$0.005 |
| GET requests (processing) | ~5,000 requests × $0.0004/1K | ~$0.002 |
| Data transfer (download from CMS to S3) | Free (CMS data is public, S3 ingress free) | $0 |
| **Total S3 Monthly** | | **~$0.50/month** |

---

### 2.2 AWS Location Service — Geocoding

**Purpose:** Convert the ~25,000 Oklahoma physician practice addresses from NPPES into lat/lng coordinates for PostGIS storage.

| Parameter | Value |
|---|---|
| Total addresses to geocode (one-time) | ~25,000 |
| Monthly new/changed addresses (NPPES refresh delta) | ~500–1,000 estimated |
| Pricing | $0.50 per 1,000 requests |

**Cost Estimate:**

| Item | Calculation | Cost |
|---|---|---|
| One-time full geocode (25K addresses) | 25,000 / 1,000 × $0.50 | **$12.50** |
| Monthly delta geocode (~750 avg) | 750 / 1,000 × $0.50 | **$0.38/month** |
| **Total geocoding (Year 1)** | $12.50 + ($0.38 × 12) | **~$17.06** |

**Notes:**
- AWS Location Service free tier: 10,000 geocode requests/month for first 3 months → one-time batch is effectively **free** if executed in first 3 months
- Rate limit: 50 requests/second by default. At 25K addresses, this is ~8 minutes. Spring Batch should respect this with appropriate throttling (50 req/sec max)
- S2-01 integration already exists — Frank should confirm the Place Index is configured with `HERE` or `Esri` as data provider (both support US addresses)

---

### 2.3 Compute: Spring Batch vs Lambda vs ECS

**Recommendation: Keep Spring Batch for orchestration, use ECS Fargate for heavy lifting.**

| Option | Pros | Cons | Recommendation |
|---|---|---|---|
| Spring Batch (existing) | Already built, familiar | 8GB NPPES parse may hit memory limits on dev machine, needs right-sized container | ✅ Use for job orchestration logic |
| AWS Lambda | Cheap, serverless | 15-min timeout, 10GB memory max — won't work for 8GB file parse in one shot | ❌ Not for NPPES bulk parse |
| ECS Fargate | Right-sized, no timeout limits, controllable memory | Slightly more setup | ✅ Use for NPPES parse task |
| AWS Glue | Managed ETL, handles large CSV | $0.44/DPU-hour, overkill for structured CMS data | ⚠️ Optional — not needed if Fargate works |
| Step Functions | Good for orchestrating multi-step pipeline | Additional cost, complexity | ⚠️ Nice-to-have, not required for Sprint 3 |

**Sprint 3 Recommendation:**
- Run Spring Batch jobs inside **ECS Fargate task** with 4 vCPU / 16GB RAM
- NPPES download → S3 → Fargate parse → PostgreSQL
- No Step Functions needed for Sprint 3; add in Sprint 4 if needed

**ECS Fargate Cost (one-time ingestion, ~2 hours):**

| Item | Calculation | Cost |
|---|---|---|
| 4 vCPU × $0.04048/vCPU-hour × 2h | | $0.32 |
| 16 GB × $0.004445/GB-hour × 2h | | $0.14 |
| **One-time ingestion total** | | **~$0.46** |
| **Monthly refresh run (30 min)** | ~$0.04 | **~$0.04/month** |

---

### 2.4 RDS vs Docker PostgreSQL

**Recommendation: Stay on Docker PostgreSQL for Sprint 3. Evaluate RDS for production.**

| Option | Pros | Cons | Recommendation |
|---|---|---|---|
| Docker PostgreSQL (current) | Free, already running, PostGIS easy to add | Not managed, no automated backups, single point of failure | ✅ Sprint 3 is fine here |
| RDS PostgreSQL | Managed, automated backups, Multi-AZ available, PostGIS supported | $0.085–$0.17/hour for db.t3.medium | Consider for production launch |
| Aurora PostgreSQL | Better performance at scale | More expensive, overkill at 25K physicians | ❌ Not needed |

**If/when moving to RDS:**
- Use `db.t3.medium` (2 vCPU, 4GB RAM) — sufficient for 25K physician records
- Enable PostGIS extension post-creation: `CREATE EXTENSION postgis;`
- Cost: ~$62/month for db.t3.medium in us-east-1
- Enable automated daily snapshots (7-day retention, ~$0.095/GB-month)

---

### 2.5 Total AWS Cost Summary

| Category | One-Time | Monthly (ongoing) |
|---|---|---|
| S3 Storage & Requests | $0.01 | $0.50 |
| AWS Location Service (geocoding) | $12.50 (free in first 3 months) | $0.38 |
| ECS Fargate (ingestion compute) | $0.46 | $0.04 |
| RDS (if migrated from Docker) | $0 | $62.00 (optional) |
| Step Functions (optional, Sprint 4) | $0 | ~$1.00 (optional) |
| **TOTAL (Docker PostgreSQL)** | **~$13** | **~$0.92/month** |
| **TOTAL (with RDS migration)** | **~$13** | **~$62.92/month** |

> **Bottom line:** Real data ingestion costs approximately **$13 one-time** (mostly geocoding, free if done in free tier window) and under **$1/month** ongoing on the current Docker stack. Budget $63/month if we migrate to RDS. This is extremely cheap for the data value we're getting.

---

## 3. Schema Changes Required

Current Flyway version: V7. All changes below require V8+ migrations.

### 3.1 `physicians` Table — Required Column Additions

| Column | Current State | Required Change | Migration |
|---|---|---|---|
| `taxonomy_code` | EXISTS but NULL for all rows | Populate from NPPES `Healthcare_Provider_Taxonomy_Code_1` | V8 (populate, not add) |
| `gender` | EXISTS but NULL for all rows | Populate from NPPES `Provider_Gender_Code` (M/F) | V8 (populate, not add) |
| `enumeration_date` | EXISTS but NULL for all rows | Populate from NPPES `Provider_Enumeration_Date` | V8 (populate, not add) |
| `npi_type` | Check if exists | Add `SMALLINT` column: 1=Individual, 2=Organization | V8 |
| `deactivation_date` | Likely missing | Add `DATE NULL` — for tracking deactivated NPIs | V8 |
| `last_updated` | Check if exists | Add `TIMESTAMP` — track when NPPES record last changed | V8 |
| `npi_status` | Likely missing | Add `VARCHAR(20)`: 'A' (active), 'D' (deactivated) | V8 |

**Note:** The current 4,500 synthetic records must be **truncated** before real NPPES import. Do not attempt to update synthetic rows — the NPIs are fake and won't match anything. Charlie should add a Flyway migration that truncates `physicians` (and all enrichment tables via CASCADE) before populating with real data.

### 3.2 `physician_specialties` Table

Currently empty. Should be populated from NPPES taxonomy codes using the NUCC taxonomy reference.

**Required schema (verify against current DDL):**

```sql
CREATE TABLE physician_specialties (
    id BIGSERIAL PRIMARY KEY,
    npi VARCHAR(10) NOT NULL REFERENCES physicians(npi),
    taxonomy_code VARCHAR(20) NOT NULL,
    taxonomy_description VARCHAR(255),
    taxonomy_grouping VARCHAR(100),
    taxonomy_classification VARCHAR(100),
    taxonomy_specialization VARCHAR(100),
    is_primary BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_physician_specialties_npi ON physician_specialties(npi);
CREATE INDEX idx_physician_specialties_taxonomy ON physician_specialties(taxonomy_code);
```

**Population logic:**
1. Download NUCC taxonomy CSV from `nucc.org/index.php/code-sets-mainmenu-41/provider-taxonomy-mainmenu-40/csv-mainmenu-57`
2. Load taxonomy reference into a `taxonomy_reference` lookup table
3. For each NPPES row, insert up to 15 taxonomy codes per physician (fields `Healthcare_Provider_Taxonomy_Code_1` through `_15`)
4. Mark `is_primary = TRUE` for `_Code_1` where `Healthcare_Provider_Primary_Taxonomy_Switch_1 = 'Y'`

### 3.3 `physician_addresses` Table

Currently empty. Must be populated from NPPES and then geocoded via AWS Location Service.

**Required schema (verify against current DDL):**

```sql
CREATE TABLE physician_addresses (
    id BIGSERIAL PRIMARY KEY,
    npi VARCHAR(10) NOT NULL REFERENCES physicians(npi),
    address_type VARCHAR(20) NOT NULL, -- 'PRACTICE' or 'MAILING'
    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(2),
    zip VARCHAR(10),
    phone VARCHAR(20),
    geom GEOMETRY(Point, 4326),  -- populated by AWS Location Service geocoding
    geocoded_at TIMESTAMP,
    geocode_confidence VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_physician_addresses_npi ON physician_addresses(npi);
CREATE INDEX idx_physician_addresses_geom ON physician_addresses USING GIST(geom);
```

**Key point:** The PostGIS `geom` column on `physician_addresses` should store the **real geocoded coordinates** from AWS Location Service, not city centroids. The `physicians` table may retain a `geom` column for quick proximity queries, but it should be kept in sync with the primary address.

### 3.4 `open_payments` Table

**Verify these columns exist; add any missing:**

| Column | Type | Source Field | Notes |
|---|---|---|---|
| `npi` | VARCHAR(10) | `Covered_Recipient_NPI` | FK to physicians |
| `company_name` | VARCHAR(500) | `Applicable_Manufacturer_or_Applicable_GPO_Making_Payment_Name` | Long names common |
| `payment_amount` | DECIMAL(12,2) | `Total_Amount_of_Payment_USDollars` | |
| `payment_date` | DATE | `Date_of_Payment` | |
| `payment_nature` | VARCHAR(255) | `Nature_of_Payment_or_Transfer_of_Value` | |
| `payment_type` | VARCHAR(50) | Derived: General/Research/Ownership | Add if missing |
| `program_year` | SMALLINT | `Program_Year` | Add if missing — needed for annual filtering |
| `record_id` | VARCHAR(50) | `Record_ID` from CMS | Add for idempotent upserts |

**Migration needed (V9):** Add `program_year` and `record_id` columns if not present. Add unique constraint on `record_id` for idempotent re-ingestion.

### 3.5 `part_b_services` Table

**Verify/add:**

| Column | Type | Source Field |
|---|---|---|
| `npi` | VARCHAR(10) | `Rndrng_NPI` |
| `hcpcs_code` | VARCHAR(10) | `HCPCS_Cd` |
| `hcpcs_description` | VARCHAR(500) | `HCPCS_Desc` |
| `total_beneficiaries` | INTEGER | `Tot_Benes` |
| `total_services` | DECIMAL(12,2) | `Tot_Srvcs` |
| `total_medicare_payment` | DECIMAL(14,2) | `Tot_Mdcr_Pymt_Amt` |
| `avg_medicare_payment` | DECIMAL(12,2) | `Avg_Mdcr_Pymt_Amt` |
| `service_year` | SMALLINT | Derived from dataset year | Add if missing |

### 3.6 `part_d_drugs` Table

**Verify/add:**

| Column | Type | Source Field |
|---|---|---|
| `npi` | VARCHAR(10) | `Prscrbr_NPI` |
| `brand_name` | VARCHAR(255) | `Brnd_Name` |
| `generic_name` | VARCHAR(255) | `Gnrc_Name` |
| `total_claims` | INTEGER | `Tot_Clms` |
| `total_30day_fills` | DECIMAL(10,2) | `Tot_30day_Fills` |
| `total_day_supply` | DECIMAL(12,2) | `Tot_Day_Suply` |
| `total_drug_cost` | DECIMAL(14,2) | `Tot_Drug_Cst` |
| `total_beneficiaries` | INTEGER | `Tot_Benes` |
| `drug_year` | SMALLINT | Derived from dataset year | Add if missing |

### 3.7 New Tables Needed

**`taxonomy_reference`** — NUCC lookup for resolving taxonomy codes to human-readable specialty names:

```sql
CREATE TABLE taxonomy_reference (
    taxonomy_code VARCHAR(20) PRIMARY KEY,
    grouping VARCHAR(100),
    classification VARCHAR(100),
    specialization VARCHAR(100),
    definition TEXT,
    notes TEXT,
    display_name VARCHAR(255) -- computed: classification + specialization
);
```

**`nppes_import_log`** — Track NPPES import runs for audit/debugging:

```sql
CREATE TABLE nppes_import_log (
    id BIGSERIAL PRIMARY KEY,
    import_date TIMESTAMP DEFAULT NOW(),
    source_file VARCHAR(500),
    records_processed INTEGER,
    records_inserted INTEGER,
    records_updated INTEGER,
    records_skipped INTEGER,
    errors INTEGER,
    status VARCHAR(20), -- 'RUNNING', 'COMPLETE', 'FAILED'
    error_detail TEXT
);
```

### 3.8 Flyway Migration Sequence

| Version | Description |
|---|---|
| V8 | Add missing columns to `physicians` (npi_status, deactivation_date, last_updated, npi_type); truncate synthetic data |
| V9 | Add `program_year`, `record_id` to `open_payments`; add `service_year` to `part_b_services`; add `drug_year` to `part_d_drugs` |
| V10 | Create `taxonomy_reference` table; create `nppes_import_log` table |
| V11 | Add missing columns to `physician_addresses` (geocoded_at, geocode_confidence, address_type); verify `geom` index |
| V12 | Add indexes for NPI columns across all enrichment tables (performance for join queries) |

---

## 4. Implementation Tickets — Sprint 3 Candidates

### S3-01 — Schema Migrations (Flyway V8–V12)
**Type:** Tech Debt / Foundation  
**Owner:** Charlie  
**Estimate:** 3 points  
**Description:**  
Write and test Flyway migrations V8 through V12 as defined in Section 3.8. Includes truncating synthetic data, adding missing columns, creating new lookup tables, and adding performance indexes. Must run cleanly on a fresh Docker PostgreSQL instance.  
**Acceptance criteria:**
- All V8–V12 migrations execute without error on clean DB
- Rollback scripts documented (or Flyway repeatable migrations used where safe)
- `nppes_import_log` and `taxonomy_reference` tables exist and empty
- All `npi` FK columns indexed

---

### S3-02 — Remove Synthetic Data Generator; Add Real NPPES Parser
**Type:** Bug / Tech Debt  
**Owner:** Charlie  
**Estimate:** 5 points  
**Description:**  
The existing Spring Batch job (S2-10) generates mock physician data. This story removes the synthetic generator and replaces it with a real NPPES CSV parser.  
- Download NPPES Full Replacement ZIP from `download.cms.gov/nppes/NPI_Files.html` (automate or manual for Sprint 3)
- Upload to S3 staging bucket
- Spring Batch reader: parse large CSV using streaming (not load-all-into-memory) — use `FlatFileItemReader` with `BufferedReaderFactory` for streaming
- Filter: `Entity_Type_Code = 1`, `Provider_Business_Mailing_Address_State_Name = OK`
- Write to `physicians` table (upsert on NPI)
- Write practice addresses to `physician_addresses` table
- Log run in `nppes_import_log`  

**Acceptance criteria:**
- ~25K real Oklahoma physicians imported successfully
- Zero synthetic NPIs remaining in DB
- All imported records have real NPI, name, address
- Spring Batch job logs start/end counts to `nppes_import_log`

---

### S3-03 — Load NUCC Taxonomy Reference + Populate `physician_specialties`
**Type:** Story  
**Owner:** Charlie  
**Estimate:** 3 points  
**Description:**  
Download NUCC taxonomy CSV and load into `taxonomy_reference` table. Extend NPPES import (S3-02) to also populate `physician_specialties` from taxonomy fields.  
- Download NUCC CSV
- Bulk load into `taxonomy_reference`
- For each NPPES record processed in S3-02, insert all non-null taxonomy codes into `physician_specialties`
- Populate `physicians.taxonomy_code` with primary taxonomy  
**Acceptance criteria:**
- `taxonomy_reference` fully populated with NUCC codes
- `physician_specialties` populated for all physicians with at least one taxonomy
- `physicians.taxonomy_code`, `physicians.gender`, `physicians.enumeration_date` all non-null for imported records

---

### S3-04 — Address-Level Geocoding via AWS Location Service
**Type:** Story  
**Owner:** Charlie + Frank  
**Estimate:** 5 points  
**Description:**  
Use the existing AWS Location Service integration (S2-01) to geocode all ~25K practice addresses imported in S3-02. Update `physician_addresses.geom` with real lat/lng coordinates.  
- Create Spring Batch geocoding job: read `physician_addresses` where `geom IS NULL`
- Call AWS Location Service SearchPlaceIndexForText (or SearchPlaceIndexForPosition) with full address string
- Parse response: extract lat/lng, confidence score
- Update `physician_addresses.geom` (PostGIS Point, SRID 4326), `geocoded_at`, `geocode_confidence`
- Throttle to 50 req/sec (AWS Location Service default limit) — use Spring Batch throttle or `Thread.sleep`
- Handle failures gracefully: log failed addresses, continue processing  
**Acceptance criteria:**
- ≥ 90% of addresses successfully geocoded
- `physician_addresses.geom` populated for geocoded records
- City-centroid fallback only for addresses that fail geocoding (not as primary)
- Frank: confirm Place Index exists in AWS, correct data provider configured

---

### S3-05 — Open Payments Data Import
**Type:** Story  
**Owner:** Charlie  
**Estimate:** 3 points  
**Description:**  
Download CMS Open Payments General Payments dataset (most recent year available), stage in S3, and import into `open_payments` table.  
- Download General Payments CSV from `openpaymentsdata.cms.gov`
- Filter: `Covered_Recipient_Type = 'Covered Recipient Physician'`
- Join/filter to NPIs present in `physicians` table (Oklahoma physicians only)
- Upsert on `record_id` for idempotent re-ingestion
- Populate `program_year` from dataset  
**Acceptance criteria:**
- `open_payments` populated for all Oklahoma physicians found in dataset
- Duplicate-safe: re-running import does not create duplicate records
- Record count logged

---

### S3-06 — Medicare Part B Import
**Type:** Story  
**Owner:** Charlie  
**Estimate:** 3 points  
**Description:**  
Download Medicare Part B Provider Utilization dataset (most recent year), stage in S3, import into `part_b_services`.  
- Download from `data.cms.gov`
- Filter to NPIs present in `physicians` table
- Populate `service_year`
- Bulk insert (no upsert needed if we truncate by year before re-import)  
**Acceptance criteria:**
- `part_b_services` populated for matching Oklahoma physicians
- `service_year` populated
- Record count logged

---

### S3-07 — Medicare Part D Import
**Type:** Story  
**Owner:** Charlie  
**Estimate:** 3 points  
**Description:**  
Download Medicare Part D Prescribers by Provider and Drug dataset, stage in S3, import into `part_d_drugs`.  
- Download from `data.cms.gov`
- Filter to NPIs present in `physicians` table
- Populate `drug_year`
- Bulk insert  
**Acceptance criteria:**
- `part_d_drugs` populated for matching Oklahoma physicians
- `drug_year` populated
- Record count logged

---

### S3-08 — S3 Staging Bucket + ECS Fargate Task Definition (Frank)
**Type:** Infrastructure  
**Owner:** Frank  
**Estimate:** 3 points  
**Description:**  
Create S3 staging bucket and ECS Fargate task definition for running Spring Batch ingestion jobs outside of local Docker.  
- S3 bucket: `medical-sales-raw-data` with appropriate IAM policies (read for ECS task, write for download automation)
- ECS Fargate task definition: 4 vCPU / 16GB RAM, Spring Boot app container
- IAM role with permissions: S3 read/write, Location Service geocode, RDS/PostgreSQL access
- CloudWatch log group for job output  
**Acceptance criteria:**
- S3 bucket exists, correct permissions
- ECS task definition works for Spring Batch job execution
- IAM least-privilege role configured
- Logs visible in CloudWatch

---

### Sprint 3 Summary

| Ticket | Type | Owner | Points | Priority |
|---|---|---|---|---|
| S3-01 | Schema / Flyway V8–V12 | Charlie | 3 | 🔴 Must (blocker for all others) |
| S3-02 | NPPES Import (replace synthetic) | Charlie | 5 | 🔴 Must |
| S3-03 | Taxonomy Reference + Specialties | Charlie | 3 | 🔴 Must |
| S3-04 | Geocoding (AWS Location) | Charlie + Frank | 5 | 🔴 Must |
| S3-08 | S3 + ECS Fargate infra | Frank | 3 | 🔴 Must (enables S3-02) |
| S3-05 | Open Payments import | Charlie | 3 | 🟡 Should |
| S3-06 | Medicare Part B import | Charlie | 3 | 🟡 Should |
| S3-07 | Medicare Part D import | Charlie | 3 | 🟡 Should |
| **Total** | | | **28 points** | |

**Recommended sequencing:**
1. S3-08 (Frank, parallel) — infra first
2. S3-01 → S3-02 → S3-03 → S3-04 (Charlie, serial — each depends on previous)
3. S3-05, S3-06, S3-07 (Charlie, can run in parallel after S3-02 complete)

---

## 5. Risks & Dependencies

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| NPPES 8GB file — download/parse time | High | High | Stream CSV (don't load into memory); use ECS Fargate not local Docker; expect 1–2 hour parse time |
| NPPES format changes (CMS changes headers without notice) | Medium | High | Pin import to specific column names; add schema validation step that checks header row before processing |
| Data quality — missing NPPES fields | High | Medium | ~15% of providers have missing address or taxonomy fields; import what exists, log nulls, don't block on clean data |
| NPI mismatch between NPPES and CMS datasets | Medium | Medium | Some physicians appear in Open Payments/Part B/D but not NPPES (deactivated NPIs, out-of-state). Use LEFT JOIN logic: enrich only NPIs present in `physicians` table |
| AWS Location Service rate limits (50 req/sec default) | Medium | Medium | Build throttle into Spring Batch geocoding job; ~25K addresses at 50/sec = ~8 min, well within limits |
| AWS Location Service geocode quality | Medium | Low | Some rural Oklahoma addresses will get low-confidence results; store confidence score, use city centroid as fallback |
| NPPES monthly refresh — delta vs full replacement | Medium | Low | NPPES publishes a full replacement file monthly (not incremental). Strategy: truncate and re-import `physicians` monthly, or implement NPI-based upsert. Upsert recommended to preserve FK integrity with enrichment tables |
| Licensing / terms of use | Low | Low | All CMS datasets (NPPES, Open Payments, Part B, Part D) are explicitly public domain per CMS data use policies. No attribution required. NUCC taxonomy is also publicly available. Zero licensing risk. |
| Sprint 3 capacity (28 points) | Medium | Medium | Consider moving S3-05/06/07 (enrichment tables) to Sprint 4 if velocity is tight. Core value is real physicians + geocoding (S3-01 through S3-04). |

---

## 6. Recommended Execution Sequence

### Phase 1 — Foundation (Week 1)
1. **Frank:** Create S3 bucket + ECS Fargate task definition (S3-08)
2. **Charlie:** Write Flyway V8–V12 migrations (S3-01)
3. **Manual:** Download NPPES Full Replacement File, upload to S3 staging

### Phase 2 — Core Data Import (Week 1–2)
4. **Charlie:** Remove synthetic data generator, build NPPES CSV parser (S3-02)
5. **Charlie:** NUCC taxonomy load + `physician_specialties` population (S3-03)
6. **Run:** Execute import job in ECS Fargate → ~25K real Oklahoma physicians in DB

### Phase 3 — Geocoding (Week 2)
7. **Charlie + Frank:** AWS Location Service geocoding batch job (S3-04)
8. **Run:** Execute geocoding job → real lat/lng on all practice addresses

### Phase 4 — Enrichment (Week 2–3, can slip to Sprint 4)
9. **Charlie:** Open Payments import (S3-05)
10. **Charlie:** Part B import (S3-06)
11. **Charlie:** Part D import (S3-07)

---

## Appendix A — Key URLs

| Resource | URL |
|---|---|
| NPPES Download | `https://download.cms.gov/nppes/NPI_Files.html` |
| NPPES Data Dictionary | `https://www.cms.gov/files/document/npi-enumeration-type-one-field-descriptions.pdf` |
| Open Payments Data | `https://openpaymentsdata.cms.gov/dataset/General-Payment-Data-Detailed-Dataset-2023-Reporting/vq63-hu5i` |
| Medicare Part B | `https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service` |
| Medicare Part D | `https://data.cms.gov/provider-summary-by-type-of-service/medicare-part-d-prescribers/medicare-part-d-prescribers-by-provider-and-drug` |
| NUCC Taxonomy CSV | `https://nucc.org/index.php/code-sets-mainmenu-41/provider-taxonomy-mainmenu-40/csv-mainmenu-57` |
| AWS Location Service Docs | `https://docs.aws.amazon.com/location/latest/developerguide/what-is.html` |
| AWS Location Pricing | `https://aws.amazon.com/location/pricing/` |

---

## Appendix B — Spring Batch NPPES Streaming Pattern

```java
// FlatFileItemReader configured for streaming large NPPES CSV
@Bean
public FlatFileItemReader<NppesRecord> nppesItemReader() {
    return new FlatFileItemReaderBuilder<NppesRecord>()
        .name("nppesReader")
        .resource(new FileSystemResource(nppesFilePath))
        .linesToSkip(1) // skip header
        .delimited()
        .names(NPPES_FIELD_NAMES)
        .fieldSetMapper(new NppesFieldSetMapper())
        .build();
}

// Critical: use chunk-oriented processing, not load-all-into-memory
// Recommended chunk size: 500 records
@Bean
public Step nppesImportStep() {
    return stepBuilderFactory.get("nppesImportStep")
        .<NppesRecord, Physician>chunk(500)
        .reader(nppesItemReader())
        .processor(nppesProcessor()) // filter OK state, NPI_Type=1
        .writer(nppesWriter())       // upsert to physicians + physician_addresses
        .faultTolerant()
        .skipLimit(100)
        .skip(FlatFileParseException.class)
        .build();
}
```

---

*Document prepared by Dennis Reynolds, PM*  
*"I have a system. I always have a system."*
