# 108 Ting Ecosystem — Production-Readiness Bar

> **Status:** Proposed standard, 2026-06-16. The missing "definition of done for production" that the
> [`PRODUCTION_READINESS_AUDIT.md`](PRODUCTION_READINESS_AUDIT.md) identified as the #1 systemic gap.
> **Purpose:** one shared, repo-agnostic gate every service passes before it enters a shared test
> environment and again before production. Complements (does not replace) the process rules in
> [`../ENGINEERING_CONSTITUTION.md`](../ENGINEERING_CONSTITUTION.md).
> **How to use:** copy the per-service sign-off block (bottom) into the service's `.ai_context`/
> `HANDOFF.md`, tick each item with evidence (file path / CI link), and record the gate decision.

---

## Gate levels

- **GATE-T (Testing entry):** the minimum to safely run the service in a *shared* test/staging
  environment without leaking secrets, corrupting shared data, or going un-deployable. **All
  BLOCKER-tagged items must pass.**
- **GATE-P (Production):** everything in GATE-T plus all HIGH-tagged items, plus a load/security pass.

A service may sit at GATE-T for a while; it may not skip to GATE-P.

---

## The 11 dimensions

Each item: tag = the gate it belongs to. `[B]` = blocks GATE-T, `[H]` = blocks GATE-P, `[R]` = recommended.

### 1. Build & compile
- `[B]` `cargo check`/`build` (or `tsc`/`flutter analyze`) is clean on a fresh checkout; no disabled/broken workspace members.
- `[H]` Release build is reproducible **in the deploy image** (not just locally).
- `[R]` Warnings tracked; clippy `-D warnings` enforced.

### 2. Tests
- `[B]` Unit tests cover core domain logic; the suite runs green in CI.
- `[B]` **Integration/DB tests actually execute in CI** (a Postgres/Redis service container exists — `#[ignore]`d-but-never-run does not count).
- `[H]` Critical flows have e2e coverage; financial/control systems have invariant/property tests.
- `[R]` Coverage trend tracked; crash-recovery / idempotency-under-failure tested.

### 3. CI/CD
- `[B]` CI gates PRs: fmt + lint (`-D warnings`) + test.
- `[H]` A build→publish→deploy path exists (image tagged & pushed; migrations applied on deploy) — not a manual ritual.
- `[R]` Coverage gate; SBOM on release.

### 4. Configuration & secrets
- `[B]` No secrets/credentials committed (verify `.gitignore`; `.env` files hold dev placeholders only).
- `[B]` Required config is documented (`.env.example`) and **validated at startup, fail-closed** — a missing/default secret must refuse to boot in the prod tier (no "the comment says it's rejected" without code).
- `[H]` Secrets come from a manager (Vault/KMS/SOPS/ESO), not raw env, in production.
- `[H]` Security-relevant defaults fail **closed** (CORS, rate-limiting, internal-auth), not open.

### 5. Database & migrations
- `[B]` Migrations are present, ordered, additive; no unreviewed destructive migration.
- `[H]` Rollback story exists (down-migrations or a documented restore path).
- `[H]` A backup + **rehearsed** restore drill exists for any money/critical data.

### 6. Observability
- `[B]` Real `/health/live` + `/health/ready` (readiness probes its backing deps).
- `[H]` Metrics exposed (Prometheus `/metrics`) including domain signals (queue/outbox backlog, dead-letters, lag, auth success/fail, ledger imbalance).
- `[H]` Structured logging + distributed tracing export (OTLP); error tracking.
- `[H]` A dead-letter / quarantine sink that is **inspectable and replayable** (not log-and-drop).

### 7. Security
- `[B]` AuthN + AuthZ enforced on every non-public endpoint; multi-tenant isolation is **token-bound**, not client-asserted.
- `[B]` Input validation on all external input (esp. money amounts, IDs).
- `[H]` Dependency-vuln scan in CI (`cargo audit`/`cargo deny`, `pnpm audit`, Trivy on images).
- `[H]` Rate-limiting that works across replicas (shared store); secret scanning; TLS at the edge.

### 8. API contracts
- `[B]` The wire contract is defined (OpenAPI/proto/typed client) — not only prose; producer↔consumer verified for any cross-service seam.
- `[H]` Versioning/negotiation strategy; FE/BE contract checked in CI (codegen or contract test).

### 9. Error handling & resilience
- `[B]` No `unwrap`/`expect`/`panic!` on a fallible request/worker path (audited; test-only is fine).
- `[H]` Timeouts, retries with backoff, and idempotency on every external call and every at-least-once consumer.
- `[H]` Background workers/schedulers are actually **wired into `main`** and have a drain/retry/DLQ loop.

### 10. Deploy artifacts
- `[B]` A working container image (builds the *whole* workspace, release profile, non-root, `HEALTHCHECK`).
- `[H]` IaC (compose/k8s/Helm) + a deploy/rollback runbook.
- `[R]` Image scanned & pinned base digest.

### 11. Docs / runbooks
- `[B]` README + `.ai_context`/HANDOFF reflect *current* reality (no stale "greenfield skeleton" text on a mature service).
- `[H]` Operational runbook: deploy, rollback, backup/restore, on-call signals.

---

## Cross-service gates (ecosystem-level, beyond a single repo)

- `[B]` Each cross-service seam (POS108↔AccountZing↔Data↔Identity↔108Zing monolith) has the producer
  and consumer contract **verified against each other**, not assumed.
- `[H]` A cross-service e2e test environment exists and exercises the money path end-to-end before any
  financial domain reaches production.
- `[H]` One owner per capability — no undefined dual-source-of-truth (see the Notification/Identity
  cases in the audit).

---

## Per-service sign-off block (copy into the service's HANDOFF / `.ai_context`)

```
## Production-Readiness Gate — <service> — <date>

Target gate: [ ] GATE-T (testing entry)   [ ] GATE-P (production)

1.  Build           [ ] pass   evidence:
2.  Tests           [ ] pass   evidence:
3.  CI/CD           [ ] pass   evidence:
4.  Config/secrets  [ ] pass   evidence:
5.  DB/migrations   [ ] pass   evidence:
6.  Observability   [ ] pass   evidence:
7.  Security        [ ] pass   evidence:
8.  API contracts   [ ] pass   evidence:
9.  Resilience      [ ] pass   evidence:
10. Deploy artifact [ ] pass   evidence:
11. Docs/runbooks   [ ] pass   evidence:

Open BLOCKERS:
Gate decision (owner):  [ ] GO   [ ] NO-GO    by ______  on ______
```

---

## Current standing (2026-06-16 audit, refreshed 2026-06-17)

The 2026-06-16 audit found **no service clearing GATE-T cleanly**. As of the
2026-06-17 refresh (verified against `origin/main`), most core BLOCKERs are
merged — AccountZing now has auth (#8), Media now ships deploy artifacts (#3),
Identity requires prod secrets (#8), pos108/api builds + exposes `/metrics`
(#302/#310), Notification has a transactional outbox + reaper (#56/#55).
**Remaining GATE-T blockers (as of 2026-07-02):** the Data-Platform **value** reconciliation strategy
— the durability pre-check is shipped + inline-enforced (#50–#55; value strategy pending). AccountZing
deploy artifacts are now shipped (#12) and the recon durability gate is enforced — both prior blockers
closed. See the
[Follow-up status (2026-06-17)](PRODUCTION_READINESS_AUDIT.md#follow-up-status-updated-2026-06-17--verified-against-originmain)
for the full merged/open breakdown.
