# Control-Doc Coverage — 108 Ting Ecosystem

> Living checklist: แต่ละ repo มีเอกสารควบคุมครบตามมาตรฐาน `templates/` ไหม
> มาตรฐาน 6 ตัว: **H**=HANDOFF.md · **C**=CLAUDE.md · **A**=docs/ARCHITECTURE.md · **M**=docs/MILESTONES.md · **D**=docs/DO_NOT_TOUCH.md · **ai**=.ai_context/
> SoT ของกฎ/เขตห้ามแตะ: `~/.claude/CLAUDE.md` + `CLAUDE.md` + `docs/DO_NOT_TOUCH.md` (ดู ENGINEERING_CONSTITUTION.md)
> สำรวจล่าสุด: **2026-07-25** (บน Mac Studio, `git pull --ff-only` ก่อนสแกน) — ตัวเลขตรงกับ [`DOCS_GRAPH.md`](DOCS_GRAPH.md).
> ⚠️ สแกนตาม branch ที่ checkout อยู่จริงต่อ repo (บาง repo อยู่บน feature branch / มี local edits) — ดู "Survey caveats" ใน DOCS_GRAPH.md

## Matrix (✓ = มี · · = ไม่มี)

| repo | H | C | A | M | D | ai |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| ~/108-POS/core ✅ครบ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| AccountZing-Platform ✅ครบ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Payment-Platform/Gateway 🔴 | ✓ | ✓ | · | · | ✓ | ✓ |
| Data-Platform | ✓ | ✓ | · | · | · | · |
| BipByte-Platform/server | ✓ | ✓ | ✓ | · | · | ✓ |
| BipByte-Platform/engines | ✓ | ✓ | · | · | · | ✓ |
| BipByte-Platform/realtime-edge | ✓ | ✓ | · | · | · | · |
| BipByte-Platform/apps/admin-web | · | ✓ | · | · | · | ✓ |
| BipByte-Platform/apps/web | · | ✓ | · | · | · | ✓ |
| BipByte-Platform/apps/flutter-app | · | ✓ | · | · | · | ✓ |
| ~/108-POS/admin | ✓ | ✓ | · | · | · | ✓ |
| ~/108-POS/terminal | ✓ | ✓ | · | · | · | · |
| ~/108-POS/orders | · | ✓ | · | · | · | · |
| ~/108-POS/store | ✓ | ✓ | · | · | · | · |
| ~/108-POS/slot-api | ✓ | ✓ | · | · | · | · |
| ~/108-POS/slot-front | ✓ | ✓ | · | · | · | · |
| Commerce-Platform/product-vision | ✓ | ✓ | · | · | · | ✓ |
| Commerce-Platform/shop108 *(skipped — fork)* | · | · | · | · | · | · |
| Creator-Platform/creator | ✓ | ✓ | ✓ | · | · | ✓ |
| Logistics-Platform/delivery | ✓ | ✓ | ✓ | · | · | ✓ |
| IoT-Platform/Smart-Farm | ✓ | ✓ | ✓ | · | · | ✓ |
| IoT-Platform/Smart-Home | ✓ | ✓ | · | · | · | ✓ |
| **108-platform-services** ✅ครบ (monorepo: identity/customer/loyalty/notify/secrets) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Platform-Services/Media | ✓ | ✓ | ✓ | · | · | · |
| Platform-Services/Payroll | ✓ | ✓ | · | · | · | ✓ |
| Platform-Services/platform-console ⚠️local-only/un-pushed | ✓ | · | · | · | · | · |
| **รวม (/26)** | **21** | **24** | **8** | **3** | **4** | **16** |

✓ DO_NOT_TOUCH ของ AccountZing+Gateway merged 2026-06-18 (PR #14, #16) · ✅ครบ = ครบ 6 doc หลัก (ปัจจุบัน **3/26**)
> อัปเดต 2026-07-25: **Payroll เพิ่ม root `HANDOFF.md`** (PR #12) → **2/6 → 3/6**. ระหว่างเขียนพบว่า
> `.ai_context/HANDOFF.md` เดิม claim ว่า CI แดงเพราะ Actions billing wall — **เช็คจริงแล้ว CI เขียวบน main**
> (`050ae85`, 2026-07-22); และ `active/extraction-task.md` claim ว่า Slice 4b/5 ยังค้าง — **จริงๆ merge แล้วทั้งคู่**
> (PR #6, #7) ตาม top-block ของ `.ai_context/HANDOFF.md` เอง. service นี้ **code-complete** แล้ว งานที่เหลือเป็น
> cross-repo (AccountZing consumer) หรือ deploy ops ไม่ใช่โค้ด. รวม **26 repos, coverage 75 → 76/156 (≈49%)**.
> ก่อนหน้า: **AccountZing ครบ 6/6** (PR #33+#34) — repo ที่ 3 ที่ครบ ต่อจาก pos108-core + 108-platform-services.
> ก่อนหน้านั้น: **`108-platform-services` ครบ 6/6** (PR #21) ต่อจาก CLAUDE #17 + HANDOFF #19. repo เดิม
> (Identity-/Customer-/Loyalty-/Notification-/Secrets-Platform) **archived 2026-07-24** ตัดออกจาก matrix
> พร้อม `customer-plat` (superseded). **`pos108-core` ยังเป็น repo แยก** (Wave-3 fold-in ถูก revert,
> 108-platform-services #15).
> **CLAUDE.md rollout** (ก่อนหน้านั้น) — เติม CLAUDE.md ให้ทุก repo ที่ยังขาด → **CLAUDE coverage 8/26 → 24/26**
> (ทุก repo มีแล้ว ยกเว้น `shop108` fork ที่ข้าม + `platform-console` local-only).
> `Payment-Platform/Gateway-scb-soft` = working copy ไม่มี `.git` → ไม่นับ. ตัวเลขตรงกับ DOCS_GRAPH.md.

## Priority follow-ups
1. ✅ **AccountZing + Gateway DO_NOT_TOUCH**: merged 2026-06-18 (PR #14, #16) — ทั้งคู่มี D แล้ว, รายการนี้ปิด
2. ✅ **AccountZing ครบ 6/6** (2026-07-25, PR #33+#34)
3. ✅ **CLAUDE.md rollout เสร็จ** (2026-07-25) + **108-platform-services ครบ 6/6** + **Payroll HANDOFF เพิ่มแล้ว** (PR #12). ถัดไป: HANDOFF.md ให้ tixtox-web/flutter ที่ยังขาด
4. 🟠 **MILESTONES.md** มีแค่ pos108-core + 108-platform-services + AccountZing — เติม repo ที่มี roadmap ชัด
5. 🟢 **leaf apps** (pos108-orders, shop108, BipByte/apps/{web,flutter-app}): ตัดสินใจชัดว่าจะลง HANDOFF อย่างเดียว หรือยกเว้น แล้วบันทึกใน templates/README.md

## วิธี refresh matrix
รันบน Mac Studio (`ssh macstudio-ts` — ที่เก็บ checkouts ทั้งหมด): `git pull --ff-only` ต่อ repo แล้วเช็คไฟล์
H/C/A/M/D + `.ai_context` ต่อ repo. สคริปต์เต็ม (พร้อม repo list ปัจจุบัน) อยู่ใน section "How to refresh" ของ
[`DOCS_GRAPH.md`](DOCS_GRAPH.md) — อัปเดตทั้งสองไฟล์พร้อมกัน.
