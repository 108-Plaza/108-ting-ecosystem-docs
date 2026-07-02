# Quick Sale Groups — Slice 2: offline branch sync (design)

> **Status:** DESIGN (2026-07-01). Slice 1 (groups model + API, cloud-side) is merged
> (#504). This slice projects quick-sale config down to the **branch-local DB** so the
> **offline terminal** renders groups without a live cloud call.
> _Confirmed 2026-07-02:_ #504 is on pos108 `main` (`eaa870b`); Slice 2 (this doc — offline
> branch-sync projection) is **not yet built** (no `project_master_data` quick_sale arm / no
> `MASTER_DATA_TYPES` quick_sale entry in `main`).
>
> ⚠️ This touches the **OWNER-LOCKED offline-first sync layer** (memory
> `pos108-offline-first-data-platform-baseline`, 7 invariants). Build in small,
> verified steps with review — do not rush.

## 1. How master-data sync works today (grounded)

Cloud → branch replication for catalog master data:

1. **Produce** — catalog writes build an event payload via
   `crates/pos-commerce/src/application/events/catalog.rs`
   (`product.created/updated/discontinued.v1`, `category.*`, `brand.*`; event-type
   constants in `pos_infra::domain_event::event_types`) and **append** it to
   `outbox_events` (`infrastructure::messaging::outbox::append()`), in the same tx
   as the write.
2. **Ship** — the `OutboxRelay` delivers; the branch pulls events into its local
   `sync_inbox`.
3. **Apply** — `SyncInboxApplier` (`src/application/services/sync/inbox_applier.rs`):
   if `MASTER_DATA_TYPES.contains(aggregate_type)` → `project_master_data(row)`,
   which **upserts a single row** (`jsonb_populate_record` into `products` /
   `categories`) or applies a `*.discontinued.v1` tombstone (targeted UPDATE).
   `MASTER_DATA_TYPES = ["product","category","branches"]`.

Key properties: single-row upsert, **tenant-wide** (every branch projects the same
catalog), idempotent, dead-letters after `INBOX_MAX_ATTEMPTS`.

## 2. Why quick-sale is different (the two hard parts)

1. **Per-branch, not tenant-wide.** A `quick_sale` config belongs to ONE branch;
   only that branch should receive it. Catalog events broadcast to all branches —
   quick-sale must be **branch-targeted**. → Need to confirm/extend how a
   branch-scoped master-data event is routed to exactly that branch's inbox (vs the
   tenant-wide catalog fan-out).
2. **Multi-row aggregate.** A config = `quick_sale_configs` (1) + `quick_sale_groups`
   (N) + `quick_sale_entries` (N). `project_master_data` only does single-row
   upserts. Quick-sale needs a **full-replace projection** (the same atomic
   delete-then-insert the cloud `replace_config` does), inside the inbox tx.

## 3. Proposed design

### 3.1 Event (production — cloud `QuickSaleService::replace_config`)
Append one outbox event **in the existing replace tx** (so it ships iff the write
commits):
- `aggregate_type = "quick_sale"`
- `aggregate_id   = branch_id` (the config's natural key within a tenant; configs
  have a composite PK `(tenant_id, branch_id)`, so `branch_id` identifies it)
- `event_type     = "quick_sale.replaced.v1"`
- `schema_version = 1`
- `payload` = the full config: `{ branchId, isEnabled, version, groups:[{id,name,position,isActive}], entries:[{productId,position,isVisible,groupId}] }`
  (reuse `config_to_json` shape).

Add `"quick_sale"` to `MASTER_DATA_TYPES`.

### 3.2 Projection (branch — new `project_master_data` arm)
For `aggregate_type == "quick_sale"`, apply a **full replace** in the inbox tx:
1. UPSERT `quick_sale_configs (tenant_id, branch_id)` from the payload (set
   `is_enabled`, `version`, `updated_at`). Use the payload's `version` as the row's
   version (cloud is the source of truth; no optimistic-lock on the branch).
