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
| AccountZing-Platform 🔴 | · | ✓ | · | · | ✓ | ✓ |
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
| **108-platform-services** (monorepo: identity/customer/loyalty/notify/secrets) | · | ✓ | · | · | · | · |
| Platform-Services/Media | ✓ | ✓ | ✓ | · | · | · |
| Platform-Services/Payroll | · | ✓ | · | · | · | ✓ |
| Platform-Services/platform-console ⚠️local-only/un-pushed | ✓ | · | · | · | · | · |
| **รวม (/26)** | **18** | **24** | **6** | **1** | **3** | **15** |

✓ DO_NOT_TOUCH ของ AccountZing+Gateway merged 2026-06-18 (PR #14, #16) · ✅ครบ = ครบ 6 doc หลัก (ปัจจุบัน 1/26)
> อัปเดต 2026-07-25: **CLAUDE.md rollout** — เติม CLAUDE.md ให้ทุก repo ที่ยังขาด (template resolve ต่อ repo
> พร้อม build/test จริง + guardrails) → **CLAUDE coverage 8/26 → 24/26** (ทุก repo มีแล้ว ยกเว้น `shop108` fork
> ที่ข้าม + `platform-console` local-only). รวม **26 repos, coverage 51 → 67/156 (≈43%)**. คอลัมน์ C ยืนยันกับ
> GitHub จริงหลัง rollout; H/A/M/D/ai มาจาก Mac Studio scan.
> **การรวม backend** — `identity`/`customer`/`loyalty`/`notify`/`secrets` = `services/*` ใน monorepo
> **`108-platform-services`** (repo เดียว, ตอนนี้ **1/6** — เพิ่ม CLAUDE แล้ว); repo เดิม
> (Identity-/Customer-/Loyalty-/Notification-/Secrets-Platform) **archived 2026-07-24** ตัดออกจาก matrix พร้อม
> `customer-plat` (superseded). **`pos108-core` ยังเป็น repo แยก** (Wave-3 fold-in ถูก revert, 108-platform-services #15).
> `Payment-Platform/Gateway-scb-soft` = working copy ไม่มี `.git` → ไม่นับ. ตัวเลขตรงกับ DOCS_GRAPH.md.

## Priority follow-ups
1. 🔴 **AccountZing + Gateway**: commit `docs/DO_NOT_TOUCH.md` ที่ร่างไว้ (ผ่าน PR ของแต่ละ repo) — repo การเงิน guardrail สำคัญสุด
2. 🔴 **AccountZing**: ยังขาด HANDOFF/ARCHITECTURE/MILESTONES ระดับ repo (มี CLAUDE + .ai_context/ + DO_NOT_TOUCH + SPEC แล้ว) — เติมให้ครบ
3. ✅ **CLAUDE.md rollout เสร็จ** (2026-07-25): เติมครบทุก repo ยกเว้น `shop108` (fork ข้าม) + `platform-console` (local-only). ถัดไป: **HANDOFF.md** ให้ 108-platform-services + tixtox-web/flutter + Payroll ที่ยังขาด
4. 🟠 **MILESTONES.md** มีแค่ pos108-core — เติม repo ที่มี roadmap ชัด
5. 🟢 **leaf apps** (pos108-orders, shop108, BipByte/apps/{web,flutter-app}, Payroll): ตัดสินใจชัดว่าจะลง HANDOFF อย่างเดียว หรือยกเว้น แล้วบันทึกใน templates/README.md

## วิธี refresh matrix
รันบน Mac Studio (`ssh macstudio-ts` — ที่เก็บ checkouts ทั้งหมด): `git pull --ff-only` ต่อ repo แล้วเช็คไฟล์
H/C/A/M/D + `.ai_context` ต่อ repo. สคริปต์เต็ม (พร้อม repo list ปัจจุบัน) อยู่ใน section "How to refresh" ของ
[`DOCS_GRAPH.md`](DOCS_GRAPH.md) — อัปเดตทั้งสองไฟล์พร้อมกัน.
