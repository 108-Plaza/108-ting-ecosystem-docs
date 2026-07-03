# KBank as a Generic Payment Provider — Implementation, Deploy & Test

**Date:** 2026-06-28
**Status (updated 2026-06-28):** MERGED to main (gateway #71 + enum-dispatch #72, relay #476, admin #259)
and **DEPLOYED to K3s staging** (payment `sha-c131929`, pos108 `sha-074393a`, pos-admin `sha-ffd6b74`).
Gateway DB migrated 18→25 (0022 backfilled 2 live SCB creds; old tables dropped; pre-migration backup
`/tmp/payment-pre-kbank-*.dump` on the K3s host). **E2E browser test PASSED** (SA): created a real KBank
QR app via UI (keyed to `payment_credentials`, catalog shows provider=KBank). A read-back 500 bug found in
testing (relay decoded gateway-nullable `terminalId`/`envId` as non-optional `String`) was fixed
([pos108 #477](https://github.com/108-Plaza/pos108-core/pull/477)) and redeployed (`sha-b69368f`); edit prefill re-verified working.
**Only remaining (as of 2026-06-28):** live KBank QR **mint** (blocked on the mTLS client cert +
source-IP allowlist) — VERIFY current cert/allowlist state.
**Companion:** `docs/PAYMENT_PLATFORM_MULTIPROVIDER_SURVEY.md` (the survey that scoped this).
**Memory SoT:** `kbank-qr-second-provider`.

> **Update (2026-07-02):** the same generic seam has since carried further providers — 2C2P (card PSP,
> #76/#77/#82), a dark BBL QR adapter (#78), and scaffolded GSB/BAY/TTB/KTB stub providers (#79).
> `PaymentProvider` is now a **7-variant enum** (`Scb, Kbank, TwoCTwoP, Gsb, Bay, Ttb, Ktb`), not
> SCB+KBank. See `PAYMENT_PLATFORM_QR_BANKS_EXPANSION.md`.

## Goal
An operator creates a **KBank QR app** in the pos108-admin UI, assigns it to a branch, and that
branch mints a **real KBank QR** — through the same flow as SCB, with no parallel/duplicate code
path. Provider is **generic** end-to-end (`{provider}`), not an SCB mirror.

## Design (one sentence per layer)
- **Gateway** already stores all credentials in one generic `payment_credentials` table (provider
  column). Routing now **derives the provider from the merchant's active credential** instead of a
  separate (never-written) routing table.
- **Relay** (pos108 cloud) exposes one **provider-parameterized** credential plane and forwards to
  the gateway's existing per-provider admin routes.
- **Admin UI** has one dialog with a **provider/bank selector** + data-driven per-provider fields.

## The three PRs (merge order 1 → 2 → 3; deploy 2 + 3 together)

| # | Repo | PR | What |
|---|---|---|---|
| 1 | Payment-Platform (Gateway) | [#71](https://github.com/108-Plaza/Payment-Platform/pull/71) (+ enum-dispatch [#72](https://github.com/108-Plaza/Payment-Platform/pull/72)) | `ProviderRouter` derives provider from `payment_credentials` (drops the dead `merchant_payment_provider` table via migration **0025**); enriches `GET /v1/admin/merchants` with `provider` per merchant. |
| 2 | pos108 (api / cloud relay) | [#476](https://github.com/108-Plaza/pos108-core/pull/476) | Generic credential relay: `CredentialGatewayPort`, routes `/payment-gateway/apps` + `/{id}/credentials` (clean break of `/scb-apps`), routes by `provider` to gateway `/scb_credentials` or `/kbank_credentials`. |
| 3 | pos108-admin | [#259](https://github.com/108-Plaza/pos108-admin/pull/259) | Generic multi-provider catalog UI: provider selector + `PROVIDER_FIELDS` map + provider column + bank-agnostic i18n + e2e. |

### Generic API contract (relay ↔ admin)
```
GET    /api/v1/payment-gateway/apps                        -> [{id,name,email,isActive,provider}]
POST   /api/v1/payment-gateway/apps                        -> create merchant {name,email,password}
PUT    /api/v1/payment-gateway/apps/{id}/credentials       -> body {provider, ...fields}
GET    /api/v1/payment-gateway/apps/{id}/credentials?provider=scb|kbank
DELETE /api/v1/payment-gateway/apps/{id}/credentials?provider=scb|kbank
POST   /api/v1/payment-gateway/apps/{id}/credentials/reactivate?provider=scb|kbank
```
PUT body — shared: `provider, appName, description?, baseUrl?, sandbox`. SCB: `appId, appSecret?,
signingKey?, billerId?, ref3Prefix?`. KBank: `consumerId, consumerSecret?, partnerId?,
partnerSecret?, merchantCode?, terminalId?, qrType?(3|4|5), envId?`. Secrets are write-only
(omit/blank = keep stored).

## Local verification (all green)
- **Gateway #65**: `cargo check/fmt/clippy` clean; 3 routing unit tests + 15 DB integration tests
  (all migrations incl. 0025; `merchant_payment_provider` confirmed dropped).
- **Relay #476**: `cargo check` (pos-payment + bin) + fmt + clippy clean; `pos-payment` 5 + `scb_gateway_client` 20 tests (incl. KBank DARK-degrade).
- **Admin #259**: `tsc` + `lint` clean; `vitest` 362; `playwright` smoke (payment-gateway 3 + branches 4).

## Deploy to staging (owner-gated)
1. **Merge** #71 → Payment-Platform `main` (owner approval). Apply migration 0025 + deploy the
   gateway image to staging.
2. **Merge** #476 (pos108) and #259 (pos108-admin) → `main` (owner approval). They are a **clean
   break** (relay drops `/scb-apps`; admin targets `/apps`) so **deploy them together**.
3. Staging image build fires on push to each repo's `deploy/staging` branch (owner merges
   `main → deploy/staging`); roll out via the K3s/cloud path. Ensure the gateway **admin key**
   (`PAYMENT_GATEWAY__SCB_ADMIN_KEY` on the pos108 cloud relay) is set, else the credential plane
   is DARK (reads empty, writes error).

## Browser test (after deploy) — create a KBank app & mint a real KBank QR
1. Log in to `admin.staging.108plaza.net` as a **Super Admin** (capability `payments:scb:manage`).
2. **Payment Gateway → Add** → select provider **KBank** → fill the KBank fields (consumerId,
   consumerSecret, partnerId/secret, merchantCode, terminalId, qrType=3, envId, KBank **sandbox**
   base URL, sandbox=true) → Save. Catalog row should show provider **KBank**.
3. Assign the app to **CAFE01** (branch edit → QR app selector). Capture the once-shown branch key.
4. Mint a QR on a CAFE01 device (branch front door, `X-Branch-Key`): the gateway resolves the
   merchant → derives provider = kbank → KBank registry → OAuth `/v2/oauth/token` →
   `/v1/qrpayment/request` → real EMVCo QR.

## Open unknowns / risks (Slice 4 — not code)
- **mTLS client cert**: KBank Open API sandbox may require a client certificate for direct calls
  (`KBANK_CLIENT_CERT_PATH` on the gateway). Owner has consumer creds but **no cert yet** — this can
  block the mint at OAuth even after deploy. apiportal "เชื่อมต่อ API" (production go-live) is still
  **locked**, but sandbox test access works (Try-API 15/15 passed).
- **Source-IP allowlist**: KBank may IP-allowlist the gateway egress ("403 Access Denied ⇒
  allowlist"). Staging egress IP may need registering.
- **Migration 0025** drops `merchant_payment_provider` (empty/never-written → inert) — low risk but
  a schema change. (It did land as 0025, not the originally-drafted 0023: Gateway `main` advanced past
  0022 before merge, so the drop was renumbered; there is no 0023 migration.)
- A merchant could technically hold two active provider credentials (PK `(merchant_id, provider)`);
  routing uses a deterministic `ORDER BY provider` tiebreak. Hardening (deactivate-others-on-upsert)
  is a follow-up.

## Notes
- Built each slice in an **isolated git worktree** because the shared Gateway clone was being
  operated concurrently by another session (branch switches reset uncommitted work).
- Relay & admin kept some internal `scb_*` symbol/file names to minimize churn; the public contract
  and behaviour are generic. Capability gate `payments:scb:manage` kept (RBAC rename = follow-up).
