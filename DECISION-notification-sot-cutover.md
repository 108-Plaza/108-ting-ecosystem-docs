# Decision — Notification source-of-truth cutover + durability (audit BLOCKER #8)

> **Status: RESOLVED (code) — Option A chosen; both durability gaps closed.**
> Remaining work is **operational** (the B1.6 cutover runbook below), owned by
> the ops/notification owner.
> **Owners:** Notification-Platform + the monolith team. **Decided:** 2026-06-17.
> Related: `PRODUCTION_READINESS_AUDIT.md` §BLOCKER #8 + Discrepancy #3,
> BipByte `docs/PLATFORM_BRIDGE_PLAN.md` §B1.

## Outcome at a glance

| Question | Resolution |
|---|---|
| **Ownership (Decision 1)** | **Option A — Notification-Platform is the SoT** for external-channel delivery. The monolith is a *producer of intent*. |
| **Which monolith?** | **BipByte `crates/06-notifications-delivery/notification-service`** — the service holding the `NOTIFY_CHANNEL_BACKEND_*` flags. In-app + preferences + realtime stay there; only *external channel delivery* (email/push) moves. |
| **Durability (Decision 2)** | **Done both sides.** Transactional outbox + reaper on the platform; durable delivery outbox on the monolith. |
| **Transport** | HTTP `POST /api/v1/notifications` with a per-`(notification,channel[,token])` idempotency key (`tixtox-notif-{id}-{channel}`), `X-API-Key` tenant auth. Not an event bus. |
| **Cutover style** | Per-channel flag flip behind a flag (no dual-send), rollback = flip back to `local`. Soak before declaring the wave closed. |

## What shipped (code complete)

**Platform side — `Platform-Services/Notification` (PR #56, `c038771`):**
- Transactional outbox: lifecycle event written in the **same DB tx** as the
  status change (`notification_outbox`), relay publishes at-least-once.
- Reaper: reclaims rows stuck in `processing` (PR #55).
- Observability: outbox backlog / dead-letter gauges.

**Monolith side — BipByte `notification-service` (PR #287, `0b5e5be`):**
- Channel routing to the platform already E2E-green for **email** (B1.4) with the
  **push** address resolver in place (B1.2b).
- **Durable delivery outbox** (migration 0004 `delivery_outbox`): the HTTP
  dispatch point was fire-and-forget — a failed send (platform down / network
  blip) was only logged, so the external delivery was lost while the in-app
  notification persisted. Now (inline-first + outbox backstop): inline send is
  tried for latency; on failure the notification id is enqueued and a background
  drainer re-sends through the same router (idempotent on the platform bridge).
  Mirrors the in-service `realtime_outbox` (0003) and the platform's own outbox.

Net: the platform is the source of truth for external delivery **end to end,
near-no-loss**, on both the producer and consumer sides.

## Scope notes / non-goals

- **In-app + realtime + preferences stay in the monolith** (audit §6.7). Only
  external channel delivery is the platform's.
- **sms / webhook are NOT in scope of this cutover** and are *not yet domain
  channels* in the monolith (`DeliveryChannel` = Email | Push | InApp). They each
  hit a contract gate (see Open items) and are tracked separately, not blocking A.

---

## B1.6 — Cutover runbook (operational; owner-driven)

> Goal: route **email then push** to the Notification-Platform in production with
> zero loss and an instant rollback. Each channel is flipped independently.

### 0. Preconditions (verify before touching prod)
- [ ] Notification-Platform reachable from the BipByte server (`/health` 200) on
      the prod network (intranet/VPC URL, not public).
- [ ] Platform DB is the durable Postgres (so the platform outbox + reaper run);
      backups + restore drill in place (see `DECISION-bipbyte-launch-readiness.md`).
- [ ] BipByte `notification-service` runs with `DATABASE_URL` set — required for
      the **delivery outbox** to be active (no PG ⇒ prior best-effort behavior).

### 1. Provision the tenant (B1.1)
- [ ] Create tenant **"tixtox"** via the platform Admin API (`X-Admin-Key`).
- [ ] Create templates matching current notification kinds (new_follower, gift,
      post_liked, order, …). Email kinds need a subject.
- [ ] Store the tenant `X-API-Key` in the BipByte server `.env` /secret manager —
      **never in the repo, never logged.**

### 2. Wire the bridge env (still `local` by default — no behavior change yet)
- [ ] `NOTIFICATION_PLATFORM_URL` = platform base URL
- [ ] `NOTIFICATION_PLATFORM_API_KEY` = the tixtox key
- [ ] `AUTH_SERVICE_URL` (email address resolution), `PUSH_SERVICE_URL` (device
      tokens), `INTERNAL_SERVICE_TOKEN` (resolver auth)
- [ ] Leave `NOTIFY_CHANNEL_BACKEND_EMAIL` / `_PUSH` = `local`.
- [ ] Deploy + confirm: logs say "bridge enabled" but routing still local; any
      missing resolver URL fails **safe** to local with a warning.

### 3. Flip EMAIL → platform (canary, then full)
- [ ] Set `NOTIFY_CHANNEL_BACKEND_EMAIL=platform` (canary instance first if
      possible). Routing is exclusive per channel — no dual-send.
- [ ] Verify: an enqueue lands `created` on the platform → `sent` + a delivery
      attempt; recipient receives the email; logs clean.
- [ ] Verify durability: with the platform briefly unreachable, the BipByte log
      shows "delivery failed inline; enqueued to outbox for retry", and a
      `delivery_outbox` row appears; on recovery the drainer marks it published
      (`SELECT count(*) FROM notification_service.delivery_outbox WHERE
      published_at IS NULL` returns to ~0). The reaper handles platform-side
      `processing` rows.
- [ ] Soak: **≥ 1 week, no regression.**

### 4. Flip PUSH → platform
- [ ] Set `NOTIFY_CHANNEL_BACKEND_PUSH=platform`. Push fans out per device token,
      idempotent per `(notification, token)`.
- [ ] Same verify + durability checks as email; soak ≥ 1 week.

### 5. Close the wave
- [ ] Both channels default `platform`, 1 week clean → declare B1 done.
- [ ] The 06 local channel services (email/sms/push) become **dormant** (not
      removed — decommission is Phase 4).

### Rollback (any time, no data loss)
- Flip the channel flag back to `local` (or unset the platform env). Local senders
  are untouched and resume immediately. In-flight `delivery_outbox` rows still
  drain through whatever the router currently points at.

### Dashboards / alerts to watch
- Platform: outbox backlog gauge, dead-letter count, `processing`-age.
- Monolith: `delivery_outbox` pending count (alert if it grows monotonically →
  platform unreachable), `bridge_requests_total{platform,op,status}`.

---

## Open items (tracked separately — NOT blocking the A cutover)

1. **sms channel** — would need a phone-number resolver, but `auth-service`
   *deliberately* exposes only email (`internal.rs`: "ONLY the email — never
   credentials, phone"). Adding sms requires a new internal endpoint + a privacy
   decision (PII export to the platform tenant). Also a new `DeliveryChannel`
   variant. **Decision required before any work.**
2. **webhook channel** — HMAC signing secret must either move to the platform or
   stay on the monolith dispatcher. **Decision required.** Also not a domain
   channel today.
3. **Decommission (Phase 4)** — removing the dormant 06 local channel services is
   out of scope here.
