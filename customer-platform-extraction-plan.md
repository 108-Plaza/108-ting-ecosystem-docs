# Customer-Platform Extraction Plan

> **Status:** DRAFT / doc-only — owner approved offline model **A** (central + cloud→branch mirror),
> requested detailed plan before any scaffold (2026-06-28). No code written yet.
> **SoT** for this plan until the service repo exists; then move to `customer-plat/docs/`.
> Sibling pattern: [loyalty-service-extraction], [payroll-service-extraction], AccountZing.
>
> **Owner decisions locked (2026-06-28):** repo name = **`customer-plat`** (under `108-Plaza`);
> business key = **phone-first** — `(tenant_id, phone)` is the customer business key, unified with
> Loyalty-Platform's key. Open follow-up: customers with no phone (see §7).

---

## 1. Goal & Scope

Extract the **customer bounded context** out of the POS108 monolith
(`Commerce-Platform/pos108/api`, crate `pos-customer`) into a **standalone central
`Customer-Platform` service** that owns the master customer registry for the whole
ecosystem (POS108, shop108, BipByte, Loyalty).

**In scope:** customer master data — `id, tenant_id, code, name, phone, email, credit_limit`,
soft-delete, CRUD API, multi-tenant isolation.

**Out of scope (stays where it is):**
- Loyalty points / accrual / redemption → already its own `Loyalty-Platform` ([loyalty-service-extraction]).
  `current_points` is a *projection* loyalty owns; Customer-Platform does NOT store points.
- Accounts-receivable balances → stay in pos108 finance (only the `customer_id` ref crosses).
- Bottle-keep inventory → stays in pos108 (only the `customer_id` ref crosses).
- Sales/orders → stay in pos108 (`customer_id` already a soft column, no FK).

---

## 2. The offline-first constraint (why customer ≠ loyalty)

Owner-locked baseline ([pos108-offline-first-data-platform-baseline]) classifies
`pos-customer` as **tier A (Transactional / sell-path)** and lists **"customers" in the
cloud→branch sync set already**. Unlike loyalty (zero branch coupling, redemption
online-only), customer is **read on the offline sell path**:

- attach a customer to a sale (`sales.customer_id`) — happens offline
- open accounts-receivable (`accounts_receivable.customer_id`) — happens offline
- bottle-keep (`bottle_keeps.customer_id`) — happens offline

**Invariant #6** forbids querying any central/external system during open-bill / payment /
refund. Therefore a *pure online* customer service (loyalty-style) would break offline selling.

### Chosen model: **A — Central registry + cloud→branch replicated mirror**

- Customer-Platform is the **system of record** (master registry, cloud).
- Each branch keeps a **local read mirror** of customers (it already does today — customers
  are in the cloud→branch pull set). The offline sell path reads the **local mirror**, never
  the central service → air-gapped branches keep selling.
- Writes (create/edit customer): support **both** paths —
  - cloud/online write → propagates cloud→branch via existing sync pull;
  - branch offline write → buffered in branch outbox → branch→cloud sync → central registry
    applies upsert-if-newer.
- Conflict resolution: **upsert-if-newer keyed on `(id, version)`** (same rule the DP bridge
  already uses). `code` uniqueness conflicts across offline branches need a tie-break rule
  (see Open Decisions §7).

This preserves all 7 offline-first invariants: critical path touches only local DB; the
central service is reached only by background sync workers, never inline.

---

## 3. Current coupling map (verified 2026-06-28)

### 3a. `pos-customer` crate — CLEAN, near-verbatim lift
- `crates/pos-customer/Cargo.toml` deps: only `pos-kernel` + `pos-error`. No workspace crate
  depends on `pos-customer`.
- Domain already has a repository **trait** + commands + entity (no `pos_infra` in domain).
- Binary consumers (only 3): `src/presentation/http/customer.rs`,
  `src/presentation/dto/customer_dto.rs`, DI in `src/bootstrap/container.rs`.

