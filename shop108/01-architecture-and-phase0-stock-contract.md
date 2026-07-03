# shop108 — Marketplace Design (Architecture + Phase 0: Stock Contract)

> สถานะ: DRAFT **v0.5** · 2026-06-21 · ยืนยัน sale path = `POST /sales` channel="ONLINE" (retail ไม่มีโต๊ะ); ปิด #4 (reserve ไม่ idempotent) #5 (lifecycle=soft-hold) #7 (channel field มีจริง). canonical: `~/108-Ting-Ecosystem/~/108-POS`
>
> ขอบเขต: **ร้านขายสินค้าทั่วไป (retail)** — ไม่มีคอนเซปต์ร้านอาหาร (จองโต๊ะ/ครัว/lady drink). "reservation" ในเอกสารนี้ = **stock soft-hold** (กันของช่วง checkout) เท่านั้น ไม่ใช่จองโต๊ะ
> 🎯 **GOAL = omnichannel** (ดู `00-omnichannel-vision.md`): ร้านเดียวขายครบ 3 ช่องทาง — ออฟไลน์ `channel=POS` (มีแล้ว) · ออนไลน์ `channel=ONLINE` (shop108) · BipByte `channel=BIPBYTE` (ต้องเติม enum). Phase 0 นี้ = ฐานร่วมที่ปลดล็อกทั้ง online + BipByte. `SaleChannel` ปัจจุบัน = {Pos, Online, Delivery, Mobile, Reservation} — **ยังไม่มี Bipbyte** + map ค่าไม่รู้จัก fallback POS เงียบ (ต้องแก้)
> ขอบเขตเอกสารนี้: ภาพรวมสถาปัตยกรรม + **Phase 0 contract เรื่องสต็อก/สินค้า pos108 → shop108**
> แพลตฟอร์ม: marketplace หลายร้าน · สต็อก source-of-truth = pos108

> **สรุปสำคัญจากการส่อง pos108 จริง:** Phase 0 ~70% **มีของอยู่แล้ว** — ระบบ reserve สต็อก, catalog events, availability event, transport (outbox+Redis), idempotency, channel `ONLINE` มีครบ งานที่เหลือคือ "เชื่อม" ไม่ใช่ "สร้างใหม่" (รายละเอียด §1.5)

---

## 0. หลักการออกแบบ (ยึดให้มั่น)

1. **ไม่สร้างกระดูกสันหลังซ้ำ** — Identity / Payment / AccountZing / Notification / Media / Logistics / Data-Platform ใช้ของเดิม เรียกผ่าน API/event เท่านั้น
2. **pos108 = source of truth ของ "สินค้า" และ "สต็อก" เสมอ** — shop108 เป็น read-model + ชั้นขายออนไลน์ ไม่ถือ truth ของสต็อก
3. **คุยข้ามแพลตฟอร์มผ่าน outbox + event** — ของจริงคือ **Postgres transactional outbox (`outbox_events`) + OutboxRelay (poll 5s) + Redis pub/sub** ดู §1.4
4. **ทุก cross-service call ใช้ tenant-scoped token**
5. **Idempotency + version-guard ทุกจุดตัดสต็อก/เงิน** — pos108 มี idempotency (Redis) + per-entity monotonic `version` (Producer rule R1) อยู่แล้ว consume ตามนี้

---

## 1. แผนที่บริการ (service map)

### สร้างใหม่ใน repo `shop108` (Rust modular workspace `api/crates/*`)

| Crate | หน้าที่ | Phase |
|---|---|---|
| `catalog` | สินค้า online (read-model จาก `product.*.v1` ของ pos108 + field online-only: desc ยาว, gallery, attribute, SEO, หมวดหมู่ marketplace) | 0–1 |
| `availability` | mirror สต็อก online ได้ จาก `inventory.availability.snapshotted.v1` + เรียก reserve กลับ pos108 | **0** |
| `cart` | ตะกร้า + saved items | 1 |
| `order` | order lifecycle + แตก sub-order ต่อร้าน | 1–2 |
| `pricing` | โปร / ค่าส่ง / VAT | 1 |
| `seller` | onboarding ร้าน, KYC, ผูก tenant pos108, % commission | 2 |
| `review` | รีวิว/เรตติ้งสินค้า+ร้าน | 3 |
| `search` | index + ค้นหา (Meilisearch/Typesense) feed จาก Data-Platform | 3 |
| `fulfillment` | ผูก Logistics, เลขพัสดุ, tracking | 3 |
| `storefront-bff` | API รวมให้ buyer app / web | 1 |