2. `DELETE FROM quick_sale_entries WHERE tenant_id=? AND branch_id=?`
3. `DELETE FROM quick_sale_groups   WHERE tenant_id=? AND branch_id=?`
4. INSERT groups, then entries (with `group_id`), from the payload.
   Allocate fresh entry ids on the branch (ids are server-side; not in the payload),
   keep group ids from the payload (entries reference them).

Idempotent: re-applying the same envelope yields the same final state (full replace).
Ordering: a newer `version` always wins; if envelopes arrive out of order, guard with
`WHERE EXCLUDED.version >= quick_sale_configs.version` on the config upsert and skip
the row-replace when the incoming version is older (avoids an older replace
clobbering a newer one).

### 3.3 No tombstone needed
`replace_config` already represents "disabled/empty" as a config with
`isEnabled=false` / no entries — a normal replaced.v1 envelope covers it. (No
`quick_sale.discontinued` path.)

## 4. Open questions

### Q1 — Branch-targeted routing — **RESOLVED (2026-07-01)**

The branch pulls via `POST /api/sync/pull` → `SyncPullService::changes_since`
(`src/application/services/sync/pull.rs`). It reads `outbox_events`
`WHERE aggregate_type = ANY(collections) AND created_at > cursor` — **and does NOT
filter by branch** (`let _ = branch_id; // reserved — collection subscriptions could
be branch-scoped later`). So master data is **tenant-wide pull**: every branch that
subscribes to a collection gets every event of that type. Fine for catalog
(product/category are tenant-wide); **wrong for a per-branch config** — branch B would
receive + project branch A's quick-sale.

