# 108 Ting Ecosystem — K3s Deployment Allocation (staging + production)

> **The concrete "which service uses which IP / port / host" map.** Companion to
> [`DEPLOYMENT_STANDARD_K3S.md`](DEPLOYMENT_STANDARD_K3S.md) (the *why/standard*) and
> [`K3S_ONBOARDING_PLAYBOOK.md`](K3S_ONBOARDING_PLAYBOOK.md) (the *how*). This doc is the
> *where, exactly* — the allocation table you reach for when wiring a service.
>
> **Status:** authored 2026-06-19. Decisions are made (not open questions); assumptions
> are stated inline.

---

## Locked decisions

- **Edge = host nginx → K3s NodePort.** Proven (Notification live). No Traefik. Per
  service: nginx vhost → `127.0.0.1:<NodePort>` → pod; `certbot --nginx` for TLS.
- **Frontends are same-origin.** A frontend + the API it calls share ONE host:
  `<host>/` → frontend NodePort, `<host>/api/…` → pos108 API. So
  `NEXT_PUBLIC_API_URL = https://<frontend-host>` (no separate API host, no CORS).
- **pos108 API = ONE binary, not split.** All POS frontends hit the same `/api`.
- **NodePort scheme — family-encoded, within K3s's `30000–32767`** (so literal
  `801X`/`901X` can't be NodePorts — encoded as the 3rd digit instead):

  **`3` + env(`0`=staging / `1`=prod) + family + 2-digit index**

  | Family | digit | staging block | prod block |
  |--------|-------|---------------|------------|
  | pos108 / Commerce API | **8** | `308XX` | `318XX` |
  | BipByte API | **9** | `309XX` | `319XX` |
  | Frontends (all) | **3** | `303XX` | `313XX` |
  | Platform-Services API (identity/notification/media) | **1** | `301XX` | `311XX` |
  | Finance API (payment) | **2** | `302XX` | `312XX` |
  | Other backends (creator/logistics/data) | **4** | `304XX` | `314XX` |

  Max port `319XX = 31999 < 32767` ✓. NodePorts bind `0.0.0.0` → firewall so only
  local nginx reaches them.
- **Databases:** staging = in-cluster (`pg-central`/`redis-central` in ns `staging`);
  prod = Mac mini `.68` (external, via Endpoints).

---

## 1. Infrastructure IPs

| Role | IP | Runs |
|------|-----|------|
| **Dell tower** (`ibrowe`) | **`103.27.202.40`** (public) | K3s single node (ns `prod`+`staging`) · host **nginx** edge (`:80/:443`, certbot) · BIND (`ns1`/`ns2.108jobs.com`) · existing legacy services |
| **Mac mini** (`.68`) | `<MACMINI_LAN_IP>` (LAN) | **prod** Postgres@16 + Redis (external, NOT a K3s node) |
| K3s pod / svc CIDR | `10.42.0.0/16` / `10.43.0.0/16` | in-cluster only |

DNS: `*.staging.108plaza.net` + `*` (`= *.108plaza.net`) → **`103.27.202.40`**. Everything
public (staging *and* prod) resolves to the Dell; nginx routes by hostname.

---

## 2. NodePort allocation

