# Data Platform (Stratum) — Connection Survey

> Read-only survey of how the Data Platform connects to the rest of the 108 Ting Ecosystem.
> Verified against actual code/config 2026-06-14. Umbrella doc (this folder is NOT a git repo).
> Source of truth for code = `Data-Platform/` @ `main 3b4f6f1`, working tree clean.

## TL;DR

- **Stratum's connection surface is small and deliberate:** 2 real inbound paths
  (POS108 HMAC webhook + 108Zing Kafka tap), 1 testing inbound (generic `/ingest`),
  and a read/query serving API outbound — plus infra (object store, 2× Postgres, bus, Prometheus).
- **Two inbound paths are built; one carries verified live traffic:** the 108Zing
  `realtime.events` tap (canary-green). The POS108 catalog pipe is **built end-to-end
  in code** — product/category emitters ARE wired into the write paths (in-tx
  `outbox::append`), brand is snapshot-only by design (no emitter), the HMAC dispatcher
  is live, and Stratum DP-1..DP-4 are merged. What's missing to make it flow in prod =
  the ops subscription/secret exchange, the backfill/replay APIs (POS-4/5 + DP-5), and a
  consumer that actually reads it. **[CORRECTED 2026-06-14: an earlier draft wrongly said
  "no emitters / Checkpoint 3 unbuilt" — code-verified false; see
  `DATA_PLATFORM_INTEGRATION_TARGET_VS_ACTUAL.md`.]**
- **The outbound serving API (`GET /catalog`, `POST /query`) is LIVE but has zero
  ecosystem consumers wired** — the 108Zing "Z-track" DP client is planned, not built.
- **Most platforms are NOT connected** to the Data Platform (Logistics, Payment,
  Creator, IoT, Identity, pos108 slot-api). Notification publishes to Kafka but to a
  different topic Stratum doesn't tap.

---

## 1. Stratum's connection surface (the hub)

Repo: `Data-Platform/` (Rust workspace, codename Stratum). API crate = `stratum-api`.

### INBOUND — producers → Stratum