### ใช้ของเดิม (ห้ามทำซ้ำ)

| ระบบ | บทบาทใน shop108 |
|---|---|
| Identity | auth ผู้ซื้อ/ผู้ขาย (JWT HS256) |
| Payment-Platform | checkout, escrow, payout เข้าร้าน |
| AccountZing | settlement, ledger คอมมิชชัน, payout |
| Notification-Platform | แจ้งออเดอร์/จัดส่ง/โปร |
| Media + face-beauty/imageprocessing | รูป/วิดีโอสินค้า |
| Logistics (scaffold) | จัดส่ง/พัสดุ |
| Data-Platform | analytics, recommendation, search ranking |
| **pos108** | **SoT สินค้า + สต็อก + ระบบ reserve (§1.5)** |
| BipByte/tixtox | shoppable post (Phase 4) |

---

## 1.4 Event transport ของจริง (คำตอบข้อ 3 ที่ไปส่องมา)

อยู่ใน crate **`pos-infra`** (`api/crates/pos-infra`) — desc: *"event envelope, outbox, redis pub/sub, dead-letter, cache, idempotency"*

| ชิ้น | ของจริง | ไฟล์ |
|---|---|---|
| **Outbox** | Postgres table `outbox_events` (id, aggregate_type, aggregate_id, event_type, schema_version, correlation_id, payload, created_at, processed_at, next_retry_at). write ธุรกิจ + insert outbox = **tx เดียวกัน** | `pos-infra/src/messaging/outbox.rs` |
| **Relay** | `OutboxRelay` background worker poll ทุก **5s** → publish ไป broker/webhook → mark `processed_at` | (infrastructure::workers) |
| **Envelope** | `EventEnvelope` หุ้ม payload + `EventName` แบบ **`{domain}.{aggregate}.{action}.v{version}`** (เช่น `inventory.stock.reserved.v1`) | `pos-infra/src/events/envelope.rs` |
| **In-cluster bus** | Redis pub/sub (broadcaster + subscriber) | `pos-infra/src/redis_bus/` |
| **Idempotency** | Redis-backed idempotency keys | `pos-infra/src/idempotency/` |
| **Dead-letter** | handler มีอยู่ | `pos-infra/src/dead_letter/` |
| **Ordered replay** | `domain_events` มี `sequence_no` ต่อ aggregate (outbox **ไม่** การันตี order) | `pos-infra/src/messaging/domain_events.rs` |

**ผลต่อ contract (สำคัญ):**
- delivery = **at-least-once + ไม่การันตีลำดับ** → consumer ของ shop108 **ต้อง idempotent + ใช้ `version` guard** (ดู §2.2) — ตรงกับ Producer rule R1 ที่ pos108 ทำไว้แล้ว
- การส่งออกนอก pos108 ใช้ **webhook dispatcher** ที่ apply `project_external` (ตัด `cost_price`, `currency`, field ภายใน POS ออกที่ขอบ — "Checkpoint C") → shop108 จะได้ payload ที่ projected แล้ว ไม่เห็นต้นทุน

---

## 1.5 ⭐ ของจริงใน pos108 ที่ Phase 0 ใช้ซ้ำได้ (เปลี่ยนแผนจาก v0.1)

ไปส่อง code มาแล้ว — สิ่งที่ v0.1 จะ "สร้างใหม่" จริง ๆ **มีอยู่แล้ว**:

