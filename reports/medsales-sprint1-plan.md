# Medical Sales Intelligence & CRM Platform
## Sprint 1 Planning Document

**Prepared by:** Dennis Reynolds, Project Manager  
**Date:** March 1, 2026  
**Sprint:** Sprint 1 — Foundation & Open Questions Resolution  
**Duration:** 2 weeks (March 2 – March 13, 2026)  
**PRD Reference:** PRD v1.2  
**Status:** OQ-1 through OQ-4 resolved — Sprint 1 dev tasks S1-02, S1-06, S1-11, S1-13 unblocked (updated Mar 2, 2026)

---

## 1. Sprint 1 Objective

Sprint 1 does not ship user-facing features. I want to be very clear about that so nobody wastes two weeks building the wrong thing.

What Sprint 1 accomplishes:

1. **Answers every open question from PRD Section 8** — dev cannot start in earnest until the 9 open questions have documented answers. We are not guessing. We are not "starting and adjusting later." We get answers, we document them, we update the PRD to v1.2, and then we build.
2. **Establishes the technical foundation** — stack decisions, repo structure, CI/CD pipeline, data infrastructure skeleton. Nothing fancy, but it has to be correct, because every decision made in Sprint 1 will echo for the next year.
3. **Produces the finalized Sprint 2 backlog** — by end of Sprint 1, the team knows exactly what they're building in Sprint 2 and how.

This is the sprint that makes or breaks the velocity of everything that follows. I take it seriously. The team will take it seriously.

---

## 2. Open Questions — MUST Resolve Before Dev Starts

These are the 9 open questions from PRD Section 8. Each one has a blocking consequence if left unanswered. They need answers from Sean by **end of day, March 3, 2026** (Day 2 of Sprint 1). Not "we'll figure it out." Documented answers.

---

### OQ-1: SaaS vs. Internal Tool ✅ RESOLVED — March 2, 2026

**Decision:** SaaS — multi-tenant architecture. This is a product sold to multiple pharma/device companies. Multi-tenancy is required from day one. All CRM tables must include `org_id` for full org isolation.

**Impact on PRD sections:** NFR-019, Section 5.0, and CRM data model — all updated in PRD v1.2. S1-06 (DB schema) and S1-13 (CRM schema) are now **UNBLOCKED**.

---

### OQ-2: Mobile Platform — React Native vs. Flutter ✅ RESOLVED — March 2, 2026

**Decision:** React Native, iOS-first. Phase 1 = iOS 16+. Android follows in a later phase. My recommendation was correct. Dennis Reynolds: 1, open questions: 0.

**Impact:** S1-11 (mobile app skeleton) is **UNBLOCKED**. Developer starts with React Native configured for iOS 16+. NFR-025 updated in PRD v1.2.

---

### OQ-3: Hosted vs. Self-Hosted ✅ RESOLVED — March 2, 2026

**Decision:** Cloud-hosted on AWS. No self-hosted or on-premises option in v1. Infrastructure: ECS or EKS for containers, RDS PostgreSQL + PostGIS for primary database.

**Impact:** S1-05 (AWS environment setup) is **UNBLOCKED**. DevOps starts AWS provisioning immediately. Section 5.0 updated in PRD v1.2.

---

### OQ-4: CRM Integration — Which CRM First? ✅ RESOLVED — March 2, 2026

**Decision:** Standalone CRM for v1. We are the CRM. No Salesforce, Veeva, or HubSpot integration in Phase 1. CSV export is Phase 2 (FR-048). Bidirectional API integration is Phase 3.

**Impact:** S1-13 (CRM schema design) is **UNBLOCKED**. Developer designs `CRMCall`, `CRMTask`, etc. as standalone tables with `org_id` isolation — no external CRM mapping tables needed. PRD v1.2 FR-048 updated with this decision.

---

### OQ-5: Commercial Data Supplement Budget

**Question:** Is there budget to add IQVIA, Symphony Health, or Definitive Healthcare data to supplement the free CMS datasets?

