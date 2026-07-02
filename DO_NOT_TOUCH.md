# 108 Ting Ecosystem Do Not Touch List

> **Source of Truth (SoT) ของ "เขตห้ามแตะ".** ไฟล์อื่น (`CLAUDE.md` → Forbidden Scope,
> `templates/HANDOFF.template.md` → Do Not Touch, per-repo `docs/DO_NOT_TOUCH.md`) ต้อง **ชี้มาที่ไฟล์นี้**
> ไม่ลิสต์ซ้ำ เปลี่ยนรายการ → แก้ที่นี่ที่เดียว

Claude must not modify or work on these areas unless explicitly approved by the user.

## Forbidden Business Areas
- payroll
- accounting
- GL-posting
- finance DAG
- canary extraction
- TikTok app

> **Approved exception (2026-06-15) — AccountZing-Platform:** `accounting` / `GL-posting` / `finance DAG`
> ห้าม *งานบัญชีภายใน platform อื่น* (เช่น POS108-local GL) เท่านั้น — **ไม่** ครอบคลุม repo standalone
> ที่ได้รับอนุมัติ **`AccountZing-Platform/`** (central ledger ของ POS108 + 108-Zing, owner-approved
> 2026-06-15). ทำงานจาก session ที่ root = `AccountZing-Platform/` ตาม docs ของ repo นั้นเอง
> (`docs/SPEC-001-*.md`, `docs/build-plan/`, `.ai_context/`) — ไม่ใช่ pos108 docs. รัน git/CI/lifecycle ของตัวเอง
> นี่คือข้อยกเว้น accounting เดียวของ Forbidden Scope. *(SoT ของ exception นี้อยู่ที่ไฟล์นี้ — ที่อื่นชี้มา)*

## Platform Scope Rules
- Do not work outside the active work area.
- Do not switch from POS108 to another platform unless explicitly approved.
- Do not infer that Payment, Logistics, Data, IoT, Creator, BipByte, or Platform-Services need changes unless the handoff says so.

## Technical Restrictions
- Do not modify production database settings.
- Do not run production migrations.
- Do not modify deployment configuration.
- Do not access or modify secrets/credentials.
- Do not rewrite large parts of the ecosystem.
- Do not create a new architecture when a handoff exists.

## Allowed Default Work
- Read docs.
- Inspect files.
- Update approved handoff/control documents.
- Propose next exact action.
- Work inside the active work area only after confirmation.
