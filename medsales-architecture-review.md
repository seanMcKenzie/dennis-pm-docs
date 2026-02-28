# MedSales Architecture Review
## Formal Assessment Report — Medical Sales Intelligence & CRM Platform

**Prepared by:** Dennis Reynolds, Project Manager  
**Date:** February 28, 2026  
**Architecture Author:** Frank Reynolds, DevOps & Solutions Architect  
**Document Version:** 1.0  
**Status:** Final

---

> *I went through every document. I read every diagram. I followed every ADR to its conclusion.
> Here is what I found — the good, the gaps, and what needs to happen before I sign off on this.*

---

## 1. Executive Summary

Frank produced a **competent and substantially complete architecture** for the MedSales platform. This is not a throwaway set of slides — it is a genuine, reasoned technical plan with database schemas, actual SQL, working Dockerfiles, a CI/CD pipeline, disaster recovery procedures, and a dedicated geocoding microservice that is frankly the best part of the document. The architecture handles the hardest problems in this platform correctly: geospatial search, multi-source data joins, multi-tenant CRM isolation, and the cost trap of geocoding five million addresses.

**Overall Confidence Level: 7.5 / 10**

This architecture is ready for development to begin on the data layer and API backbone. It is **not** ready for greenlight on the full platform because several underspecified components — most critically, the identity/auth provider — could cause significant rework if left unresolved. There are also cost estimate gaps that Sean needs to understand before budgeting.

**Bottom line:** This is a GO on Phase 1 development, conditioned on resolving five specific gaps documented in Section 4. Do not start building until those are answered.

---

## 2. Strengths — What Frank Got Right

### 2.1 Data Domain Separation

Separating the **Physician Intelligence DB** (shared, read-heavy, public CMS data) from the **CRM DB** (per-org, read-write, tenant-isolated) is the single most important architectural decision in this platform, and Frank got it right. These two data domains have fundamentally different access patterns, consistency requirements, and scaling strategies. Combining them would have been a time bomb. The separation is clean and enforced throughout the schema design.

### 2.2 PostgreSQL + PostGIS (ADR-001)

The right call. PostgreSQL with PostGIS handles five million geospatial points with sub-second `ST_DWithin` queries when indexed correctly. The schema shows the GIST index on `physician_address.geom` — that is the index that makes the entire app work. Frank knows this. The alternatives considered (MongoDB, DynamoDB, Elasticsearch as primary) were correctly rejected. The table partitioning strategy for Part B (250 million rows) and Part D (80 million rows) by `data_year` is not optional — it is mandatory — and Frank identified it before it became a problem.

### 2.3 Geocoding Service Architecture

This is the standout piece of the entire document. Frank built a complete, production-grade geocoding microservice with:

- **Multi-provider fallback** (Google Maps → Mapbox → US Census) with automatic failover
- **Cache-first strategy** with SHA-256 address hashing — identical normalized addresses share one geocode result
- **Address normalization pipeline** that improves the cache hit rate from ~20% to ~60%
- **Census batch geocoding first** on initial load — this reduces the five million address geocoding cost from ~$25,000 (naive Google-everything approach) to approximately **$2,500–5,000**

This is the kind of engineering thinking that saves real money. The decision to use the US Census Bureau's free batch geocoder for the first pass (85% coverage, $0 cost) and only pay Google for the remaining ~15% is the right move.

### 2.4 Search Strategy — Defer Elasticsearch (ADR-002)

ADR-002's decision to use PostgreSQL + PostGIS + `pg_trgm` for Phase 1 search, with a defined trigger to evaluate Elasticsearch in Phase 2, is correct. Many engineering teams reach for Elasticsearch too early and then spend months managing data sync pipelines they didn't need yet. Frank defined clear performance thresholds (p95 > 3 seconds) that would justify the switch. The materialized view strategy to pre-aggregate common search patterns is the right mitigation.

### 2.5 Security Architecture

The security posture is solid for a v1:

