# Data Platform — Target vs Actual Integration Audit

> Read-only audit. Continues from `DATA_PLATFORM_CONNECTIONS_SURVEY.md`.
> Target = intended design from architecture docs; Actual = code/git verified 2026-06-14.
> No code changed. Umbrella doc (this folder is not a git repo).

## Sources
- **Target:** `Data-Platform/docs/{DATA_PLATFORM_ARCHITECTURE.md (§5 ownership, §11 roadmap),
  ECOSYSTEM_TARGET_ARCHITECTURE.md (§7 consumers, §13 12 gaps), COMMERCE_ERP_INTEGRATION_PLAN.md
  (§5 10-event contract, §8 gates, §9 phases A–D), PHASE1_IMPLEMENTATION_PLAN.md (§7 PR backlog,
  §5 G1–G7 gates)}`.
- **Actual:** Data-Platform git `main 3b4f6f1` + `.ai_context`; POS108 `pos108/api` code; ecosystem repo grep.

> ⚠️ **Doc-vs-code drift found & resolved by code:** `PHASE1_IMPLEMENTATION_PLAN.md` still reads
> "PLANNING ONLY — all 15 PRs NOT STARTED", but git/code prove DP-1..DP-4 merged **and** POS108
> product/category emitters built. The prior survey made the opposite error (trusting POS108's
> stale `DATA_PLATFORM_SEPARATION_PLAN.md` "Checkpoint 3 unbuilt"). **This audit uses code/git as
> the source of truth.** Status legend: **LIVE** = built+deployed+traffic · **PARTIAL** = built in
> code, not fully operational/consumed · **PLANNED** = designed, not built · **NOT CONNECTED**.

---

## 1. Integration Matrix

### 1A. Producers → Stratum (inbound)

