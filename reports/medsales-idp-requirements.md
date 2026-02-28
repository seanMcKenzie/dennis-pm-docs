# MedSales Identity Provider (IdP) — Functional Requirements & Cost Assessment

**Prepared by:** Dennis Reynolds, Project Manager  
**Date:** February 28, 2026  
**Version:** 1.0  
**Status:** Draft — Pending Engineering Review  
**Linked PRD:** medical-sales-platform-prd.md v1.1  
**References:** NFR-015 (MFA + SSO), NFR-016 (RBAC), NFR-017 (API keys), NFR-018 (audit logs), NFR-019 (org isolation)

---

## 1. Executive Summary

### Why We're Building This

Sean has made the call: we build our own Identity Provider. Let me explain why that's actually defensible — and what it's going to cost us.

The MedSales platform is a multi-tenant SaaS product serving pharmaceutical and medical device sales organizations. Each organization has its own users, roles, and data isolation requirements. The authentication surface includes mobile apps (iOS + Android), a web companion, and system integration APIs (Salesforce export, etc.). Enterprise customers will eventually require SAML 2.0 SSO so their reps can log in through their corporate identity systems (Okta Workforce, Azure AD, Ping Identity).

The core tension in the build-vs-buy decision is this: **off-the-shelf IdPs are priced for scale and recoup their development cost across a large customer base. At 10K MAU they're expensive. At 50K MAU they're very expensive. At 100K+ MAU with enterprise SAML features, you're looking at $100K–$500K/year before negotiation.**