- VPC with private subnets (no direct internet to app/data tier)
- AWS WAF with OWASP Top 10 rule set in production
- TLS 1.2+ everywhere, AES-256 at rest
- Row-Level Security on all CRM tables (`ALTER TABLE crm_call ENABLE ROW LEVEL SECURITY;`) — this is database-enforced tenant isolation, not just application-layer promises
- Non-root container execution (`adduser appuser`)
- Trivy container scanning in CI/CD pipeline with hard failure on CRITICAL/HIGH CVEs
- Secrets Manager with 90-day rotation

The dual database strategy for multi-tenancy (separate `org_id` indexed columns + RLS) is belt-and-suspenders and appropriate for a SaaS product.

### 2.6 Three-Environment CI/CD with Manual Production Gate

The GitHub Actions pipeline with dev (auto-deploy on `develop`), staging (auto-deploy on `main`), and production (manual approval after staging QA) is the correct model. The rollback procedure (redeploy previous ECS task definition) is documented and fast (5-minute RTO). Good.

### 2.7 Data Ingestion Pipeline Design

The five-stage ingestion pipeline (Acquire → Validate → Transform → Load → Verify) is thorough. Specifically:

- **Schema validation before loading** — catches CMS format changes that have broken integrations in the past (the NPPES V1→V2 migration in 2026 would have destroyed a naive pipeline)
- **Address diff detection** — only re-geocodes addresses that actually changed, not all five million on every monthly refresh
- **Data lineage logging** — every ingestion run logged with file name, row counts, timing, and success/failure status. This is how you know the data is current.
- **NPI Luhn validation** at ingestion — invalid NPIs quarantined, not silently imported

### 2.8 React Native + Offline Architecture (ADR-003)

Appropriate choice. Native bridge for maps (Apple Maps on iOS, Google Maps on Android) avoids WebView performance issues. WatermelonDB for local SQLite-backed offline storage is the right library for this use case — it has a migration system, reactive queries, and handles the sync queue pattern cleanly. The offline scope is realistic: last 100 accessed profiles, queued writes, today's task list.

### 2.9 Deployment Architecture — Multi-AZ with DR

Production in multi-AZ with RDS failover and a DR standby in `us-west-2` behind WAL streaming is appropriate for a 99.9% SLA target. The 4-hour RTO / 1-hour RPO is achievable with this setup. The ECS rollback to a previous task definition (5-minute RTO for bad deployment) is documented and correct.

---

## 3. Risks & Gaps — What's Missing or Concerning

### 🔴 GAP-001: Identity Provider Is Completely Unspecified (CRITICAL)

Every diagram shows a box labeled "Identity Provider / SSO / SAML 2.0" or "Auth Service." There is no ADR for this. There is no decision. Options include:

- **AWS Cognito** — managed, integrates natively with ALB, SAML/OIDC support, per-MAU pricing (~$0.0055/MAU for 50K+ users)
- **Keycloak** — self-hosted, full-featured, adds operational overhead
- **Auth0 / Okta** — expensive at scale but enterprise-familiar
- **Custom JWT service** — most dangerous option; do not do this

This is not a detail. Authentication and authorization are foundational. Every API endpoint, every tenant isolation mechanism, every audit log references `org_id` and `user_id` that come from auth. You cannot build this system without deciding what issues the tokens.

**Resolution required before development begins.**

### 🔴 GAP-002: No Connection Pooler in Deployment Architecture (CRITICAL)

ADR-001 correctly identifies that PgBouncer or RDS Proxy is required for 10,000 concurrent users. At 10K concurrent users, even if only 20% are actively querying at any moment, that's 2,000 connections attempting to hit PostgreSQL. PostgreSQL handles perhaps 500-1,000 connections before memory pressure degrades performance significantly.

The deployment architecture shows ECS tasks connecting directly to RDS. **There is no PgBouncer or RDS Proxy in any of the architecture diagrams or Terraform structure.** This will cause production failures at scale.

**RDS Proxy** (AWS managed, ~$0.015/hour per GB of compute capacity) is the path of least resistance here given the ECS Fargate deployment model. Add it.

### 🟡 GAP-003: SQS vs. RabbitMQ Decision Not Made

