# 108 Commerce — Omnichannel Vision (offline · online · BipByte)

> สถานะ: DRAFT v0.1 · 2026-06-21
> **GOAL:** ร้านค้าหนึ่งราย ขายได้ครบ **3 ช่องทาง** ด้วยสินค้า/สต็อก/บัญชี **ชุดเดียวกัน**
> 1. **ออฟไลน์** — POS หน้าร้าน (pos108, มีอยู่แล้ว)
> 2. **ออนไลน์** — ร้านค้าออนไลน์ shop108 (marketplace, กำลังออกแบบ)
> 3. **BipByte** — ขายในฟีด/โพสต์ของ BipByte (social commerce)

---

## 1. หลักการเดียวที่ทำให้ goal นี้เป็นจริง

> **เครื่องยนต์เดียว (pos108) · หลายหน้า (channel) · ของชุดเดียว**

ทุกช่องทาง "ขายจริง" = ลงเป็น **pos108 sale** ผ่าน `POST /api/v1/sales` โดยติดป้าย `channel` ต่างกัน
→ สต็อกตัดที่เดียว, บัญชี (GL/COGS) ลงที่เดียว, AccountZing settlement ที่เดียว
→ ไม่มีสต็อก/บัญชีแยกกันต่อช่องทาง = ไม่มีทางขายชนกันหรือยอดเพี้ยน

```
        OFFLINE              ONLINE                 BIPBYTE
     POS cashier app      shop108 storefront     BipByte shoppable post
          │                     │                      │
          │              ┌──────┴───────────────────────┐
          │              │  shared commerce layer        │   ← ชั้นบาง (ใหม่)
          │              │  catalog · cart · checkout     │     ใช้ร่วม online+BipByte
          │              └──────┬───────────────────────┘
          ▼                     ▼                      ▼
   POST /sales            POST /sales            POST /sales
   channel=POS            channel=ONLINE         channel=BIPBYTE
          └──────────────────────┼──────────────────────┘
                                  ▼
                     ┌────────────────────────┐
                     │  เครื่องยนต์ pos108     │  สินค้า · สต็อก · ขาย · บัญชี
                     └────────────┬───────────┘
                                  ▼
              Identity · Payment · AccountZing · Logistics  (บริการกลาง ใช้ร่วม)
```

- **ออฟไลน์** = แอป POS เดิมยิง pos108 ตรง ๆ (ไม่ผ่านชั้น commerce) — มีอยู่แล้ว
- **ออนไลน์ + BipByte** = ใช้ **ชั้น commerce ร่วมกัน** (catalog/cart/checkout ของ shop108) ต่างกันแค่ "หน้า" ที่ลูกค้าเข้า และ channel tag ตอน book sale
- ⇒ BipByte = "อีกประตูเข้า" ของระบบ online เดียวกัน ไม่ใช่ระบบขายที่สาม

## 2. สถานะของจริงต่อช่องทาง (ส่อง pos108 แล้ว)

| ช่องทาง | channel value | สถานะใน pos108 | ต้องทำ |
|---|---|---|---|
| ออฟไลน์ | `POS` (default) | ✅ ครบ — แอป POS ขายจริงอยู่แล้ว | — |
| ออนไลน์ | `ONLINE` | ✅ enum + `POST /sales` รับได้ | สร้าง storefront + commerce layer (ดู `01`) |
| BipByte | `BIPBYTE` *(ยังไม่มี)* | ⚠️ `SaleChannel` = {Pos, Online, Delivery, Mobile, Reservation} **ไม่มี BIPBYTE**; `POST /sales` map channel ที่ไม่รู้จัก → **fallback เป็น POS เงียบ ๆ** (เสีย attribution) | **เติม `SaleChannel::Bipbyte`** + map "BIPBYTE" (อย่าให้ fallback เงียบ) + ผูก shoppable post → product |

> ความเสี่ยงที่เจอ: การ map channel แบบ `_ => Pos` ทำให้ค่าแปลก ๆ กลายเป็น POS เงียบ ๆ — ตอนเพิ่ม BIPBYTE ควรเปลี่ยนให้ **reject ค่าที่ไม่รู้จัก** แทน fallback (กันยอด BipByte ไปโผล่เป็นยอดหน้าร้าน)

## 3. ช่องทาง BipByte (social commerce) — ออกแบบ

BipByte มี integration กับ catalog อยู่แล้ว (catalog read Z-track, ZING_TAP) → ต่อยอดเป็น shoppable:

1. **Shoppable post** — โพสต์/วิดีโอใน BipByte แท็ก `product_id` (จาก catalog pos108 ที่ shop108 mirror)
2. **กดดู → กดซื้อ** — เปิดหน้า product / mini-checkout ที่ **ใช้ commerce layer ของ shop108** (ตะกร้า/เช็คเอาท์ตัวเดียวกับ online)
3. **จ่ายเงิน** — Payment-Platform (เดียวกับ online)
4. **book sale** — `POST /sales channel=BIPBYTE` → ตัดสต็อก + บัญชี (เหมือน online ทุกอย่าง ต่างแค่ channel tag เพื่อรู้ว่ายอดมาจาก BipByte)
5. **settlement** — AccountZing แยกยอด/คอมมิชชันตาม channel ได้ (เพราะ tag ติดมาแล้ว)

> ทางเลือก: BipByte จะ "ฝัง" หน้า checkout ของ shop108 (web view / shared BFF) หรือเรียก API เอง — แนะนำ **ฝัง/แชร์ commerce layer** เพื่อไม่ให้มี cart/checkout 2 ชุด

## 4. ทำไม goal นี้ "ทำได้จริงและไม่หนัก"

- เครื่องยนต์ (สต็อก/ขาย/บัญชี) + บริการกลาง = **มีครบแล้ว** ไม่สร้างใหม่
- ออฟไลน์ = มีแล้ว · ออนไลน์ = ช่องทาง ONLINE พร้อม · BipByte = เติม channel value เดียว + ต่อ shoppable
- งานจริงกระจุกที่ **ชั้น commerce ร่วม (shop108)** ที่ใช้ได้ทั้ง online และ BipByte — ทำครั้งเดียวได้ 2 ช่องทาง

## 5. ลำดับสู่ goal (channel-by-channel)

- **C0 — ฐานร่วม (Phase 0 ใน `01`)**: catalog sync + stock soft-hold + book `POST /sales` (channel tag) + service token. ปลดล็อกทั้ง online และ BipByte พร้อมกัน
- **C1 — ออนไลน์ (shop108)**: storefront + cart + checkout + order → ขาย `channel=ONLINE` ได้จริง
- **C2 — BipByte**: เติม `SaleChannel::Bipbyte` + shoppable post → product → reuse checkout → `channel=BIPBYTE`
- **C3 — multi-seller + settlement**: seller onboarding + คอมมิชชันแยก channel ผ่าน AccountZing
- ออฟไลน์: ✅ ไม่ต้องทำเพิ่ม (ของเดิม)

---

## ถัดไป / ที่เกี่ยวข้อง
- `01-architecture-and-phase0-stock-contract.md` — รายละเอียด stock/sale contract (เครื่องยนต์ ↔ ชั้น commerce)
- `02-product-catalog-model.md` — หน้าตาสินค้า (ดึงจาก `product.*.v1`) ใช้ร่วม online+BipByte
- เปิดประเด็น: เติม `SaleChannel::Bipbyte` (pos108) · BipByte ฝัง vs เรียก checkout · คอมมิชชันแยก channel
