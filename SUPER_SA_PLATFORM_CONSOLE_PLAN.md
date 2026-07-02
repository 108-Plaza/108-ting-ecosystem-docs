# Super-SA Platform Console — Design Plan

> **Status:** STATUS (2026-07-02) — **Phase 2 backend SHIPPED (no longer doc-only).** All six pos108
> backend PRs are merged on `main`: #492 (`31756e7`) platform-tier guards on AuthCtx, #494 (`dd5a4fb`)
> Super-SA impersonation mint, #496 (`56c6bf5`) delivery-scope shadowing fix, #498 (`3b937d5`) re-gate
> control endpoints to Super-SA, #499 (`6a6fc9c`) tenant-overview endpoint, #500 (`a51c3d4`) narrow SA
> write-management, #501 (`5252503`) short dedicated impersonation-token TTL. The §4.3 step-0 offline
> token-validation spike is **DONE** and no longer blocks. **Remaining:** step 5b MFA/step-up,
> multi-instance registry, and the platform-console app itself — a **local-only scaffold** (repo
> `platform-console`, 3 commits, **no git remote / not on the approved-work list**; see the ⚠️ note in
> `ACTIVE_WORK.md`). §6's checklist is already up to date.
> Decisions locked: **(1)** Super-SA lives in a **separate platform console** (new
> `platform-console` repo, NOT inside `pos108-admin`); **(2)** plan-before-build;
> **(3)** §4.2 resolved — **federate via Identity-Platform** + **per-tenant
> impersonation tokens** (see §4.2 + §4.2a).
> _Original (2026-06-30): DESIGN / doc-only, no code yet._
> SoT for the platform-console design until the dedicated repo exists.

## 1. Why