### (A) ระบบ reserve สต็อก — มีครบ **+ เปิด HTTP API แล้ว** ✅✅ `pos-inventory-allocation::InventoryAllocationService`
lifecycle จริง:
```
reserve()  → inventory_reservations(ACTIVE) + inventory_stocks.reserved_qty += qty
   ├─ release()  → RELEASED  + reserved_qty -= qty      (ยกเลิก/หมดเวลา)
   └─ allocate() → CONSUMED  + inventory_allocations(ALLOCATED)
                      └─ fulfill() → FULFILLED          (เขียน stock_movement)
expire() → คืน reservation ที่หมดอายุ
```
- `reserve()` **TOCTOU-safe**: `reserve_stock_in_tx` atomically check available + เพิ่ม `reserved_qty` ใน UPDATE เดียว → **ครอบ invariant I1 ให้แล้ว**
- stock row: `inventory_stocks(qty, reserved_qty, available_qty, version)` ต่อ **`stock_location_id`** → **multi-location จริง**

**HTTP API มีอยู่แล้ว** (`api/src/presentation/http/inventory.rs`, scope `/api/v1/inventory`):

| Method + path | handler | perm | หมายเหตุ |
|---|---|---|---|
| `GET /inventory/stock` | get_stock_balance | — | คืน (qty, reserved, **available** เป๊ะ) |
| `GET /inventory/stock-balances` | list_stock_balances | — | |
| `POST /inventory/reservations` | create_reservation | `inventory:reserve` | body: stock_location_id, product_id, variant_id, qty, reference_id, **expires_minutes**(TTL); `reference_type` hardcode `"manual"` |
| `POST /inventory/reservations/{id}/release` | release_reservation | (reserve) | |
| `POST /inventory/reservations/{id}/allocate` | allocate_reservation | `inventory:allocate` | |
| `POST /inventory/stock-adjust` | adjust_stock | | |

> ⇒ **shop108 ไม่ต้องสร้าง reservation API เลย — มันมีอยู่แล้ว!** แต่ดู ⚠️ ข้างล่างก่อน

> ⚠️⚠️ **สำคัญสุด (ส่อง SQL จริงแล้ว):** lifecycle นี้เป็น **"soft-hold ledger" ล้วน ไม่ใช่การขาย** —
> - `reserve` = `reserved_qty += qty, available_qty -= qty` (on-hand `qty` **ไม่ลด**)
> - `allocate` = ตั้ง reservation→CONSUMED + insert allocation row → **ไม่แตะ stock เลย** (reserved_qty/available_qty/qty คงเดิม)
> - `fulfill` = แค่ flip allocation status → **ไม่มีใครเรียกในโค้ดทั้ง repo** (caller เดียวของ `allocate` คือ HTTP handler เอง)
> - การ **ตัด on-hand จริง + เขียน `stock_movement` + GL/COGS** อยู่ที่ selling flow `create_sale` → `deduct_stock_and_record_movement_in_tx` (reference_type="sale") ซึ่ง **ไม่ใช้ reservation เลย**
>
> ⇒ **เรียก reserve+allocate ผ่าน HTTP เฉย ๆ "ไม่ได้ขาย" อะไร** — แค่จองนุ่ม ๆ แล้วค้าง reserved_qty ไว้. การขายจริงจาก shop108 ต้อง **book เป็น pos108 sale** (ดู §2.4 ที่แก้ใหม่)

### (B) Catalog events — มีครบ ✅ `pos-commerce/application/events/catalog.rs`
- `product.created.v1` / `product.updated.v1` / `product.discontinued.v1`(tombstone), `category.*`, `brand.*`
- **fat event** (แบกทั้ง row → consumer upsert ได้ไม่ต้อง read-back) + per-entity monotonic **`version`** + **`watermark`**
- สร้างมาเพื่อ **Data Platform Phase 1 + branch catalog sync** อยู่แล้ว → **shop108 = consumer รายใหม่ของ stream เดิม**
- contract-pinned ด้วย fixtures `tests/fixtures/contracts/catalog/*.json` (เปลี่ยนใน v1 ต้อง additive เท่านั้น)
- egress `project_external` ตัด field ภายในออก → shop108 ได้เฉพาะ allow-list

> ⇒ **เลิกใช้ `pos108.catalog.v1` ที่คิดเองใน v0.1** — consume `product.*.v1` ของจริงแทน. ส่วน online-only (gallery, desc ยาว, SEO) เก็บใน shop108 `catalog` เอง

