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
| AccountZing-Platform 🔴 | · | · | · | · | ✓ | ✓ |
| Payment-Platform/Gateway 🔴 | ✓ | · | · | · | ✓ | ✓ |
| Data-Platform | ✓ | · | · | · | · | · |
| BipByte-Platform/server | ✓ | ✓ | ✓ | · | · | ✓ |
| BipByte-Platform/engines | ✓ | ✓ | · | · | · | ✓ |
| BipByte-Platform/realtime-edge | ✓ | · | · | · | · | · |
| BipByte-Platform/apps/admin-web | · | ✓ | · | · | · | ✓ |
| BipByte-Platform/apps/web | · | · | · | · | · | ✓ |
| BipByte-Platform/apps/flutter-app | · | · | · | · | · | ✓ |
| ~/108-POS/admin | ✓ | ✓ | · | · | · | ✓ |
| ~/108-POS/terminal | ✓ | · | · | · | · | · |
| ~/108-POS/orders | · | · | · | · | · | · |
| ~/108-POS/store | ✓ | · | · | · | · | · |
| ~/108-POS/slot-api | ✓ | · | · | · | · | · |
| ~/108-POS/slot-front | ✓ | · | · | · | · | · |
| Commerce-Platform/product-vision | ✓ | ✓ | · | · | · | ✓ |
| Commerce-Platform/shop108 | · | · | · | · | · | · |
| Creator-Platform/creator | ✓ | · | ✓ | · | · | ✓ |
| Logistics-Platform/delivery | ✓ | · | ✓ | · | · | ✓ |
| IoT-Platform/Smart-Farm | ✓ | · | ✓ | · | · | ✓ |
| IoT-Platform/Smart-Home | ✓ | · | · | · | · | ✓ |
| Platform-Services/Security/identity | ✓ | ✓ | · | · | · | · |
| Platform-Services/Notification | ✓ | · | ✓ | · | · | ✓ |
| Platform-Services/Media | ✓ | ✓ | ✓ | · | · | · |
| Platform-Services/Loyalty | · | ✓ | · | · | · | ✓ |
| Platform-Services/Payroll | · | · | · | · | · | ✓ |
| Platform-Services/customer-plat ⚠️local-only/un-pushed | · | · | · | · | · | ✓ |
| Platform-Services/platform-console ⚠️local-only/un-pushed | ✓ | · | · | · | · | · |
| **รวม (/29)** | **20** | **9** | **7** | **1** | **3** | **18** |

✓ DO_NOT_TOUCH ของ AccountZing+Gateway merged 2026-06-18 (PR #14, #16) · ✅ครบ = ครบ 6 doc หลัก (ปัจจุบัน 1/29)
> อัปเดต 2026-07-25: `pos108` monorepo แยกเป็น per-repo checkouts ใต้ `~/108-POS` (core/admin/terminal/orders/store
> + slot-api/slot-front); Platform-Services เพิ่ม **Loyalty** + **Payroll** (5→7). รวม **29 repos, coverage 58/174 (≈33%)**.
> Doc gains: Media 1→3 (เพิ่ม CLAUDE+ARCHITECTURE), Identity 1→2 (เพิ่ม CLAUDE). `Payment-Platform/Gateway-scb-soft`
> เป็น working copy ไม่มี `.git` → ไม่นับ. `customer-plat` + `platform-console` ยัง **local-only ไม่ push**
> (ดู ⚠️ ใน ACTIVE_WORK.md) — ตัวเลขตรงกับ DOCS_GRAPH.md.

## Priority follow-ups
1. 🔴 **AccountZing + Gateway**: commit `docs/DO_NOT_TOUCH.md` ที่ร่างไว้ (ผ่าน PR ของแต่ละ repo) — repo การเงิน guardrail สำคัญสุด
2. 🔴 **AccountZing**: ขาด HANDOFF/CLAUDE/ARCHITECTURE/MILESTONES ระดับ repo (มีแต่ .ai_context/ + SPEC) — เติมให้ครบ
3. 🟠 กระจาย **CLAUDE.md** ไป repo ที่ขาด (มี template, ส่วนใหญ่แค่ resolve placeholder) — เหลือ 20/29 ที่ยังไม่มี
4. 🟠 **MILESTONES.md** มีแค่ pos108-core — เติม repo ที่มี roadmap ชัด
5. 🟢 **leaf apps** (pos108-orders, shop108, BipByte/apps/{web,flutter-app}, Payroll): ตัดสินใจชัดว่าจะลง HANDOFF อย่างเดียว หรือยกเว้น แล้วบันทึกใน templates/README.md

## วิธี refresh matrix
รันบน Mac Studio (`ssh macstudio-ts` — ที่เก็บ checkouts ทั้งหมด): `git pull --ff-only` ต่อ repo แล้วเช็คไฟล์
H/C/A/M/D + `.ai_context` ต่อ repo. สคริปต์เต็ม (พร้อม repo list ปัจจุบัน) อยู่ใน section "How to refresh" ของ
[`DOCS_GRAPH.md`](DOCS_GRAPH.md) — อัปเดตทั้งสองไฟล์พร้อมกัน.
