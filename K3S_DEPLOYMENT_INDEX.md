# 108 Ting Ecosystem — K3s Deployment Docs Index & Plan

> The map of the K3s deployment documentation: what each doc/artifact is, the order to
> read them, what's authored vs still to write. Start here.
>
> **Status:** 2026-06-24. Architecture proven; live in staging: Notification, **pos108 API
> (`30810`, rev 4 `sha-8824860`)** + pos108-admin, BipByte (ns `tixtox`), image-processing-engine.

---

## Reading order (newcomer → operator)

1. **[DEPLOYMENT_STANDARD_K3S.md](DEPLOYMENT_STANDARD_K3S.md)** — *the standard (why/where).*
   Locked decisions, topology, placement matrix, subdomain naming, the **proven
   nginx-edge → NodePort** model. Read the ⚡ banner first.
2. **[DEPLOYMENT_ALLOCATION.md](DEPLOYMENT_ALLOCATION.md)** — *the exact map.* Every
   service × env → hostname, NodePort, `/api` backend, IP. The registry you wire from.
3. **[K3S_ONBOARDING_PLAYBOOK.md](K3S_ONBOARDING_PLAYBOOK.md)** — *the how.* Cluster
   baseline checklist, the shared chart, per-service onboarding gate, rollout order.
4. **[../deploy/k3s/README.md](../deploy/k3s/README.md)** — *baseline runbook.* Apply
   order for the Wave-0 manifests + BIND zone (`108plaza.net.zone`).
5. **[../deploy/charts/ting-service/README.md](../deploy/charts/ting-service/README.md)**
   — *the shared chart.* One chart, per-service values; NodePort + probes + secrets.

---

## Document & artifact inventory

| Artifact | Kind | Covers | Status |
|----------|------|--------|--------|
| `docs/DEPLOYMENT_STANDARD_K3S.md` | reference | standard, decisions, placement, naming, nginx-edge, image-build trigger (§4a) | ✅ current |
| `docs/DEPLOYMENT_ALLOCATION.md` | reference | IP/port/host per service × env | ✅ current |
| `docs/K3S_ONBOARDING_PLAYBOOK.md` | guide | executable onboarding plan + gotchas | ✅ current |
| `docs/K3S_DEPLOYMENT_INDEX.md` | index | this map | ✅ current |
| `deploy/k3s/*.yaml` + `README.md` | manifests | namespaces, quota, netpol, datastores, NATS | ✅ applied (staging) |
| `deploy/k3s/108plaza.net.zone` | config | corrected BIND zone (`*.staging` → Dell) | ✅ live |
| `deploy/k3s/bootstrap.sh` | script | one-shot baseline apply (idempotent) | ✅ used |
| `deploy/k3s/nginx/*.conf` | templates | host nginx vhost (backend + same-origin frontend) | ◐ notify done |
| `deploy/charts/ting-service/` | helm chart | shared service chart (+ examples) | ✅ v0.2.0, proven |
| `deploy/k3s/04-*` (Traefik/RFC2136) | manifests | wildcard TLS in K3s | 🅿️ **parked** (nginx-edge used instead) |

---

## Documentation still to write (the doc-work plan)

Ordered by when they're needed:

1. **Frontend deploy reference** `[next]` — the Next.js (`output: standalone`) Dockerfile
   pattern + CI image-build + same-origin nginx vhost, produced while onboarding
   **pos108-admin**. Becomes the template for pos/orders/slot/tixtox-web.
2. **Per-service onboarding records** — a short block per service (image, NodePort, DB
   role, host, status) appended to `DEPLOYMENT_ALLOCATION.md` §6 as each lands. (Registry-style.)
3. **Day-2 operations runbook** `[before more prod traffic]` — cert renewal (certbot
   timer), backup/restore (Mac mini prod DB + rehearsed drill), log access, rolling
   redeploy, NodePort firewalling, capacity watch on the single node.
4. **Observability setup** — install kube-prometheus-stack, flip
   `serviceMonitor.enabled=true`, dashboards + alerts (outbox backlog, consumer lag,
   ledger imbalance, pod restarts/OOM).
5. **Legacy→K3s cutover** — moving live hosts (`api.108plaza.net`→:18000,
   `api.pos108`→:8090) from the existing nginx backends onto K3s NodePorts without
   downtime; the order and the rollback.
6. **GitOps/repo move** — put `deploy/` in a git repo (`108-Plaza/platform-deploy`) so
   the Dell `git pull`s instead of rsync/scp; optionally CI `helm upgrade` from a
   self-hosted runner. (Kills the current sync friction.)
7. **Secrets hardening** — migrate the manual `kubectl create secret` to **Sealed
   Secrets** (git-committed) once GitOps lands; document the seal workflow.

> Items 1–2 ride along with the next service deploys (no separate effort). 3–7 are
> standalone docs, write them when their trigger hits.

---

## Conventions (quick reference)

- **Hosts:** `<svc>.staging.108plaza.net` (staging) / `<svc>.108plaza.net` (prod);
  internal services (AccountZing) get **no host**.
- **NodePorts:** `300xx` staging backend · `301xx` staging frontend · `310xx` prod
  backend · `311xx` prod frontend (cluster-wide unique — see allocation §2).
- **Frontends:** same-origin — `NEXT_PUBLIC_API_URL = https://<own-host>`, nginx
  `/api → pos108`, `/ → frontend`.
- **TLS:** `certbot --nginx -d <host>` per host (HTTP-01). No wildcard/Traefik.
- **Secrets:** `kubectl create secret` now → Sealed Secrets after GitOps.
- **Image:** `ghcr.io/108-plaza/<svc>:<tag>`, **built only on the repo's `deploy/staging`
  branch** (never on `main` — see standard [§4a](DEPLOYMENT_STANDARD_K3S.md)); pull via
  `ghcr-pull` (classic PAT, `read:packages`).

*Authored 2026-06-19. The single entry point for K3s deployment docs — keep the
inventory + doc-work plan current as docs land.*
