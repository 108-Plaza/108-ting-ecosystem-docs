# 108 Ting Ecosystem Architecture

> **Status:** overview refreshed 2026-07-24. Live-state facts (ports/hosts) are VERIFY, not
> git-checkable; per-service detail lives in each platform's own docs.

## Purpose
108 Ting Ecosystem is the root workspace that contains multiple platform domains. It provides the high-level organization and cross-platform boundaries for the 108 product ecosystem.

## Live Platform Map (as of 2026-07-24)

All k3s services run on the single Dell node fronted by host nginx (see
[`DEPLOYMENT_ALLOCATION.md`](DEPLOYMENT_ALLOCATION.md)).

| Platform / repo | Role | Live state |
|-----------------|------|-----------|
| **pos108-core** | POS cloud API + control-plane (one binary; branch nodes run the same binary in branch mode) | k3s staging NodePort `30810` |
| **pos108-terminal** | Rust/Slint POS terminal — the ACTIVE POS front-end (i18n, self-update, Mac/Win/Linux bundles) | shipped to customer devices |
| **pos108-admin** | Next.js back-office for tenants | k3s staging (`admin.staging.108plaza.net`) |
| **pos108-orders** | table-QR / walk-in public ordering front-end | k3s staging |
| **pos108-store** | "108 Online" storefront SPA | LIVE `shop.108plaza.com`, NodePort `30830`; BFF staff token retired 2026-07-24 |
| **pos108-sell** | 108plaza.com marketing/sales site + operator console `admin.108plaza.com` | LIVE on k3s **prod** NodePort `30820` (pm2 retired 2026-07-22) |
| **Identity-Platform** | central auth (EdDSA JWTs; OTP passwordless; Google OAuth via env vars) | LIVE `id.108plaza.net`, NodePort `30110` |
| **Customer-Platform** | central customer/CRM registry (phone-first; split from Loyalty: Customer = identity, Loyalty = points) | LIVE k3s staging NodePort `30114` |
| **Loyalty-Platform** | central loyalty: points / wallet / gift-card, phone-keyed | repo active; loyalty features go HERE, not pos108 |
| **Notification-Platform** | email + OTP delivery (Postfix SMTP; transactional email wired into sell; `POST /api/v1/otp` for Identity) | LIVE k3s staging NodePort `30710` |
| **Secrets-Platform** | our own TRUE-E2EE secrets vault (ECDH P-256 + AES-GCM; vault stores ciphertext only; auth = Identity JWT `aud=secrets`) | LIVE k3s staging NodePort `30913` |
| **Media-Platform** | shared media services | see platform docs |
| **Payment-Platform** | payment gateway (SCB QR, 2C2P, multi-bank providers) | see platform docs |
| **AccountZing-Platform** | central accounting ledger (double-entry GL) for POS108 + 108-Zing | standalone repo, internal (ClusterIP, no host) |
| **BipByte-Platform** | 108Zing social-commerce stack (57 Rust services, ns `tixtox`) | api-gateway on k3s NodePort `30910` |

### Repo naming (renamed 2026-07-03 — old names are stale)
- `pos108` → **pos108-core** · `pos-rs` → **pos108-terminal** · `shoping-online` → **pos108-store**
- The Flutter POS client is **archived**; the Rust/Slint terminal is the active front-end.

### Domain map
- `108plaza.com` — sell site (marketing/sales, k3s prod)
- `shop.108plaza.com` — 108 Online store SPA
- `admin.108plaza.com` — operator console (owner SaaS back-office, on the sell process)
- `id.108plaza.net` — Identity
- `*.108plaza.net` subdomains — tenant shops / platform services (`.net` = tenant, `.com` = 108's own)

### Org-wide standards
- **sqlx 0.9 everywhere** (all 16 Rust repos; cargo-deny bans < 0.9; `AssertSqlSafe` for dynamic SQL).
- **Deploy** = `git push origin main:deploy/staging` → CI builds the GHCR image → k3s rollout.
- **CI** = 5 shared org runners (`dell-org-108plaza` + `dell-org-ci-1..4`) serve all 108-Plaza repos; no new per-repo runners for org repos.

### Tenancy model
**1 deploy = 1 tenant = 1 DB.** pos108 is NOT shared-DB multi-tenant; never seed two tenants
on one node.

## Platform Domains (workspace layout)
- Data-Platform: data services, analytics, data pipelines, or shared data infrastructure
- IoT-Platform: IoT device and hardware-connected services
- Logistics-Platform: logistics, delivery, routing, warehouse, or fulfillment services
- Payment-Platform: payment services and payment integrations
- Creator-Platform: creator/content-related services
- BipByte-Platform: 108Zing-related product services
- Commerce-Platform: commerce, POS, sales, pricing, customer, branch, promotion, and related services
- Platform-Services: shared platform services and internal utilities
- AccountZing-Platform: central accounting ledger (double-entry GL) for POS108 + 108-Zing (owner-approved standalone repo, 2026-06-15)

## Current Active Architecture Area
~/108-POS/core

## Boundary Rules
- Root ecosystem docs describe global direction only.
- Each platform owns its technical details.
- Do not couple unrelated platforms without explicit approval.
- Do not move code across platforms without a migration plan.
- Do not infer dependencies between platforms from names alone.

## Current Unknowns
- Exact platform ownership must be confirmed from existing docs/code.
- Cross-platform integration points must be documented before changes.
- POS108 API dependency graph must be confirmed inside ~/108-POS/core.