### (C) Availability event — มีแล้ว ✅ แต่ **coarse bucket ล้วน** `inventory.availability.snapshotted.v1`
- emit ผ่าน `pos-leader` (leader-locked, single publisher) → projector `api/src/infrastructure/workers/availability_snapshot.rs`; contract-pinned `tests/fixtures/contracts/inventory/inventory.availability.snapshotted.v1.json`
- **payload จริง (อ่าน fixture แล้ว)** — แบกแค่สถานะหยาบ ไม่มีจำนวน ไม่มีต้นทุน:
```jsonc
{ "type":"inventory.availability.snapshotted.v1", "schema_version":1, "source":"pos108",
  "tenant_id":"...", "occurred_at":"...",
  "data":{ "branch_id":"...", "tenant_id":"...", "version":1, "taken_at":"...",
    "availability":[ {"product_id":"...","status":"IN_STOCK"},     // | LOW | OUT_OF_STOCK
                     ... ] } }       // เรียงตาม product_id, debounce: bucket เดิม=byte เดิม=ไม่ re-emit
```
- classify(available, reorder_point): `≤0`→OUT_OF_STOCK, `≤threshold`→LOW, else IN_STOCK
> ⚠️ ⇒ **shop108 mirror "จำนวน" จาก event นี้ไม่ได้** — ได้แค่ป้าย พร้อมส่ง/ใกล้หมด/หมด. **จำนวนจริงต้องดึงตอนกดซื้อ** ผ่าน `GET /inventory/stock` หรือผล `reserve` (ดู §2.3 ที่แก้ใหม่)

### (D) Channel "ONLINE" — มีแล้วบางส่วน ✅/⚠️
- `SaleChannel::Online` + `OrderSource::OnlineOrder` มีในโดเมน sales/order แล้ว → ขายออนไลน์ถูกแท็กเป็น channel ได้
- ⚠️ **แต่ยังไม่มี "โควตาสต็อกแยกช่องทาง" (online_allocation)** — `reserved_qty` เป็น pool เดียวรวมทุก channel

### (E) Auth ของ reserve API — gate แบบ branch-scoped (จุดที่ต้องต่อ)
- `create_reservation` เรียก `ctx.require("inventory:reserve")` + **branch-isolation**: resolve branch ของ `stock_location_id` แล้ว `ctx.require_branch(...)` — **caller scope `global` ผ่านได้** (เห็นทุก branch)
- ⇒ shop108 ต้องมี **service principal** (ออกผ่าน Identity, tenant-scoped token) ที่ถือ perm `inventory:reserve` + `inventory:allocate` และ scope ที่ผ่าน branch gate (global หรือ branch ที่ designated). pattern token นี้มีใช้แล้วใน BipByte↔Data
- **ช่องว่างที่ต้องเติม:** (1) `reference_type` ตอนนี้ hardcode `"manual"` — ควรเพิ่มค่า `"SHOP108_ORDER"` เพื่อ recon แยกออก; (2) `create_reservation` **ยังไม่เห็นการรับ Idempotency-Key** — at-least-once ข้ามแพลตฟอร์มควรกันยิงซ้ำ (ตอนนี้กันด้วย `reference_id` เท่านั้น — ต้องยืนยันว่าซ้ำ reference_id แล้วไม่เกิด reservation ซ้อน); (3) ยังไม่เห็น HTTP `fulfill` (มีแค่ allocate) — ต้องเช็กว่า allocate พอ consume reservation ไหม หรือ stock_movement เขียนทางอื่น

---

## 2. Phase 0 — Stock & Product Contract (ฉบับปรับตามของจริง)

### 2.1 การตัดสินใจหลัก: โควตาช่องทาง (channel allocation)

มี 2 ทาง — เลือกก่อนลงมือ:

- **ทาง A (แนะนำเริ่มต้น) — Shared pool:** ออนไลน์ reserve จาก `reserved_qty/available_qty` pool เดียวกับหน้าร้าน ผ่าน `InventoryAllocationService` ที่มีอยู่ **ไม่ต้องแก้ pos108 schema** กัน oversell ได้ทันทีเพราะ reserve atomic. ข้อเสีย: คุมไม่ได้ว่า "กันของไว้ขายออนไลน์เท่าไร" (ใครกดก่อนได้ก่อน)
- **ทาง B — Channel quota (ฟีเจอร์ใหม่ใน pos108):** เพิ่มคอลัมน์/ตาราง `online_allocation` ต่อ (tenant, sku, location) + guard `online_reserved ≤ online_allocation`. ยืดหยุ่นกว่า แต่ต้องแก้ pos108 + เพิ่ม event `inventory.online_allocation.changed.v1`