Per-branch "Cloud Lease / Rental Invoices" UI was just **removed** from the
`pos108-admin` Edit Branch dialog (PR #271) because licensing moved to
**tenant-level** on the backend (#486, *tenant-level licensing package — replaces
per-branch*). Two follow-on needs:

- **(A) SA-facing "Package" page** — a tenant owner (SA) views/manages **their own
  tenant's** cloud rental package (lease + invoices). Direct replacement for the
  removed per-branch UI. **Backend already exists** → buildable now (Phase 1).
- **(B) Super-SA platform console** — a tier **above** the tenant SA that manages
  SAs **across tenants** and **issues** rental packages to them (the landlord /
  platform operator). New surface, cross-tenant → needs design (this doc).

## 2. Foundation that already exists (don't rebuild)

- **Role-level hierarchy** (`pos-auth/.../role_level.rs`):
  `PLATFORM_OWNER 100` (global, intentionally unreachable from inside) >
  `SUPER_ADMIN 90` (global, cross-tenant admin) > `TENANT_OWNER 80` (= today's
  `admin`/SA) > `TENANT_ADMIN 70` > `BRANCH_ADMIN 60` > `MANAGER 40` > `STAFF 20`
  > `CASHIER 10`. `RoleLevel::outranks()` already enforces the ladder.
  → **"Super SA above SA" = SUPER_ADMIN(90) over TENANT_OWNER(80). Concept exists.**
- **Role scope** (`role_scope.rs`): `Global` scope is **reserved for PLATFORM_OWNER
  & SUPER_ADMIN**; **custom roles cannot be created at Global scope from inside the
  app**. → A Super-SA **cannot** be minted through the normal Roles UI; it needs a
  dedicated provisioning path.
- **Tenant-licensing API** (#486, `presentation/http/tenant_license.rs`, scope
  `/api/v1/tenant`): `PUT/GET/DELETE /tenant/license`, `POST/GET /tenant/invoices`,
  `POST /tenant/invoices/{id}/pay`, `POST .../pay-link`. **Gated `branches:license`
  + `require_tenant_scope`** → operates on the **caller's own tenant only**.
- **Platform-level endpoints already present**: `platform_catalog.rs`,
  `platform_delivery.rs`, `bootstrap.rs` (cross-tenant / provisioning seams).
- Central auth context (memory `ecosystem-identity-strategy`): Identity-Platform is
  the single auth authority; tokens carry platform+tenant+branch; **pos108 is the
  offline-federation exception** — which is the crux of the console's auth question.

## 3. Phase 1 — SA tenant "Package" page = **VIEW + PAY only** (done)

Frontend-only in `pos108-admin`. Page `/dashboard/package` ("Cloud Rental", System
HQ group, gated `branches:license`) consuming `/api/v1/tenant/license` +
`/api/v1/tenant/invoices`.

**Owner decision (2026-06-30): the package is controlled by the Super-SA per tenant,
NOT self-served by the tenant.** So the SA page is **read-only + pay**:
- **Cloud Lease** card — READ-ONLY display: status, expiry, grace, max-branches /
  max-terminals quotas, live usage, notes. (`GET /tenant/license` only.)
- **Rental Invoices** card — list + **Pay** (per OPEN invoice → `pay-link` QR).
  NO issue-lease, NO set-quota, NO create-invoice, NO mark-paid (all = platform control).

Shipped: tenant page (#272) + per-branch removal (#271); reworked to view+pay
(`fix/tenant-package-view-pay-only`).

> **Backend gating follow-up (Phase 2):** the control endpoints — `PUT/DELETE
> /tenant/license`, `POST /tenant/invoices`, the mark-paid/settle path — are today
> gated `branches:license` + `require_tenant_scope`, so a tenant SA *could* still
> call them directly. They must move to **SUPER_ADMIN-gated platform endpoints**
> (cross-tenant, `tenantId`-parameterised). The SA keeps only `GET /tenant/license`,
> `GET /tenant/invoices`, and `pay-link`. Until that re-gate lands, the SA's *UI*
> can't issue/bill, but the API restriction is not yet enforced — track in §4.3.

## 4. Phase 2 — Super-SA platform console — backend SHIPPED (2026-07-02); console app scaffold-only

A **separate** console app (own repo/deploy), strictly gated to `SUPER_ADMIN`,
managing tenants cross-tenant. The hard parts are **provisioning** and **cross-tenant
auth** — the role tiers exist, but almost nothing cross-tenant is wired for write.

### 4.1 What it manages
- **Tenants**: list all; (optionally) create/suspend a tenant.
- **SAs per tenant**: create/reset/disable the `TENANT_OWNER` of each tenant.
- **Packages (licensing)**: issue/extend a tenant's lease + create/settle invoices
  **for any tenant** (the landlord side of #486).
- **Overview**: seats/usage, lease status, who's in grace / cut off.

### 4.2 Key design decisions (RESOLVED — owner, 2026-06-30)

1. **Super-SA provisioning** → **(c) Federate via Identity-Platform.** Super-SA is
   created in central Identity-Platform with a platform/Global claim and federated
   into pos108. **No in-app self-service** (by design). Candidates rejected: (a)
   bootstrap/CLI seed, (b) PLATFORM_OWNER-gated endpoint.
2. **Cross-tenant AuthN/AuthZ** → **(b) Per-tenant impersonation/scoped tokens.** The
   console does **not** add cross-tenant `/platform/tenants/{tenantId}/...` endpoints;
   instead it obtains a **tenant-scoped token per target tenant** and calls the
   **existing** `require_tenant_scope` endpoints as that tenant. (Doc previously
   leaned (a) platform endpoints; owner chose impersonation.)
3. **Where it physically runs** → **New repo `platform-console`** — own deploy + own
   auth boundary.
4. **Blast radius / safety** — cross-tenant write is the highest-privilege action in
   the system. Mandatory: full audit log, no destructive defaults, MFA/step-up for
   Super-SA, read-mostly with explicit confirms for issue/suspend. (Unchanged — not a
   choice; a build requirement.)
5. **Identity integration** → through **central Identity-Platform** (follows from #1).

### 4.2a Implications of the resolved decisions (read before build)

The two identity-leaning choices (#1 federate + #2 impersonate) reshape the build:

- **Central unknown is now the token-validation path.** pos108 is the **offline-
  federation exception** (memory `ecosystem-identity-strategy`): it does **not** sit
  behind central Identity at runtime. So "Identity-Platform mints a Super-SA platform
  claim + per-tenant impersonation token" only works if **pos108 can validate that
  token offline** — i.e. trust Identity's signing key / federation envelope without a
  live call. **This is the first thing to design in Phase 2; everything else waits on
  it.** (Pick: shared-trust signed token pos108 verifies locally, vs. a pos108-local
  platform credential that Identity hands off to — the latter partly walks back #5.)
- **"Re-gate" (§3 / §4.3-2) changes shape.** Original plan: move control endpoints
  onto **new `SUPER_ADMIN` platform endpoints**. Under **impersonation**, the console
  calls the **same** `require_tenant_scope` endpoints, so they stay tenant-scoped. The
  §3 self-service gap is instead closed by: **(i)** removing `branches:license`
  **write** capability from the `TENANT_OWNER` role (SA keeps only `GET` +
  `pay-link`), and **(ii)** letting **only** a Super-SA-minted impersonation token
  carry the control capability. No new cross-tenant endpoint surface to build/audit —
  but the **impersonation-mint** path becomes the highest-privilege seam and must be
  audited in its place.
- **Audit moves to the mint seam.** With no `tenantId`-parameterised platform
  endpoint, the auditable "Super-SA acted on tenant X" record must be emitted **at
  token-mint time** (who minted, for which tenant, scope, TTL), not at the pos108
  endpoint (which only sees a normal tenant-scoped call).

### 4.3 Build order (reflects the resolved impersonation/federation model)
0. **Token-validation design spike — DONE 2026-06-30 (grounded in pos108 code).**
   Findings: pos108 JWT is **HS256 symmetric, self-issued**, issuer pinned `"pos108"`,
   trusts **only tokens it minted** — no asymmetric/external-issuer path
   (`crates/pos-auth/.../jwt.rs`). pos108 is **single-tenant per instance**
   (`tenant_id` is a startup constant, not from JWT) — "cross-tenant" lives **above**
   instances, not inside one. `role_level`/`role_scope` already ride in the token +
   `TenantContext`; no `require_role_level()`/`require_global_scope()` guard exists yet.
   → **Fork (a)** (pos108 verifies an Identity-issued token at runtime) **contradicts
   the OWNER-LOCKED offline-federation invariant** + is a large lift. **Decision:
   reconciliation below** — Identity is the *human* IdP; pos108 validation stays local;
   cross-instance action via a pos108-local-credential-gated mint endpoint.

   **Reconciled architecture (honors #1/#2/#5 + the offline lock):**
   - Human Super-SA logs into `platform-console` via **Identity-Platform** (#1/#5, IdP layer).
   - Each pos108 instance keeps **self-issued HS256, offline** validation (invariant).
   - To act on a tenant, the console presents a **pos108-local platform service
     credential** to a narrow per-instance **mint endpoint**, which returns a
     **tenant-scoped token** the existing `require_tenant_scope` endpoints accept (#2
     impersonation). **Audit emitted at the mint** (who/tenant/scope/TTL).
   - ⚠ Introduces a **new pos108-local platform service credential/secret** — owner
     sign-off required before build (secret + cross-repo surface).
1. **Impersonation-mint seam** — a `SUPER_ADMIN`/platform-gated path (in Identity-
   Platform per #1/#5) that mints a **tenant-scoped token** for an explicit `tenantId`,
   carrying the control capability. Emit the **audit record here** (who/tenant/scope/TTL).
2. **Close the §3 self-service gap (re-gate, owner: sequence here — not now).** Strip
   `branches:license` **write** from the `TENANT_OWNER` role so a tenant SA can no
   longer call `PUT/DELETE /tenant/license`, `POST /tenant/invoices`, or mark-paid
   directly; SA keeps `GET /tenant/license`, `GET /tenant/invoices`, `pay-link`. The
   **control capability rides only on the Super-SA-minted impersonation token** (the
   endpoints themselves stay `require_tenant_scope`). (No new cross-tenant endpoints —
   see §4.2a.)
3. **Tenant list / SA-management read+write** for the console — reachable via
   impersonation or a thin platform read endpoint (TBD in step 0/1).
4. Separate **console app** (`platform-console` repo): auth → tenant list →
   per-tenant package + SA mgmt, calling existing endpoints via minted tokens.
5. Audit (at mint seam) + step-up auth (MFA for Super-SA).

## 5. Out of scope / non-goals
- PLATFORM_OWNER (100) self-service — intentionally unreachable from inside; stays
  bootstrap/operator-only.
- Putting cross-tenant management inside `pos108-admin` (rejected — separate console).

## 6. Immediate next steps
- [x] Owner resolves §4.2 decisions (2026-06-30): federate via Identity-Platform •
  per-tenant impersonation tokens • new `platform-console` repo • re-gate sequenced
  (not now). See §4.2 + §4.2a.
- [x] **Phase 1** SA package page on `/tenant/*` (view+pay) — shipped (#271/#272).
- [x] **§4.3 step 0 — token-validation spike DONE** (2026-06-30): pos108 = self-issued
  HS256, single-tenant-per-instance, offline-locked. Reconciled architecture in §4.3.0
  (Identity = human IdP; pos108 validation stays local; mint endpoint gated by a new
  pos108-local platform service credential). **Awaiting owner sign-off on the
  architecture** (new secret + cross-repo).
- [x] (Zero-risk, in pos108/api, fork-independent) `AuthCtx::require_role_level()` +
  `require_global_scope()` guards — the primitive every Super-SA gate needs. **PR
  108-Plaza/pos108#492** (additive, not yet wired; fmt/clippy/check+test green;
  pre-existing `cargo-deny`/"Supply chain" failure on main is unrelated — zero dep
  change). Awaiting owner merge approval.
- [x] Owner signed off on reconciled architecture (2026-06-30).
- [x] **§4.3 step 1 — mint seam (slice 1a): MERGED.** `POST
  /api/v1/platform/console/impersonate` (**PR 108-Plaza/pos108#494**, merged `dd5a4fb`).
  - **Follow-up fix MERGED (#496, `56c6bf5`):** the broad `web::scope("/api/v1/platform")`
    in `platform_delivery` was registered before the specific `/catalog` + `/console`
    scopes → actix shadowed them → both 404 on cloud (also revealed the **Data
    Platform catalog feed had been dead** the same way). Reordered specific-before-broad
    + regression test `cloud_platform_scopes_are_not_shadowed`.
  - **Staging deploy state (2026-06-30):** a concurrent session deployed `sha-e98ed85`
    (#494 + #497) — so the mint endpoint is present on staging **but still shadowed
    (404)** because **#496 is on main, not yet on `deploy/staging`**. Next
    `main→deploy/staging` deploy un-shadows it (mint → 401 dark, catalog → 401). Owner
    chose "merge only" — redeploy deferred. **Reuses the
  existing `X-Platform-Key`/`platform_credentials` mechanism** → the "new pos108-local
  platform service credential" needs **no new secret and no migration** (the
  Data-Risk gate dissolved — `issued_by` is a plain uuid, no FK). Gated to one
  `system_code=PLATFORM_CONSOLE` (catalog/delivery keys can't mint), **dark** until
  that row is registered, cloud-only, mints a tenant-scoped `SUPER_ADMIN`/`Global`
  token (minimal perms, short TTL), audit-logged at the mint. All CI green
  (fmt/clippy/check+test/cargo-deny). Awaiting owner merge approval.
  - **Ops to un-dark:** owner registers a `platform_credentials` row
    `system_code='PLATFORM_CONSOLE'` and hands the raw key to the console.
- [x] **§4.3 step 2 — re-gate `/tenant/*` control endpoints: MERGED + DEPLOYED**
  (**PR 108-Plaza/pos108#498**, `3b937d5`). `require_platform_control`
  (`require_role_level(SUPER_ADMIN)`+`require_global_scope`) on the 4 control handlers
  (set/clear license, create invoice, pay) → a tenant SA now gets 403; SA keeps
  GET + pay-link. Endpoint-level (no role-seeding change). **Deployed to staging
  `sha-6717100`** (with #496) — verified: `console/impersonate`→401 (un-shadowed,
  dark), `catalog/replay`→401 (feed revived), health green.
  - ⚠ Contract: tightens gating — owner to confirm `pos108-admin` SA page never
    calls a control endpoint directly (plan says it's already view+pay-only #271/#272).
- [x] **§4.3 step 3 (slice 3a) — console overview: MERGED + DEPLOYED**
  (**PR 108-Plaza/pos108#499**, `6a6fc9c`; staging `sha-b2333c3`, `/overview`→401 dark).
  `GET /api/v1/platform/console/overview` — read-only per-tenant snapshot (tenant id,
  lease+usage, derived lease status, SA accounts). Module renamed
  `platform_impersonate`→`platform_console` (one scope for `/impersonate`+`/overview`).
  - [x] **slice 3b — SA write-management: MERGED + DEPLOYED** (**PR
    108-Plaza/pos108#500**, `a51c3d4`; staging `sha-936ee6c`, `/sa` valid-body+no-key
    →401 dark). Owner chose **narrow dedicated endpoints** (not widening the minted
    token): `POST /sa`, `POST /sa/{id}/reset-password`, `PATCH /sa/{id}/status` — reuse
    the audited `auth_service`, bounded TENANT_OWNER-only (404 on non-SA, no
    enumeration), create assigns the system TENANT_OWNER role + must_change_password,
    disable refuses the last active SA.
- [~] **§4.3 step 4 — `platform-console` app: LOCAL TRACER SCAFFOLD DONE.**
  `Platform-Services/platform-console` (SolidStart SSR, mirrors the storefront;
  local-only git repo `8249f46`, **no org repo — outward-facing, owner decides**).
  BFF: human auth via Identity-Platform → encrypted httpOnly server session;
  pos108 reached with the **server-held `PLATFORM_CONSOLE` key** (browser never sees
  it). Tracer: login → tenant overview (lease + usage + SA accounts) + SA
  create/enable-disable/reset-password, wired to the real `/api/v1/platform/console/*`.
  Verified: typecheck + SSR build + runtime smoke (`/`→302 login). Dark-aware.
- [x] **§4.3 step 5a — human Super-SA authorization: DONE** (console `84dbe0c`).
    `requireSuperSa` gates every console query/action: logged in AND the Identity
    token's `roles[]` holds `SUPERSA_REQUIRED_ROLE` (default `platform:super_sa`,
    configurable to match what the owner grants via Identity `/rbac/roles`) →
    else `/login` or `/forbidden`. Closes the tracer's top gap (was: any Identity
    login + server key = access). Two gates now — machine (platform key) + human
    (role). Verified typecheck/build/smoke.
- [x] **Console licensing UI: DONE** (console `28c9837`). BFF mints a SUPER_ADMIN
    token per request (`/impersonate`) and drives the re-gated `/tenant/*`: issue/
    renew/clear lease, list/raise/pay invoices. UI on the overview page, all
    `requireSuperSa`-gated. Verified typecheck/build/smoke.
- [ ] **§4.3 step 5b — step-up auth (MFA)** for Super-SA (Identity MFA endpoints).
- [ ] Console remaining: multi-instance registry (today one URL = one tenant),
    deploy + org repo (outward-facing — owner). See console HANDOFF.md.

**Status (2026-06-30):** Phase 2 backend COMPLETE + on staging; console does
login→overview→SA-mgmt→licensing with human Super-SA authz. Left: MFA,
multi-instance, deploy.

**✅ END-TO-END PROVEN on staging (2026-06-30):** a `PLATFORM_CONSOLE` credential
was registered and the full backend chain validated live — **11/11 PASS**:
overview → impersonate → set lease → create+pay invoice → create SA →
disable/enable → reset-password, plus the two security checks (no-token→401
re-gate, non-SA→404 blast-radius guard). All 6 backend PRs verified against a real
instance.
- **Finding → FIXED + DEPLOYED** (#501, `5252503`; staging `sha-35fde1a`): the
  minted impersonation token reused the 24h instance access TTL. Added
  `JwtService::issue_with_ttl`; mint now uses a dedicated `IMPERSONATION_TTL_SECS
  = 600` (10 min). Verified live: `impersonate` → `expiresInSecs = 600`.
- Console endpoints are now LIVE (not dark) on staging behind that key. Test data
  left: a lease (+1yr, maxBranches=5), one paid invoice, SA `e2e_sa_13195`.
- [ ] **Ops to activate end-to-end:** owner registers a `platform_credentials` row
  `system_code='PLATFORM_CONSOLE'` (un-darks the mint) + builds the console (step 4).
