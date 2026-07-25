# 108 Ting Ecosystem Architecture

> **Status:** overview refreshed 2026-07-24; structure diagram added 2026-07-25. Live-state facts
> (ports/hosts) are VERIFY, not git-checkable; per-service detail lives in each platform's own docs.

## Purpose
108 Ting Ecosystem is the root workspace that contains multiple platform domains. It provides the high-level organization and cross-platform boundaries for the 108 product ecosystem.

## Structure diagram

Live service map — how clients reach the edge, which frontends fan into the single POS
cloud API, and how the API talks to the central platforms and data stores. Ports are staging
NodePorts unless marked `(prod)`; the port families are explained in
[`DEPLOYMENT_ALLOCATION.md`](DEPLOYMENT_ALLOCATION.md).

```mermaid
graph TB
  classDef client fill:#e7ecf6,stroke:#274690,color:#0e1b3a;
  classDef edge   fill:#efeafa,stroke:#5b3ea6,color:#241247;
  classDef fe     fill:#faeeda,stroke:#854f0b,color:#412402;
  classDef core   fill:#e1f5ee,stroke:#0f6e56,color:#04342c;
  classDef plat   fill:#eaf3de,stroke:#3b6d11,color:#173404;
  classDef data   fill:#fcebeb,stroke:#a32d2d,color:#501313;

  TERM["pos108-terminal<br/>Rust/Slint POS · branch devices"]:::client
  GUEST["Guests / customers<br/>table-QR · walk-in · web"]:::client

  NGINX["host nginx edge — Dell node (ibrowe)<br/>routes by hostname · certbot TLS"]:::edge

  subgraph FE["Frontends / apps · NodePort family 303xx"]
    direction LR
    ADMIN["pos108-admin · 30310<br/>tenant back-office (Next.js)"]:::fe
    ORDERS["pos108-orders · 30312<br/>table-QR / walk-in (SolidJS)"]:::fe
    STORE["pos108-store · 30830<br/>shop.108plaza.com (SolidJS+BFF)"]:::fe
    SELL["pos108-sell · 30820 (prod)<br/>108plaza.com + operator console"]:::fe
  end

  CORE["pos108-core · 30810<br/>POS cloud API + control-plane<br/>branch nodes = same binary, branch mode"]:::core

  subgraph PLAT["Central platforms"]
    direction LR
    ID["Identity · 30110<br/>EdDSA JWT · OTP · Google OAuth"]:::plat
    CUST["Customer · 30114<br/>CRM registry · phone-first"]:::plat
    LOY["Loyalty<br/>points · wallet · gift-card"]:::plat
    NOTIF["Notification · 30710<br/>email + OTP · Postfix SMTP"]:::plat
    SEC["Secrets · 30913<br/>TRUE-E2EE vault"]:::plat
    MEDIA["Media<br/>uploads · CDN · live"]:::plat
    PAY["Payment · 30210<br/>SCB QR · 2C2P · multi-bank"]:::plat
  end

  AZ["AccountZing<br/>double-entry GL · ClusterIP (internal)"]:::plat
  BIP["BipByte / 108Zing · 30910<br/>57-service Rust stack · ns tixtox"]:::plat

  PG["pg-central + redis-central (staging)<br/>Mac mini .68 · PG16 + Redis (prod)"]:::data

  TERM -->|X-Terminal-Id| CORE
  GUEST --> NGINX
  NGINX --> ADMIN
  NGINX --> ORDERS
  NGINX --> STORE
  NGINX --> SELL
  ADMIN -->|same-origin /api| CORE
  ORDERS -->|public token| CORE
  STORE -->|BFF platform-key| CORE
  SELL -->|auth| ID
  BIP -->|POST /api/v1/sales| CORE

  CORE --> ID
  CORE --> CUST
  CORE -. sale.completed .-> LOY
  CORE --> PAY
  CORE -->|outbox events| NOTIF
  CORE --> AZ
  ID -->|OTP adapter| NOTIF
  CUST --- LOY

  CORE --> PG
  ID --> PG
  CUST --> PG
  LOY --> PG
  NOTIF --> PG
  SEC --> PG
  PAY --> PG
```

Legend: blue = clients · purple = edge · amber = frontends · green = POS core ·
olive = central platforms · red = data. Solid arrow = synchronous call; dotted = async
event (`sale.completed` → Loyalty accrual, order-independent); plain line = same phone-key
domain (Customer ↔ Loyalty).

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
