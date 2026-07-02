# 108 Ting Ecosystem — K3s Onboarding Playbook

> **Companion to** [`DEPLOYMENT_STANDARD_K3S.md`](DEPLOYMENT_STANDARD_K3S.md) (the
> *where / naming / locked decisions*). This doc is the **how**: turning the standard
> into an executable plan that loads the ecosystem onto the **K3s cluster that is
> already running on the Dell tower**.
>
> **Status:** plan, 2026-06-19. No service code/CI changed by this doc — each step is
> its own slice + PR in the owning repo.
>
> **Starting point (changed):** K3s is **already installed** on the Dell. So "Wave 0"
> is **configure**, not install. The real work is **artifacts + onboarding**, because
> most services still lack a deployable chart/manifest (see §4).
>
> ## ⚡ PROVEN ingress model (2026-06-19) — nginx-edge, NOT Traefik
>
> The Dell already runs **system nginx on `:80/:443`** (certbot) fronting live sites;
> K3s has **no ingress controller**. So K3s services are **NodePort** + the existing
> nginx reverse-proxies each host → `127.0.0.1:<nodePort>` + `certbot --nginx` per host.
> **Traefik/RFC2136 is parked.** **Notification is LIVE this way:**
> `https://notify.staging.108plaza.net/health` → HTTP/2 200. Per-service recipe:
> image (GHCR) → `kubectl create secret` (DB url, redis, JWT/HMAC) → DB role → `helm
> upgrade --install … --set image.tag=latest --set metrics.serviceMonitor.enabled=false`
> → NodePort → nginx vhost (`deploy/k3s/nginx/`) → certbot. (Mind the NetworkPolicy:
> intra-namespace must allow ALL ports — fixed.)

---

## 0. The honest gap — what "make it a system" actually needs

The cluster is the *easy* part (it exists). Three gaps stand between "K3s running" and
"the 108 ecosystem running as a system":

1. **Platform baseline not yet wired** — namespaces +
   isolation, secrets, registry pull, external-DB wiring, staging infra. One-time. §1.
2. **Per-service deploy artifacts mostly missing** — only Notification (Helm) and
   Creator (k8s) are deploy-ready today; everything else needs a chart/values, and
   **AccountZing needs a Dockerfile from zero**. This is the bulk of the effort. §3–§4.
3. **No onboarding gate** — no shared "definition of in-cluster," so each service
   would improvise. §3 fixes that with one checklist + one shared chart.

---

## 1. Cluster baseline — configure the existing K3s (one-time, before any app)

> **Manifests drafted + YAML-validated** at [`../deploy/k3s/`](../deploy/k3s/) (numbered
> apply order + REPLACE markers + prereqs in its `README.md`). The checklist below maps
> 1:1 to those files.

Run these against the running cluster. Each is a checkable gate.

- [ ] **Access + facts** — confirm `kubectl`/`helm` access (copy `/etc/rancher/k3s/k3s.yaml`
      → `~/.kube/config`); note `k3s` version, default StorageClass (`local-path`),
      `metrics-server` on (HPA), and that there is **no ingress controller** (edge = host nginx).
