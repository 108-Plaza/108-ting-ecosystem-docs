# Control-Doc Coverage — 108 Ting Ecosystem

> Living checklist: แต่ละ repo มีเอกสารควบคุมครบตามมาตรฐาน `templates/` ไหม
> มาตรฐาน 6 ตัว: **H**=HANDOFF.md · **C**=CLAUDE.md · **A**=docs/ARCHITECTURE.md · **M**=docs/MILESTONES.md · **D**=docs/DO_NOT_TOUCH.md · **ai**=.ai_context/
> SoT ของกฎ/เขตห้ามแตะ: `~/.claude/CLAUDE.md` + `CLAUDE.md` + `docs/DO_NOT_TOUCH.md` (ดู ENGINEERING_CONSTITUTION.md)
> สำรวจล่าสุด: 2026-07-02

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
| ~/108-POS/admin | ✓ | ✓ | · | · | · | · |
| ~/108-POS/apps/pos | ✓ | ✓ | · | · | · | · |
| ~/108-POS/orders | · | · | · | · | · | · |
| ~/108-POS/slot-front | ✓ | · | · | · | · | · |
| ~/108-POS/slot-api | ✓ | · | · | · | · | · |
| Commerce-Platform/product-vision | ✓ | ✓ | · | · | · | ✓ |
| Commerce-Platform/shop108 | · | · | · | · | · | · |
| ~/108-POS/store | ✓ | · | · | · | · | · |
| Creator-Platform/creator | ✓ | · | ✓ | · | · | ✓ |
| Logistics-Platform/delivery | ✓ | · | ✓ | · | · | ✓ |
| IoT-Platform/Smart-Farm | ✓ | · | ✓ | · | · | ✓ |
| IoT-Platform/Smart-Home | ✓ | · | · | · | · | ✓ |
| Platform-Services/Notification | ✓ | · | ✓ | · | · | ✓ |
| Platform-Services/Media | ✓ | · | · | · | · | · |
| Platform-Services/Security/identity | ✓ | · | · | · | · | · |
| Platform-Services/customer-plat ⚠️local-only/un-pushed | · | · | · | · | · | ✓ |
| Platform-Services/platform-console ⚠️local-only/un-pushed | ✓ | · | · | · | · | · |

✓ DO_NOT_TOUCH ของ AccountZing+Gateway merged 2026-06-18 (PR #14, #16) · ✅ครบ = ครบ 6 doc หลัก (ปัจจุบัน 1/27)
> อัปเดต 2026-07-02: เพิ่ม product-vision, shop108, storefront (Commerce) + customer-plat, platform-console
> (Platform-Services, **local-only ยังไม่ push** — ดู ⚠️ ใน ACTIVE_WORK.md). รวม 27 repos, coverage 53/162 (≈33%)
> — ตัวเลขตรงกับ DOCS_GRAPH.md.

## Priority follow-ups
1. 🔴 **AccountZing + Gateway**: commit `docs/DO_NOT_TOUCH.md` ที่ร่างไว้ (ผ่าน PR ของแต่ละ repo) — repo การเงิน guardrail สำคัญสุด
2. 🔴 **AccountZing**: ขาด HANDOFF/CLAUDE/ARCHITECTURE/MILESTONES ระดับ repo (มีแต่ .ai_context/HANDOFF + SPEC) — เติมให้ครบ
3. 🟠 กระจาย **CLAUDE.md** ไป repo ที่ขาด (มี template, ส่วนใหญ่แค่ resolve placeholder)
4. 🟠 **MILESTONES.md** มีแค่ pos108/api — เติม repo ที่มี roadmap ชัด
5. 🟢 **leaf apps** (pos108/apps/{orders,slot}, BipByte/apps/{web,flutter-app}): ตัดสินใจชัดว่าจะลง HANDOFF อย่างเดียว หรือยกเว้น แล้วบันทึกใน templates/README.md

## วิธี refresh matrix
รันสคริปต์เช็ก (จาก controller): `ssh macstudio` แล้ว loop `find ~/108-Ting-Ecosystem -maxdepth 5 -name .git` เช็คไฟล์ H/C/A/M/D + `.ai_context` ต่อ repo
