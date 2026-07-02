# Decision / checklist — BipByte launch set (audit BLOCKER #7)

> **Status:** OPEN — ops/creds owner actions (not code-only). **Owner:** ibrowe108
> / BipByte ops. **Date:** 2026-06-17. Related: `PRODUCTION_READINESS_AUDIT.md`
> §BLOCKER #7. BipByte already has a thorough launch runbook + 5,684 tests; this
> tracks the **launch-blocking ops gaps** the audit found.

## The five launch-blocking items

| # | Item | Why it blocks | Decision / action needed |
|---|---|---|---|
| 1 | **realtime-edge E5 not deployed** | No edge ⇒ no realtime fan-out (core BipByte feature is dead) | Decide deploy target for the edge (managed? self-host?) + ship E5 |
| 2 | **External creds unset & un-live-verified** — SendGrid / Twilio / FCM / SCB | Notifications, SMS, push, payments silently fail in prod | Provision real creds in the secret store + run a **live** smoke per provider (not mocked) |
| 3 | **Managed Postgres + rehearsed restore drill** | Self-managed PG + no proven restore = data-loss risk at launch | Choose managed PG; run a **restore-from-backup drill** and record RTO/RPO |
| 4 | **S3 / CDN not wired** | Media upload/serve path unproven | Wire S3 + CDN creds/buckets; verify an end-to-end upload→serve |
| 5 | **`/health/ready` probes DB only** (codeable now) | Broken S3 stays invisible until the first upload fails | Extend readiness to probe S3 (+ critical externals) so orchestration sees S3 outages |

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