Every document says "SQS / RabbitMQ" with no decision. These are architecturally different:
- **SQS** — managed, AWS-native, dead letter queues built in, no infrastructure to maintain. Appropriate for this platform.
- **RabbitMQ** — more flexible routing, but requires managing a broker cluster (another operational surface area)

For a platform on AWS with ECS Fargate and the batch processing patterns described, **SQS is the obvious choice**. There is no reason to operate a RabbitMQ cluster here. This decision should be formalized in ADR-004.

### 🟡 GAP-004: Production Cost Estimates Exclude Ongoing Geocoding (SIGNIFICANT)

Frank's deployment cost estimate of **~$3,300/month** for production is missing a line item: ongoing geocoding.

The geocoding service doc covers this well internally, but it never makes it into the deployment cost summary. Monthly geocoding costs (NPPES refresh + rolling re-geocode) are:

- NPPES monthly refresh: ~200K addresses × 40% cache miss = 80K API calls × $5/1,000 = **~$400/month**
- Rolling re-geocode (1/12th of DB/month, 60% cache hit): ~417K addresses × 40% miss = 167K × $5/1,000 = **~$835/month** (tapering as cache warms; see Section 5)

This adds **$600–1,200/month** in steady-state geocoding costs that are absent from the summary. Sean needs to know this number.

### 🟡 GAP-005: Elasticsearch Migration Path Is Underplanned

ADR-002 says moving to Elasticsearch in Phase 2 is "~2-3 weeks engineering." This is optimistic. The actual scope includes:
- Setting up a managed Elasticsearch cluster (AWS OpenSearch or Elastic Cloud)
- Building a Change Data Capture (CDC) sync pipeline (Debezium + Kafka, or a custom sync worker)
- Defining the Elasticsearch index mappings for physician data (non-trivial with 10+ dimensions)
- Maintaining eventual consistency between PG and ES in production
- Migrating existing search queries
- Load testing

A realistic estimate is **4-8 weeks** if the trigger criteria are hit, and the migration must be seamless with no downtime. There is no documented migration plan, only a diagram. If there's any chance Elasticsearch is needed in Phase 2, start designing the sync pipeline now, not when search is already broken in production.

### 🟡 GAP-006: Mobile Offline Conflict Resolution Is Hand-Wavy

ADR-003 states: "Conflict resolution: last-write-wins with server timestamp; conflicts flagged for user review."

Last-write-wins is acceptable for call logs and task notes, but the protocol is not specified. What happens when a rep logs a call offline at 2pm, manager updates the same physician's status online at 3pm, and the rep syncs at 5pm? Which wins? What does "flagged for user review" look like in the mobile UI?

This needs a simple, documented conflict resolution protocol before the sync engine is built. Fix it on paper now. It is very expensive to fix in production.

### 🟡 GAP-007: Staging Environment Lacks WAF

The environment sizing table shows WAF = No for both Dev and Staging, Yes for Production. Staging is supposed to be "production-like." If WAF is in production, WAF should be in staging — otherwise you will discover WAF-related issues (rule conflicts, false positives blocking legitimate API calls) in production, not staging.

### 🟠 GAP-008: No Load Testing Plan Documented

The architecture targets 10,000 concurrent users and 3-second p95 search response times. These targets appear in the NFRs but there is no load testing specification, no identified tooling (k6, JMeter, Locust), and no plan for benchmarking PostGIS queries against production-scale data (5M physicians, 250M Part B rows) before go-live.

This is not just a nice-to-have. The ADR-002 decision to defer Elasticsearch explicitly depends on PostGIS meeting the performance targets under load. If it doesn't, and there's no load test before launch, the first time you discover it is when sales reps are in the field.

### 🟠 GAP-009: Re-Geocoding Budget Is Underrepresented

The geocoding service doc mentions a rolling 12-month re-geocode cycle (1/12th of DB/month) as a good practice. At 5M total addresses, that's ~417K addresses/month for re-geocoding on top of the 200K NPPES delta. Frank mentions this generates ~$700/month in additional cost. This is not included in any budget figure in the deployment architecture document. It needs to be accounted for.

