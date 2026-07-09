# 108 Ting Ecosystem Active Work

## Active Work Area
~/108-POS/core

## Active Goal
POS108 service separation / POS108 API stabilization.

## Active Sub-Project Handoff
~/108-POS/core/.ai_context/ (tracked control system; start with README.md → CONTROL_TOWER.md)

## POS108 — Recently Shipped (rollup updated 2026-07-09)
Repo HEADs at this rollup: **pos108-core → #547**, **pos108-admin → #300**, **pos108-terminal → #22**
(supersedes the "#509" figure referenced in the 2026-07-02 SCB snapshot below). All merged to their
default branches and (except where noted) deployed to staging. Grouped by theme; PR numbers are the
source of truth.

- **Void/discount approvals (manager maker-checker, C2 flow).** Terminal raises a request → 409/pending
  → manager approves on admin; Separation-of-Duties enforced (the raiser cannot self-approve).
  core #522 (status endpoint), #523 (branch-scoped live pending list), #524 (real approval method);
  admin #283/#284 (approve page + server-side grant verify), #285 (live pending list), #286/#287
  (global notifier badge/toast/beep), #291 (prefill from terminal QR deep link); terminal #12 (void +
  approval flow), #13 (void menu), #19 (https QR deep link → prefilled admin page).
- **RBAC branch-scope + branch-manager UX.** core #528 (branch-scope roles + backfill migration +
  cashier POS-grid reads), #520 (`GET /users?branchId=` pinned resolution), #542 (server-side display
  names for branch-scoped admins); admin #288 (roles UI mirrors RBAC hierarchy), #278 (branch-context
  nav), #295 (names-not-ids, store-type menu gating, hide HQ-only rows).
- **Branch-locked UI hardening (2026-07-09).** admin #300 (hide branch selectors for branch-locked
  users across ~19 pages + stop the spurious `GET /branches` 403 → "Failed to load" toast on
  stock-locations/storage-bins/goods-receipts) + #298 (quick-sale uses the active branch). Design fact:
  **no cross-branch operation — every user works only within their own branch.**
- **Shifts, cash drawer & bank deposit.** core #525 (drawer-kick on cash sale), #532 (no-sale
  open-drawer + drawer-kick print), #531 (bank deposit + reconciliation + sales-by-terminal report);
  admin #289 (bank-deposit UI + reconciliation + report), #290 (PromptPay ทวนยอด manager key-in +
  owner report); terminal #16 (open-drawer + mid-shift cash-count), #18 (shift drawer/cash-count).
- **Per-branch cash safe (ตู้เซฟ).** SHIPPED design = core #544 (`branches.has_safe` toggle +
  `CASH_PICKUP` movement) + admin #296 ("มีตู้เซฟ" per-branch toggle in branch edit) + terminal safe UI.
  ⚠️ A parallel `CASH_DROP` implementation (core #546 / admin #297 / terminal #21) was a **duplicate and
  was REVERTED** (core #547 / admin #299 / terminal #22). Use the #544/#296 model.
- **Real-time PromptPay confirm (coffee-shop).** core #533 (personal PromptPay overrides gateway +
  credit-notification auto-confirm), #534 (real-time confirm + terminal routing + shift-close ทวนยอด);
  admin #281 (per-branch personal PromptPay QR config); terminal #17 (one-tap confirm + WS paid push +
  chime).
- **Table-QR pre-order + call-staff.** core #535 (staff accept-gate + acceptance expiry), #539 (product
  modifiers + table call-staff + strict branch-scoped user management); terminal #18 (table-QR queue).
- **Kitchen default station.** core #537 (route kitchen-unbound lines to a per-branch default station);
  admin #293 (default-station toggle).
- **Partial refunds (terminal).** terminal #14 (per-line partial refund) + #15 (poll/complete +
  amount-cap fixes).
- **Misc.** core #527 (pickup-queue handover `POST /orders/{id}/complete`), #530 (block PO of a
  manufactured product), #536 (fix payment-method read-back mislabel), #541 (per-request-tenant settings
  resolution), #545 (productKind list-filter test coverage — the filter contract itself shipped earlier
  in #245); money/qty thousand-separators core-side + admin #292 + terminal #20.