### 3b. Inbound references to `customers` (cross-boundary)
| Referencer | Type | Action at cutover |
|---|---|---|
| `loyalty_accounts.customer_id` | FK ON DELETE RESTRICT | Loyalty already re-keyed by phone + dropped FK in its own DB → no action in Customer-Platform |
| `accounts_receivable.customer_id` | FK | pos108: drop FK → soft `customer_id` ref |
| `bottle_keeps.customer_id` | FK ON DELETE RESTRICT (migration `20260628000000`) | pos108: drop FK → soft ref |
| `sales.customer_id` | column, no FK | already soft ✅ |
| `orders.customer_id` | column, no FK | already soft ✅ |
| reporting / analytics | — | never JOINs customers ✅ no action |

### 3c. The one read-path enrichment
`PgCustomerRepository` LEFT JOINs `loyalty_accounts` to fill `current_points`. After split,
`current_points` is NOT Customer-Platform's data. Options: (a) drop it from the customer
response and have the client call Loyalty-Platform; (b) Customer-Platform calls Loyalty as an
online enrichment (online-only, not on sell path). **Recommend (a)** — keeps boundary clean.

---

## 4. Slice plan (proven Loyalty/Payroll cadence)

Each slice = local-green gate (fmt + clippy -D + check + tests) before push; owner-gated merge.

- **S1 — Scaffold** (genesis): Cargo workspace (`customer` binary + verbatim `crates/pos-kernel`
  + `crates/pos-error`); `crate::shared::*` shim; lift `domain` verbatim (entities/commands/
  repository-trait); `/health` tracer; CI/deny/env/README/`.ai_context` + `docs/EXTRACTION_PLAN.md`
  (move this doc there). Pin toolchain 1.96.0. Gate: domain unit tests.
- **S2 — Infra + own DB**: lift `PgCustomerRepository` verbatim (runtime sqlx, no `query!` macro);
  migration `init_customer` = `customers` table, **drop FK→tenants → soft `tenant_id`**; **swap the
  unique key to partial `UNIQUE (tenant, phone) WHERE phone <> '12345'` (phone-first; `12345` = the
  no-phone temp placeholder)** with normalized-phone column, `code` demoted to a non-unique alias;
  add `version` column for upsert-if-newer; drop the `loyalty_accounts` LEFT JOIN
  (current_points removed). Add `find_by_phone`. DB integration test (skips w/o `TEST_DATABASE_URL`);
  CI `postgres:16`. *(Repo currently keys/looks-up by `code` → this is the one non-verbatim change.)*
- **S3 — HTTP + auth**: lift `http/customer.rs` scope + `customer_dto` verbatim; Identity EdDSA
  `AuthCtx` (`customers:read` / `customers:write`, audience `customer`, `AUTH_ENABLED=false`
  dev hatch) — copy from Loyalty/Payroll. End-to-end HTTP test (boot dev-open → health 200 →
  create 201 → list 200).
- **S4 — Sync seam (the model-A core)**: transactional outbox; emit `customer.upserted.v1` /
  `customer.deleted.v1`; **one-way cloud→branch** replication contract so pos108 mirror applies
  upsert-if-newer on `(id, version)`. **This is the slice that makes offline-first hold.** Per
  decision §7.5 (online-only create) the flow is one-directional — **no branch→cloud buffer, no
  phone-collision merge** — so this slice is simpler than the loyalty accrual consumer.
- **S5 — pos108 cutover** (separate gated PR into pos108): change `accounts_receivable` +
  `bottle_keeps` FKs → soft refs; repoint customer client to read local mirror / write via sync;
  keep the local mirror table + sync applier; remove `pos-customer` write authority from the binary.
  **Data Risk + schema change → hard owner gate.**
- **S6 — Deploy**: Dockerfile + ting-service Helm chart + `DEPLOYMENT_ALLOCATION` entry
  (per [k3s-deployment-standard]).

---

## 5. ADR draft — Customer offline contract

**ADR-0001 (Customer-Platform): Offline replication contract**
- **Context:** customer is read on the pos108 offline sell path; baseline forbids inline central
  reads during sell/charge/refund; customers already replicate cloud→branch.
- **Decision:** Customer-Platform is the master registry. Branches keep a synced local mirror and
  read it offline. Writes are bidirectional (cloud→branch pull + branch→cloud buffered push),
  reconciled by **upsert-if-newer on `(id, version)`**. The central service is never on the
  critical path — only background sync workers reach it.