**Context:**
- CMS Part B/D data is 18–24 months old — that's a known limitation
- Commercial data providers offer near-real-time prescribing data but cost $50,000–$500,000/year
- The free data is still genuinely useful and competitive vs. legacy tools; it is not a dealbreaker
- This decision changes the cost structure of the product significantly

**Recommendation from Dennis:** Launch v1 with free CMS data. Display data vintage prominently (NFR-032 already requires this). Add commercial data as a Phase 2/3 premium tier feature if the business case materializes from customer feedback.

**Decision needed from:** Sean  
**Deadline:** March 3, 2026 (low urgency — does not block Sprint 2, but needs to be in the business plan)

---

### OQ-6: Sample Management Compliance Level

**Question:** Is FDA-compliant sample distribution tracking required (e-signature, chain of custody), or is internal tracking sufficient?

**Why it matters:**
- FDA-compliant sample tracking (as required under the Prescription Drug Marketing Act) requires e-signature capture, audit trail, and potentially third-party compliance platform integration
- This is a significant scope expansion if required
- Many internal tools just track quantities and rely on the rep to maintain physical compliance

**Recommendation:** Internal tracking only for v1 (FR-029, FR-030 as written). FDA compliance is a Phase 3 feature requiring legal review.

**Decision needed from:** Sean  
**Deadline:** March 5, 2026

---

### OQ-7: US-Only Scope Confirmation

**Question:** US only, or will international markets (Canada, EU) be required?

**Why it matters:**
- International scope changes everything — different data sources, GDPR compliance, localization
- EU requires GDPR compliance review, data residency decisions, language support
- CMS data is entirely US-based; international would require entirely new data sourcing

**Recommendation:** US only for v1. This should be a firm constraint in the PRD.

**Decision needed from:** Sean  
**Deadline:** March 5, 2026

---

### OQ-8: Email Integration Scope

**Question:** Do reps want emails to physicians auto-logged in the CRM? (Requires Gmail/Outlook integration.)

**Why it matters:**
- This is a significant engineering effort — OAuth integration with Gmail/Outlook, email parsing, linking to physician records
- Without it, reps manually log emails (simple text field in call log)
- If required for v1, it expands scope substantially

**Recommendation:** Out of scope for v1. Manual email logging only. Phase 3 adds Gmail/Outlook integration.

**Decision needed from:** Sean  
**Deadline:** March 5, 2026

---

### OQ-9: AI Prompt Export Admin Toggle

**Question:** Should organizations be able to disable the AI prompt export feature for reps on networks with strict data handling policies?

**Why it matters:**
- RISK-NEW-002 in the PRD identifies this data privacy concern
- If yes, requires admin settings infrastructure Sprint 1/2
- If no, simpler v1

**Recommendation:** Include the toggle. It's low-effort to build into admin settings and prevents a future compliance fire. The PRD already calls for it.

**Decision needed from:** Sean  
**Deadline:** March 5, 2026

---

## 3. Sprint 1 Tasks — Prioritized

### 🔴 P0 — Must complete this sprint

| # | Task | Owner | Estimated Effort | Notes |
|---|---|---|---|---|
| S1-01 | **Open Questions documentation session with Sean** — get answers to OQ-1 through OQ-9, document in decisions log | Dennis + Sean | 2 hrs | Must happen by Day 2 |
| S1-02 | **Publish PRD v1.2** — incorporate all answered open questions, finalize stack decisions, close or update all open questions | Dennis | 3 hrs | After OQ answers received |
| S1-03 | **Technology stack decision document** — confirm: React Native vs Flutter, backend language/framework, database (PostgreSQL + PostGIS), cloud provider, CI/CD toolchain | Dev Lead | 4 hrs | Must precede S1-04 |
| S1-04 | **Create GitHub repo and project structure** — monorepo or polyrepo decision, folder structure, README, .gitignore, initial CI pipeline (GitHub Actions) | Developer | 4 hrs | After S1-03 |
| S1-05 | **AWS/cloud environment setup** — dev, staging, production accounts, IAM structure, VPC, RDS (PostgreSQL) instance provisioned, S3 buckets for data ingestion | DevOps | 6 hrs | Parallel with S1-03 |
| S1-06 | **Database schema v1** — create initial schema migrations covering Physician, PhysicianAddress, PhysicianTaxonomy, PhysicianCMSProfile, HospitalAffiliation, Hospital tables (per PRD Section 5.2) | Developer | 6 hrs | After S1-05 |
| S1-07 | **NPPES test ingestion** — download NPPES bulk file, run test import against dev database, validate row counts and NPI format (NFR-031) | Developer | 8 hrs | Key risk validation |
| S1-08 | **Finalize Sprint 2 backlog** — translate answered open questions + stack decisions into fully groomed Sprint 2 tickets with acceptance criteria | Dennis | 4 hrs | End of sprint |

