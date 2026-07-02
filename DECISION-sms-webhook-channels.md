# Decision — SMS + Webhook delivery channels (notification follow-up)

> **Status:** OPEN — needs owner decision (privacy + secret-custody calls).
> **Owner:** ibrowe108 / Notification-Platform + auth-service owner.
> **Date:** 2026-06-17 · Related: DECISION-notification-sot-cutover.md (B1),
> BipByte notification-service, Platform-Services/Notification.
> **Why this is a gate, not just code:** both channels need a decision the code
> cannot make — SMS exposes a new PII attribute (phone), webhook accepts a
> user-supplied URL (SSRF) + needs a signing-secret custody model.

## Where things stand today

- Monolith `DeliveryChannel` enum = **Push | Email | InApp** only
  (`crates/06-notifications-delivery/notification-service/src/domain/kind.rs`).
  `sms`/`webhook` are **env-reserved placeholders, not domain channels.**
- The **platform's** external wire contract already accepts
  `email|sms|push|webhook` (`platform_sender.rs`), so the platform side is ready
  to *receive* them; the monolith side has no producer + no address resolver.
- Address resolution (B1.2b): email from auth-service
  `GET /internal/identities/:id/email`, push tokens from push-service. The
  auth-service internal endpoint **deliberately exposes ONLY email — never phone,
  credentials, or any other attribute** (`auth-service/src/api/internal.rs:6-7`).
  "phone-only" identities already exist (email is `null`).

So adding SMS is blocked on a **privacy contract** (how/whether to expose phone),
and webhook is blocked on a **security contract** (URL trust + signing secret).

---

## SMS

### Decisions the owner must make
1. **Add `Sms` to `DeliveryChannel`?** (Yes/No — opens the producer + resolver work.)
2. **Phone resolution + consent.** auth-service today never exports phone. To send
   SMS we need a phone number AND a lawful basis (PDPA: explicit opt-in for SMS).
   - Do users opt into SMS per kind (OTP/security vs marketing)?
   - Where is consent recorded?
3. **Provider + creds (ops).** Twilio account, sender id / sender pool, per-region
   rules; live smoke before launch. Secrets in the secret store, never in repo.

### Options for phone exposure
| Option | Shape | Pros | Cons |
|---|---|---|---|
| **A. Dedicated consented-phone resolver** (recommended) | NEW `GET /internal/identities/:id/phone` that returns the E.164 phone **only if** the identity has an active SMS consent, guarded by `X-Internal-Token` + a narrower scope than the email endpoint | Preserves least-privilege; consent enforced at the source; symmetric with the email resolver | One new endpoint + a consent field |
| B. Widen the email endpoint to also return phone | Smallest diff | **Rejected** — breaks the explicit "email only, never phone" contract; leaks phone to every internal email caller | least-privilege violation |
| C. Phone lives only in a separate SMS-consent store | Notification owns the consented phone copy | Decouples from auth-service | Duplicate PII custody, sync/erasure burden |

**Recommendation:** **A** — a separate, consent-gated phone resolver. Do **not**
widen the email endpoint. Default SMS to **transactional/security kinds only**
(OTP, security alerts) at launch; marketing SMS requires separate opt-in.

---

## Webhook

### Decisions the owner must make
1. **Add `Webhook` to `DeliveryChannel`?**
2. **Endpoint registry.** Webhooks target a *tenant/integration-registered* URL,
   not a user attribute. Who registers URLs, and where are they stored?
3. **Signing-secret custody.** Per-tenant HMAC secret — generated where, rotated
   how, stored where (secret manager, never repo/logs)?

### Recommended contract
- **Channel = tenant integration, not per-user.** A webhook notification resolves
  its address from a per-tenant registered endpoint, not from auth-service.
- **Signature:** `X-108-Signature: sha256=<hmac>` over the raw body, HMAC-SHA256
  with a **per-tenant secret** from the secret manager. Include a timestamp +
  reject stale (replay window) — same idea raised for the Omise webhook verify.
- **SSRF protection (mandatory):** registered URLs must be `https://`, public DNS
  only; **block private/loopback/link-local/metadata ranges** at send time
  (resolve + check, not just at registration). This is the main reason webhook is
  a security gate, not just code.
- **Delivery semantics:** reuse the durable delivery outbox (PR #287) — at-least-
  once, bounded retries with backoff, dead-letter after N; consumers dedupe on the
  idempotency key. No new transport.

**Recommendation:** ship webhook **after** SMS; it needs the registry + secret
rotation + SSRF guard, which is more surface than SMS. Until then keep it a
reserved placeholder (current state) rather than a half-built channel.

---

## What is codeable once decided (no further decision)
- Add the enum variant(s) + producer mapping + drift tests.
- SMS: the consent-gated phone resolver + Twilio sender (creds = ops).
- Webhook: endpoint registry + HMAC signer + SSRF guard + outbox wiring.

## Open questions for the owner
- SMS consent model + which kinds are SMS-eligible at launch?
- Twilio account owner + secret location?
- Who registers webhook endpoints; per-tenant secret rotation policy?
- Replay window + retry/dead-letter thresholds for webhook?