## Approved Parallel Work (owner-approved 2026-06-15)
**AccountZing-Platform/** — sanctioned standalone repo (central accounting ledger for POS108 +
108-Zing). Approved active work area in parallel with POS108. Follows its own docs
(`AccountZing-Platform/docs/SPEC-001-*.md`, `docs/build-plan/`, `.ai_context/HANDOFF.md`) and runs
its own git/CI/lifecycle — open a session with cwd = `AccountZing-Platform/` to work it. This is the
approved exception to the `accounting` entry in DO_NOT_TOUCH.md. Progress (updated 2026-07-02):
M0 (#1), M1/Slice 2A (#2), M2/SLA-engine Slice 3 (#3), M4/refund+appeal Slice 4 (#4), and
M5/settlement-engine Slice 5 (#5) merged to its `main`; latest merged work is #17 (settlement
eligibility bucketed by Bangkok date) plus deploy/CI hardening — #12 (prod container image + compose),
#9 (cargo-deny + Dependabot), #23/#24 (self-hosted runner, manual-only image build during dev).

**product-vision** — owner-approved 2026-06-20 standalone repo `Commerce-Platform/product-vision/`:
visual product lookup for POS108 (photo → SKU via image embedding + barcode fallback), Rust/Actix +
pgvector API + offline-first Flutter client. Own git/CI/lifecycle + docs (`docs/DESIGN.md`,
`.ai_context/HANDOFF.md`) — open a session with cwd = `Commerce-Platform/product-vision/`. Slice 1
**MERGED to main** (squash `f305505`, PR #1, 2026-06-21; branch cleaned): API (photo→SKU embedding NN
+ barcode + multi-shot enrollment + confidence cutoff + delta-sync) + offline-first Flutter; 20 unit
tests + real-pgvector integration test green in CI. **product-vision = standalone catalog master** for
the ~5k-SKU convenience-store MVP (POS108 link deferred/optional). Next: Slice 2 (real ONNX model).

**shop108 / 108 omnichannel commerce** — **APPROVED code-work area 2026-06-21** (owner green-lit C0 +
repo creation). GOAL: one merchant sells across 3 channels off one product/stock/ledger — offline
(pos108 POS) + online (shop108 storefront) + BipByte (social commerce); every channel books a pos108
sale via `POST /api/v1/sales` with a `channel` tag → single stock decrement + single GL/AccountZing.
Locked decisions: stock = shared pool (no pos108 schema change); BipByte calls pos108 API directly.
Building **C0** = auth glue (Identity service token) + small pos108 tweaks (Idempotency-Key,
`reference_type=SHOP108_ORDER`, add `Bipbyte` channel). pos108 changes go via PR through tmux `ting`
broker. Full design + pos108-inspection findings = `docs/shop108/` (`00-omnichannel-vision.md`,
`01-...stock-contract.md` v0.5, `03-path-to-first-sale.md`) — SoT, don't restate here.

## ⚠️ UNAPPROVED / UN-PUSHED local work — NOT on any approved-work list (status note, 2026-07-02)
Two services exist locally under `Platform-Services/` with **no git remote** (never pushed to GitHub),
recorded here only so future sessions don't mistake them for sanctioned scope:
- **`customer-plat`** — 10 commits (S1 genesis → S2 own DB → S3 HTTP + Identity EdDSA auth → S4
  cloud→branch replication outbox → S5 pull endpoint/denormalize, + a pull-cursor data-loss fix). Its
  own `.ai_context/HANDOFF.md` says "owner approved model A, requested a plan **before any scaffold**"
  and "push blocked by the exfil classifier (owner does it manually)" — i.e. the build ran **past** its
  own start-gate. pos108-side work sits on three branches pushed to origin but **unmerged to main**:
  `feat/customer-sync-applier`, `feat/customer-platform-pull` (DARK), `feat/customer-denormalize-refs`.
- **`platform-console`** — 3-commit Super-SA console scaffold, no remote.

These are **not owner-approved parallel work**. Do not treat them as active work areas or as delivered
without explicit owner approval + reconciliation (sanction & push, or discard).

## SCB QR Payment — Live Status (updated 2026-06-26 — DEPLOYED; historical snapshot)
> Note (2026-07-02): the snapshot below is ~5 weeks old relative to code HEAD (pos108 was ~#465 here,
> now #509) and payment work has moved far past SCB-only (2C2P card + BBL/GSB/BAY/TTB/KTB providers,
> Payment-Platform #57–#83). Physical `.68`/`.71`/`.76` deployment claims are live-state and not
> re-verified in this pass. Treat this section as the 2026-06-26 SCB history, not current status.
Goal: "POS รับรู้การชำระเงิน" (POS sees a real SCB QR payment settle to PAID). Full detail = memory `pos108-scb-gateway-bridge` (SoT). Snapshot — **DB-binding model SHIPPED + DEPLOYED (staging + PROD .68)**:
- **Config-driven `mode` flag REMOVED → per-branch DB-binding model** (pos108 **#446**, removes `APP_PAYMENT_GATEWAY__MODE`): a branch gets QR by being **assigned an SCB QR app** in admin — no env flag; no selection ⇒ cash. **SA-only gate** on branch↔app assignment (**#450**). pos108 still holds zero payment creds.
- **Admin SCB QR-app program shipped:** catalog **list** (pos108-admin **#243**) + per-branch app **selector** in branch edit (**#245**) + catalog **Add/Edit modal** (**#247**); pos108 cloud **cred-edit relay** (**#457**). Owner screenshot-confirmed live (HQ → Payment Gateway → SCB QR app catalog; "GW Debug"/"POS108 Cloud Relay"/"Pos108 slim" all Active).
- **Gateway SCB QR mint LIVE + PROVEN on staging** (Payment-Platform **#37**: `scb_ref` uppercase `[A-Z0-9]` — real 8101 cause was ref2=lowercase UUID; ref3=`MRC`+value; `transaction_id` Optional; `gateway_reference`=ref3). Real "Pos108 slim" sandbox creds + biller keyed; login→create→confirm = 200 + real EMVCo qr_payload. nginx `pay.staging.108plaza.net/callback` vhost+TLS LIVE; gateway→cloud webhook DELIVERING.
- **DEPLOYED — two paths (pos108 #452/#455):** staging via the `deploy/staging` branch K3s image build (latest live: **cloud `sha-8da39c1` (#457)** + **admin `sha-bb7f67c` (#247)**, rolled out 2026-06-26 — supersedes the #454 `sha-bc5ab1b`/`sha-cc41d2a`; ⚠️ GHCR push auth-flakes intermittently → `gh run rerun --failed` re-pushes the already-built image); **PROD .68 via build-and-copy** — build the `pos108` binary on macstudio (.76, the dev box) → `scp` to .68 → swap + restart, keep `pos108.rollback-<sha>` (probed .68: binary mtime 16:36, running since 16:37; **#456** marks task complete). **.68 is binary-only** — its stale `.git` (HEAD `45ba9bd`, didn't reflect the copied-in binary) was removed 2026-06-26; track the deployed sha on macstudio, not via `git log` on .68. The .68 branch backend (kiosk .71 → `.68:5300`) now runs the DB-binding binary — this **resolves the earlier "blocked: config-driven 45ba9bd, fork A/B" state** by taking path (B).
- Remaining = **operational verification only** (not code/deploy): mint a QR on a real branch device, pay it in the **SCB EASY Simulator** → capture→callback→cloud-webhook→nudge→branch reconcile→PAID→POS. SCB sandbox biller validity is the one external unknown.
- Side PRs: **pos108 #447** (cloud DB pool 100→25), **pos108-admin #244** (branch-context HQ/operational menu).
- **Per-credential `ref3_prefix` MERGED 2026-06-27** (3-repo chain, order #46→#465→#254; branches cleaned): gateway [Payment-Platform #46](https://github.com/108-Plaza/Payment-Platform/pull/46) (migration 0018, lifts hardcoded `MRC` → per-credential so a 2nd biller with a different SCB "Reference 3 Prefix" can mint without 8101) + pos108 cloud relay [#465](https://github.com/108-Plaza/pos108-core/pull/465) + admin Ref3-prefix form field [pos108-admin #254](https://github.com/108-Plaza/pos108-admin/pull/254). Blank ⇒ gateway defaults `MRC`. Code merged; **deploy = separate owner step**. (The SCB callback-never-arrives blocker above is unrelated/unchanged.)

## Current Rule
All implementation work must stay inside the active work area unless the user explicitly approves a scope change.

## Next Action
Read the active sub-project handoff and continue only from that milestone.

## Scope Change Process
Before changing active work area:
1. Explain why the scope change is needed.
2. List files/platforms affected.
3. Ask for explicit approval.
4. Update this file and root HANDOFF.md after approval.