> คำแนะนำ: **เริ่มทาง A** (เชื่อมของที่มี ปล่อยขายได้เร็ว) แล้วยก B เป็น Phase ถัดไปถ้าธุรกิจต้องการกันโควตาจริง

### 2.2 Consume catalog (ใช้ event เดิม)
- subscribe `product.created.v1 / product.updated.v1 / product.discontinued.v1` (+ `category.*`)
- upsert เข้า `catalog` read-model โดย **idempotent ตาม `id` + ทิ้ง event ที่ `version` ≤ ที่เก็บไว้** (กัน out-of-order/at-least-once)
- tombstone `discontinued` → ซ่อนสินค้า ไม่ลบจริง (กัน resurrection ตอน replay)
- รับเฉพาะ `is_online_sellable = true` (ธงจาก pos108) มาแสดง

### 2.3 Consume availability (coarse) + ดึงจำนวนตอนซื้อ
- subscribe `inventory.availability.snapshotted.v1` → เก็บ **coarse status** (IN_STOCK/LOW/OUT_OF_STOCK) ต่อ (tenant, branch, product), guard ด้วย `version`
- ใช้โชว์ป้าย "พร้อมส่ง/ใกล้หมด/หมด" บนหน้า list/รายละเอียด (eventually-consistent)
- ⚠️ **จำนวนจริงไม่อยู่ใน event** — เวลาแสดง "เหลือ N ชิ้น" หรือก่อนยืนยันตะกร้า ให้เรียก `GET /api/v1/inventory/stock` (คืน qty/reserved/available เป๊ะ) และตัวเลขที่ผูกพันจริงคือผล `reserve` (atomic)

### 2.4 Flow จริง: soft-hold ตอน checkout → book sale ตอนจ่าย (แก้ใหญ่ตาม §1.5A)
reservation ใช้เป็น **soft-hold กันแย่งของช่วง checkout เท่านั้น** ส่วน "การขายจริง" book ผ่าน sale path:
```
1) checkout (กันของช่วงผู้ซื้อกำลังจ่าย)  [optional แต่กัน oversell ระหว่าง add-to-cart→จ่าย]
   POST /api/v1/inventory/reservations { ..., reference_id=order_id, expires_minutes=15 }
      → reserved_qty+ / available_qty-     (atomic; 409 ถ้าไม่พอ)

2) ชำระเงินสำเร็จ (Payment escrow) → book เป็น pos108 sale ผ่าน **`POST /api/v1/sales`**:
   { branch_id, stock_location_id, channel:"ONLINE", items[], payment_method, payment_amount,
     cash_session_id?:null (ออนไลน์ไม่มีลิ้นชัก), customer_id?, order_id? }
      → deduct_stock_and_record_movement_in_tx (ตัด on-hand จริง + stock_movement)
      → GL/COGS + outbox → AccountZing
   แล้ว **release reservation** (ข้อ 1) มิฉะนั้น available_qty ถูกลดซ้ำ (hold + ขาย = นับสองเด้ง)

3) ยกเลิก/ไม่จ่าย → POST /inventory/reservations/{id}/release | expire() auto คืน
```
> ✅ **`POST /sales` = retail sale ทั่วไป ไม่มีโต๊ะ/ครัว** — `CreateSaleRequest` ไม่มี table_id; มี field `channel: Option<String>` รับ `"ONLINE"` ได้ตรง (ปิดข้อ tag channel), `cash_session_id`/`customer_id` optional. เหมาะร้านขายของทั่วไปพอดี
> ⚠️ จุดที่ต้องออกแบบ: **กันนับสองเด้ง** soft-hold (`available_qty-`) vs sale deduct (`available_qty-` ผ่าน stock_writer ที่ guard ไม่ต่ำกว่า reserved pool) — release hold ตอน/ก่อน book sale
> ของที่ต้องเติม: idempotency บน reserve + sale (กันยิงซ้ำ at-least-once), service token (§1.5E)
> หมายเหตุ path อื่น: `POST /sales/bills`=เปิดบิล (ร้านอาหาร), `/order`+/pay=pre-order (`order_id` ลิงก์เข้า /sales ได้), `quick_sale`=config ปุ่มลัดบนจอ POS (ไม่ใช่การขาย), `public_preorder`=พรีออเดอร์สาธารณะ — **สำหรับ retail ออนไลน์ใช้ `POST /sales` ตรง ๆ พอ**
Invariants (pos108 รับประกันให้แล้วเป็นส่วนใหญ่):
- **I1** atomic check+increment (มีแล้วใน `reserve_stock_in_tx`)
- **I2/I3** idempotent ตาม Idempotency-Key / reservation_id + state machine one-way (ACTIVE→CONSUMED|RELEASED) — ใช้ idempotency ของ pos-infra
- **I4** ตัด stock จริงตอน `fulfill()` เท่านั้น (ก่อนหน้านั้นแค่ `reserved_qty`)