| System / repo | Target connection | Actual connection | Status | Evidence | Blocker |
|---|---|---|---|---|---|
| **POS108 — catalog (product, category)** | Owner of catalog; emit `product.*`/`category.*` via outbox→HMAC webhook→`/ingest/pos108/webhook`; Stratum serves read model | **Producer BUILT & wired:** in-tx `outbox::append` on every product/category create/update/discontinue; egress allow-list projection; HMAC dispatcher live. Stratum DP-1/2/3/4 **merged**. | **PARTIAL** (built end-to-end; consumer shipped Z-1/Z-2; backfill shipped both ends — POS-4/5 #279 + Stratum DP-5 #33; remaining = ops subscription only) | emitters `pos-commerce/.../infrastructure/product/mod.rs:153,422,475`, `category_repository.rs:97,272,353`; payload `application/events/catalog.rs`; dispatch `pos-integration/.../webhook_dispatcher.rs:52,65`; backfill API `pos108 src/presentation/http/platform_catalog.rs` (#279) + CLI `Data-Platform crates/stratum-cli/src/backfill.rs` (#33) | ops subscription + secret exchange (pending) — backfill path now complete |
| **POS108 — brand** | brand read model in Stratum | snapshot-only by design — **no emitter** (grep empty); served left-join in DP-4 | **PARTIAL (by design)** | no `BRAND_*` emitter in `pos-commerce/src`; DP arch §13 gap #2 | brand has no CRUD/authoring API (gap #2) |
| **POS108 — order / payment / inventory / warehouse / pricing / promotion / customer** | 10-event canonical contract → Stratum (Phase A/A.5/B); order→Gold sales, etc. | pos108 emits sales/inventory.stock/purchasing/etc. to its **own** outbox subscribers (branch sync), but **no DP mapping/contract** in Stratum for them | **PLANNED** (Phase A.5/B) | `COMMERCE_ERP_INTEGRATION_PLAN.md §5,§9`; Stratum `stratum-ingest` has only `pos108`(catalog)+`zing` mappers | cross-repo build; financial needs PII+reconciliation gates (Phase C) |
| **POS108 — accounting / ledger (real money)** | CDC + reconciliation gate → Gold analytics only (Phase C) | not started | **PLANNED (blocked)** | `§8` PII+reconciliation gates; `stratum-cdc` = polling baseline, no delete capture | gates not built; **DO NOT TOUCH: GL/accounting forbidden scope** |
| **108Zing / server — behavioral (engagement, social, content, live, moderation, notification)** | Owner; publish `realtime.events` → Stratum tap → Bronze→Silver→Gold analytics | **LIVE:** 9 services w/ `realtime_outbox` → `realtime.events`; Stratum `stratum tap` (group `stratum-zing-tap`) canary-green; Bronze→Silver→Gold pipelines live (notification/social) | **LIVE** (Wave 1) | `shared-kernel/.../realtime/event_type.rs`; Stratum `zing_tap.rs`, `zing.rs`, `deploy/staging-canary/`; `.ai_context` B3.x | none for Wave 1 |
| **108Zing — financial (wallet/ledger/payment/payout/earning)** | CDC + reconciliation (Wave 2 / Phase C) | not started (no CDC role, no reconciliation) | **PLANNED (blocked)** | `DATA_PLATFORM_MIGRATION_PLAN.md` Wave 2; `§8` gates | read-only CDC role (ops) + reconciliation + PII gates |
| **Platform-Services / Media — live-service** | live events via realtime backbone | publishes `realtime.events` (`config.realtime_event_topic`) → in tap scope | **LIVE** (via realtime.events) | `Media/live-service/src/main.rs:118`; `tests/pg_repo.rs` | same-cluster assumption |
| **Platform-Services / Notification** | notification analytics (target lists notification on realtime.events) | publishes Kafka topic **`notifications`** — Stratum does **not** tap it | **NOT CONNECTED** | `Notification/.../config.rs:135`; Stratum taps only `realtime.events` (`stratum-config/src/lib.rs:474`) | **topic decision** (tap `notifications` vs repoint to realtime.events) |
| **Logistics-Platform / delivery — fulfillment/shipment** | Owner of `shipment_created`/`shipment_delivered` → Stratum (Phase B, §5.3) | delivery repo has **zero** stratum refs (POS108↔delivery has its own `X-Platform-Key` delivery-status API, not a DP link) | **PLANNED / NOT CONNECTED** | `pos108 .../platform_delivery.rs` (separate); no stratum in `Logistics-Platform/delivery` | Phase B not started |
| **pos108 slot-api · Payment/Gateway · Creator/creator · IoT Smart-Home/Farm · Identity** | not in Phase 1–3 ownership matrix (some future) | no stratum refs; own internal outbox/webhooks (SCB/Omise) / MQTT only | **NOT CONNECTED** | repo grep; `Smart-Home/.../adr/0003-mqtt...` | out of current scope |

### 1B. Stratum → Consumers (outbound)

| Consumer / repo | Target connection | Actual connection | Status | Evidence | Blocker |
|---|---|---|---|---|---|
| **108Zing / commerce-bridge-service** | read catalog/price/availability from Stratum `GET /catalog/*`; shadow→flip→drop direct POS108 pull (Z-1..Z-4) | **Z-1+Z-2 SHIPPED** (2026-06-14): Z-1 client `6f0d232` #277; Z-2 shadow-compare `026950a` #278 — `COMMERCE_CATALOG_SOURCE` ∈ {`pos108` default, `data_platform`, `shadow`}. Shadow serves POS108 + compares DP (lag-aware) → `commerce_bridge_catalog_shadow_*` metrics. **Z-3 flip / Z-4 cleanup pending** (gated on G-6 soak of the shadow metric) | **PARTIAL** | `infrastructure/{data_platform_client,shadow_catalog_client}.rs`; `COMMERCE_CATALOG_SOURCE`/`DATA_PLATFORM_*`/`CATALOG_SHADOW_LAG_SECS` | run shadow soak ≥7d → Z-3 flip (DP primary + fallback) → Z-4 drop POS108 pull |
| **Analytics / BI** | read Gold marts via `/query` (DataFusion); JDBC/Arrow Flight future (Phase 3) | `/query` **live** (bearer); no BI tool/dashboard wired; per-consumer dashboards missing | **PARTIAL** | `stratum-api` `/query`; gap #10; `§8` "per-consumer dashboard missing" | BI connectivity + dashboards |
| **Mobile BFF · marketplace · supplier · partner · affiliate · AI** | per-consumer scoped reads (Phase 4–5) | none | **PLANNED** | `ECOSYSTEM_TARGET_ARCHITECTURE.md` App.D | Phase 4 needs per-consumer auth/quota/entitlements (gap #5) |
| 108Zing / realtime-edge | — (not a DP consumer) | consumes `realtime.events` **direct from source bus** (sibling of the tap) | **N/A** | `realtime-edge/README.md:3` | — |

---

## 2. Gap List (ranked)

### P0 — unblock Phase 1 value (catalog read path end-to-end)
1. **108Zing catalog consumer / client (Z-1..Z-4) — `Z-1+Z-2 DONE`, Z-3..Z-4 remain.**
   ~~Stratum's `GET /catalog/*` is LIVE but nothing reads it.~~ **Z-1 shipped** (`6f0d232` #277):
   `HttpDataPlatformCatalogClient`, source-selectable. **Z-2 shipped** (`026950a` #278):
   `COMMERCE_CATALOG_SOURCE=shadow` serves POS108 while comparing the Data Platform (lag-aware,
   `CATALOG_SHADOW_LAG_SECS`) → `commerce_bridge_catalog_shadow_{compared,match,mismatch,suppressed,
   secondary_error}_total` metrics. **Remaining:** run the shadow soak ≥7d (G-6: mismatch rate ≈0
   after lag suppression) → Z-3 flip (`data_platform` primary + POS108 fallback) → Z-4 drop the direct
   POS108 pull. **Ops next:** deploy with `COMMERCE_CATALOG_SOURCE=shadow` + `DATA_PLATFORM_*` set, then
   watch the shadow metrics.
2. **POS108 catalog producer — `PARTIAL`, finish the last mile (NOT a rebuild).**
   Emitters + dispatcher + projection are built. Remaining: (a) **ops** — register the Stratum
   webhook subscription + exchange the HMAC secret so events actually flow; (b) ~~POS-4/POS-5 bulk
   snapshot + replay APIs + DP-5 CLI~~ **SHIPPED** — pos108 `fbc0741` #279 (`GET /api/v1/platform/
   catalog/{replay, snapshot/products, snapshot/categories}`, X-Platform-Key, incl. soft-deleted)
   + Stratum `stratum reseed`/`replay` #33 (consumes them via the same mapper+applier as live
   ingest). **Backfill path complete**; only the ops subscription remains; (c) brand authoring API
   only if brand events are ever needed (else keep snapshot-only).
3. **Stratum production hardening.** Serving store (DP-3, Postgres) already closes the "DataFusion-
   per-request" gap #4. Remaining before prod: **per-consumer auth/quota/entitlements** (gap #5 —
   single shared bearer today), **TLS termination** (S3 remainder), **Kafka SASL/TLS + runtime
   hardening**, and a **real deploy target** (canary runs local-FS storage + in-memory bus, not prod infra).

### P1 — next wave
4. **Financial CDC (108Zing Wave 2 / POS108 Phase C) — `PLANNED (blocked)`.** Needs read-only CDC role
   (ops) **and** the two cross-cutting gates that don't exist yet: **PII governance gate** + **reconciliation
   gate** (§8). Gold financial data must not go live until both pass. (Accounting/GL stays forbidden scope.)
5. **Notification topic decision — `NOT CONNECTED`.** Platform-Services/Notification publishes to
   `notifications`, which Stratum doesn't tap. Decide: add a Stratum tap on `notifications`, or repoint
   the service to the `realtime.events` backbone. One-line config/ownership call, not a broken pipe.
6. **Consumer dashboard / BI — `PARTIAL`.** `/query` is live but there are no dashboards and no
   per-consumer lag/error monitoring (required by §10.4 go-live + G-5 read-path smoke). Add BI
   connectivity (JDBC/Arrow Flight) + per-consumer Grafana.

### P2 — later phases / out of current scope
7. **Logistics shipment events → Stratum (Phase B)** — Logistics owns fulfillment per §5.3 but
   `delivery` repo isn't connected; build `logistics.shipment*` emitters + Stratum mapper when Phase B starts.
8. **Payment / Creator / IoT / Identity / slot-api** — not in Phase 1–3 ownership matrix; revisit per
   business demand (Payment summary via POS108; identity federation = gap #6, future POS108↔108Zing work).

---

## 3. Diagram (Mermaid)

```mermaid
flowchart LR
  subgraph Producers
    Z9["108Zing · 9 services"]
    LV["Media · live-service"]
    POSc["POS108 · product/category emitters"]
    POSt["POS108 · order/financial/inventory"]
    ZF["108Zing · financial svc"]
    LOG["Logistics · delivery"]
    NOT["Notification svc"]
  end

  subgraph Stratum["STRATUM (Data Platform)"]
    TAP["zing tap (realtime.events)"]
    WH["/ingest/pos108/webhook (HMAC)/"]
    MED["Bronze → Silver → Gold"]
    SERV["serving store (PG)"]
    CAT["GET /catalog/* (read API)"]
    QRY["POST /query (SQL)"]
  end

  subgraph Consumers
    BRIDGE["108Zing commerce-bridge"]
    BI["Analytics / BI"]
    EXT["mobile BFF / partner / supplier / affiliate / AI"]
  end

  %% LIVE (solid)
  Z9 -->|realtime.events LIVE| TAP
  LV -->|realtime.events LIVE| TAP
  TAP --> MED
  MED --> QRY
  POSc -->|HMAC webhook · built, ops-pending| WH
  WH --> SERV
  SERV --> CAT

  %% PLANNED (dashed)
  POSt -. Phase A.5/B .-> WH
  ZF -. CDC Wave2/Phase C .-> MED
  LOG -. shipment events Phase B .-> WH
  CAT -. Z-track not built .-> BRIDGE
  QRY -. no dashboards yet .-> BI
  CAT -. Phase 4-5 .-> EXT

  %% DISCONNECTED
  NOT -. topic 'notifications' NOT tapped .-> Stratum

  classDef live fill:#1b5e20,stroke:#0b3d12,color:#fff;
  classDef planned fill:#5d4037,stroke:#3e2723,color:#fff,stroke-dasharray:4 3;
  classDef disc fill:#424242,stroke:#212121,color:#bbb;
  class Z9,LV,TAP,WH,MED,SERV,CAT,QRY,POSc live;
  class POSt,ZF,LOG,BRIDGE,BI,EXT planned;
  class NOT disc;
```

*Solid = built/flowing (POS108 catalog producer leg is built but ops-pending — annotated).*
*Dashed = planned. realtime-edge (sibling consumer of the source bus) omitted — not a DP edge.*

---

## 4. Final Recommendation

**The Data Platform is further along than either planning doc admits — the bottleneck is the
consumer, not the producer.** Both inbound paths are built (108Zing tap is LIVE; POS108 catalog
producer + Stratum ingest/serving/read-API are merged and code-complete). The catalog initiative
is stalled on its **last 10%**, all on the read side.

**Do, in order:**
1. **Light up one end-to-end catalog flow (P0-1 + P0-2 ops):** register the Stratum webhook
   subscription + secret in a staging tenant, confirm `product.*`/`category.*` land in the serving
   store, then build **Z-1** (DP catalog client behind the existing port) and run **Z-2 shadow-compare**.
   This converts a large amount of already-built infra into the first real consumer — highest value, lowest new code.
2. **Backfill safety net — COMPLETE.** POS-4/5 (pos108 #279) + DP-5 `stratum reseed`/`replay`
   (Data-Platform #33) ship both ends: a consumer can now bootstrap/rebuild the serving store from the
   authoritative snapshot and recover from outbox-retention loss — backstopping a trustworthy flip (G-6).
3. **Close production hardening (P0-3) before any external consumer:** per-consumer auth/entitlements
   (gap #5) + TLS + a real deploy target. The single shared bearer is fine for internal 108Zing, **not**
   for partner/supplier/affiliate (Phase 4).
4. **Defer P1/P2.** Financial CDC stays blocked until the PII + reconciliation gates exist — build those
   gates first, treat Gold-financial as gated. Make the Notification topic call (cheap). Everything else
   (Logistics/Payment/Creator/IoT/Identity) waits for its phase.

**Cross-repo guardrail:** POS-4/5 touch the **POS108 repo** and Z-1..Z-4 touch the **108Zing repo** —
both outside the current active area (`Commerce-Platform/pos108/api` is in-scope for POS-4/5; 108Zing is
**not**). Per project rules, **stop and get explicit approval before editing 108Zing**, and keep
accounting/GL/finance untouched (forbidden scope). This audit changed no code.