| Service | Family | Staging | Prod | Host (staging) |
|---------|--------|---------|------|----------------|
| identity | platform(1) | `30110` | `31110` | `auth.staging.108plaza.net` |
| **notification** | platform(1) | **`30084`** ✅ live¹ | `31111` | `notify.staging.108plaza.net` |
| media | platform(1) | `30112` | `31112` | `media.staging.108plaza.net` |
| payment | finance(2) | `30210` | `31210` | `pay.staging.108plaza.net` |
| accountzing | finance(2) | — *(ClusterIP)* | — | *(internal, no host)* |
| **pos108 API** (cloud) | pos108(8) | **`30810`** ✅ live⁴ | `31810` | `api.staging.108plaza.net` |
| creator | other(4) | `30410` | `31410` | `creator.staging.108plaza.net` |
| delivery | other(4) | `30420` | `31420` | `delivery.staging.108plaza.net` |
| data-platform (stratum) | other(4) | `30430` | `31430` | `stratum.staging.108plaza.net` |
| **BipByte API** (api-gateway) | bipbyte(9) | **`30910`** ✅ live² | `31910` | `bipbyte.staging.108plaza.net` |
| BipByte realtime-edge | bipbyte(9) | `30911` | `31911` | `bipbyte-ws.staging.108plaza.net` |
| BipByte image-processing-engine | bipbyte(9) | — *(ClusterIP)*³ | — | *(internal, queue worker)* |
| pos108-admin (fe) | frontend(3) | `30310` | `31310` | `admin.staging.108plaza.net` |
| pos108-pos (fe) | frontend(3) | `30311` | `31311` | `pos.staging.108plaza.net` |
| pos108-orders (fe) | frontend(3) | `30312` | `31312` | `orders.staging.108plaza.net` |
| pos108-slot (fe) | frontend(3) | `30313` | `31313` | `slot.staging.108plaza.net` |
| BipByte web (fe) | frontend(3) | `30314` | `31314` | `bipbyte-web.staging.108plaza.net` |
| BipByte admin (fe) | frontend(3) | `30315` | `31315` | `bipbyte-admin.staging.108plaza.net` |

¹ **Notification is LIVE at `30084`** (assigned before this scheme). It logically belongs
at `30111`; renumber when convenient (needs a redeploy + its nginx vhost `proxy_pass` change).

² **BipByte is a 57-service stack** (`tix-tox-clone`), deployed to ns **`tixtox`** —
not the old single "server" binary. (The 3 C/C++ engines are excluded — no images built; sibling
`tixtox-engines` repo — so the deployable stack is 57 Rust services, per `deploy/tixtox/README.md` +
`service-list.txt`.) Only **api-gateway** is exposed (NodePort `30910`, `/health` 200 from
`103.27.202.40:30910`); the other **56** Rust services (auth, feed, media, payment, live, …) are
**ClusterIP**, reached internally via the gateway's `*_SERVICE_URL` env. Manifests: root project
`deploy/tixtox/` (kustomize, `api-gateway-ext` NodePort in `91-api-gateway-nodeport.yaml`); images
`tixtox-*:latest` built on-node (`IfNotPresent`).
No ingress controller here (Traefik disabled) so `90-ingress.yaml` is inert — NodePort only.

³ **image-processing-engine is a Redis-Streams queue worker, not an HTTP service** — no
NodePort, no host vhost. Deployed (ns **`staging`**) via the shared `ting-service` chart;
LIVE since 2026-06-20. It consumes `image:jobs` → ONNX beauty filter → `image:results`.
Runbook: [`../deploy/k3s/IMAGE_PROCESSING_ENGINE_STAGING.md`](../deploy/k3s/IMAGE_PROCESSING_ENGINE_STAGING.md);
values: `deploy/charts/ting-service/examples/values.image-processing-engine.staging.yaml`.