- [ ] **TLS / edge** — *no Traefik step.* The existing system nginx is the edge; TLS is
      per-host `certbot --nginx -d <host>` when each service is exposed (§9 of the
      standard). Use `certbot --staging` first if testing, then the real cert.
      (Appendix A's Traefik+RFC2136 wildcard path is parked.)
- [ ] **Namespaces** — create `prod` + `staging`, each with `ResourceQuota`,
      `LimitRange`, default-deny `NetworkPolicy`, and a `PriorityClass`
      (`prod` > `staging`). (§3 of the standard.)
- [ ] **Secrets** — for manual bring-up, create secrets **directly** with
      `kubectl create secret` (proven simplest). **Sealed Secrets** controller is only
      needed once you commit secrets to git (GitOps) — defer it until then.
- [ ] **Registry pull** — create the **GHCR** `ghcr-pull` Secret in both namespaces
      from a **classic PAT with `read:packages`** (`kubectl create secret docker-registry
      ghcr-pull --docker-server=ghcr.io …`). Fine-grained PAT is finicky with org packages.
- [ ] **External DB wiring** — `ExternalName` Service (or `Endpoints`) `pg-central` in
      `prod` → Mac mini `.68` LAN IP:5432. Confirm the Dell can reach `.68:5432`/`:6379`
      (firewall/route). Provision per-service DB + least-privilege role on `.68`
      (reuse `deploy/postgres-init/01-init-databases.sql`).
- [ ] **Staging infra** — small StatefulSets in `staging`: `postgres:16-alpine`,
      `redis:7-alpine`, `nats:2-alpine` (mirrors `deploy/docker-compose.infra.yml`),
      PVCs on `local-path`.
- [ ] **Observability** — `monitoring` ns: Prometheus + Grafana; scrape `/metrics`
      (pos108 #310, Notification, Payment, Data already expose it).

> When all boxes are checked, the cluster can host its first app. Nothing above
> touches a service repo.

---

## 2. The accelerator — ONE shared Helm chart (not N bespoke charts)

Most ecosystem backends are **homogeneous**: Rust/axum, listen `:8080`, `/health/live`
+ `/health/ready`, env-only config, migrations at boot, graceful shutdown,
stateless-multinode. That is exactly the shape BipByte's own design (D-PD2) flagged
for "Helm parameterized over a service table." So:

- **Build one `ting-service` Helm chart** (model on the proven Notification chart:
  Deployment + Service + ConfigMap + sealed Secret `envFrom` + HPA + hardened
  securityContext + probes + optional in-chart PG for staging). Publish it once as an
  **OCI chart** in GHCR (`oci://ghcr.io/108-plaza/charts/ting-service`).
- **Each service = a `values.<env>.yaml`** (image tag, host slug per §10, env keys,
  secret keys, replicas, resources). No per-repo chart authoring.
- Each repo's **CI (self-hosted runner)** runs
  `helm upgrade --install <svc> oci://…/ting-service -n <env> -f values.<env>.yaml`.
- **Exceptions keep their own chart:** Notification (already has a complete chart) and
  any service that is genuinely not the homogeneous shape (Data-Platform's
  Redpanda/MinIO stack; BipByte's 60-service compose; edge backends).

> **Decision: APPROVED (owner, 2026-06-19)** — shared `ting-service` chart is the
> central standard. **Prototype drafted + validated** at
> [`../deploy/charts/ting-service/`](../deploy/charts/ting-service/) (modeled on the
> Notification chart; `helm lint` clean, renders correctly for public / prod-HPA /
> internal-only via the three files in `examples/`). Next: publish it as the OCI
> chart and onboard services with per-env values files.

---

## 3. Per-service onboarding gate — "definition of in-cluster"

A service is **not** "in K3s" for an environment until every box is checked. Copy this
block into the owning repo's HANDOFF per env (`staging` then `prod`):

```
[ ] Image: multi-stage, --release, non-root, HEALTHCHECK; pushed ghcr.io/108-plaza/<svc>:<git-sha>
[ ] DB: prod → role+db on Mac mini .68 ; staging → in-cluster StatefulSet
[ ] Secrets: kubectl create secret (DB url, redis, JWT/HMAC, API keys); --set existingSecret
[ ] values.<env>.yaml: service.type NodePort + nodePort=<3xxxx>; ingress.enabled=false
[ ] Route: nginx vhost (deploy/k3s/nginx/) host=<slug>.staging.108plaza.net → 127.0.0.1:<nodePort>
[ ] Cert: certbot --nginx -d <host> ; https://<host> 200 ; 401→200 auth behaves
[ ] Migrations: applied at boot, _sqlx_migrations count matches
[ ] Health: /health(/ready) = 200 via NodePort and via https
[ ] Metrics: ServiceMonitor (after kube-prometheus-stack) or pod annotations
[ ] Rollback: prior image tag redeploys cleanly (additive-only migrations)
```

This is the per-service application of the ecosystem GATE-T/GATE-P bar
(`PRODUCTION_READINESS_BAR.md`).

---

## 4. Ordered execution — mapped to real artifact readiness

> Onboard in dependency order; prove each in `staging` before `prod`. The constraint
> is **artifact readiness**, not the cluster.

### Step 1 — Deployable TODAY (artifacts already exist) → prove the platform
| Service | What's there | Action |
|---|---|---|
| **Notification** | ✅ **LIVE in staging (2026-06-19)** | `https://notify.staging.108plaza.net/health` → HTTP/2 200. Deployed via the shared chart (`values.notification.staging.yaml`, NodePort 30084) → image `:latest` (its own ci.yml) → kubectl secrets → nginx vhost + certbot. **The proven reference recipe** for every other service. Next: `prod`. |
| **Creator** | ✅ k8s manifests | Adapt to namespaces + Traefik + Sealed Secrets; deploy `staging`. |

### Step 2 — Need values on the shared chart (Dockerfile OK, no manifests yet)
Author `ting-service` chart (§2) first, then a values file each. Order = boot spine:
**Identity → pos108(cloud) → Payment**, then **Media**, **Logistics**, **Data-Platform**.
| Service | Gap | Notes |
|---|---|---|
| **Identity** | values only | Boot #1 (issuer/JWKS). Prod secrets now required. Host `auth.*`. |
| **pos108 (cloud)** | values only (Dockerfile fixed #302) | `APP_ENVIRONMENT=cloud` only. Host `pos.*`. Branch nodes stay at-site. |
| **Payment** | values only | Consumer; inbound SCB webhooks → host `pay.*`. |
| **Media** | values (k8s after #3) | 9 services; host `media.*`. |
| **Logistics** | values | Host `delivery.*`. |
| **Data-Platform** | own manifests (Redpanda/MinIO) | Async; **not** POS-critical; money-Gold still gated by reconciliation. Lower priority/quota. |

### Step 3 — Build artifacts FIRST, then onboard
| Service | Gap | Notes |
|---|---|---|
| **AccountZing** | 🔴 **no Dockerfile at all** | Build multi-stage Dockerfile + values **before** placement. **Internal-only — no subdomain** (ClusterIP; ledger off the internet). Auth/scheduler already wired (#7/#8). |

### Step 4 — Aligned later / their own track
| Service | Notes |
|---|---|
| **BipByte (108Zing)** | Honor its own approved decision (compose-on-VM first, k8s post-launch). 60 services → **capacity-check the single node** before prod. Public: `bipbyte.*`, `bipbyte-ws.*`; ~50 internal stay ClusterIP. |
| **Frontends** (pos/admin/orders/slot, bipbyte-web/admin) | Next.js → Deployments; Flutter web → static (nginx pod or CDN). Hosts per §10. |
| **IoT cloud backends** (Smart-Farm/Home) | Cloud portion may join later; **edge gateways stay at-site**. |

---

## 5. Single-node gotchas to plan around

- **No HA** — one K3s node + prod DB on one Mac mini. Resilience = **backup/restore +
  fast redeploy**, not failover. Schedule `pg_dump`/PITR on `.68`; rehearse a restore.
- **Capacity** — the node hosts `prod` + `staging` + (eventually) BipByte's 60 svc.
  Enforce `ResourceQuota` so staging can't starve prod; size BipByte before committing.
- **Boot ordering** — Identity must be ready before pos108, pos108 before consumers
  (the outbox needs a live producer). In k8s use readiness gates + retries, not
  compose `depends_on`.
- **External DB reachability** — pods → `.68:5432/6379` crosses the LAN; lock the
  firewall to the Dell only, and `NetworkPolicy` so only the namespaces that need it
  egress there.
- **NetworkPolicy intra-namespace = ALL ports** — a port-restricted ingress rule
  (e.g. 8080-only) silently blocks app→`postgres:5432`/`redis:6379`, so the app hangs
  on DB connect and never binds; probes show `connection refused`. (Real bug hit +
  fixed 2026-06-19 in `03-networkpolicy.yaml`.)
- **ServiceMonitor needs the Prometheus CRDs** — deploy with
  `--set metrics.serviceMonitor.enabled=false` until kube-prometheus-stack is installed.
- **helm vs k3s kubeconfig** — `k3s kubectl` auto-finds `/etc/rancher/k3s/k3s.yaml`;
  standalone `helm` does not → copy it to `~/.kube/config` (chown to you).
- **Let's Encrypt rate limits** — `certbot --staging` while wiring, then the real cert.
- **Migrations at boot** — a failed migration aborts startup; the old pod keeps
  serving until the new one is `Ready` (safe rollback). Keep migrations additive-only.

---

## 6. Status + next actions

**Done (2026-06-19):** ✅ `*.staging` DNS → Dell · ✅ baseline §1 staging (namespaces,
priority, quota, **NetworkPolicy** [bug fixed], pg-central/redis-central/nats) · ✅
`ting-service` shared chart (NodePort support) · ✅ **Notification LIVE** at
`https://notify.staging.108plaza.net` (image `:latest` from its ci.yml, kubectl secrets,
nginx vhost + certbot).

**Next:**
1. **Reduce friction** — put `deploy/` in a git repo (`108-Plaza/platform-deploy`) so
   the Dell `git pull`s instead of rsync/scp per file.
2. **Onboard Identity → pos108(cloud) → Payment** to `staging` via the proven recipe
   (NodePort + values + DB role + nginx vhost + certbot). Mind: Identity is the boot
   spine; pos108 = `APP_ENVIRONMENT=cloud` only.
3. **Build AccountZing's Dockerfile** (parallel — the one hard blocker; internal-only,
   no nginx vhost).
4. **kube-prometheus-stack** → flip `serviceMonitor.enabled=true` → metrics in Grafana.
5. **Promote to `prod`** namespace once the staging set is stable (external `.68` DB).

---

*Authored 2026-06-19 as the K3s onboarding plan for the existing Dell-tower cluster.
Companion to DEPLOYMENT_STANDARD_K3S.md. Update as waves complete.*