### 🟠 GAP-010: Specialty Normalization Table Not Provided

RISK-005 in the PRD correctly identifies that NPPES, Provider Data Catalog, and Open Payments use three incompatible specialty naming conventions. The data architecture creates the `specialty_mapping` table. No mapping data is provided. This table has to be populated before specialty filtering works at all, and it requires manual curation of 200+ taxonomy codes. This is engineering work that is not accounted for in any timeline I can see.

---

## 4. Recommendations — Changes Required Before Greenlight

### REC-001 (REQUIRED): Create ADR-004 for Identity Provider

**Action:** Frank and the dev team must evaluate and select an authentication provider this week. My recommendation: **AWS Cognito** for Phase 1. Reasons: managed, no servers to run, native ALB integration, SAML 2.0 and OAuth 2.0 support for enterprise SSO in Phase 3, and predictable per-MAU pricing. Document this in ADR-004 with the same rigor as ADR-001 through ADR-003.

### REC-002 (REQUIRED): Add RDS Proxy to Deployment Architecture

**Action:** Update the Terraform modules and deployment architecture to include RDS Proxy between ECS tasks and RDS. Size it appropriately for the connection pool. This is a blocking gap for production scaling. Cost impact: ~$50–100/month (see Section 5).

### REC-003 (REQUIRED): Decide SQS and Create ADR-004 or ADR-005

**Action:** Formally select Amazon SQS, remove all references to "SQS / RabbitMQ" ambiguity from the architecture documents, and document the decision. Configure SQS queues (main job queue, geocode queue, notification queue, dead letter queues) in the Terraform modules.

### REC-004 (REQUIRED): Write a Load Testing Specification

**Action:** Before any performance-critical code is declared production-ready, define and execute a load test that:
- Populates a staging database with realistic data volumes (5M physicians, 250M Part B rows, actual PostGIS indexes)
- Simulates 1,000 concurrent users performing radius searches (NFR-002: < 3 second p95 target)
- Simulates 10,000 concurrent users for API throughput baseline
- Documents the results and compares against NFR targets

If the PostGIS search queries don't meet targets under load, **do not launch**. Pivot to Elasticsearch Phase 2 early and build the sync pipeline. This is the decision Frank's ADR-002 depends on.

### REC-005 (REQUIRED): Document Offline Conflict Resolution Protocol

**Action:** Write a one-page conflict resolution specification before the mobile sync engine is built. Cover: call logs, task updates, physician flags, and future opportunity records. Define the merge rules, the UI presentation for flagged conflicts, and the API contract for the sync endpoint. "Last-write-wins" is a design, not an implementation.

### REC-006 (RECOMMENDED): Add WAF to Staging

**Action:** Enable AWS WAF on staging and configure the same rule groups as production. This has minimal cost impact (~$5/month for staging) and will catch WAF-related issues before they reach production.

### REC-007 (RECOMMENDED): Revise Deployment Cost Summary

**Action:** Update the cost summary in `deployment-architecture.md` to include:
- Ongoing geocoding: ~$400–800/month steady-state (after initial load)
- RDS Proxy: ~$75/month production
- SQS: ~$20/month
- ECR storage: ~$10/month
- Revised total: **~$4,000–4,500/month for Phase 2 production** (vs. the current $3,300 estimate which excludes geocoding)

### REC-008 (RECOMMENDED): Begin Specialty Normalization Table

**Action:** Assign engineering resources to produce the initial `specialty_mapping` table data before the ingestion pipeline is built. This is a content/data problem, not a code problem, but it is blocking. Use the NUCC taxonomy code list as the canonical source and map to CMS specialty names. Estimated effort: 3-5 days for a developer with access to the taxonomy documentation.

### REC-009 (RECOMMENDED): Elasticsearch Phase 2 Migration Plan