⁴ **pos108 API (cloud) is LIVE in staging** since 2026-06-24 — helm release `pos108` (ns `staging`,
chart `ting-service` 0.2.0, NodePort `30810`), **rev 4 = `ghcr.io/108-plaza/pos108:sha-8824860`**
(= origin/main #426); `/health/ready` ok (DB `pg-central.staging` + Redis). Reached today via each
frontend's same-origin `/api` (admin/pos.staging). The direct host `api.staging.108plaza.net` vhost is
prepared but **not yet applied** — cert pending (owner runs `certbot --nginx`; staged at Dell
`/tmp/api.staging.108plaza.net`).

---

## 3. Hostname → backend — STAGING (`*.staging.108plaza.net` → 103.27.202.40)

**Backends** (nginx `location /` → NodePort): use the host + NodePort from §2.
AccountZing is ClusterIP-only (`accountzing.staging.svc` — no host, money ledger off the internet).

**Frontends** (same-origin: `/` → frontend NodePort, `/api/` → pos108 API `30810`):

| Host | `/` → | `/api/` → | `NEXT_PUBLIC_API_URL` |
|------|-------|-----------|------------------------|
| `admin.staging.108plaza.net` | 30310 | pos108 30810 | `https://admin.staging.108plaza.net` |
| `pos.staging.108plaza.net` | 30311 | pos108 30810 | `https://pos.staging.108plaza.net` |
| `orders.staging.108plaza.net` | 30312 | pos108 30810 | `https://orders.staging.108plaza.net` |
| `slot.staging.108plaza.net` | 30313 | pos108 30810 | `https://slot.staging.108plaza.net` |

> **pos108 (cloud) is now on K3s** (`30810`, live 2026-06-24): the frontends' `/api/` point at
> `30810` (admin.staging proven). The legacy `127.0.0.1:8090` / `api.pos108.108plaza.net` backend
> is superseded for staging.

---

## 4. Hostname → backend — PRODUCTION (`*.108plaza.net` → 103.27.202.40)

Same slugs **without** `.staging`; **prod** NodePorts (swap `30`→`31`: pos108 `31810`,
admin `31310`, …); DB on Mac mini `.68`. `api.108plaza.net` currently → legacy `:18000`
— cut over to K3s prod pos108 (`31810`) when prod is promoted (coordinate, it's live).

---

## 5. nginx vhost patterns

**Backend** — `deploy/k3s/nginx/<host>.conf`:
```nginx
server {
    listen 80; listen [::]:80;
    server_name notify.staging.108plaza.net;
    location / { proxy_pass http://127.0.0.1:30084; include /etc/nginx/proxy_params; }
}
# certbot --nginx -d notify.staging.108plaza.net
```

**Frontend (same-origin)** — `/api` to pos108, the rest to the frontend:
```nginx
server {
    listen 80; listen [::]:80;
    server_name admin.staging.108plaza.net;
    location /api/ { proxy_pass http://127.0.0.1:30810; include /etc/nginx/proxy_params; }  # pos108 API
    location /     { proxy_pass http://127.0.0.1:30310; include /etc/nginx/proxy_params; }  # admin frontend
}
# certbot --nginx -d admin.staging.108plaza.net
```

---

## 6. Status & rollout order

- ✅ **Notification** (`30084`) — LIVE at `https://notify.staging.108plaza.net`.
- ✅ **pos108 API** (`30810`) — LIVE, `/health/ready` ok (DB+Redis), bootstrap tenant
  seeded. Reached via each frontend's same-origin `/api`. **Image rev 4 = `sha-8824860`**
  (= origin/main #426), built via `release-image.yml` workflow_dispatch, 2026-06-24. Direct
  host `api.staging.108plaza.net` vhost prepared, cert pending (owner applies).
- ✅ **pos108-admin** (`30310`) — LIVE PUBLIC at `https://admin.staging.108plaza.net`
  (same-origin `/api`→pos108 proven, 401 JSON). Image #228 + lockfile fix #229.
  Staging admin login: `admin` / (set at deploy, see secret `pos108-secrets`).
- ✅ **BipByte API** (`30910`) — LIVE in ns `tixtox`: full `tix-tox-clone` stack (**57 Rust
  services**; the 3 C/C++ engines excluded), api-gateway `/health` 200 (internal +
  `103.27.202.40:30910`). Deployed via **`deploy/tixtox/`** kustomize (on-node images); only
  api-gateway exposed (NodePort), rest ClusterIP. nginx vhost `bipbyte.staging.108plaza.net` →
  `127.0.0.1:30910` **is wired** (`deploy/k3s/nginx/bipbyte.staging.108plaza.net.conf`); TLS via
  certbot — VERIFY cert issuance (live-state).
- **Next (staging):** pos108-pos (`30311`) + orders (`30312`) + slot (`30313`)
  frontends (same recipe, same-origin /api). Identity (`30110`) + Payment (`30210`).
  AccountZing = build Dockerfile first, stays ClusterIP.
- **Prod:** promote the staging set into ns `prod` (`31xxx` ports, `.68` DB) once stable.

> ⚠️ **NetworkPolicy gotcha:** the staging `allow-ingress-traefik-and-intra` rule must
> allow intra-namespace on ALL ports (2 rules), not just 8080 — else app→postgres:5432
> is blocked (manifests as "pool timeout"). Both prod+staging fixed in `03-networkpolicy.yaml`.

> This table is the **registry** — add a row / confirm the port when you assign one.

*Authored 2026-06-19. Update when a NodePort/host is assigned or a service goes live.*