### 2.5 Reconciliation
- nightly recon: เทียบ `reserved_qty(pos108)` ส่วนที่ ref=SHOP108_ORDER กับ reservation ACTIVE ใน shop108 → ต่าง = alert (ใช้ pattern recon ที่ Data-Platform มี)
- orphan sweeper: ใช้ `expire()` ของ pos108 (มีอยู่แล้ว) เป็นตัวคืนโควตา

### 2.6 Failure modes ที่ต้องเทสต์
| สถานการณ์ | ต้องได้ผล | กลไกที่รองรับแล้ว |
|---|---|---|
| reserve ack หาย → กดซ้ำ | reservation เดิม | Idempotency-Key (pos-infra) |
| จ่ายสำเร็จ แต่ allocate timeout | retry allocate (idempotent) ห้าม release | reservation state machine |
| ทิ้งตะกร้า | TTL → `expire()` คืนของ | `expires_at` + expire() |
| event มาสลับลำดับ | ทิ้งตัว version เก่า | per-entity `version` (R1) |
| pos108 ล่มตอน reserve | order `awaiting_stock`, ไม่เก็บเงิน | (ออกแบบใน shop108 order) |
| ขายชนหน้าร้าน | ฝั่งมาทีหลังได้ 409 | atomic reserve (TOCTOU fix) |

---

## 3. Order → Payment → Payout (สรุป — รายละเอียดเต็มใน doc Phase 2)
```
ตะกร้า(หลายร้าน) → แตก sub-order ต่อ seller/tenant
  → reserve ทุก line (ต้องสำเร็จก่อน) → Payment เก็บเงินรวม (escrow)
    → allocate+fulfill (ตัด stock) → Notification แจ้งร้าน → ร้านส่ง (Logistics)
      → ผู้ซื้อรับของ → AccountZing รับรู้รายได้ − commission → Payment payout เข้าร้าน
```

---

## 4. Deploy / repo
| สิ่ง | ที่ | อ้างอิง pattern |
|---|---|---|
| repo `108-Plaza/shop108` | ใหม่ | โครงเดียวกับ pos108 (`api/crates/*`, `apps/`) |
| `apps/buyer` (Flutter) | ใน repo | เหมือน pos108 `apps/pos` |
| `apps/seller-center` | ใน repo | หลังบ้านร้าน |
| `shop108-admin` | repo แยก/`apps/admin` | อนุมัติร้าน/คอมมิชชัน/ข้อพิพาท |
| k3s ns `shop108` | node เดิม | gateway NodePort + ingress เหมือน tixtox |
| `deploy/` overlay | net `108-core` | gate healthcheck Identity→Payment→pos108→shop108 |

---