| # | Path / mechanism | Transport | Auth | Source | Status | Code |
|---|---|---|---|---|---|---|
| 1 | `POST /ingest/pos108/webhook` | HTTP | **HMAC-SHA256** `X-POS108-Signature: t=,v1=` (skew 300s) | POS108 catalog dispatcher | **Built end-to-end** (product/category emitters wired; brand snapshot-only). Ops subscription pending; no consumer reading yet | [stratum-api/src/lib.rs:288](../Data-Platform/crates/stratum-api/src/lib.rs#L288), mapper [pos108.rs](../Data-Platform/crates/stratum-ingest/src/pos108.rs) |
| 2 | `stratum tap` consumer | **Kafka** topic `realtime.events`, group `stratum-zing-tap` | consumer-group offset | 108Zing realtime backbone | **LIVE — canary-green** | [zing_tap.rs](../Data-Platform/crates/stratum-ingest/src/zing_tap.rs), mapper [zing.rs](../Data-Platform/crates/stratum-ingest/src/zing.rs) |
| 3 | `POST /ingest/:id` | HTTP | Bearer (`jwt_secret`) | any JSON producer | scaffolded / testing | [stratum-api/src/lib.rs:258](../Data-Platform/crates/stratum-api/src/lib.rs#L258) |

- **Webhook contract (1):** envelope `id/type/schema_version/source/tenant_id/occurred_at/data`;
  `type` → collection.action; `created→Insert, updated→Update, discontinued→Delete`;
  datasets `pos108.products` / `pos108.categories`; stamps headers `entity_version` (from
  `data.version`), `watermark`, `tenant_id`. Bronze-first, then version-guarded serving applier.
- **Tap contract (2):** 108Zing envelope (schemaVersion 2, camelCase); one Bronze dataset per
  event type (`<type>` dots→`_`, camelCase→snake → `zing.<...>`); unknown well-formed types still
  map (flagged). Only topic Stratum consumes = `realtime.events` (default
  [stratum-config/src/lib.rs:474](../Data-Platform/crates/stratum-config/src/lib.rs#L474)).

### OUTBOUND — Stratum → consumers

| # | Path | Transport | Auth | Returns | Status | Consumers wired? |
|---|---|---|---|---|---|---|
| 1 | `GET /catalog/:collection` (+ `/:id`) | HTTP | Bearer | read model: `doc` jsonb + `entity_version` + `watermark` + `tenant_id`; keyset paging | **LIVE (DP-4)** | **none yet** |
| 2 | `POST /query` | HTTP | Bearer | DataFusion SQL over lakehouse | LIVE | none yet |
| 3 | `GET /metrics`, `/health` | HTTP | public | Prometheus text / `ok` | LIVE | Prometheus |

- Serving store = Postgres `serving` schema (`products`/`categories`/`brands`), one row per
  `(tenant_id, id)`, version-guarded upsert. Code [stratum-serve/src/lib.rs](../Data-Platform/crates/stratum-serve/src/lib.rs).

### INFRA connections

object store (local/S3/GCS/Azure via `object_store`), catalog Postgres (`[catalog]`),
serving Postgres (`[serving]`), internal event bus (memory | Kafka `[stream]`, topics
`stratum.<dataset>`), in-process Prometheus. Config: `stratum.toml.example` / `.env.example`
(`STRATUM_*` env, `__` nesting).

### Key external-connection config

`STRATUM_POS108__WEBHOOK_SECRET` (+`MAX_SKEW_SECS`), `STRATUM_ZING__{BROKERS,TOPIC,GROUP}`,
`STRATUM_SECURITY__JWT_SECRET` (serving/query bearer), `STRATUM_SERVING__DATABASE_URL`
(+`AUTO_MIGRATE`/`APPLY_ENABLED`), `STRATUM_CATALOG__DATABASE_URL`, `STRATUM_STORAGE__*`,
`STRATUM_STREAM__{BACKEND,BROKERS}`.

---

## 2. The ecosystem ends — who actually connects

| Repo | Direction | Transport | Status | Evidence |
|---|---|---|---|---|
| **Commerce / pos108 api** | → DP (produce) | HMAC webhook dispatcher + transactional outbox | **producer BUILT** — product/category emitters wired in-tx (`outbox::append`); egress allow-list projection live; brand snapshot-only (no emitter). Missing = ops subscription + backfill (POS-4/5) + consumer | emitters `pos-commerce/.../infrastructure/product/mod.rs:153,422,475` + `category_repository.rs:97,272,353`; payload `pos-commerce/.../application/events/catalog.rs`; dispatcher `pos-integration/.../webhook_dispatcher.rs:65`; constants `pos-infra/src/domain_event.rs:119` |
| **108Zing / server** | → DP (produce) | Kafka `realtime.events` (9 services w/ `realtime_outbox`, schemaVersion 2) | **LIVE — Wave 1 canary-green** (`zing.social_followed`, `zing.notification_created` → Bronze) | `shared-kernel/src/realtime/event_type.rs`; docs `DATA_PLATFORM_MIGRATION_PLAN.md`, `PLATFORM_BRIDGE_PLAN.md` |
| **Platform-Services / Media (live-service)** | → DP (produce) | Kafka `realtime.events` backbone (`realtime_event_topic`) | LIVE producer; in tap scope if same cluster | `live-service/src/main.rs:118` (`config.realtime_event_topic`), `tests/pg_repo.rs` (`PgRealtimeOutboxSink`) |
| 108Zing / realtime-edge | — (sibling) | Kafka `realtime.events` consumer (`phoenix-edge`) | **sibling consumer of the SOURCE topic, NOT a Stratum downstream** | `realtime-edge/README.md:3` |
| Platform-Services / Notification | — (not connected) | Kafka publisher to topic **`notifications`** (default) | **NOT tapped by Stratum** (Stratum only taps `realtime.events`) — potential future tap | `Notification/backend/crates/api/src/config.rs:135` |
| Commerce / pos108 slot-api | none | — | not connected | no stratum/ingest/outbox refs |
| Logistics / delivery | none | — | not connected | — |
| Payment / Gateway | none | — | internal outbox + SCB inbound webhook only | `backend/crates/api/src/{relay.rs,scb_callback.rs}` |
| Creator / creator | none | — | Omise payment webhooks only | `platform-api/src/routes/membership.rs` |
| IoT / Smart-Home, Smart-Farm | none | — | MQTT, deliberately not Kafka | `Smart-Home/docs/adr/0003-mqtt-edge-gateway-emqx.md` |
| Platform-Services / Security (identity) | none | — | not connected | — |

---

## 3. Connection map (live vs planned)

```
 PRODUCERS                         STRATUM (Data Platform)              CONSUMERS
 ─────────                         ───────────────────────              ─────────
 108Zing server (9 svc) ┐
 Media/live-service     � ──Kafka realtime.events──▶ [zing tap] ──▶ Bronze→Silver→Gold
                        ┘   (LIVE, canary-green)                       │
                                                                       ├─▶ GET /catalog  ◀╴╴ (no consumer
 POS108 api ──HMAC webhook /ingest/pos108/webhook──▶ [webhook]         │     (LIVE)          wired yet)
   (pipe LIVE, NO catalog events emitted yet)         │                ├─▶ POST /query   ◀╴╴ (Z-track
                                                      └──▶ serving PG ──┘     (LIVE)          planned)

 NOT CONNECTED: Notification (other topic), realtime-edge (sibling consumer of source),
                slot-api, Logistics, Payment, Creator, IoT, Identity.
```

## 4. Gaps / what "continue" means next

1. **POS108 catalog producer (Checkpoint 3)** is the missing half of the webhook pipe —
   `product.*`/`category.*`/`brand.*` emitters don't exist; until built, the HMAC webhook
   carries no catalog traffic. Cross-repo, POS108-side — **do not touch pos108 without approval.**
2. **No consumer reads the serving API** — outbound `GET /catalog` is LIVE but unused; the
   108Zing "Z-track" DP-client-behind-a-port + shadow-compare + flip is planned, not built.
3. **Notification → Stratum** would just need a tap on the `notifications` topic (or repoint it
   to `realtime.events`) — currently a non-connection, not a broken one.
4. **Wave 2 (financial CDC)** for 108Zing money DBs + reconciliation/PII gates — not started.

See per-track detail: `Data-Platform/.ai_context` (DP-1..5, B3 tap, Commerce-ERP plan),
`Data-Platform/HANDOFF.md`, and the POS108 / 108Zing docs cited in §2.