**Action:** Write a one-paragraph architecture sketch for the Elasticsearch migration path now — identify the CDC tool (Debezium preferred over a custom sync worker), the Kafka or direct-to-ES topology, and the estimated timeline realistically (call it 6 weeks, not 2-3). This way it's on the shelf, not invented under pressure.

---

## 5. Cloud Infrastructure Cost Estimate

### 5.1 Cost Assumptions

| Assumption | Value |
|---|---|
| Cloud provider | AWS (us-east-1 primary) |
| Pricing model | On-demand (Reserved Instances would reduce costs 30-40% for stable workloads) |
| MVP definition | Phase 1 launch: 2 API tasks, 1 worker, 1 DB primary + 1 replica, small Redis |
| Phase 2 definition | Scaled production: 4+ API tasks (auto-scale), 2 workers, larger DB, Redis cluster, WAF, Elasticsearch if needed |
| Geocoding | Uses Census-first strategy for initial load; Google Maps for deltas |
| Initial geocoding | ~5M addresses, Census batch first → ~$2,500–5,000 one-time |

---

### 5.2 MVP (Phase 1) Monthly Cost Estimate

*2 API tasks, 1 ingestion worker, 1 geocoding worker — single AZ acceptable for MVP*

| Service | Spec | Est. Monthly Cost |
|---|---|---|
| **ECS Fargate — API** | 2 tasks × 2 vCPU / 4 GB RAM | ~$140 |
| **ECS Fargate — Ingestion Worker** | 1 task × 2 vCPU / 4 GB RAM | ~$70 |
| **ECS Fargate — Geocoding Service** | 1 task × 1 vCPU / 2 GB RAM | ~$35 |
| **RDS PostgreSQL** | db.r6g.large (2 vCPU, 16 GB) primary only | ~$175 |
| **RDS Read Replica** | db.r6g.large | ~$175 |
| **RDS Storage** | 250 GB gp3 | ~$29 |
| **ElastiCache Redis** | cache.r6g.large (1 node) | ~$100 |
| **ALB** | 1 dedicated | ~$50 |
| **S3** | 20 GB raw CMS files + exports | ~$15 |
| **CloudFront CDN** | Light traffic (static assets + mobile bundle) | ~$30 |
| **NAT Gateway** | 1 AZ | ~$45 |
| **CloudWatch** | Logs + basic metrics | ~$50 |
| **SQS** | 3 queues, moderate volume | ~$10 |
| **Secrets Manager** | ~10 secrets | ~$4 |
| **Route 53** | 1 hosted zone + queries | ~$5 |
| **ECR** | ~5 GB image storage | ~$0.50 |
| **Data transfer** | Inter-service + internet out | ~$30 |
| **Geocoding — monthly NPPES delta** | ~80K API calls (60% cache hit on 200K) | ~$400 |
| **Subtotal — MVP Infrastructure** | | **~$1,363/month** |
| **Geocoding ongoing (steady-state)** | Monthly refresh only | **~$400/month** |
| **MVP TOTAL** | | **~$1,400–1,800/month** |

> **MVP Infrastructure note:** Reserve Instances on RDS (1-year, no upfront) reduce DB costs by ~30%, saving ~$100/month. RDS Proxy adds ~$50/month.

---

### 5.3 Phase 2 (Scaled Production) Monthly Cost Estimate

*4+ API tasks (auto-scaling), 2 workers, larger DB with 2 read replicas, Redis cluster, WAF, multi-AZ*