---

### 🟡 P1 — Should complete this sprint if time allows

| # | Task | Owner | Estimated Effort | Notes |
|---|---|---|---|---|
| S1-09 | **Geocoding provider decision and cost estimate** — evaluate Google Maps vs. Mapbox for geocoding ~5M records; get pricing, select vendor | Developer + DevOps | 2 hrs | Needed before data pipeline build |
| S1-10 | **Specialty normalization mapping table design** — map NPPES taxonomy codes → CMS plain-English → normalized display names (RISK-005) | Developer | 4 hrs | Can stub, does not need to be complete |
| S1-11 | **Mobile app skeleton** — initialize React Native (or Flutter) project, configure iOS build target (iOS 16+), confirm simulator runs | Developer | 4 hrs | Unblocks Sprint 2 mobile work |
| S1-12 | **Data architecture decision document** — PostgreSQL + PostGIS for geospatial queries, pg_trgm for text search (or Elasticsearch), confirm or change PRD recommendations | Developer + DevOps | 3 hrs | Written record for the team |
| S1-13 | **CRM schema design** — design org-isolated CRM tables (CRMCall, CRMTask, CRMOpportunity, CRMOrder, CRMSample) per PRD Section 5.2. Confirm multi-tenant structure if OQ-1 answered as SaaS | Developer | 4 hrs | After OQ-1 answered |

---

### 🟢 P2 — Stretch goals

| # | Task | Owner | Estimated Effort | Notes |
|---|---|---|---|---|
| S1-14 | **Provider Data Catalog test ingestion** — download CMS National Downloadable File, test join against NPPES records by NPI | Developer | 4 hrs | Stretch |
| S1-15 | **CI/CD pipeline smoke test** — confirm GitHub Actions triggers build on push, runs linting and basic schema migration check | DevOps | 3 hrs | Stretch |

---

## 4. What Does NOT Ship in Sprint 1

To be extremely clear: none of the following are Sprint 1 scope.

- No user-facing UI (not even a login screen)
- No actual data in production
- No physician profiles
- No CRM functionality
- No mobile app published to any store
- No routing features
- No API endpoints exposed externally

The entire team is laying foundation and answering questions. That is the sprint. Anyone who tries to run ahead and start building features before the open questions are resolved will create rework. I am not cleaning up avoidable rework.

---

## 5. Dependencies & Blockers