- **Consequences:** air-gapped branches keep full customer read/attach capability; eventual
  consistency on edits; `code` collision across offline branches needs a tie-break (§7);
  `current_points` leaves the customer payload (Loyalty owns it).

**ADR-0002 (Customer-Platform): Identity & customer key** — DECIDED phone-first
- Service auth via central Identity EdDSA bearer (audience `customer`). Customer **business key =
  `(tenant_id, phone)`** (phone-first), unified with Loyalty-Platform (keys `(tenant, phone)`) →
  one membership identity across the ecosystem. `code` demoted to an optional human-facing alias
  attribute, no longer the uniqueness key. `id` (UUID) remains the technical PK / mirror sync key.
- Consequence: phone normalization is now **mandatory** (was a gap) — normalize to canonical
  digits before the unique key. **No-phone-yet customers** use a TEMP value `12345` that is
  overwritten by the real phone once captured (transient sentinel, not a permanent identity); the
  `(tenant, phone)` unique index is therefore **partial — `WHERE phone <> '12345'`** so many temp
  rows coexist (keyed by UUID), uniqueness bites only for real phones.

---

## 6. Known gotchas (from Loyalty/Payroll)

- **data-exfil classifier** blocks `git push`/`git remote add` of a verbatim tree lifted from
  private pos108 → **owner must create empty repo + push manually** after confirming exact repo name.
- **Never leave design docs untracked in pos108** — concurrent git activity wipes them. SoT lives
  in the service repo (or this umbrella `docs/` until the repo exists).
- Host cargo shim broken → run via `~/.rustup/toolchains/stable-aarch64-apple-darwin/bin`.
- Stacked-PR squash-merge cascade gotcha (see [payroll-service-extraction]) — prefer per-slice
  branch off main, or merge bottom-up with `--merge`.

---

## 7. Open decisions (need owner before / during build)

1. ~~Repo name~~ — **DECIDED: `customer-plat`** (under `108-Plaza`; classifier needs it explicit
   before owner pushes).
2. ~~Business key~~ — **DECIDED: phone-first `(tenant, phone)`**, unified with Loyalty.
3. ~~No-phone customers~~ — **DECIDED: `12345` is a TEMP value**, not a permanent key. When a
   customer is created without a phone yet, phone = `12345` temporarily, and it is **overwritten by
   the real phone** once captured. So `12345` is a transient sentinel, never a long-lived identity.
   ⚠️ **S2 implementation note:** because several customers can sit at temp-`12345` at the same time,
   the `(tenant, phone)` unique index must be **partial — `WHERE phone <> '12345'`** (uniqueness
   bites only real phones; temp rows are distinguished by UUID `id` until updated). Phone
   normalization (canonical digits/country code) required for real numbers.
4. **`current_points`** — drop from customer response (client calls Loyalty) vs online enrichment.
   *Plan recommends drop.*
5. ~~Offline customer creation~~ — **DECIDED: online-only create.** Air-gapped branches read the
   local mirror only; new customers are created online (cloud) and flow cloud→branch via the
   existing pull. **No branch→cloud buffer, no phone-collision merge needed** → S4 is one-way
   (cloud→branch replication) and materially simpler.
6. ~~Forbidden-Scope check~~ — **CLEARED (2026-06-28):** customer is NOT on `docs/DO_NOT_TOUCH.md`
   (forbidden = payroll/accounting/GL-posting/finance-DAG/canary/TikTok only). Like loyalty, no
   special exception required. *(Creating the new repo is still a platform-scope step → needs the
   explicit "start S1" go-ahead.)*

**→ All §7 decisions resolved. Plan is fully specified; awaiting only owner go-ahead to start S1.**

---

## 8. Next action

**All §7 decisions resolved (2026-06-28):** model A · repo `customer-plat` · phone-first key ·
no-phone temp `12345` · online-only create · Forbidden-Scope cleared. Plan is fully specified.
**Awaiting only the explicit "start S1 scaffold" go-ahead** (creating the new repo is a
platform-scope step). Until then this stays doc-only.
