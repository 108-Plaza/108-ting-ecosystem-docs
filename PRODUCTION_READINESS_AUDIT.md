# 108 Ting Ecosystem — Production-Readiness Audit

> ⚠️ **PARTIALLY SUPERSEDED (2026-06-17).** The scoreboard and per-system BLOCKER
> lists below are the **2026-06-16 snapshot** (kept as the historical "before").
> Most blockers have since **merged to `origin/main`** — the current truth is in
> [**§ Follow-up status (updated 2026-06-17)**](#follow-up-status-updated-2026-06-17--verified-against-originmain)
> at the bottom. Read that section for "is it OK *now*".

> **Status:** Snapshot audit, 2026-06-16. Read-only — no code was modified.
> **Scope:** Ecosystem-wide ("audit รวมของระบบ") — what is missing/incomplete to enter a real
> testing phase and then production, across all platforms.
> **Method:** Structural recon (CI/manifest/migration/deploy signals across all repos) + GitHub org
> state + 7 parallel read-only deep-dive agents (one per platform group), each scored against the
> same 11-dimension rubric now codified in [`PRODUCTION_READINESS_BAR.md`](PRODUCTION_READINESS_BAR.md).
> **Related:** [`ACTIVE_WORK.md`](ACTIVE_WORK.md) · [`DO_NOT_TOUCH.md`](DO_NOT_TOUCH.md) ·
> [`../ENGINEERING_CONSTITUTION.md`](../ENGINEERING_CONSTITUTION.md)

---

## Executive summary

**Good news:** several core services are *near-production in code quality* — `pos108/api` (~1,900
tests), BipByte `server` (5,684 tests + a thorough launch runbook), `Identity-Platform`
(production-grade auth crypto), `Notification-Platform` (CI/deploy/observability complete).

**Bad news:** almost everything missing is **not features — it is the operational shell around the
code**, and it is missing as a *repeating cross-system pattern*. The two most dangerous individual
findings: **AccountZing (the financial ledger) has no authentication at all**, and **Identity ships
with a deploy config that provisions no production secrets** (plaintext TOTP seeds, per-boot JWT key).

**Governance gap (root cause):** the `ENGINEERING_CONSTITUTION.md` defines *process* (git/scope/
lifecycle) but **no production-readiness bar** — there is no shared definition of "done for prod" and
no release gate. Each repo improvises. This audit's companion document
[`PRODUCTION_READINESS_BAR.md`](PRODUCTION_READINESS_BAR.md) fills that gap.

---

## Scoreboard

| System | Maturity (code) | Deployable? | #1 blocker |
|--------|-----------------|-------------|------------|
| **pos108/api** (POS core) | Near-prod | ❌ | Dockerfile copies only `src/`, not `crates/` → image build fails |
| **BipByte server** (108Zing) | Near-prod | ⚠️ | realtime-edge E5 not deployed + external creds unset + managed PG/restore |
| **Identity-Platform** | Prod-grade | ❌ | Prod secrets not provisioned (plaintext TOTP seeds, per-boot JWT key) |
| **Notification-Platform** | High | ⚠️ | Dual-SoT ownership undefined + no event outbox / no reaper |
| **Data-Platform** (Stratum) | Late-beta (behavioral) | ⚠️ | Reconciliation gate = 0 lines → money path cannot ship |
| **Media-Platform** | Code-mature | ❌ | No deploy artifacts at all (9 services) |
| **AccountZing** (ledger) | Tracer-bullet | ❌ | 🔴 No auth on the whole service — anyone can post/read/settle any tenant |
| **Payment-Gateway** | Near-prod core | ⚠️ | Single-PSP (SCB) + direction not reconciled with 108Zing payment |
| **Creator / Smart-Home / Logistics / slot-api** | Mid-flight | ⚠️ | Test depth thin + most lack a Dockerfile |
| **Smart-Farm** | Early | ⚠️ | Near-zero tests (but drives real pumps/valves) |
| **POS frontends** (admin/orders/pos/slot-front) | admin+pos near-prod; orders+slot-front thin | ⚠️ | slot-front: 0 tests + CI has no test gate |
| **engines** (image-proc) | Medium | ⚠️ | Hardening still design-stage; thin tests |

---

## Cross-cutting systemic gaps (fix once → helps everywhere — highest leverage)

1. **No central production-readiness bar** `[governance]` — no checklist / release gate. Quality
   swings widely between repos. **Fix:** adopt [`PRODUCTION_READINESS_BAR.md`](PRODUCTION_READINESS_BAR.md)
   as the shared gate before measuring everything else consistently.
2. **Deploy artifacts broken/missing — the most common BLOCKER.**
   - `pos108/api`: `Dockerfile` copies only `src/`, omits `crates/` (41 path-dep members) → release build fails in-image.
   - `Media`: no Dockerfile/compose/k8s anywhere (9 services).
   - `AccountZing`: no artifacts at all.
   - BipByte `server`: Dockerfile builds the **debug** profile (no `--release`).
   - `Identity`: Dockerfile runs as root, no `HEALTHCHECK`.
   - `slot-api`, `Smart-Home`, `Smart-Farm`, `Logistics`: no Dockerfile.
   - **Almost no IaC (k8s/Helm)** except `Notification` and `Creator`.
3. **No supply-chain / dependency-vuln scanning on the most sensitive services** `[HIGH]` — missing in
   `pos108/api`, `Identity`, `AccountZing`, `Media`, BipByte `server` (all the auth/finance/POS code).
   Present in `Notification` (cargo-deny + Dependabot + SBOM), `Data`, `Payment`, `Logistics`,
   `Smart-Home`. **Fix:** standardize `cargo audit`/`cargo deny` + Dependabot in CI.
4. **Observability holes on the sync/financial-critical path** `[HIGH]` —
   - `pos108/api`: no metrics/Prometheus, no OTEL, no error tracking — offline-first sync with no view of outbox backlog / dead-letters / sync lag.
   - `AccountZing`: only `/healthz`; no reconciliation job; no imbalance alerting.
   - `Data-Platform`: Kafka high-watermark is fetched then **discarded** → no consumer-lag metric; no dead-letter sink; run-history is in-memory (lost on restart).
5. **CI does not run DB/integration tests** `[HIGH]` — `Data-Platform` and `Media` have no Postgres
   service in CI, so all `#[ignore]`-gated adapter/integration tests **never run**; a regression in
   offset durability (Data) or DB adapters (Media) passes CI silently.
6. **Secrets management** `[HIGH → BLOCKER per-service]` — no Vault/KMS anywhere (raw env only);
   Identity defaults fail **open** (CORS permissive, rate-limiter falls open if Redis is down);
   `pos108/api` default `hmac_secret` is not actually rejected despite YAML comments claiming it is.
7. **No cross-service e2e** — each repo tests itself; the seams POS↔AccountZing↔Data↔Identity↔monolith
   are manual smoke at best. No automated contract/integration test across service boundaries.

---

## Per-system BLOCKERS (must fix individually — security/correctness)

1. **🔴 AccountZing — financial ledger has no auth.** `POST /v1/events` and every admin endpoint
   (settlement pay, SLA config, appeal decisions) are open; `tenant_id` comes from the request body
   untrusted → any caller can post journals or read/settle any tenant's books. SPEC §13 RBAC is
   deferred to an unbuilt slice. **Do not place on a shared test environment until RBAC + token-bound
   tenant isolation exist.** (`ingestion/http.rs`, `contract.rs`.)
2. **🔴 Identity — prod secrets not provisioned.** Shipped `docker-compose.yml`/`.env.example` set
   none of `AUTH_JWT_SIGNING_KEY`, `AUTH_OTP_PEPPER`, `AUTH_TOTP_ENC_KEY`, `SESSION_TOKEN_HASH_SECRET`,
   `AUTH_FAIL_CLOSED` → per-boot JWT key (tokens die on restart, JWKS diverges across nodes), plaintext
   TOTP seeds + Google secret, weak refresh-token hasher. **Plus `iss`/`aud` drift**
   (`identity-platform` vs the locked contract `auth-service`/`tixtox`) → tokens rejected at the
   108Zing gateway during cutover.
3. **🔴 pos108/api — broken Dockerfile** → no deployable artifact (kills every CI-image/compose path).
4. **🔴 Media — no deploy artifacts** for any of the 9 services.
5. **🔴 Data-Platform — reconciliation gate unbuilt** (zero `reconcil*` code). Owner-locked invariant:
   money Gold must balance against an independent source before publish → no financial domain can go
   to production until built. (Behavioral/108Zing path is canary-proven and much closer.)
6. **🟠 AccountZing — scheduler is dead code.** `run_refund_scheduler` + appeal time-transitions exist
   and are tested but are **never wired into `main.rs`** → refunds never auto-advance in a running
   service.
7. **🟠 BipByte launch set** — realtime-edge E5 not deployed (no edge = no realtime fan-out);
   external creds (SendGrid/Twilio/FCM/SCB) unset & un-live-verified; managed Postgres + a
   *rehearsed* restore drill; S3/CDN not wired (`/health/ready` probes DB only, so broken S3 is
   invisible until first upload).
8. **🟠 Notification — dual-SoT ownership undefined** + no transactional outbox (lifecycle events are
   fire-and-forget → lost on broker/process failure) + no reaper for crash-orphaned `processing` rows
   (those notifications never redeliver).

---

## Recommended phased roadmap (testing → production)

- **Phase 0 — Define the bar + safety gates** (mostly doc + CI templates): adopt
  [`PRODUCTION_READINESS_BAR.md`](PRODUCTION_READINESS_BAR.md); ship reusable CI templates for
  `cargo audit`/`cargo deny`, a Postgres service container, and a standard multi-stage Dockerfile +
  `HEALTHCHECK`. *Unblocks consistent measurement of everything below.*
- **Phase 1 — Make each core service deployable & safe to test:** close blockers #1–#4
  (AccountZing auth, Identity secrets + `iss`/`aud`, pos108/api Dockerfile, Media artifacts).
- **Phase 2 — Wire the seams + observability:** metrics/lag/dead-letter; reconciliation gate; wire
  the AccountZing scheduler; verify the POS108↔AccountZing contract end-to-end.
- **Phase 3 — Cross-service e2e environment + load/security testing.**
- **Phase 4 — BipByte launch specifics:** edge E5, external creds, managed PG + restore drill, S3/CDN.

---

## Discrepancies vs memory/docs — confirm with owner before relying on them

1. **`pos108/api`: the `accounzing.shift_summary_emit` flag + shift-summary producer adapter
   ("shipped dark, #299") is NOT present in `main` at HEAD `cad6925`** (grep over `src/`, `crates/`,
   `configuration/` = 0 hits). Contradicts the recorded note that it shipped. Was it reverted / never
   merged / on another branch?
2. **BipByte outbox at-least-once bug is FIXED (#279), not open** — `kafka_publisher.rs` propagates
   `Err` via `map_send_result`. `docs/BUG_realtime_outbox_at_least_once.md` is stale and should be
   marked RESOLVED.
3. **Notification dual-SoT — the two sides disagree.** The monolith side considers it resolved via
   per-channel routing flags (`NOTIFY_CHANNEL_BACKEND_*`); the standalone platform still sees an
   unowned two-system split. Reconcile into one cutover decision.
4. **AccountZing vs AccounZing** naming drift — code/repo say `AccountZing`, committed SPEC says
   `AccounZing`. Cosmetic, but pick one.

---

## Notes on method / confidence

- All findings are from a read-only pass; a few `cargo check` runs were blocked by host-wide cargo
  lock contention (many concurrent builds) and lean on green CI history + static inspection — flagged
  per-repo in the source agent reports.
- The host Rust shim is broken (`~/.cargo/bin/cargo` dangling); builds were run via
  `~/.rustup/toolchains/stable-aarch64-apple-darwin/bin`.
- Severity legend: **BLOCKER** (cannot test/ship safely until fixed) · **HIGH** (must fix before
  production traffic) · **MED/LOW** (hardening).

---

## Follow-up status (updated 2026-06-17) — verified against origin/main

Re-verified by `git fetch` + reading `origin/main` on every repo below (local ==
origin/main for all; AccountZing local was 1 commit behind = its #9). The
2026-06-16 snapshot said "branches pushed, PRs pending" — **those PRs are now
merged.** Net: the ecosystem moved from "no service clears GATE-T" toward "most
core BLOCKERs cleared; ~3 real items remain."

### BLOCKERs now CLEARED on `origin/main`

| Audit BLOCKER | Resolution (merged) |
|---|---|
| 🔴 #1 AccountZing no-auth | ✅ EdDSA JWT verify + RBAC + token-bound tenant on all `/v1/*` — **#8** |
| 🟠 #6 AccountZing dead scheduler | ✅ `run_refund_scheduler` wired into entrypoint — **#7** |
| 🔴 #2 Identity secrets + iss/aud | ✅ iss/aud aligned to contract + prod-secrets required — **#8** (+ sqlx RUSTSEC **#12**) |
| 🔴 #3 pos108/api Dockerfile | ✅ `COPY crates` — **#302** |
| 🔴 #4 Media deploy artifacts | ✅ deploy artifacts — **#3**; + CI DB-adapter tests on Postgres — **#9**; RUSTSEC TLS — **#10/#11** |
| 🟠 #8 Notification durability | ✅ transactional outbox — **#56** + crash-orphan reaper — **#55**; SoT = Option A (see `DECISION-notification-sot-cutover.md`) |
| Payment audit follow-ups | ✅ C1–C3 + H1–H7 — **#12**; SSRF/idempotency/CAS/scope/claim — **#13/#15**; metrics+outbox+Docker — **#9** |
| Cross-cutting #3 supply-chain scan | ✅ cargo-deny + Dependabot merged in pos108 (**#304**), Identity (**#9**), Media (**#4/#8**), AccountZing (**#9**), Payment |
| Cross-cutting #4 observability (POS) | ✅ pos108/api Prometheus `/metrics` w/ outbox-backlog + dead-letter gauges — **#310** |
| Cross-cutting #5 CI DB tests | ✅ Media runs DB-adapter integration tests vs Postgres in CI — **#9** (Data-Platform still pending) |

### Still OPEN (the real remaining work)

| Item | Severity | Status / why |
|---|---|---|
| **Data-Platform reconciliation gate** (#5) | 🔴 BLOCKER (in progress) | **Slice 0 (durability pre-check) merged** — Data-Platform #50 (design) + #51 (`recon::DurabilityReport` fail-closed row-count gate + `RECONCILIATION_BREAKS` metric). The **value** reconciliation (A-intra/A-ext) is still unbuilt: needs Q-REC-1..5, a pinned AccountZing read contract (no client today), and the money-type call (`Float64` today). Money-Gold path still cannot ship until that lands. See `DECISION-data-reconciliation-gate.md`. |
| **AccountZing deploy artifacts** | 🔴 BLOCKER | **Still none (verified: 0 Dockerfile/compose on origin/main, only `dependabot.yml`).** Auth/scheduler done but the service has no deployable image. |
| **BipByte launch set** (#7) | 🟠 | Ops/creds — edge E5, SendGrid/Twilio/FCM/SCB creds, managed PG + restore drill, S3/CDN. Item 5 (S3 readiness probe) is codeable. `DECISION-bipbyte-launch-readiness.md`. |
| **Notification cutover** | 🟡 | Code complete; remaining = the B1.6 operational runbook (flip email→push, soak). Ops-owned. |
| **Cross-service e2e + secrets manager** | 🟡 | No automated cross-seam e2e yet; still raw-env secrets (Vault/KMS) — Identity now *requires* prod secrets but no central manager. |

**Residual (tracked):** AccountZing by-id reads (settlement/appeals/sla) are
auth-gated but not yet tenant-filtered at the repo layer.

### Discrepancies — resolved
1. **`shift_summary` flag/producer "missing from main" → FALSE ALARM.** The grep
   ran on a stale working tree. On `origin/main` (`804e5dc`) the producer is
   present (`crates/pos-shift/src/application/shift/accounzing_summary.rs`,
   `gl_async_flags.rs`, shipped via #299). The "shipped dark" note was correct.
2. **BipByte outbox at-least-once → confirmed FIXED (#279).** `kafka_publisher.rs`
   propagates `Err` via `map_send_result`. `docs/BUG_realtime_outbox_at_least_once.md`
   now carries a RESOLVED banner (`docs/resolve-outbox-bug`).