An internally-built IdP amortizes its development cost (roughly $120K–$180K in Charlie's time plus infrastructure) across the entire customer base indefinitely. The 3-year break-even against Auth0 and Okta at 50K MAU is approximately 18–24 months.

The risk is real: identity is the one place where you absolutely cannot ship bugs. A poorly implemented OAuth flow, a JWT secret rotation failure, or a TOTP implementation mistake can compromise every user account simultaneously. This is not a place for "move fast and break things." We will move deliberately and we will test exhaustively.

### What We're Building

A purpose-built, standards-compliant Identity Provider that implements:

- **OAuth 2.0 Authorization Code flow with PKCE** (mobile and web clients)
- **OpenID Connect (OIDC)** for SSO across MedSales products
- **JWT access tokens with refresh token rotation**
- **TOTP-based MFA** (Google Authenticator, Authy, etc.) with optional SMS/email OTP fallback
- **RBAC** with Rep, Manager, and Admin roles, scoped to organization-level tenancy
- **API key management** for system integrations
- **Password policies and brute-force protection**
- **Comprehensive audit logging** of all authentication events
- **(Phase 2) SAML 2.0 IdP** — so enterprise customers can connect MedSales to their corporate IdP

This is a real deliverable. It will require real engineering time. It will be maintained indefinitely. The requirements below are written to be handed directly to Charlie.

---

## 2. Functional Requirements

### 2.1 User Registration & Account Lifecycle

**FR-IDP-001** — The system shall support user registration via email and password. Registration shall require: email address (unique per organization), password (meeting complexity policy), first name, last name, and organization assignment.

**FR-IDP-002** — Upon registration, the system shall send an email verification link to the user's email address. The link shall expire after 24 hours. Unverified accounts shall not be granted API access.

**FR-IDP-003** — Organization administrators shall be able to pre-provision user accounts (invite flow), where the user receives an invitation email with a one-time setup link. The link shall expire after 72 hours.

**FR-IDP-004** — The system shall support account deactivation (soft delete). Deactivated accounts shall immediately have all active sessions invalidated and all access tokens revoked. Deactivated accounts shall retain their user record and audit log history but shall be unable to authenticate.

**FR-IDP-005** — The system shall support account re-activation by an organization administrator. Re-activation shall trigger email notification to the user.

**FR-IDP-006** — The system shall support self-service password reset via email. The reset link shall be single-use and shall expire after 1 hour. After password reset, all existing sessions shall be invalidated.

**FR-IDP-007** — Users shall be able to update their own email address, subject to re-verification of the new address. Organization administrators shall be able to update any user's email address within their organization.

---

### 2.2 Authentication — Login & Logout

**FR-IDP-008** — The system shall authenticate users via username (email) and password. Authentication shall occur over HTTPS only. Passwords shall never be transmitted or logged in plaintext.

**FR-IDP-009** — The system shall support login from the following client types:
- Native mobile app (iOS + Android) — uses Authorization Code flow with PKCE
- Web browser (SPA or server-rendered) — uses Authorization Code flow
- Server-to-server integrations — uses API key authentication (see FR-IDP-040 through FR-IDP-047)
- CLI / automation — uses OAuth 2.0 Device Authorization Grant (Phase 2)

**FR-IDP-010** — The login endpoint shall return an authorization code upon successful credential + MFA verification. The client shall exchange this code for an access token and refresh token via the token endpoint. Credentials shall never be passed directly to the client application.

**FR-IDP-011** — The system shall support a "Remember this device" option that suppresses MFA prompts for a configurable period (default: 30 days) on a recognized device. Device recognition shall use a signed, httpOnly cookie or device fingerprint — not a browser localStorage value. Organization administrators shall be able to disable this feature.

**FR-IDP-012** — The system shall support logout at two levels:
- **Local logout:** Invalidates the current session and access/refresh tokens for the active client only.
- **Global logout (all sessions):** Invalidates all active sessions and tokens across all devices for the user. Available to users and administrators.

**FR-IDP-013** — Upon logout, the system shall redirect to a configurable post-logout redirect URI registered for the client application.

---

### 2.3 OAuth 2.0 Authorization Code Flow + PKCE

**FR-IDP-014** — The system shall implement the OAuth 2.0 Authorization Code flow as specified in RFC 6749. The authorization endpoint shall accept the following required parameters: `response_type=code`, `client_id`, `redirect_uri`, `scope`, and `state`.

**FR-IDP-015** — The system shall implement Proof Key for Code Exchange (PKCE) as specified in RFC 7636. PKCE shall be **required** for all public clients (mobile apps, SPAs). The system shall support the `S256` code challenge method. The `plain` method shall not be accepted.

**FR-IDP-016** — Authorization codes shall be single-use and shall expire after 60 seconds. A code that has already been redeemed or has expired shall be rejected and an `invalid_grant` error returned.

**FR-IDP-017** — The token endpoint shall issue tokens in response to a valid authorization code exchange. Supported grant types:
- `authorization_code` — standard code exchange
- `refresh_token` — refresh token rotation (see FR-IDP-022)
- `client_credentials` — for server-to-server (confidential clients only)

**FR-IDP-018** — The token endpoint shall validate `redirect_uri` exactly against the registered redirect URIs for the client. Partial matches, wildcards, and open redirectors shall not be accepted.

**FR-IDP-019** — The system shall support OAuth 2.0 scopes. Default scopes for MedSales: `openid`, `profile`, `email`, `crm:read`, `crm:write`, `admin:read`, `admin:write`, `api:keys`. Scopes shall be validated against the requesting client's registered allowed scopes.

**FR-IDP-020** — The system shall implement OAuth 2.0 client registration. Each registered client shall have: `client_id`, `client_secret` (confidential clients only), registered `redirect_uris`, allowed `grant_types`, allowed `scopes`, and a display name.

**FR-IDP-021** — The system shall expose an OAuth 2.0 metadata discovery endpoint at `/.well-known/oauth-authorization-server` as specified in RFC 8414, returning all supported endpoints, grant types, scopes, and token signing algorithms.

---

### 2.4 OpenID Connect (OIDC) for SSO

**FR-IDP-022** — The system shall implement OpenID Connect Core 1.0 on top of the OAuth 2.0 base. When the `openid` scope is requested, the token endpoint shall return an ID token (JWT) in addition to the access token.

**FR-IDP-023** — The ID token shall include the following standard claims: `iss` (issuer), `sub` (subject — user UUID), `aud` (audience — client_id), `exp` (expiration), `iat` (issued at), `auth_time`, and optionally `nonce` (when provided by client).

**FR-IDP-024** — The ID token shall include the following MedSales custom claims: `org_id`, `org_name`, `role` (rep | manager | admin), and `email_verified`.

**FR-IDP-025** — The system shall expose a UserInfo endpoint at `/userinfo` as specified in OIDC Core. The endpoint shall require a valid Bearer access token with the `openid` scope. The response shall include all claims corresponding to the scopes granted.

**FR-IDP-026** — The system shall expose an OIDC Discovery endpoint at `/.well-known/openid-configuration` returning all required OIDC metadata fields including: `issuer`, `authorization_endpoint`, `token_endpoint`, `userinfo_endpoint`, `jwks_uri`, `scopes_supported`, `response_types_supported`, `id_token_signing_alg_values_supported`.

**FR-IDP-027** — The system shall expose a JSON Web Key Set (JWKS) endpoint at `/jwks.json` or `/.well-known/jwks.json`. The JWKS shall contain the current public keys used to verify ID token and access token signatures. The JWKS shall support key rotation without downtime (overlap period for old and new keys).

**FR-IDP-028** — ID tokens and access tokens shall be signed using RS256 (RSA + SHA-256) with a 2048-bit minimum key size. The signing key shall be rotatable without service interruption.

---

### 2.5 JWT Access Tokens + Refresh Token Rotation

**FR-IDP-029** — Access tokens shall be short-lived JWTs. Default lifetime: **15 minutes**. Configurable per client (range: 5 minutes to 1 hour). Organization administrators shall not be able to set access token lifetime beyond 1 hour.

**FR-IDP-030** — The access token JWT payload shall include: `iss`, `sub`, `aud`, `exp`, `iat`, `jti` (JWT ID for revocation tracking), `org_id`, `role`, and granted `scopes`.

**FR-IDP-031** — Refresh tokens shall be long-lived opaque tokens (not JWTs). Default lifetime: **30 days**. Configurable per client. Refresh tokens shall be stored server-side as a secure hash (SHA-256); the raw token shall only be transmitted to the client and never stored plaintext.

**FR-IDP-032** — The system shall implement **refresh token rotation**: every time a refresh token is used to obtain a new access token, a new refresh token is issued and the old one is immediately invalidated. This prevents token replay attacks.

**FR-IDP-033** — The system shall implement **refresh token reuse detection**: if a previously invalidated refresh token is presented, the system shall immediately invalidate the entire token family for that session and alert the user via email that suspicious activity was detected.

**FR-IDP-034** — Access tokens shall be validatable offline by resource servers using the public key from the JWKS endpoint. Resource servers shall not need to make a round-trip to the IdP for every request. Token introspection endpoint (RFC 7662) shall be provided for resource servers that require revocation checking.

**FR-IDP-035** — The system shall maintain a token revocation list (blacklist) for access tokens that were explicitly revoked before their expiration time. Revocation events: global logout, password change, account deactivation, admin revocation. The revocation list shall be consulted on token introspection calls. Access tokens presented to resource servers without introspection checking will be valid until expiration — this is acceptable given the short (15-minute) lifetime.

---

### 2.6 Multi-Factor Authentication (MFA)

**FR-IDP-036** — The system shall require MFA for all user accounts. MFA enforcement shall not be optional at the user level. Organization administrators may configure the MFA policy for their organization.

**FR-IDP-037** — The system shall support **TOTP (Time-based One-Time Password)** as the primary MFA method, compliant with RFC 6238. Users shall enroll by scanning a QR code in any standard authenticator app (Google Authenticator, Authy, Microsoft Authenticator, 1Password, etc.). The QR code shall encode an `otpauth://` URI.

**FR-IDP-038** — During TOTP enrollment, the system shall display backup codes (minimum 8 single-use codes). Backup codes shall be stored as bcrypt hashes server-side. Users shall be warned that backup codes are shown only once. Organization administrators shall be able to regenerate backup codes on behalf of a user.

**FR-IDP-039** — The system shall support a TOTP grace window of ±1 time step (30 seconds before or after the current window) to account for clock drift on user devices.

**FR-IDP-040** — The system shall support **email OTP** as a secondary MFA method. When selected, the system sends a 6-digit code to the user's verified email address. The code shall expire after 10 minutes and be single-use.

**FR-IDP-041** — The system shall support **SMS OTP** as an optional MFA method (requires Twilio or equivalent SMS gateway integration). SMS OTP shall be considered lower-assurance than TOTP. Organization administrators shall be able to disable SMS OTP for their organization.

**FR-IDP-042** — The MFA step shall occur after successful password authentication and before token issuance. Failed MFA attempts shall be rate-limited: 5 failed attempts triggers a 15-minute lockout of MFA for that session. This is distinct from the account lockout in FR-IDP-055.

**FR-IDP-043** — Organization administrators shall be able to reset a user's MFA configuration (e.g., if user loses their phone). The reset shall send a notification to the user's email. The reset event shall be logged in the audit log.

**FR-IDP-044** — The system shall support MFA step-up: resource servers may request step-up authentication (re-challenge MFA) for sensitive operations (e.g., admin actions, data export). Step-up shall be implemented via the `acr_values` OIDC parameter.

---

### 2.7 Role-Based Access Control (RBAC) + Organization Isolation

**FR-IDP-045** — The system shall implement RBAC with three base roles:
- **Rep** — access to own CRM data, physician search, own activity logs; cannot access other users' data
- **Manager** — access to all Rep-level data plus: team CRM data, territory reports, rep performance dashboards; cannot access system administration
- **Admin** — full access to organization configuration, user management, territory management, data export, API key management; cannot access other organizations' data

**FR-IDP-046** — Every user account shall be associated with exactly one organization. An organization is the top-level tenancy boundary. A user in Organization A shall have no access whatsoever to Organization B's data, users, or configuration.

**FR-IDP-047** — Organization identity shall be embedded in the JWT access token as `org_id`. All API endpoints shall validate `org_id` against the requested resource's organization before authorizing access. Cross-organization access is never permitted regardless of role.

**FR-IDP-048** — Organization administrators shall be able to manage users within their own organization only: create, deactivate, role-assign, and MFA-reset. System-level administration (creating new organizations, adjusting pricing, platform-wide config) is reserved for MedSales internal Superadmin accounts.

**FR-IDP-049** — The system shall support a **Superadmin** role for MedSales internal staff. Superadmins can: create and manage organizations, act as an Admin within any organization (impersonation shall be logged), and access platform-wide audit logs. Superadmin accounts shall be issued only from a designated internal email domain.

**FR-IDP-050** — Role changes (promotion or demotion) shall take effect on the next login. Active sessions shall not be forcibly invalidated on role change — this is acceptable given the short (15-minute) access token lifetime. Administrators may trigger a global logout for the affected user if immediate role enforcement is required.

**FR-IDP-051** — The system shall support future extensibility for fine-grained permissions (permission flags attached to roles). The data model shall not assume that Rep, Manager, and Admin are permanent fixed roles. Role definitions shall be stored in a configuration table, not hardcoded. (Actual fine-grained permission enforcement is Phase 2.)

---

### 2.8 API Key Management

**FR-IDP-052** — The system shall allow organization administrators to create API keys for system-to-system integrations (Salesforce data export, Veeva CRM sync, webhook consumers, etc.).

**FR-IDP-053** — API keys shall be long, randomly generated strings (minimum 256 bits of entropy). They shall be displayed to the administrator exactly once at creation time and never again. The system shall store only a SHA-256 hash of the key.

**FR-IDP-054** — Each API key shall have: a human-readable name (set by the admin), an optional description, an associated scope set (drawn from the same OAuth 2.0 scopes as user tokens), an optional expiration date, and an optional IP allowlist (CIDR notation). Keys without an expiration date shall be valid indefinitely until revoked.

**FR-IDP-055** — Each API key shall be scoped to the creating organization. An API key cannot access resources belonging to another organization.

**FR-IDP-056** — Organization administrators shall be able to list all API keys for their organization (name, creation date, last used timestamp, expiration, status). Administrators shall be able to revoke any key immediately.

**FR-IDP-057** — The system shall track API key usage: last used timestamp, total request count (approximate), and any failed authentication attempts. This data shall be available in the admin dashboard.

**FR-IDP-058** — API key authentication shall occur via the `Authorization: Bearer <key>` header on API requests. API key tokens shall be validated by looking up the hash in the key store, not by JWT signature. As a result, API key lookups are a database read — the key store shall be backed by a Redis cache with a short TTL (60 seconds) to avoid per-request database hits.

---

### 2.9 Password Policies & Brute-Force Protection

**FR-IDP-059** — The system shall enforce the following minimum password policy (configurable per organization, with these as floor values):
- Minimum length: 12 characters
- Must contain at least one uppercase letter, one lowercase letter, one digit, and one special character
- Must not match the user's email address, first name, or last name
- Must not be a known breached password (checked against HaveIBeenPwned API `k-Anonymity` endpoint)
- Must not be one of the user's last 10 passwords (stored as bcrypt hashes)

**FR-IDP-060** — Passwords shall be hashed using **bcrypt** with a work factor of 12 (re-evaluated for increases as hardware improves). Argon2id is the preferred alternative; the system architecture shall make hashing algorithm configurable to allow migration.

**FR-IDP-061** — The system shall implement account lockout after **5 consecutive failed login attempts** within a 15-minute window. Lockout duration: 15 minutes (auto-unlock). After 10 failed attempts within any 60-minute window, the account shall be locked until an administrator or the user unlocks it via email verification.

**FR-IDP-062** — The system shall implement per-IP rate limiting on the login endpoint: maximum 20 login attempts per IP per 5-minute window. Rate limit state shall be maintained in Redis.

**FR-IDP-063** — The system shall implement CAPTCHA challenge (hCaptcha or Cloudflare Turnstile) after 3 failed login attempts from the same IP or user account. CAPTCHA shall not be required on the first 3 attempts.

**FR-IDP-064** — The login endpoint shall return identical error messages for "user not found" and "invalid password" — preventing username enumeration. Response timing shall be normalized using a dummy bcrypt hash comparison when the user is not found, to prevent timing-based enumeration.

**FR-IDP-065** — The system shall implement credential stuffing detection using device fingerprinting and velocity anomaly detection (>10 unique accounts attempted from the same IP in 1 hour triggers IP-level throttle). Phase 2: integrate with an IP reputation feed (AbuseIPDB or similar).

---

### 2.10 SAML 2.0 Identity Provider *(Phase 2)*

**FR-IDP-066** *(Phase 2)* — The system shall implement SAML 2.0 Identity Provider functionality, enabling enterprise customers to configure their own Service Providers (their corporate IdP: Azure AD, Okta Workforce, Ping Identity, Google Workspace) to federate with MedSales.

**FR-IDP-067** *(Phase 2)* — The SAML IdP shall expose: IdP metadata XML, SSO endpoint (HTTP-POST and HTTP-Redirect bindings), and SLO (Single Logout) endpoint.

**FR-IDP-068** *(Phase 2)* — Organization administrators shall be able to configure an inbound SAML connection for their organization by providing their corporate SP metadata XML or manually configuring: Entity ID, SSO URL, and X.509 certificate. This enables SP-initiated SSO flows.

**FR-IDP-069** *(Phase 2)* — When a user authenticates via SAML federation, the system shall automatically provision (Just-In-Time provisioning) a MedSales user account if one does not already exist, mapping SAML attributes (email, first name, last name) to MedSales user fields. The default provisioned role shall be `rep`. Administrators may configure attribute mapping rules.

**FR-IDP-070** *(Phase 2)* — SAML-authenticated users shall still receive MedSales JWT access tokens at the end of the flow. The IdP shall bridge SAML → OAuth 2.0 → OIDC, so the rest of the platform sees only JWTs regardless of the upstream authentication method.

---

### 2.11 Session Management

**FR-IDP-071** — The system shall issue sessions upon successful authentication (post-MFA). A session represents a logged-in device/browser. Sessions are tracked server-side by session ID stored in a secure, httpOnly cookie (for browser clients) or in secure storage on mobile.

**FR-IDP-072** — Sessions shall have a configurable idle timeout: default **8 hours** for web, **7 days** for mobile. If a user is idle for longer than the timeout, the session is expired and the next request requires re-authentication.

**FR-IDP-073** — Sessions shall have a configurable absolute maximum lifetime, regardless of activity: default **30 days**. After 30 days, re-authentication is required unconditionally.

**FR-IDP-074** — The system shall enforce concurrent session limits per user. Default: **5 concurrent sessions** (to support a user on phone, tablet, and computer). Organization administrators shall be able to configure this limit (minimum 1, maximum 20). When the limit is exceeded, the oldest session shall be automatically terminated.

**FR-IDP-075** — Users shall be able to view all of their active sessions from the account settings page, including: device type, browser/app, approximate location (city/state from IP geolocation), last activity timestamp, and session creation timestamp. Users shall be able to terminate individual sessions or all sessions except the current one.

**FR-IDP-076** — Session cookies shall be set with: `Secure` (HTTPS only), `HttpOnly` (not accessible to JavaScript), `SameSite=Strict` (prevents CSRF), and an appropriate `Domain` and `Path` scope. CSRF protection shall not rely solely on `SameSite` — CSRF tokens shall be required for state-changing operations.

---

### 2.12 Audit Logging

**FR-IDP-077** — The system shall log every authentication-related event to an immutable audit log. The following events shall be logged at minimum:

| Event Category | Events |
|---|---|
| Authentication | login_success, login_failure, logout, session_expired, token_issued, token_refreshed, token_revoked |
| MFA | mfa_enrolled, mfa_success, mfa_failure, mfa_reset_by_admin, backup_code_used, mfa_method_changed |
| Password | password_changed, password_reset_requested, password_reset_completed, password_locked_out |
| Account | account_created, account_deactivated, account_reactivated, email_verified, email_changed, role_changed |
| Sessions | session_created, session_terminated, session_expired, concurrent_limit_exceeded |
| API Keys | api_key_created, api_key_used, api_key_revoked, api_key_expired |
| Admin Actions | user_impersonated, org_config_changed, user_invited, mfa_reset, global_logout_triggered |
| Security | brute_force_detected, ip_blocked, suspicious_token_reuse, captcha_triggered |

**FR-IDP-078** — Each audit log entry shall contain: timestamp (UTC, millisecond precision), event type, user ID (if applicable), organization ID, IP address, user agent string, session ID (if applicable), and a free-text detail field for event-specific context.

**FR-IDP-079** — Audit logs shall be append-only. No process shall have DELETE or UPDATE access to the audit log table. Audit log integrity shall be verifiable.

**FR-IDP-080** — Audit logs shall be retained for a minimum of 2 years, consistent with NFR-018 of the MedSales PRD.

**FR-IDP-081** — Organization administrators shall be able to query the audit log for their organization with filters: date range, event type, user, and IP address. Audit log data shall be exportable to CSV.

**FR-IDP-082** — Superadmins shall have access to platform-wide audit logs, including cross-organization administrative events.

---

## 3. Non-Functional Requirements

### 3.1 Performance

**NFR-IDP-001** — **Token validation** (JWT signature verification + claims check) at the resource server shall complete in **< 10 milliseconds** on the server. This is achievable via RSA public key verification locally without an IdP round-trip. This is a hard requirement — any architecture decision that adds per-request IdP latency violates this requirement.

**NFR-IDP-002** — The token endpoint (code exchange, refresh) shall respond in **< 200ms** at the 95th percentile under normal load. This includes bcrypt verification on the password grant.

**NFR-IDP-003** — The authorization endpoint (login page redirect, MFA prompt) shall respond in **< 300ms** at the 95th percentile, excluding network latency to the client.

**NFR-IDP-004** — The API key lookup (hash lookup + Redis cache hit) shall complete in **< 5ms** at the 95th percentile. Cache hit rate shall be maintained above 95% for active keys.

**NFR-IDP-005** — The JWKS endpoint shall be served from a CDN or in-memory cache. Response time shall be **< 50ms** globally. Key rotation shall not require JWKS cache invalidation until the new key is active; overlap windows handle this gracefully.

### 3.2 Availability

**NFR-IDP-006** — The IdP shall maintain **99.99% uptime** (≤ 52 minutes downtime/year). This requires:
- Multi-AZ deployment (minimum 3 availability zones)
- Active-active or active-standby failover with < 30-second failover time
- Database replication with automatic failover
- No single point of failure in the critical path: login → token → JWKS

**NFR-IDP-007** — The IdP shall degrade gracefully: if the database is unavailable, existing JWT access tokens (already issued, signature-verifiable locally) shall continue to be accepted by resource servers until they expire (15-minute window). The system shall not be a hard dependency for every API call — only for new token issuance.

**NFR-IDP-008** — Planned maintenance and key rotation operations shall be performable without service interruption. Deployments shall use blue-green or rolling deployment strategy.

### 3.3 Security

**NFR-IDP-009** — The IdP implementation shall comply with **OWASP Application Security Verification Standard (ASVS) Level 2** across all applicable controls, with Level 3 for authentication and session management controls (ASVS Chapter 3) and cryptography (ASVS Chapter 6).

**NFR-IDP-010** — All token signing keys and sensitive secrets (bcrypt pepper if used, HMAC secrets, database credentials) shall be stored in a secrets manager (AWS Secrets Manager, HashiCorp Vault, or equivalent). Keys shall not be stored in environment variables, configuration files, or source code.

**NFR-IDP-011** — The IdP shall undergo a **third-party penetration test** before production launch, specifically targeting: OAuth 2.0 flow vulnerabilities, JWT implementation, PKCE bypass attempts, session fixation, and CSRF. Pen test findings rated High or Critical must be remediated before launch.

**NFR-IDP-012** — All cryptographic primitives shall use established, audited libraries. No custom cryptographic implementations. Approved libraries: `jose` (JWT, JWKS), `bcrypt`/`argon2` (password hashing), `speakeasy`/`otplib` (TOTP), `speakeasy` (HOTP backup codes).

**NFR-IDP-013** — The IdP shall implement HTTP security headers on all responses: `Strict-Transport-Security` (HSTS, 1-year, includeSubDomains), `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Content-Security-Policy` (strict), `Referrer-Policy: no-referrer`.

**NFR-IDP-014** — All inter-service communication (IdP → database, IdP → Redis, IdP → SMS gateway) shall use mutual TLS or equivalent where supported.

### 3.4 Scalability

**NFR-IDP-015** — The IdP API layer shall be horizontally scalable (stateless). Session state shall be maintained in Redis, not in-process. Any IdP instance shall be able to handle any request without sticky sessions.

**NFR-IDP-016** — The IdP shall be designed to support **100,000 MAU** without architectural changes. Scaling beyond 100K MAU may require database sharding or read replicas — this shall be documented in the architecture.

**NFR-IDP-017** — Redis shall be used for: session state, rate limiting counters, TOTP replay prevention (used-OTP tracking), refresh token lookup cache, and API key cache. Redis cluster mode shall be supported for horizontal Redis scaling.

**NFR-IDP-018** — The authentication database (user accounts, sessions, tokens, audit logs) shall be a separate database instance from the main MedSales application database. This provides security isolation, independent scaling, and simplifies compliance auditing.

---

## 4. Build vs. Buy Cost Assessment

### 4.1 Vendor Pricing

> **Note:** Pricing below reflects publicly available rates as of Q1 2026 and industry-standard estimates where exact pricing requires vendor negotiation. Enterprise pricing is typically subject to annual contracts and volume discounts.

---

#### 4.1.1 Auth0 (Okta Customer Identity Cloud)

Auth0 is the market leader for Customer Identity and Access Management (CIAM). Pricing is MAU-based with significant feature gating between tiers.

**Tier Structure:**
- **Free:** 7,500 MAU, no MFA, no custom domains, no enterprise features
- **Essentials:** ~$23/month base, scales by MAU. Basic MFA, custom domains.
- **Professional:** ~$240/month base (up to 1,000 MAU), includes advanced MFA, organizations (multi-tenancy), custom actions/hooks
- **Enterprise:** Custom pricing; required for SAML IdP (outbound SAML), advanced rate limiting, dedicated infrastructure, SLA guarantees

**Estimated costs at MedSales scale:**

| Scale | Tier | Est. Monthly Cost | Est. Annual Cost |
|---|---|---|---|
| 10K MAU | Professional | ~$650 | ~$7,800 |
| 10K MAU + Enterprise SSO (SAML) | Enterprise | ~$2,500–$4,000 | ~$30,000–$48,000 |
| 50K MAU | Professional | ~$2,800 | ~$33,600 |
| 50K MAU + Enterprise SSO (SAML) | Enterprise | ~$6,000–$12,000 | ~$72,000–$144,000 |

**Key limitations:**
- SAML IdP (for enterprise customer SSO) is Enterprise tier only — significant price jump
- Custom MFA flows, custom token claims, and brute-force policy customization require higher tiers or custom Actions (engineering time)
- Per-organization (multi-tenancy) isolation is available but complex to configure at scale
- Vendor lock-in: proprietary rules engine, custom Actions in JavaScript executed on Auth0 infrastructure

---

#### 4.1.2 AWS Cognito

AWS Cognito is the lowest-cost option at small scale, with significant caveats in developer experience and feature completeness.

**Pricing:**
- User Pools: **First 50,000 MAU free.** Then $0.0055/MAU (50K–100K), $0.0046/MAU (100K–1M)
- Advanced Security Features (brute-force protection, anomaly detection): **$0.050/MAU** (all MAU, not just above threshold)
- SAML/OIDC federation (inbound): $0.015 per 1,000 federation events (effectively negligible)
- SMS OTP: Billed via Amazon SNS (~$0.00645/SMS, US)

**Estimated costs at MedSales scale:**

| Scale | Basic | With Advanced Security | Est. Annual |
|---|---|---|---|
| 10K MAU | $0 | ~$500/month | ~$6,000 |
| 50K MAU | $0 | ~$2,500/month | ~$30,000 |
| 100K MAU | ~$275/month | ~$5,275/month | ~$63,000 |

**Key limitations:**
- Developer experience is notoriously poor: Hosted UI is inflexible, SDK is complex, documentation is fragmented
- Refresh token rotation requires custom Lambda triggers — not built-in
- SAML outbound IdP (MedSales as IdP to enterprise SPs) is **not supported** — critical gap for Phase 2
- Custom token claims require Lambda triggers that add latency and complexity
- API key management is not provided — must be built separately regardless
- OAuth 2.0 / OIDC implementation has known edge cases and non-standard behaviors
- Lock-in to AWS ecosystem; migration from Cognito is notoriously painful

---

#### 4.1.3 Okta Workforce Identity

Okta is enterprise-grade workforce identity — it is not designed for CIAM (customer-facing users). It is included for completeness but is not a realistic option for MedSales at the stated scale.

**Pricing:**
- Workforce Identity Cloud SSO: ~$2/user/month
- MFA: ~$3/user/month (additional)
- Advanced Server Access, Lifecycle Management: additional modules
- Full platform (SSO + MFA + LCM): ~$8–$15/user/month

**Estimated costs at MedSales scale:**

| Scale | SSO Only | SSO + MFA | Full Platform |
|---|---|---|---|
| 10K users | ~$20,000/month | ~$50,000/month | ~$80,000–$150,000/month |
| 50K users | ~$100,000/month | ~$250,000/month | ~$400,000–$750,000/month |
| Annual (10K, full) | N/A | N/A | ~$1M–$1.8M/year |

**Verdict:** Okta Workforce Identity is categorically inappropriate for a CIAM use case at this scale. The pricing is 10–100x what's needed here. Okta's CIAM product is Auth0 (Okta Customer Identity Cloud) — see section 4.1.1. This line item is included only to show Sean what he would pay if anyone ever suggested Okta for end-user auth.

---

### 4.2 Build Cost Estimate

#### 4.2.1 Development Hours

Charlie is our sole developer for the MedSales platform. The IdP is a significant but bounded engineering project. Below is the breakdown.

**Phase 1 — Core IdP (OAuth 2.0, OIDC, JWT, MFA, RBAC, API Keys)**

| Component | Estimated Hours | Notes |
|---|---|---|
| Architecture & database design | 24 | Schema, Redis topology, key management design |
| OAuth 2.0 Authorization Code + PKCE | 40 | Authorization endpoint, token endpoint, client registration |
| OIDC layer (ID token, UserInfo, discovery) | 24 | On top of OAuth 2.0 base |
| JWT signing, JWKS, key rotation | 16 | RS256, JWKS endpoint, rotation without downtime |
| User auth (registration, login, logout) | 32 | Including email verification, password reset |
| MFA — TOTP enrollment + verification | 24 | QR code, backup codes, grace window |
| MFA — Email OTP | 8 | Simpler than TOTP |
| MFA — SMS OTP (Twilio) | 8 | |
| RBAC + org isolation | 24 | Role enforcement middleware, org_id validation |
| Password policy + HIBP integration | 16 | Bcrypt, policy enforcement, breach check |
| Brute-force protection + rate limiting (Redis) | 24 | Account lockout, IP throttle, CAPTCHA integration |
| Session management (Redis-backed) | 16 | Session CRUD, concurrent session limits, device list |
| API key management | 24 | Creation, hashing, Redis cache, usage tracking |
| Audit logging (append-only) | 16 | Schema, event emission, query API |
| Admin dashboard API | 24 | User management, MFA reset, session management endpoints |
| Security headers + HTTPS enforcement | 8 | HSTS, CSP, CORS policy |
| Integration tests (comprehensive) | 40 | OAuth flows, PKCE, token rotation, MFA, RBAC |
| Security review + pen test prep | 16 | Self-audit against OWASP ASVS |
| Documentation (API docs, integration guide) | 16 | Required for resource server integration |
| **Phase 1 Total** | **400 hours** | |

**Phase 2 — SAML 2.0 IdP (Enterprise SSO)**

| Component | Estimated Hours |
|---|---|
| SAML 2.0 IdP implementation (HTTP-POST, HTTP-Redirect) | 48 |
| SP configuration UI (admin dashboard) | 24 |
| JIT user provisioning | 16 |
| SAML → OAuth 2.0 bridge | 16 |
| SLO (Single Logout) | 16 |
| Testing against Azure AD, Okta, Google Workspace | 24 |
| **Phase 2 Total** | **144 hours** |

**Total Build: 544 hours (~14 weeks at full dedication, or ~6 months alongside platform development)**

#### 4.2.2 Cost at Charlie's Rate

| Scenario | Hours | Rate | Cost |
|---|---|---|---|
| Phase 1 only | 400 | $150/hr | $60,000 |
| Phase 1 + Phase 2 | 544 | $150/hr | $81,600 |
| Phase 1 only | 400 | $200/hr | $80,000 |
| Phase 1 + Phase 2 | 544 | $200/hr | $108,800 |

> Charlie's rate will drive this number. Adjust accordingly. At $150/hr, Phase 1 is $60K. At $200/hr, it's $80K. Either way, it's less than one year of Auth0 Enterprise at 50K MAU.

#### 4.2.3 Ongoing Maintenance

Post-launch, the IdP requires ongoing maintenance. This is not optional and is a real ongoing cost.

| Activity | Hours/Month | Annual Hours |
|---|---|---|
| Security patches + dependency updates | 4 | 48 |
| Key rotation (quarterly) | 2 | 8 |
| Monitoring + on-call response | 4 | 48 |
| Feature additions (minor) | 4 | 48 |
| Incident response (estimated) | 2 | 24 |
| **Total** | **16 hr/month** | **176 hr/year** |

**Annual maintenance cost: ~$26,400/year at $150/hr | ~$35,200/year at $200/hr**

#### 4.2.4 Infrastructure Cost

The IdP runs as a separate service cluster from the main MedSales application.

| Component | Monthly Cost | Annual Cost |
|---|---|---|
| Compute (2–4 instances, t3.medium, multi-AZ) | $120–$240 | $1,440–$2,880 |
| Database (RDS PostgreSQL, Multi-AZ, db.t3.medium) | $100–$180 | $1,200–$2,160 |
| Redis (ElastiCache, cache.t3.medium, cluster) | $80–$160 | $960–$1,920 |
| Load balancer (ALB) | $25 | $300 |
| Secrets Manager | $10 | $120 |
| CloudWatch logging + monitoring | $30 | $360 |
| SMS OTP (Twilio, est. 1,000 SMS/month) | $7 | $84 |
| **Total Infrastructure** | **~$375–$650/month** | **~$4,500–$7,800/year** |

---

### 4.3 Three-Year Total Cost of Ownership (TCO) Comparison

Assumptions:
- Growth: 10K MAU at launch, 50K MAU by end of Year 2, 100K MAU by Year 3
- SAML / enterprise SSO required from Year 2 onward
- Charlie's rate: $175/hr (midpoint)
- Auth0/Okta CIC: Enterprise tier required from Year 2 (SAML)
- Build cost amortized: Phase 1 in Year 1, Phase 2 in Year 2

| Cost Category | Auth0 (3-Year) | AWS Cognito (3-Year) | Build (3-Year) |
|---|---|---|---|
| **Year 1 — 10K MAU, no SAML** | | | |
| License / Build | $7,800 | $6,000 | $70,000 (Phase 1 dev) |
| Infrastructure | Included | Included | $5,500 |
| Maintenance | Included | Included | $0 (first year, dev team owns it) |
| **Year 1 Total** | **$7,800** | **$6,000** | **$75,500** |
| **Year 2 — 50K MAU, SAML enabled** | | | |
| License / Build | $96,000 (Enterprise) | $30,000 | $25,200 (Phase 2 dev, 144 hrs) |
| Infrastructure | Included | Included | $6,500 |
| Maintenance | Included | Included | $30,000 |
| **Year 2 Total** | **$96,000** | **$30,000** | **$61,700** |
| **Year 3 — 100K MAU, SAML, full features** | | | |
| License / Build | $144,000+ (Enterprise) | $63,000 | $0 (no new build) |
| Infrastructure | Included | Included | $7,500 |
| Maintenance | Included | Included | $33,000 |
| **Year 3 Total** | **$144,000+** | **$63,000** | **$40,500** |
| | | | |
| **3-Year TCO** | **$247,800+** | **$99,000** | **$177,700** |
| **Break-even vs. Auth0** | — | — | **~Month 20** |
| **Break-even vs. Cognito** | — | — | **~Month 36** |

**Key observations:**
1. Auth0 becomes dramatically more expensive when SAML is needed — and it will be needed for enterprise pharma customers.
2. Cognito is cheap on paper but cannot serve as a SAML IdP outbound (critical Phase 2 gap). Cognito would require building SAML separately anyway, making the "build" costs comparable while retaining all the Cognito limitations.
3. The build option has the highest Year 1 cost (development) but the lowest Year 3 cost. Break-even against Auth0 occurs around Month 20.
4. The build option is the correct long-term choice given the SAML requirement, multi-tenancy complexity, and the need for complete control over token claims and security policy.

---

## 5. Risks

### RISK-IDP-001 — Security Vulnerability in Custom Implementation
**Probability: Medium | Impact: Critical**

Identity is the highest-risk system to build from scratch. A flaw in the OAuth 2.0 flow (e.g., missing state parameter validation), JWT implementation (e.g., `alg=none` vulnerability, weak secret), or session management (e.g., session fixation) can compromise all user accounts simultaneously. This is not theoretical — numerous high-profile breaches have occurred from exactly these mistakes.

**Mitigation:**
- Use established, audited libraries for all cryptographic primitives (no custom JWT parsing, no custom TOTP math)
- Mandatory third-party penetration test before launch
- Code review by a senior security engineer (not Charlie reviewing his own auth code)
- Follow OWASP ASVS Level 3 for all auth/crypto controls
- Bug bounty program for the IdP endpoints post-launch

### RISK-IDP-002 — Development Timeline Underestimate
**Probability: High | Impact: High**

400 hours is a reasonable estimate for Phase 1. It is also the number that is most likely to grow. OAuth 2.0 and OIDC are well-specified, but implementation details — edge cases in PKCE, token rotation replay detection, TOTP clock drift, concurrent session tracking — take longer than expected. Security review often surfaces issues requiring redesign.

**Mitigation:**
- Phase gating: core login + JWT must be done before MFA, MFA before RBAC features, RBAC before API keys
- Do not parallelize IdP development with mobile app development — the IdP must be stable before the app integrates
- Add 25% contingency to timeline estimates (400 hours → plan for 500 hours)
- Do not ship MedSales platform features that depend on IdP until IdP passes security review

### RISK-IDP-003 — Maintenance Burden Grows Over Time
**Probability: High | Impact: Medium**

The IdP is not a feature you build and forget. Security patches to crypto libraries require fast response (hours, not weeks). Token signing key rotation must happen quarterly. Compliance requirements (SOC 2) will generate IdP-specific controls. As enterprise customers integrate SAML, each new customer may have unique SP configuration quirks.

**Mitigation:**
- Budget 16 hours/month ongoing maintenance explicitly in the development budget
- Instrument the IdP with comprehensive monitoring and alerting
- Document the key rotation procedure before the first rotation
- Consider dedicating DevOps capacity to IdP operations once SAML is live

### RISK-IDP-004 — Regulatory Compliance Scope Creep
**Probability: Medium | Impact: Medium**

Enterprise pharma customers are regulated. Some will require: SOC 2 Type II audit coverage for the IdP, HIPAA-ready configuration even though PHI is not currently processed, evidence of pen test results, and contractual SLAs on auth uptime. These requirements arrive unexpectedly and retroactively.

**Mitigation:**
- Design the IdP with SOC 2 controls in mind from the start (audit logging, access controls, encryption)
- Keep the IdP architecture documentation current — SOC 2 auditors will ask for it
- Do not promise compliance certifications to enterprise customers until the audit is complete

### RISK-IDP-005 — Library Deprecation / Ecosystem Changes
**Probability: Low | Impact: Medium**

The OAuth 2.0 and OIDC specifications have been stable for years. However, underlying library ecosystems change: `jsonwebtoken` had a critical vulnerability in 2022 (CVE-2022-23529), `passport-oauth2` has had breaking changes. Dependency on a poorly-maintained library creates long-term risk.

**Mitigation:**
- Choose mature, actively maintained libraries with broad community adoption
- Pin dependency versions explicitly; review changelogs before every upgrade
- Monitor CVE databases for all production dependencies (Dependabot or Snyk)

### RISK-IDP-006 — Token Rotation Edge Cases at Scale
**Probability: Medium | Impact: Medium**

Refresh token rotation with replay detection is simple in theory and complicated in practice. Mobile apps with poor network connectivity may retry token refresh requests, causing false reuse detection and locking users out. At 50K MAU, even a 0.1% edge case rate is 50 users locked out per refresh cycle.

**Mitigation:**
- Implement a short replay detection grace window (5 seconds): if the same token is presented twice within 5 seconds, treat as retry, not reuse
- Alert on anomalous family-invalidation rates — sudden spikes indicate a bug, not attacks
- Thoroughly test mobile reconnection scenarios (airplane mode → wifi restoration → app resume)

---

## 6. Recommendation — Phased Build Approach

Sean has made the call. We build. This section defines how we do it without destroying the MedSales launch timeline.

### Phase 1 — Core IdP (Months 1–4, alongside platform MVP)
**Goal:** Ship a secure, working IdP that handles MedSales MVP authentication needs.

**Scope:**
- OAuth 2.0 Authorization Code + PKCE
- OIDC (ID tokens, UserInfo, Discovery)
- JWT access tokens (RS256) + refresh token rotation
- MFA: TOTP + email OTP (SMS deferred to Phase 1.5)
- RBAC: Rep, Manager, Admin + org isolation
- API key management (basic — for Salesforce export)
- Password policies + account lockout + rate limiting
- Session management + audit logging
- Penetration test before launch

**What Phase 1 does NOT include:**
- SAML 2.0 IdP (Phase 2)
- SMS OTP (Phase 1.5 — add after launch)
- CAPTCHA integration (Phase 1.5)
- Device Trust / "Remember this device" (Phase 1.5)
- Fine-grained permissions (Phase 3)
- Okta/Azure AD SAML federation for enterprise customers (Phase 2)

### Phase 1.5 — Hardening (Month 4–5, post-MVP launch)
**Scope:**
- SMS OTP via Twilio
- CAPTCHA (hCaptcha/Turnstile)
- IP reputation integration
- "Remember this device"
- Session management UI (view/revoke active sessions)
- Respond to pen test findings

### Phase 2 — SAML 2.0 IdP (Month 6–9)
**Goal:** Enable enterprise customers to use their corporate IdP (Azure AD, Okta, Google Workspace) to log into MedSales.

**Scope:**
- SAML 2.0 IdP implementation
- SP configuration admin UI
- JIT user provisioning
- SAML → OAuth 2.0 bridge
- SLO
- Testing against 3 enterprise SAML SPs (Azure AD, Okta, Google)

### Phase 3 — Advanced Enterprise (Month 10+)
**Scope:**
- Fine-grained permissions beyond role-based
- OAuth 2.0 Device Authorization Grant (CLI tools)
- SCIM 2.0 user provisioning (enterprise HR systems)
- SOC 2 audit coverage documentation
- Advanced threat detection / IP reputation

---

### Sequencing with Platform Development

The IdP is a prerequisite, not a parallel track. Charlie cannot build authenticated features in MedSales until the IdP's core flows (OAuth, JWT, RBAC) are stable. The recommended sequencing:

1. **Weeks 1–4:** IdP architecture, database, OAuth 2.0 + PKCE, basic user auth
2. **Weeks 5–8:** JWT, JWKS, OIDC, MFA (TOTP)
3. **Weeks 9–12:** RBAC, org isolation, API key management, audit logging
4. **Weeks 13–16:** Session management, brute-force protection, integration tests, security review
5. **Week 17:** Penetration test
6. **Week 18+:** Pen test remediation → **IdP is production-ready**
7. **Weeks 19+:** Mobile app integration + MedSales platform features resume

---

*Prepared by Dennis Reynolds, Project Manager*
*This is the plan. Every requirement is numbered. Every risk is documented. Every cost is estimated.*
*The process is correct because I designed the process.*