→ **Per-branch routing does not exist today; it must be added deliberately.** The code
already anticipates exactly this ("collection subscriptions could be branch-scoped
later"). Two options:

- **(A) Branch-scoped pull filter (recommended — the reserved path).** Mark
  `quick_sale` as a *branch-scoped* collection; in `changes_since`, for such
  collections add `AND payload->>'branchId' = $branch_id::text`. The branch pulls only
  its own config. Touches the locked pull query, but it is the extension the comment
  reserved, and keeps bandwidth minimal.
- **(B) Branch-side projection guard.** Keep the tenant-wide pull; in
  `project_master_data`, skip a `quick_sale` envelope unless `payload.branchId` equals
  this node's branch. Lower blast radius on the pull, but every branch pulls every
  branch's quick-sale events, and `inbox_applier` currently has **no node `branch_id`**
  to compare against (would need plumbing).

**Recommendation: (A)** — it matches the reserved design, filters at the source, and
needs no new node-identity plumbing. It is an **extension of the OWNER-LOCKED pull
contract**, so it needs owner sign-off before coding.

### Q2 — Reconciliation — **RESOLVED (2026-07-01)**
`reconciliation.rs` counts the cloud side via `reconcile_count(branch, collection)`,
whose query is `FROM outbox_events WHERE aggregate_type = $1` — **counted by type, NOT
by branch**. With option-A's branch-scoped pull, the branch pulls only its own
quick-sale events, so `branch_count < cloud_count` (cloud counts every branch's) →
**a false reconciliation diff**. → The same branch filter (`payload->>'branchId' =
branch`) **must also be applied to the cloud `reconcile_count` query** for
branch-scoped collections. (Confirmed touch point; without it reconciliation alarms.)
`duplicate_counter` keys on `event_id` so it is unaffected.

### Q3 — Initial backfill — **RESOLVED (2026-07-01)**
The outbox **is purged** (B9 `outbox_purge` worker trims terminal rows past retention).
So a branch enrolling *after* a config was set — and after that event aged out — pulls
from epoch and gets **nothing** → its quick-sale would be empty until the next cloud
edit. Catalog solves this with a **B8 snapshot** (authoritative paginated read).
→ For quick-sale v1 (one small per-branch config), the cheapest robust fix is to
**re-emit the current config to the outbox when a branch enrolls** (a fresh, unpurged
`quick_sale.replaced.v1` the branch then pulls) — no new snapshot endpoint needed.
Alternative: a tiny snapshot read on first sync. Pick re-emit-on-enrollment for v1.

### Q4 — schema_version — `sync_inbox.schema_version` is `SMALLINT` (i16); keep v1.

## 4b. BETTER APPROACH FOUND (2026-07-01) — mirror the `"branches"` precedent
`default_pull_collections` already includes **`"branches"`** — a **per-branch config**
(PromptPay) that pulls **tenant-wide** (no pull filter) and projects via a
**branch-id-keyed UPDATE that no-ops when the branch row isn't local** (a slim node
only holds its own branch). This is the same per-branch-config-over-tenant-wide-pull
problem quick-sale has — solved **without** any `changes_since`/`reconcile` change.

→ **Revised plan (supersedes option A):** mirror `"branches"`.
- Pull `quick_sale` **tenant-wide** (no `changes_since` filter, no `reconcile_count`
  filter — counts still match because the branch pulls every event and just no-ops
  foreign ones). **Drops touch points #2 and #3 (the riskiest locked-pull changes).**
- Projection guards with **"branch exists locally"**: apply the full-replace only when
  `payload.branchId` exists in the local `branches` table — foreign-branch events
  no-op on a slim node (exactly how `"branches"`/PromptPay behaves).

**Revised touch points (5, not 7):** #1 emit · #4 projection (guarded full-replace) ·
#5 `MASTER_DATA_TYPES` · #6 `pull_collections` · #7 enrollment re-emit (backfill).
Only #4/#5 touch the LOCKED inbox (where `"branches"` already lives) — much lower risk.

## 4a. Original option-A touch points (superseded by §4b)
1. `QuickSaleService::replace_config` — `outbox::append("quick_sale", branch_id,
   "quick_sale.replaced.v1", payload{tenant_id, branchId, isEnabled, version, groups,
   entries})` inside the existing tx. *(application crate — not locked)*
2. **`changes_since`** — branch-scoped-collection filter `payload->>'branchId' = branch`
   for `quick_sale`. *(LOCKED pull)*
3. **`reconcile_count`** cloud query — same branch filter for `quick_sale`. *(LOCKED)*
4. **`project_master_data`** — `quick_sale` full-replace arm (config upsert w/ version
   guard → delete groups+entries → insert from payload). *(LOCKED inbox)*
5. `MASTER_DATA_TYPES += "quick_sale"`. *(LOCKED inbox)*
6. `settings.sync.pull_collections += "quick_sale"` (+ mark it branch-scoped). *(config)*
7. Branch enrollment — re-emit current quick-sale config (backfill). *(enrollment path)*

## 5. Build steps (Q1 resolved → option A; needs owner sign-off on the pull extension)
1. Cloud: emit `quick_sale.replaced.v1` to `outbox_events` inside `replace_config`'s
   tx (mirror the catalog append; payload embeds `branchId` + full config). Unit-test
   the payload shape.
2. **Pull (locked-layer extension):** add `quick_sale` to the branch-scoped collection
   set; in `changes_since`, filter such collections by `payload->>'branchId'`. Add
   `quick_sale` to the branch's default `pull_collections`.
3. Branch: add the `quick_sale` arm to `project_master_data` (full-replace, version
   guard) + `MASTER_DATA_TYPES`. DB-backed test: apply an envelope → rows land;
   re-apply → idempotent; older version → skipped.
4. Backfill (Q3) if required.
5. E2E on staging: edit on cloud → pull on a branch node → verify the branch's
   `quick_sale_*` tables match.

## 6. Recommendation
**Q1 is resolved** (§4): per-branch routing must be added via option **A** (branch-
scoped pull filter — the reserved path). That is an extension of the **OWNER-LOCKED
pull contract**, so the one thing left before coding is **owner sign-off on adding
branch-scoped collections to `changes_since`**. Once approved, build steps 1→3 are
self-contained and DB-testable; do each as a separate verified PR (cloud emit → pull
filter → branch projection), then the staging E2E. Steps 2 (pull) and 3 (projection)
are the locked-layer touches — review carefully against the 7 offline-first invariants.
