# Payment Platform — QR Banks Expansion (BBL / KTB / GSB / ttb / Krungsri)

> Preparation plan (2026-06-28). Goal: add merchant-receive **QR Code** providers for
> Bangkok Bank (กรุงเทพ / BBL), Krungthai (กรุงไทย / KTB), GSB (ออมสิน / Government Savings
> Bank), ttb (TMBThanachart), and **Krungsri (กรุงศรี / BAY)** to **Payment-Platform/Gateway**,
> on the **existing generic-provider seam** (same one that carries SCB + KBank + 2C2P today).
> Krungsri was added by the owner during planning (2026-06-28) → **5 new banks**.

> **STATUS (2026-07-02) — partly shipped, and the router shape differs from the plan below:**
> - **GSB/BAY/TTB/KTB variants already exist** in the enum, dispatched to a single shared
>   `StubBankGateway` via `stubs.for_provider(p)` — **not** as new generic type-params (PR #79). So
>   `ProviderRouter` stayed `<S, K, T>` (scb/kbank/2c2p) + one stub arm covering all four dark banks;
>   the "grow to 7/8 type-params" concern (Decision 2 / §5) did **not** materialize for the stubs.
> - **BBL is still absent from the enum** — its adapter `bbl_gateway.rs` exists but is dark/unwired
>   (PR #78), so for BBL the "add enum variant" step still stands.
> - `PaymentProvider` now lives at `crates/application/src/ports.rs:226` (the §2 "`:218`" reference is
>   stale). Per-bank work remaining = replace each stub with a real adapter + portal-verify the contract.

## 0. Decisions locked (owner, 2026-06-28)

1. **Banks (5):** BBL, KTB, GSB, ttb, **Krungsri/BAY** — owner reports portal access + sandbox creds
   for all five. (Contract still must be portal-verified per bank — Gate 0, §5.)
2. **Router architecture = (A) keep static dispatch, grow type-params.** ⚠️ With SCB+KBank+2C2P
   already present, 5 new banks ⇒ **8 providers / 8 type-params** in `ProviderRouter<…>` and an 8-arm
   `match` per method. Honored as chosen; revisit (B) dyn-map is fair around bank #3–4 if `main.rs`
   wiring gets unwieldy (the dyn cost is noise next to the HTTP round-trip).
3. **Driver = settlement at the merchant's own bank** → **per-merchant routing** (the active-credential
   model already in place). Confirms the design in §4; no new product surface.
4. **Pilot = BBL first, end-to-end** as the template that proves the post-genericization "add a bank" path.
   Then parallelize KTB / GSB / ttb / Krungsri as their contracts clear.
>
> This is a **prepare/plan** doc — no code is written yet. It records exactly what one new
> provider touches, the open decisions that gate the work, and the per-bank slicing.
> SoT for the existing provider work: memory `kbank-qr-second-provider`,
> `docs/PAYMENT_PLATFORM_KBANK_GENERIC_PROVIDER.md`, `docs/PAYMENT_PLATFORM_MULTIPROVIDER_SURVEY.md`.

## 1. Foundation already in place (the big head start)

The generic-provider chain is **merged + deployed to staging** (2026-06-28). It already abstracts
providers so a 4th–7th bank plugs into the same seam:

- **One generic credential store** — `payment_credentials` (PK `merchant_id + provider`;
  `secret1_enc/last4`, `secret2_enc/last4`, `config JSONB`, `provider TEXT`). Any provider's
  fields live in `config` JSON + two encrypted secrets. **⇒ adding a bank needs NO DB migration.**
- **Routing derived from the active credential** — `ProviderRouter` reads the merchant's active
  credential to pick the provider; no per-branch provider table (the old `merchant_payment_provider`
  was dropped as dead code). ⇒ a bank works the moment a merchant is keyed with that bank's cred.
- **Relay is provider-agnostic** (pos108 #476): clean-break routes `/payment-gateway/apps` +
  `?provider=` pass-through.
- **Admin form is data-driven** (pos108-admin #259): `PROVIDER_FIELDS: Record<Provider, …>` +
  `PROVIDERS[]` + bank-agnostic i18n. A bank = one map entry + labels.

## 2. What ONE new bank touches (grounded in real files)

### Gateway — `Payment-Platform/Gateway/backend`
1. **Provider enum** — `crates/application/src/ports.rs:218` `enum PaymentProvider`: add variant
   (e.g. `Bbl`) + `as_str()` (`"bbl"`) + `parse_or_scb()` arm.
2. **Adapter** — `crates/infrastructure/src/<bank>_gateway.rs`: new struct impl `PaymentGateway`
   (`authorize` → create QR, `inquiry` → status). Mirror `kbank_gateway.rs` (OAuth token cache,
   retry-once-on-401, mTLS via cert env if the bank requires it).
3. **Per-merchant registry** — `crates/infrastructure/src/<bank>_gateway_registry.rs`: builds a
   per-merchant client from the generic cred store + caches it + `GatewayCacheInvalidator`. Mirror
   `kbank_gateway_registry.rs`. Falls back to an `Unconfigured…Gateway` that DECLINES (never mock).
4. **Config parsing** — `crates/infrastructure/src/cred_config.rs`: read the bank's `config` JSON
   keys via `cfg_str(...)` into the bank's `Config` struct.
5. **Router** — `crates/infrastructure/src/provider_router.rs`: **see §3 (architecture decision)** —
   today this is `ProviderRouter<S, K, T>` with one concrete type param per provider + a `match` arm
   in every method.
6. **Callback** — `POST /v1/webhooks/<bank>`: if the bank signs callbacks, add a `<bank>_callback_verifier.rs`;
   if unsigned (like KBank), fail-closed via `reconcile_via_inquiry` (trust only the correlation id,
   settle from a fresh inquiry, ACK in the bank's expected shape).
7. **Wiring** — `api/src/main.rs`: construct the registry + pass into the router; `.env.example`
   documents the new `<BANK>_*` env (base URL, env-id, mTLS cert path if any).
8. **Migration** — **none for creds.** Only if a bank needs a side-channel column (KBank needed
   `gateway_txn_no` for exact-txn void) does a migration appear — decide per bank from its contract.

### Relay — `Commerce-Platform/pos108/api` (actix monolith, active work area)
- Already routes by `?provider=`. Likely **near-zero code** per bank — confirm the relay's provider
  allowlist/validation accepts the new string and that the admin-key relay degrades dark when unset.
  (Per-bank credential decode structs use `Option<String>` for nullable fields — the read-back null
  bug fixed in pos108 #477 is the template to follow.)

### Admin — `pos108-admin` (`Commerce-Platform/pos108/apps/admin`)
- `src/features/payment-gateway/scb-app-form-dialog.tsx`: add to `PROVIDERS[]` and `PROVIDER_FIELDS`
  (idKey, secretKey, per-bank fields). `src/lib/api/scb-app.ts`: extend the `Provider` union.
- i18n `messages/{en,th}.json` namespace `paymentGateway.provider`: bank display name (keep copy
  bank-agnostic elsewhere).

**Net per-bank cost (post-genericization): 1 adapter + 1 registry + enum variant + router arm +
callback route + 1 admin map entry. No DB migration. ~½–1 of the KBank effort each.**

## 3. DECISION — ProviderRouter scaling (static N type-params vs dyn map)

`ProviderRouter<S, K, T>` was deliberately refactored to **static enum-driven dispatch**
(commit `7a3f2be`, one concrete `Arc<S>/<K>/<T>` field per provider — no per-call vtable). Adding
BBL+KTB+GSB+ttb pushes this to **7 type params** (`<S,K,T,B,Kt,G,Tt>`) and a 7-arm `match` in every
method. Two ways forward:

- **(A) Keep static dispatch, grow to 7 type params.** Preserves the owner's no-vtable decision;
  cost is verbose generics + every call site in `main.rs` names 7 concretes. Tolerable but noisy.
- **(B) Switch to `HashMap<PaymentProvider, Arc<dyn PaymentGateway>>`.** One field, register any N
  providers, one lookup; reintroduces a single `dyn` hop per call (negligible vs a network round-trip).
  Cleaner for a 7-provider future; reverses the `7a3f2be` micro-optimization.

Recommendation: **(B)** once provider count crosses ~4 — the dyn cost is noise next to the HTTP call,
and the generics stop scaling. Flagging because it reverses a deliberate prior decision → owner call.

## 4. STRATEGIC NOTE — Thai QR is interoperable (what's actually different per bank)

Thai QR / PromptPay is an **EMVCo interoperable rail**: a QR minted via *any* bank's merchant API is
payable from *any* customer's banking app. So the value of N bank integrations is **NOT** "more
customers can pay" — it's:
- **Settlement destination** — funds land in the merchant's account *at that bank* (a BBL merchant
  wants BBL, an ออมสิน merchant wants GSB).
- **Per-bank inquiry/callback/void** — each bank confirms PAID and reverses through its own API.

Implication: the **QR-create payload is similar across banks** (EMVCo PromptPay), but **OAuth +
inquiry + callback + settlement/void differ per bank** — that's where the real per-bank contract
work is. This also raises a product question (§5 Q3): is multi-bank driven by merchants who bank
elsewhere, or is one acquirer enough?

## 5. The real blocker — per-bank contract verification (Gate 0)

KBank only moved once its contract was **verified endpoint-by-endpoint from the bank's API portal**
(OAuth, QR create, inquiry, callback both directions, void, error codes, field lengths, mTLS/IP
rules). The same Gate 0 applies to each new bank, and is the **prerequisite to any adapter code**.
None of these four contracts are verified yet. Per bank we need, from that bank's developer/partner portal:

Owner reports portal access + sandbox creds for all five (2026-06-28). What is still **unverified** is
each bank's *contract shape* — the endpoints, fields, callback signing, mTLS/IP rules — which must be
read from the portal before any adapter is written (do not assume a KBank-shaped contract).

| Bank | Portal + sandbox creds | Contract portal-verified (Gate 0) | mTLS / IP-allowlist? |
|------|------------------------|-----------------------------------|----------------------|
| BBL (กรุงเทพ)   | ✅ owner has | 🟡 portal docs read 2026-06-28 → `Gateway/docs/bbl-qr-provider-plan.md` (pending Try-API confirm) | ✅ none (TLS only) |
| KTB (กรุงไทย)   | ✅ owner registered 2026-06-29 | ⛔ **self-serve portal has NO merchant-receive QR** (confirmed 2026-06-29: only Fund-Transfer-OUT incl. to-PromptPay, Direct Debit, Account, and "QR Scan"=Paotang LOGIN). Merchant-collect = corporate-gated → email sent to KTB. `Gateway/docs/ktb-qr-provider-plan.md` | ⚠️ yes (KTB IP-allowlists) |
| GSB (ออมสิน)    | ✅ owner has | ❌ pending | ❓ |
| ttb             | ❌ **no portal access** (owner not registered, confirmed 2026-06-29) → SKIPPED until owner registers | ❓ |
| Krungsri (กรุงศรี/BAY) | ✅ owner has | ⏸️ **PARKED 2026-06-29** — Swaggers read; **no MPM mint-QR API** (only RTP request-to-pay [needs payer phone] + CPM scan-customer-QR + KMA). Doesn't fit MPM. `Gateway/docs/krungsri-qr-provider-plan.md` | ✅ none (Bearer + X-Client-Transaction-ID) |

> Agent cannot self-serve bank developer portals. Gate 0 per bank = owner shares the portal API docs
> (or grants access / pastes the endpoint specs) → agent verifies + writes `docs/<bank>-qr-provider-plan.md`.

## 6. Per-bank slicing (mirror KBank, once Gate 0 passes)

For each bank, after the contract is verified:
- **S0 Contract verification** (owner + agent on the portal) → write `docs/<bank>-qr-provider-plan.md`.
- **S1 Adapter (dark)** — `<bank>_gateway.rs` impl `PaymentGateway`, token cache, unit tests; built
  only when `<BANK>Config::from_env()` is wired. No behavior change.
- **S2 Cred + registry + router** — config-key parsing, `<bank>_gateway_registry.rs`, enum variant,
  router arm (or map entry per §3). Dark by default.
- **S3 Callback** — `POST /v1/webhooks/<bank>` fail-closed (or signed verifier). Route inert until
  the bank calls.
- **S4 Relay + Admin** — provider allowlist + `PROVIDER_FIELDS` entry + i18n.
- **S5 Stage on staging** — key a test merchant, mint→pay→inquiry=PAID on the bank's sandbox.
- **S6 Prod go-live** — owner ops (real creds, mTLS cert, IP allowlist, prod cutover).

Banks are **independent** — each can land on its own timeline as its contract clears. Suggest doing
**one bank end-to-end first as the template** (proves the post-genericization "add a bank" path), then
parallelize the rest.

## 7. Owner questions — RESOLVED 2026-06-28 (see §0)

Architecture (A static), banks (5 incl. Krungsri), driver (settlement/per-merchant), pilot (BBL) — all
answered and recorded in §0. One remaining confirmation: this spans **3 repos** (Payment-Platform/Gateway
+ pos108 relay + pos108-admin), like the KBank chain — runs as an approved parallel area to the
`pos108/api` active area (KBank/2C2P already did).

## 8. BBL Gate 0 — DONE (portal docs read 2026-06-28)

BBL QR Payment contract captured in `Payment-Platform/Gateway/docs/bbl-qr-provider-plan.md` from
`apiportal.bangkokbank.com`. Headlines: OAuth Basic+client_credentials (24h token); `qr-generate`→
`qrCodeId`+EMV `qrData`; `payment-inquiry`→statusCode 00/UK/01/02; **3-phase refund**; **TWO inbound
callbacks** incl. a pre-pay verification handshake; **every request JWT-signed with merchant RSA
private key**; notification callback signed+Basic-auth. **No mTLS / no IP-allowlist** (BBL clears the
two unknowns that block KBank). Cred map: secret1=consumerSecret, secret2=RSA private key, config=
{billerId,baseUrl,reference1Prefix}. **No DB migration.**

Decisions locked (RSA keypair = gateway-generated; pre-pay verify = always-approve).

**S1 SHIPPED (draft, dark) 2026-06-28** — [Payment-Platform #78](https://github.com/108-Plaza/Payment-Platform/pull/78):
`bbl_gateway.rs` `BblPaymentGateway` (authorize→qr-generate, capture→payment-inquiry, RS256-signed,
OAuth 24h). Additive + DARK (not wired into router/main.rs). No DB migration. 11 unit tests +
fmt/clippy/workspace green local; contract not yet Try-API-exercised (S0). **Next:** S0 sandbox confirm →
S2 refund (3-phase) → S3 callbacks → S4 relay + admin `PROVIDER_FIELDS["bbl"]` + per-merchant keygen →
enum variant + router wiring. Krungsri/KTB/ttb still need owner-supplied field contracts; GSB needs a link.
