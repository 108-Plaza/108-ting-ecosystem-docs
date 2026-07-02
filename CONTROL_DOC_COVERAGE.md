# Control-Doc Coverage — 108 Ting Ecosystem

> Living checklist: แต่ละ repo มีเอกสารควบคุมครบตามมาตรฐาน `templates/` ไหม
> มาตรฐาน 6 ตัว: **H**=HANDOFF.md · **C**=CLAUDE.md · **A**=docs/ARCHITECTURE.md · **M**=docs/MILESTONES.md · **D**=docs/DO_NOT_TOUCH.md · **ai**=.ai_context/
> SoT ของกฎ/เขตห้ามแตะ: `~/.claude/CLAUDE.md` + `CLAUDE.md` + `docs/DO_NOT_TOUCH.md` (ดู ENGINEERING_CONSTITUTION.md)
> สำรวจล่าสุด: 2026-06-18

## Matrix (✓ = มี · · = ไม่มี)

| repo | H | C | A | M | D | ai |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Commerce-Platform/pos108/api ✅ครบ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| AccountZing-Platform 🔴 | · | · | · | · | ✓ | ✓ |
| Payment-Platform/Gateway 🔴 | ✓ | · | · | · | ✓ | ✓ |
| Data-Platform | ✓ | · | · | · | · | · |
| BipByte-Platform/server | ✓ | ✓ | ✓ | · | · | ✓ |
| BipByte-Platform/engines | ✓ | ✓ | · | · | · | ✓ |
| BipByte-Platform/realtime-edge | ✓ | · | · | · | · | · |
| BipByte-Platform/apps/admin-web | · | ✓ | · | · | · | ✓ |
| BipByte-Platform/apps/web | · | · | · | · | · | ✓ |
| BipByte-Platform/apps/flutter-app | · | · | · | · | · | ✓ |
| Commerce-Platform/pos108/apps/admin | ✓ | ✓ | · | · | · | · |
| Commerce-Platform/pos108/apps/pos | ✓ | ✓ | · | · | · | · |
| Commerce-Platform/pos108/apps/orders | · | · | · | · | · | · |
| Commerce-Platform/pos108/apps/slot | · | · | · | · | · | · |
| Commerce-Platform/pos108/slot-api | ✓ | · | · | · | · | · |
| Creator-Platform/creator | ✓ | · | ✓ | · | · | ✓ |
| Logistics-Platform/delivery | ✓ | · | ✓ | · | · | ✓ |
| IoT-Platform/Smart-Farm | ✓ | · | ✓ | · | · | ✓ |
| IoT-Platform/Smart-Home | ✓ | · | · | · | · | ✓ |
| Platform-Services/Notification | ✓ | · | ✓ | · | · | ✓ |
| Platform-Services/Media | ✓ | · | · | · | · | · |
| Platform-Services/Security/identity | ✓ | · | · | · | · | · |

✓ DO_NOT_TOUCH ของ AccountZing+Gateway merged 2026-06-18 (PR #14, #16) · ✅ครบ = ครบ 5 doc หลัก (ปัจจุบัน 1/22)

## Priority follow-ups
1. 🔴 **AccountZing + Gateway**: commit `docs/DO_NOT_TOUCH.md` ที่ร่างไว้ (ผ่าน PR ของแต่ละ repo) — repo การเงิน guardrail สำคัญสุด
2. 🔴 **AccountZing**: ขาด HANDOFF/CLAUDE/ARCHITECTURE/MILESTONES ระดับ repo (มีแต่ .ai_context/HANDOFF + SPEC) — เติมให้ครบ
3. 🟠 กระจาย **CLAUDE.md** ไป repo ที่ขาด (มี template, ส่วนใหญ่แค่ resolve placeholder)
4. 🟠 **MILESTONES.md** มีแค่ pos108/api — เติม repo ที่มี roadmap ชัด
5. 🟢 **leaf apps** (pos108/apps/{orders,slot}, BipByte/apps/{web,flutter-app}): ตัดสินใจชัดว่าจะลง HANDOFF อย่างเดียว หรือยกเว้น แล้วบันทึกใน templates/README.md

## วิธี refresh matrix
รันสคริปต์เช็ก (จาก controller): `ssh macstudio` แล้ว loop `find ~/108-Ting-Ecosystem -maxdepth 5 -name .git` เช็คไฟล์ H/C/A/M/D + `.ai_context` ต่อ repo