| Service | Spec | Est. Monthly Cost |
|---|---|---|
| **ECS Fargate — API** | 4–6 tasks × 2 vCPU / 4 GB (auto-scale) | ~$280–420 |
| **ECS Fargate — Workers** | 2 tasks × 2 vCPU / 4 GB | ~$140 |
| **ECS Fargate — Geocoding** | 2 tasks × 2 vCPU / 4 GB | ~$140 |
| **RDS PostgreSQL — Primary** | db.r6g.2xlarge (8 vCPU, 64 GB) | ~$350 |
| **RDS Read Replicas** | 2 × db.r6g.2xlarge | ~$700 |
| **RDS Storage** | 500 GB gp3 (auto-expand) | ~$58 |
| **RDS Proxy** | Connection pooler (production-grade, required) | ~$75 |
| **ElastiCache Redis** | cache.r6g.xlarge cluster (2 shards, 2 replicas) | ~$450 |
| **ALB** | 1 dedicated | ~$75 |
| **AWS WAF** | OWASP rule set + custom rules | ~$100 |
| **S3** | 50 GB raw + exports + backups | ~$40 |
| **CloudFront CDN** | ~1 TB/month data transfer | ~$100 |
| **NAT Gateway** | 2 AZs | ~$90 |
| **CloudWatch** | Full logs + metrics + dashboards | ~$150 |
| **SQS** | 5 queues + DLQs | ~$20 |
| **SNS Alerts** | Ops notifications | ~$5 |
| **Secrets Manager** | ~20 secrets, 90-day rotation | ~$8 |
| **Route 53** | DNS + health checks | ~$8 |
| **ECR** | ~10 GB image storage | ~$1 |
| **AWS Cognito** | 10,000 MAU × $0.0055 = $55 (first 50K free) | ~$0–55 |
| **Inter-AZ Data Transfer** | ~500 GB/month @ $0.01/GB each way | ~$10 |
| **General Data Transfer** | Internet out ~1 TB | ~$90 |
| **Geocoding — Monthly NPPES delta** | 80K calls (60% cache hit) | ~$400 |
| **Geocoding — Rolling re-geocode** | ~167K calls/month (1/12th DB, 60% cache hit) | ~$835 |
| **Subtotal — Phase 2 Infrastructure** | | **~$3,345/month** |
| **Geocoding steady-state** | Included above | — |
| **Phase 2 TOTAL** | | **~$3,800–4,300/month** |

> **Optional Phase 2 add: Elasticsearch (AWS OpenSearch)** — If PostGIS search performance does not meet targets under load, add AWS OpenSearch:
> - 3-node cluster, r6g.large.search: ~$450/month
> - Revised Phase 2 total with Elasticsearch: **~$4,300–4,800/month**

---

### 5.4 One-Time Costs

| Item | Estimate | Notes |
|---|---|---|
| **Initial geocoding** | $2,500–5,000 | Census-first strategy; ~3.2M unique addresses; ~480K Google calls at $5/1K |
| **Terraform setup & automation** | Engineering time (~2 weeks dev) | IaC modules for all environments |
| **Mobile app store registration** | $100 (Apple $99/yr + Google $25 one-time) | Standard |
| **SSL/TLS certificates** | $0 | AWS ACM is free for ALB-terminated certs |
| **SOC 2 Type II audit** | $30,000–50,000 | Required per NFR-024; plan for 18 months post-launch |
| **Load testing run (k6 / Locust)** | Engineering time | ~1 week for setup + execution |

---

### 5.5 Cost Summary Table

| Phase | Infrastructure | Geocoding (Monthly) | Total Monthly |
|---|---|---|---|
| **MVP (Phase 1)** | ~$900–1,000 | ~$400 | **~$1,400–1,800** |
| **Phase 2 (Scaled)** | ~$3,000–3,500 | ~$800–1,200 | **~$3,800–4,700** |
| **Phase 2 + Elasticsearch** | ~$3,500–4,000 | ~$800–1,200 | **~$4,300–5,200** |

> **Note for Sean:** Frank's deployment architecture estimated ~$3,300/month for production. That estimate is approximately correct for the infrastructure itself, but it does not include ongoing geocoding costs ($800–1,200/month), RDS Proxy ($75/month), or Cognito (~$55/month at 10K MAU). The real steady-state number is **$4,000–4,700/month** for Phase 2 production. Budget accordingly.

---

## 6. Overall Verdict

### GO — With Conditions

**The architecture is sound.** Frank made the right calls on database, search strategy, mobile framework, geocoding cost optimization, and security posture. The documentation is thorough enough to build from. The ADRs show genuine consideration of alternatives, not just post-hoc justification of decisions already made. The geocoding service in particular is excellent engineering thinking that saves tens of thousands of dollars.