## 5. Definition of Done — Phase 0 (เน้น "เชื่อม" — reserve API + availability event มีแล้ว)
- [ ] **เลือกทาง A หรือ B** (§2.1) — แนะนำ A (ใช้ reserve API ที่มีเลย ไม่แก้ pos108 schema)
- [ ] **Auth glue (งานชิ้นแรกจริง):** Identity ออก service token ให้ shop108 ถือ perm `inventory:reserve`+`inventory:allocate` + scope ผ่าน branch gate
- [x] ~~เลือก endpoint book sale + tag channel~~ → **`POST /sales` channel="ONLINE"** (retail, ไม่มีโต๊ะ)
- [ ] pos108 idempotency: unique `(tenant_id, reference_type, reference_id)` + `ON CONFLICT` บน reserve **และ** บน sale-booking (กันยิงซ้ำ at-least-once) + `reference_type="SHOP108_ORDER"`
- [ ] flow กันนับสองเด้ง: release soft-hold ใน tx เดียวกับ book sale (§2.4)
- [ ] shop108 `catalog`: consume `product.*.v1` (+`category.*`) idempotent + version-guard + tombstone + กรอง `is_online_sellable`
- [ ] shop108 `availability`: consume `inventory.availability.snapshotted.v1` (coarse) + version-guard; จำนวนจริงดึง `GET /inventory/stock`
- [ ] shop108 `order`: เรียก reserve→allocate→release ตาม flow §2.4 + สถานะ `awaiting_stock`
- [ ] nightly recon (ref=SHOP108_ORDER) + alert; orphan ใช้ `expire()` เดิม
- [ ] integration test ครบ failure modes §2.6

---

## 6. ข้อตัดสินใจ/ของต้องเช็กต่อ
1. **ทาง A vs B** (§2.1) — ของหลัก (แนะนำ A)
2. ~~availability payload~~ ✅ **ตอบแล้ว** — coarse bucket ล้วน (§1.5C) ไม่มีจำนวน → mirror count ไม่ได้ ต้องดึง `GET /inventory/stock`
3. ~~HTTP reserve เปิดข้ามแพลตฟอร์มไหม~~ ✅ **ตอบแล้ว** — มีแล้ว `POST /inventory/reservations|/release|/allocate` (§1.5A) gate ด้วย perm + branch-isolation (global ผ่าน) → **งานชิ้นแรก = auth glue ไม่ใช่สร้าง endpoint**
4. ~~reference_id กัน reservation ซ้อนพอไหม~~ ✅ **ตอบแล้ว: ไม่กัน** — `create_reservation` INSERT ไม่มี `ON CONFLICT` (ออก `ReservationId::new()` ทุกครั้ง) + `reserve_stock_in_tx` เพิ่ม reserved_qty ทุกครั้ง ⇒ ยิงซ้ำ reference_id = **2 reservation + ลด available_qty สองเด้ง**. **ต้องเติม idempotency** — แนะนำ unique index `(tenant_id, reference_type, reference_id) WHERE status='ACTIVE'` + `ON CONFLICT DO NOTHING` คืน id เดิม (หรือ Idempotency-Key ผ่าน pos-infra ที่ HTTP layer)
5. ~~allocate พอไหม / fulfill แยก~~ ✅ **ตอบแล้ว: ทั้งคู่ไม่ตัด stock** — reserve→allocate→fulfill = soft-hold ledger; การตัด on-hand จริงอยู่ที่ `create_sale`/`deduct_stock_and_record_movement_in_tx` ⇒ **shop108 book เป็น sale ผ่าน `POST /sales`** (§2.4)
6. **กันนับสองเด้ง** soft-hold vs sale deduct (§2.4) — ต้อง release hold ใน flow เดียวกับ book sale
7. ~~tag channel=ONLINE~~ ✅ **ตอบแล้ว:** `CreateSaleRequest` มี field `channel: Option<String>` ส่ง `"ONLINE"` ได้ตรง (ไม่มี table_id, cash_session/customer optional) — retail sale ทั่วไป
8. **หลายสาขาต่อร้าน** — ออนไลน์ reserve/ขายจาก `stock_location_id` + `branch_id` ไหน (designated ต่อ tenant? routing rule?)
9. **canonical doc** — code อ้าง `docs/ECOSYSTEM_TARGET_ARCHITECTURE.md §10` (Producer rule R1) แต่ไม่อยู่ใน `pos108/docs` → อยู่อีก repo?

---

## ถัดไป
- `02-product-catalog-model.md` — map `product.*.v1` → shop108 catalog + online-only fields
- `03-reservation-integration-endpoint.md` — สเปก endpoint ครอบ `InventoryAllocationService`
- `04-order-and-payout-phase2.md` — multi-seller order, escrow, commission, AccountZing legs
