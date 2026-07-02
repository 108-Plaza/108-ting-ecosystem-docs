# 03 — Path to First Sale (เส้นทางสั้นที่สุดสู่การขายจริง ครบ 3 ช่องทาง)

> สถานะ: DRAFT v0.1 · 2026-06-21 · operationalize GOAL ใน `00-omnichannel-vision.md`
> เป้า: ลำดับงานที่เล็กที่สุดที่ทำให้ "ร้านขายได้จริง" ทีละช่องทาง โดยใช้ของที่มีให้มากที่สุด

---

## สรุปสถานะ (เริ่มจากตรงไหน)

| ช่องทาง | ขายได้จริงแล้ว? | เหลืออะไร |
|---|---|---|
| **ออฟไลน์ (POS)** | ✅ **ขายได้แล้ววันนี้** | ไม่มี — pos108 POS ใช้งานจริง |
| **ออนไลน์ (shop108)** | ❌ ยัง | ฐานร่วม C0 + storefront C1 |
| **BipByte** | ❌ ยัง | ฐานร่วม C0 + channel BIPBYTE + shoppable C2 |

⇒ goal เหลือจริง ๆ แค่ 2 ช่องทาง (ออนไลน์ + BipByte) และทั้งคู่ "นั่งบนฐานร่วม C0 เดียวกัน"

---

## C0 — ฐานร่วม (ปลดล็อก online + BipByte พร้อมกัน) 🔑

งานที่ทำครั้งเดียว ใช้ได้ทั้ง 2 ช่องทาง:

1. **Service identity** — Identity ออก token ให้ commerce layer (perm `inventory:reserve`, `inventory:allocate`, `sales:create`) + scope ผ่าน branch gate
2. **pos108 tweaks (เล็ก):**
   - เติม `SaleChannel::Bipbyte` (+map "BIPBYTE", เปลี่ยน `_ => Pos` เป็น reject ค่าไม่รู้จัก)
   - Idempotency-Key บน `POST /sales` + `POST /inventory/reservations` (กันยิงซ้ำ at-least-once)
   - `reference_type="SHOP108_ORDER"` บน reservation
3. **catalog mirror** — consume `product.*.v1` → read-model สินค้า (กรอง `is_online_sellable`)
4. **stock flow** — reserve (soft-hold) → จ่าย → `POST /sales channel=...` → release hold (กันนับสองเด้ง)

**Done = ยิง `POST /sales channel=ONLINE` ทดสอบแล้วเห็นสต็อกตัด + GL ลง + AccountZing ได้รับ**

## C1 — ออนไลน์ขายได้ (shop108 storefront)
1. catalog read-model + หน้า list/รายละเอียดสินค้า (รูปจาก Media, badge availability จาก `inventory.availability.snapshotted.v1`)
2. cart + checkout → Payment
3. order orchestration → เรียก C0 (reserve → sale → release)
4. หน้า "ออเดอร์ของฉัน" + สถานะ

**Done = ลูกค้าซื้อจากเว็บ shop108 ได้จริง 1 รายการ end-to-end**

## C2 — BipByte ขายได้ (social commerce)
1. shoppable post — แท็ก `product_id` (จาก catalog เดียวกับ C1) ในโพสต์/วิดีโอ
2. กดซื้อ → เปิด checkout ของ shop108 (ฝัง/แชร์ commerce layer — ไม่สร้าง cart ชุดที่สอง)
3. book `POST /sales channel=BIPBYTE` (ใช้ C0)

**Done = กดซื้อจากโพสต์ BipByte ได้จริง ยอดติดป้าย BIPBYTE แยกจาก ONLINE**

## C3 — multi-seller + settlement (ทำให้เป็น "marketplace" จริง)
- seller onboarding + ผูก tenant/branch pos108
- แตก order หลายร้าน + escrow + payout
- คอมมิชชันแยกตาม channel ผ่าน AccountZing (tag ติดมาแล้วจาก C0)

---

## ลำดับแนะนำ & เหตุผล
```
C0 (ฐานร่วม)  ──►  C1 (online ขายได้)  ──►  C2 (BipByte ขายได้)  ──►  C3 (marketplace เต็ม)
   🔑 หัวใจ          เห็นเงินเข้าเร็วสุด        ต่อยอด C1 ราคาถูก         ขยายเป็นหลายร้าน
```
- ทำ **C0 ก่อนเสมอ** — ถ้าฐานนี้ผิด (channel/idempotency/double-decrement) จะลามทุกช่องทาง
- **C1 ให้ผลเร็วสุด** (ช่องทางที่คนคุ้น) → ได้ feedback จริงก่อนลงทุน C2/C3
- **C2 ถูกมาก** ถ้า C1 ทำชั้น commerce แบบแชร์ได้ (BipByte = อีกประตู)

## สิ่งที่ยัง gate ไม่ให้เริ่ม "ลงมือโค้ด"
1. **เจ้าของอนุมัติ** เปิดเป็น code-work area + สร้าง repo `shop108` (ตอนนี้ design-only ตาม Scope Change Process)
2. **เคาะ 3 ข้อนโยบาย** (จาก `01` §6): shared-pool vs channel-quota · double-decrement guard · BipByte ฝัง vs เรียก checkout

> เมื่อ 2 ข้อนี้ผ่าน → เริ่ม C0 ได้ทันที (spec พร้อมแล้วใน `00`/`01`/`03`)
