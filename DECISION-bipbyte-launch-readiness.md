# Decision / checklist — BipByte launch set (audit BLOCKER #7)

> **Status:** OPEN — ops/creds owner actions (not code-only). **Owner:** ibrowe108
> / BipByte ops. **Date:** 2026-06-17. Related: `PRODUCTION_READINESS_AUDIT.md`
> §BLOCKER #7. BipByte already has a thorough launch runbook + 5,684 tests (test count
> as of 2026-06-17 — VERIFY/recount, the server has advanced since); this tracks the
> **launch-blocking ops gaps** the audit found.
>
> **STATUS (2026-07-02):** the two codeable items are resolved. **Item 5 DONE** — `media-service`
> `/health/ready` now probes S3 via `head_bucket` bounded by a 3s timeout, 503 on failure (#286 +
> #290). **Item 4 code plane DONE** — real `S3ObjectStore` (aws-sdk-s3) + `S3_BUCKET`/`S3_REGION`/
> `CDN_BASE_URL` config wired (#231; blanks treated as unset); only ops provisioning + the end-to-end
> smoke remain. **Item 1**: E5 is **built + merged** (realtime-edge #2/#5; consumed by web #28 +
> flutter #43) but **deployment is unproven** — the edge repo has no deploy manifests (VERIFY with ops).
> **Items 2 & 3** remain OPEN (ops-only: live creds + managed-PG restore drill). See per-item notes.

## The five launch-blocking items

| # | Item | Why it blocks | Decision / action needed |
|---|---|---|---|
| 1 | **realtime-edge E5** — ~~not deployed~~ **built + merged; deploy unproven** (VERIFY) | No edge ⇒ no realtime fan-out | Feature shipped (edge #2/#5; web #28, flutter #43). Remaining: decide deploy target + actually deploy (no manifest in repo yet) |
| 2 | **External creds unset & un-live-verified** — SendGrid / Twilio / FCM / SCB | Notifications, SMS, push, payments silently fail in prod | **OPEN (ops)** — provision real creds in the secret store + run a **live** smoke per provider (not mocked) |
| 3 | **Managed Postgres + rehearsed restore drill** | Self-managed PG + no proven restore = data-loss risk at launch | **OPEN (ops)** — choose managed PG; run a **restore-from-backup drill** and record RTO/RPO |
| 4 | **S3 / CDN** — ~~not wired~~ **code plane DONE (#231); ops smoke pending** | Media upload/serve path | Code wired (real `S3ObjectStore` + CDN config). Remaining ops-only: provision bucket/CDN creds + verify end-to-end upload→serve |
| 5 | ~~**`/health/ready` probes DB only**~~ ✅ **DONE (#286/#290)** | (was: broken S3 stays invisible) | Readiness now probes S3 (`head_bucket`, 3s timeout, 503 on failure) — shipped |

## What is codeable headless (no creds needed)
- **Item 5** — extend `/health/ready` to include an S3 reachability check (HEAD on
  the bucket) behind a short timeout, like pos108's DB+Redis readiness. This is the
  one item that does not need live creds/ops and can ship as a normal PR.

## What needs the owner (ops/creds — cannot be done headless)
- Items 1–4: deploy decisions, real third-party credentials, a managed-PG choice,
  and a rehearsed restore drill. These need someone with cloud/provider access and
  a maintenance window.

## Recommended sequencing
1. Ship **item 5** now (readiness probe) so the rest become observable.
2. Provision creds (item 2) + S3/CDN (item 4) in staging; live-smoke each.
3. Stand up managed PG + restore drill (item 3).
4. Deploy realtime-edge E5 (item 1) last, once the data/creds plane is proven.

## Open questions for the owner
- Cloud/provider + region for edge, PG, S3/CDN?
- Who holds the SendGrid/Twilio/FCM/SCB accounts; where do prod secrets live?
- Target RTO/RPO for the restore drill?
