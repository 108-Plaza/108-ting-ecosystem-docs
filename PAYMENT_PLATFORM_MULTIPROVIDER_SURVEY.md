# Payment Platform — Multi-Provider (KBank) Readiness Survey

**Date:** 2026-06-28
**Scope of survey:** read-only across 3 layers — Payment-Platform/Gateway, pos108 cloud relay
(`Commerce-Platform/pos108/api`), pos108-admin UI (`Commerce-Platform/pos108/apps/admin`).
**Question that started it:** "pos108-admin UI รองรับ new design ของ Payment Platform แล้วหรือยัง?"

Related SoT: memory `kbank-qr-second-provider` (backend slice-by-slice history),
`single-payment-gateway-program`, `pos108-scb-gateway-bridge`.

---

> **Implementation (2026-06-28):** the generic-provider build that this survey scoped is now
> code-complete across 3 repos — see `docs/PAYMENT_PLATFORM_KBANK_GENERIC_PROVIDER.md` for the
> design, the 3 PRs ([Payment-Platform #65](https://github.com/108-Plaza/Payment-Platform/pull/65),
> [pos108 #476](https://github.com/108-Plaza/pos108/pull/476),
> [pos108-admin #259](https://github.com/108-Plaza/pos108-admin/pull/259)), deploy sequence, and the
> browser test. Deploy + live KBank mint are pending owner merge + the mTLS/IP unknowns below.

## 1. Headline

- **SCB per-branch DB-binding design → fully supported + live** (admin catalog + per-branch
  selector + cred relay + ref3_prefix; deployed staging + PROD .68).
- **Multi-provider design (KBank) → backend MERGED, but the two layers above the gateway
  (relay + admin UI) are still SCB-only.** KBank is greenfield: code exists, **never tested
  against a real KBank endpoint.**

## 2. Layer-by-layer status

| Layer | Multi-provider? | Evidence |
|---|---|---|
| **Gateway** (Payment-Platform) | ✅ SCB + KBank merged to main | `PaymentProvider::{Scb,Kbank}` (`crates/application/src/ports.rs`); generic `payment_credentials` store (PR #54 / migration 0022); KBank cred admin API (PR #53); KBank adapter (PR #45/#48); ProviderRouter (PR #51 / migration 0020) |
| **Relay** (pos108 cloud / api) | ❌ SCB-only | `src/infrastructure/payments/scb_gateway_client.rs` calls only `/v1/admin/merchants/{id}/scb_credentials`; route scope `web::scope("/payment-gateway/scb-apps")` (`src/presentation/http/payment.rs:84`); **zero KBank references** |
| **Admin UI** (pos108-admin) | ❌ SCB-only | catalog/form/selector hardcode `scb-app-*` (`src/features/payment-gateway/*`, `src/features/branches/qr-app-section.tsx`); no provider dropdown; no `kbank`/`provider` references |

**Live confirmation (2026-06-28, admin.staging.108plaza.net):** drove the staging admin via browser.
Payment Gateway page renders **"You don't have permission to manage the payment gateway"** for the
logged-in account (gate = `payments:scb:manage`, SA-only). Bundle scan of the 19 loaded JS chunks
on that page: `scbApp`=2, `ref3Prefix`=1, **`kbank`=0, provider/bankSelector=0, consumerId=0** →
staging admin build is **SCB-only** (no KBank in catalog). ⇒ **Creating a KBank QR via the UI is
not possible on current staging admin**; needs the Step-3 frontend work first.

Visually confirmed (owner screenshot, SA account): the catalog Add modal is titled **"Add SCB QR
app"** with SCB-only fields — Merchant name, Account email, Password, App name, API Key (App ID),
API Secret, Biller ID, Ref3 prefix (default MRC), Base URL, Signing key, Sandbox mode,
Description. **No bank/provider selector, no KBank fields** (consumerId/partnerId/merchantCode).
KBank frontend = not deployed.

## 3. Key architectural finding — `merchant_payment_provider` is dead code

The per-merchant provider router (PR #51) routes by reading table `merchant_payment_provider`.
**That table is never written:**

- Whole-repo search: only `CREATE TABLE` (migration 0020) + a single `SELECT`. **No
  INSERT/UPDATE/DELETE anywhere** in code or migrations.
- Therefore `PgPaymentProviderRepository::provider_for()` always returns `None`
  (`crates/infrastructure/src/postgres/payment_provider_repository.rs`), and the router always
  falls back to the platform default `PAYMENT_GATEWAY_PROVIDER` (= `scb`)
  (`crates/api/src/main.rs:177`).
- `provider_for` has **exactly one caller** — the router. No use case / service / handler
  depends on it. The repo is constructed once and injected only into the router.

**Implication:** per-merchant provider routing as a *dedicated table* is redundant. The provider
a merchant uses is already a function of which credential it holds — `payment_credentials`
carries a `provider` column (PK `(merchant_id, provider)`). The existing `ScbGatewayRegistry` /
`KBankGatewayRegistry` already resolve per-merchant by loading creds keyed `(merchant_id,
provider)`.

**Recommended refactor (cheap now — table empty, KBank untested, no data to migrate):**
1. `ProviderRouter` source: replace `providers.provider_for()` with a new
   `creds.active_provider_for(merchant_id)` = `SELECT provider FROM payment_credentials WHERE
   merchant_id=$1 AND is_active LIMIT 1`.
2. Delete trait `PaymentProviderRepository` + `PgPaymentProviderRepository` + StubProviders test.
3. New migration: `DROP TABLE merchant_payment_provider`.
4. `main.rs`: drop `provider_repo` wiring.
5. Keep the `ProviderRouter` decorator (still dispatches SCB↔KBank at mint) — only its input
   source changes.

Caveat: PK `(merchant_id, provider)` technically allows a merchant to hold two providers at
once. In the per-branch catalog model (1 catalog app = 1 merchant = 1 provider) this never
happens — enforce "≤1 active provider per merchant" and the derive is unambiguous.

**Effect on the admin question:** with routing derived from creds, the work to enable KBank via
UI shrinks to: catalog Add form picks provider type → hits the right creds endpoint; relay
passes through KBank creds. **No per-branch provider selector and no set-provider endpoint
needed** — branch→merchant binding already disambiguates.

## 4. Test-data-first path (validate KBank without building UI)

Decision (owner, 2026-06-28): seed test data directly so the KBank adapter is proven before any
UI work. Environment = **staging**. Creds on hand: **consumer_id/secret yes, mTLS cert NO.**

Seed recipe (no UI):
```
POST /v1/merchants                                  -> merchant_id (name it KBANK-TEST)
PUT  /v1/admin/merchants/{id}/kbank_credentials     (X-Admin-Key)  -> KBank creds
INSERT merchant_payment_provider (merchant_id,'kbank')   -- routing (current code) OR do §3 refactor
POST /v1/auth/tokens   -> JWT;  POST /v1/payment_intents -> intent;  .../confirm -> authorize()
```
authorize() → OAuth `POST /v2/oauth/token` → `POST /v1/qrpayment/request` → `qrCode`
(`crates/infrastructure/src/kbank_gateway.rs`).

**First unknown to clear = does KBank sandbox require mTLS?** Probe (no gateway/DB/staging
needed) = single OAuth call with consumer creds, no cert:
`scratchpad/kbank_oauth_probe.sh` — 200+token ⇒ cert not needed; SSL/TLS curl error ⇒ mTLS is a
hard blocker (chase cert from onboarding). ⚠️ KBank token quota = 5 req / 30 min.

## 4b. KBank apiportal certification — PASSED 15/15 (2026-06-28)

Owner reports the apiportal Try-API set for app `pos108` is **15/15 green** (OAuth 2.0, Generate
Thai QR, Generate QR Credit Card, all Inquiry/Cancel/Void variants incl. settlement &
over-the-day). **Now pending KBank approval** (production access / provisioning).

App card state (apiportal): **ทดสอบ API 15/15 ✅** + **ข้อมูลการสมัคร ✅** (registration complete)
but **เชื่อมต่อ API 🔒 LOCKED** (greyed, lock icon). So "รออนุมัติ" = KBank unlocking
**เชื่อมต่อ API = production go-live**. This gates PRODUCTION only — **sandbox test access still
works** (the 15/15 proves it), so testing our gateway → KBank sandbox on staging can proceed now;
only the real prod cutover waits on KBank's unlock.

What this proves: the QR contract + the consumer creds are valid. **What it does NOT cover:** the
portal runs the calls from KBank's own side. Our gateway makes its *own outbound* HTTPS calls
from staging → so two transport-layer unknowns remain, independent of KBank's approval:
- **mTLS client cert** required for direct (non-portal) calls?
- **source-IP allowlist** — memory note "403 Access Denied implies KBank IP-allowlists the
  source"; staging egress IP may need allowlisting.
The OAuth-without-cert probe still answers "can our box reach KBank directly?" (≠ "did KBank's
own test pass?").

## 5. Remaining blockers / open items

1. **mTLS cert + source-IP allowlist** — for our gateway's *direct* calls; not proven by the
   portal 15/15. Probe pending.
2. **Staging gateway build** — verify the staging image includes KBank (migrations 0020–0022 on
   staging DB). Deploy is via the `deploy/staging` branch image build; main does not build images.
3. **Migration 0022 is Data Risk** — it DROPs `scb_credentials`/`kbank_credentials` (backfill
   into `payment_credentials`); applying to staging/prod is owner ops.
4. **Step 2 (relay) + Step 3 (admin UI)** — not started; file-level scope already mapped in
   memory `kbank-qr-second-provider`.
5. **Mae Manee (3rd provider)** — design-only (PR #50); no code in any layer; `PaymentProvider`
   enum has no `Maemanee` variant.