**Five conditions must be met before development begins in earnest:**

| # | Condition | Owner | Deadline |
|---|---|---|---|
| 1 | Select and document identity provider (ADR-004) | Frank + Dev Lead | Before sprint 1 begins |
| 2 | Add RDS Proxy to deployment architecture and Terraform | Frank | Week 1 |
| 3 | Formally select SQS (over RabbitMQ), document in ADR | Frank | Week 1 |
| 4 | Write offline sync conflict resolution protocol | Dev Lead | Before mobile development begins |
| 5 | Define and schedule load testing against production-scale data | QA + Dev | Before staging → production gate |

**Additional work I want before I'm fully comfortable:**

- Specialty normalization table populated (blocks search filtering entirely)
- WAF enabled on staging
- Geocoding costs added to the deployment cost summary
- Elasticsearch Phase 2 migration plan sketched (one page; not a full ADR, just a plan on the shelf)

**If these conditions are met:** I am prepared to greenlight development. The data pipeline, database schema, geocoding service, and API architecture are ready to build. Phase 1 is achievable. The platform has a realistic path to production.

**If these conditions are not met:** I will block the sprint. Not to be difficult. Because I have seen what happens when auth is undefined and you build six services on top of it. We are not doing that.

---

## 7. Appendix — Document Coverage Matrix

| Architecture Document | Reviewed | Assessment |
|---|---|---|
| README.md | ✅ | Clean index; nothing missing |
| system-architecture-overview.md | ✅ | Solid; auth provider gap noted |
| component-diagram.md | ✅ | Comprehensive; Route Export Service is correct per PRD v1.1 |
| data-architecture.md | ✅ | Excellent; SQL schemas are implementation-ready |
| deployment-architecture.md | ✅ | Good; missing RDS Proxy, geocoding costs, WAF on staging |
| geocoding-service.md | ✅ | Best document in the set. Ship it. |
| ADR-001-database.md | ✅ | Correct decision, well-argued |
| ADR-002-search.md | ✅ | Correct decision; Phase 2 timeline is optimistic |
| ADR-003-mobile.md | ✅ | Correct decision; stack is appropriate |
| PRD (medical-sales-platform-prd.md) | ✅ | Architecture addresses PRD v1.1 faithfully |

**PRD Coverage Check:**

| PRD Section | Architecture Coverage | Status |
|---|---|---|
| FR-001–011 (Physician Data) | Full schema, API endpoints, ingestion pipeline | ✅ Covered |
| FR-012–014 (Multi-site practices) | physician_address table, location endpoints | ✅ Covered |
| FR-015–018 (Search & Filter) | Search Service, PostGIS, pg_trgm, Redis cache | ✅ Covered |
| FR-019–021 (Location/Map) | GPS via RN, Map Service, PostGIS | ✅ Covered |
| FR-NEW-001–005 (Route Export) | Route Export Service documented, no external APIs | ✅ Covered |
| FR-026–033 (CRM Calls/Tasks) | CRM schema, endpoints, offline sync | ✅ Covered |
| FR-034–039 (Pipeline/Opportunities) | CRM schema includes opportunities, orders, dashboards | ✅ Covered |
| FR-040–043 (Territory Management) | Territory Service, territory tables | ✅ Covered |
| FR-044–048 (Data Refresh/Admin) | Ingestion pipeline, admin endpoints | ✅ Covered |
| NFR-010 (99.9% uptime) | Multi-AZ, DR in us-west-2, ECS health checks | ✅ Covered |
| NFR-012 (RPO 1hr / RTO 4hr) | WAL streaming, RDS PITR | ✅ Covered |
| NFR-015 (SSO/SAML) | **Auth provider not selected — GAP-001** | ❌ Gap |
| NFR-026 (Offline mode) | WatermelonDB, sync engine | ✅ Covered |
| NFR-031 (NPI validation) | Luhn check at ingestion | ✅ Covered |

---

*This is a solid foundation. Build on it. But fix the five conditions first. That is not a suggestion.*

*— Dennis Reynolds, Project Manager*