| Dependency | Blocks | Status | Owner |
|---|---|---|---|
| Sean's answers to OQ-1 (SaaS vs. internal) | S1-02, S1-06, S1-13 | ✅ RESOLVED — SaaS multi-tenant (Mar 2, 2026) | Sean |
| Sean's answers to OQ-2 (mobile platform) | S1-03, S1-11 | ✅ RESOLVED — React Native, iOS-first (Mar 2, 2026) | Sean |
| Sean's answers to OQ-3 (hosted vs. self-hosted) | S1-05 | ✅ RESOLVED — AWS cloud-hosted (Mar 2, 2026) | Sean |
| Sean's answers to OQ-4 (CRM integration) | S1-13 | ✅ RESOLVED — Standalone CRM v1, no external integration (Mar 2, 2026) | Sean |
| Cloud provider selection (OQ-3) | S1-05 | ✅ RESOLVED — AWS (ECS/EKS + RDS PostgreSQL) | DevOps |
| Stack decision document (S1-03) | S1-04, S1-06, S1-11 | ⚠️ NOT STARTED | Dev Lead |
| AWS environment (S1-05) | S1-06, S1-07 | ⚠️ NOT STARTED | DevOps |
| Database schema v1 (S1-06) | S1-07 | ⚠️ NOT STARTED — **UNBLOCKED** (OQ-1 resolved) | Developer |
| NPPES bulk file download (~8GB) | S1-07 | ⚠️ NOT STARTED | Developer |
| Geocoding vendor decision (S1-09) | Sprint 2 data pipeline | ⚠️ NOT STARTED | Developer |

---

## 6. Definition of Done — Sprint 1

Sprint 1 is **done** when ALL of the following are true:

- [x] OQ-1 through OQ-4 answered by Sean (Mar 2, 2026) — documented in PRD v1.2 and sprint plan
- [ ] OQ-5 through OQ-10 remaining (non-blockers for Sprint 2 — deadline March 5, 2026)
- [x] PRD v1.2 is published with OQ-1–4 resolved and all downstream sections updated
- [ ] A technology stack decision document exists in the repo and has been reviewed by the full team
- [ ] GitHub repository is created with initial project structure and CI pipeline running green
- [ ] AWS (or equivalent) dev environment is provisioned: RDS PostgreSQL instance running, S3 bucket created, dev/staging environments exist
- [ ] Database schema v1 migration is applied to dev database and reviewed
- [ ] At least one successful NPPES bulk import test has run against dev database with row count validation
- [ ] Sprint 2 backlog is fully groomed: tickets exist, have acceptance criteria, have estimates, and are prioritized
- [ ] All P0 tasks (S1-01 through S1-08) are marked complete in the task tracker

---

## 7. Sprint 1 Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Sean does not answer open questions by Day 2 | Medium | High — blocks S1-02 through S1-13 | Dennis escalates daily. No passive waiting. |
| NPPES bulk ingestion fails or takes >4 hours | Medium | Medium — validates NFR-006 early | Test in dev only; failure is informative, not catastrophic in Sprint 1 |
| Stack disagreement on React Native vs. Flutter | Low | Medium — delays S1-11 | Decision log documents the choice and rationale; team alignment meeting Day 1 |
| Cloud setup takes longer than estimated | Low | Medium | DevOps starts Day 1; should not be the critical path |
| Scope creep — someone starts building UI | Low | High | Dennis is watching. This will not happen. |

---

## 8. Sprint 1 Success Metrics

At sprint review, we should be able to demonstrate:

1. Read out every open question with its documented answer
2. Show the PRD v1.2 with open questions resolved
3. Show the GitHub repo running a green CI build
4. Show `psql` connected to dev RDS instance with schema applied
5. Show NPPES row count in dev database (even partial) — proves the pipeline works
6. Walk through the Sprint 2 backlog — every ticket has a description, acceptance criteria, and an estimate

If we can do all six, Sprint 1 is a success and we are on track for Sprint 2 to ship real features.

---

## 9. Sprint 2 Preview (Pending OQ Resolution)

Assuming Sean answers OQ-1 (SaaS), OQ-2 (React Native, iOS-first), OQ-3 (AWS cloud-hosted), Sprint 2 likely covers:

- Full NPPES + Provider Data Catalog ingestion pipeline to production
- Physician master record API endpoint (GET /physicians/{npi})
- Geocoding pipeline for practice addresses
- Basic radius search query (PostGIS ST_DWithin)
- Mobile app login screen + basic physician list view
- Authentication (JWT-based, or Cognito on AWS)

Sprint 2 is where we start building. Sprint 1 is where we make sure we build the right thing.

---

*Prepared by Dennis Reynolds, Project Manager*  
*These are the questions. Get me the answers. Then we build.*
