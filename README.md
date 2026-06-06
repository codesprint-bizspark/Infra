# Infra — Bizpark GitOps

Kubernetes (K3s) manifests for the **BizSpark AI** platform, delivered via **ArgoCD** GitOps. Everything that runs in the cluster is declared here; ArgoCD continuously reconciles the cluster to match this repo.

ArgoCD tracks:

```text
apps/bizpark/overlays/prod
```

The BE repository's CI updates `apps/bizpark/overlays/prod/kustomization.yaml` with immutable `ghcr.io/codesprint-bizspark/*` image **digests** after every successful build on `main`. (The frontend image is pinned by `sha-<commit>` tag in `base/workloads.yaml` — its CI auto-pin flakes, so it's bumped manually.)

> **Primary domain: `bizspark.online`** (migrated from `randitha.net`). `randitha.net` hosts still resolve and are redirected to `bizspark.online` at the Cloudflare edge.

---

## Deployment Architecture

### CI/CD — GitOps flow (how code reaches the cluster)

```
  ┌─────────────────────┐   ┌──────────────────────┐
  │ Bizpark--AI-BE      │   │ BizSpark-AI---FE     │   push to main
  │ (API/Admin/Commerce │   │ (SaaS dashboard)     │ ──────────────┐
  │  /Web/Runner/MCP)   │   └──────────┬───────────┘               │
  └──────────┬──────────┘              │                           │
             │ push to main            │                           ▼
             ▼                         ▼                ┌────────────────────────┐
   ┌────────────────────────────────────────┐          │  GitHub Actions (CI)   │
   │  GitHub Actions — build 7 images        │─────────▶│  build → push → digest │
   └──────────────────────┬──────────────────┘          └───────────┬────────────┘
                          │ push images                             │ write image digests
                          ▼                                         ▼
                 ┌──────────────────┐                  ┌──────────────────────────┐
                 │  GHCR registry   │                  │  Infra repo (THIS repo)  │
                 │  ghcr.io/...     │                  │  overlays/prod/kustomize │
                 └────────┬─────────┘                  └────────────┬─────────────┘
                          │  pull (by digest)                       │ watches main
                          │                                         ▼
                          │                              ┌────────────────────┐
                          └─────────────────────────────▶│  ArgoCD (in-cluster)│
                                                         │  app: bizpark-prod  │
                                                         └─────────┬──────────┘
                                                                   │ sync → apply
                                                                   ▼
                                                          K3s cluster (below)
```

### Runtime topology (single GCP VM, K3s)

```
        Internet (HTTPS)
              │
   ┌──────────▼───────────┐  apex + named hosts: Cloudflare-proxied,
   │  Cloudflare (proxy)  │  Universal SSL (*.bizspark.online), SSL mode Full,
   └──────────┬───────────┘  edge → origin via cf-origin-cert
              │ :443         tenant subdomains: DNS-only (direct to VM, LE cert)
   ┌──────────▼───────────────────────────────────────────────────────────┐
   │                         K3s VM (Debian, GCP)                          │
   │  ┌────────────────────┐                                               │
   │  │  Traefik (ingress) │  host-based routing (base/ingress.yaml)       │
   │  └─────────┬──────────┘  priorities: exact host > tenant wildcard > catch-all
   │   ┌────────┼─────────────┬───────────────┬────────────────────────┐   │
   │   ▼        ▼             ▼               ▼                        ▼   │
   │ frontend  api         commerce       commerce-web    admin + mcp      │
   │ :3000   :3000/api      :3003            :3004      :3002 / :3005       │
   │ (bizspark.online)   (commerce.b.o)   (store.b.o /  (admin.b.o)        │
   │                                       <slug>.b.o)                     │
   │   api ──► Redis (in-cluster) ◄── runner (BullMQ worker)               │
   │   ▲                                                                   │
   │   │  envFrom  ┌───────────────────────────────┐                       │
   │   └──────────│ SealedSecret bizpark-runtime-env│ (sealed-secrets ctrl) │
   │              └───────────────────────────────┘                       │
   │   cert-manager ── LE wildcard *.bizspark.online (DNS-01 via Cloudflare)│
   │   ArgoCD ── argocd.randitha.net                                       │
   └──────────────────────────┬────────────────────────────────────────────┘
                              │
            ┌─────────────────┼──────────────────────────────┐
            ▼                 ▼                              ▼
     Neon Postgres      Neon Postgres (Commerce)     External services
     (api/admin/runner) (tenant_<id> schemas)        OpenAI · Gemini · MiniMax
                                                     Meta (FB/IG) · PayHere
                                                     Cloudflare DNS · Claude (MCP)
```

- **Outbound** integrations: OpenAI/Gemini/MiniMax (AI), Meta (social publishing), PayHere (billing), Cloudflare DNS (cert-manager DNS-01; per-tenant subdomains are covered by a wildcard, so the API no longer creates per-record DNS).
- **Inbound** integration: Claude (Desktop/Web) connects **into** `bizpark-mcp` to read tenant store data — see the BE repo's *AI Connect (MCP)*.

---

## Repository layout

```
apps/bizpark/
├── base/
│   ├── namespace.yaml
│   ├── workloads.yaml                  # Deployments (7 services)
│   ├── services.yaml                   # ClusterIP services
│   ├── redis.yaml                      # in-cluster Redis
│   ├── ingress.yaml                    # Traefik ingress (bizspark.online + tenant wildcard)
│   ├── cert-manager.yaml               # LE ClusterIssuer + *.bizspark.online wildcard Certificate
│   ├── bizpark-runtime-env-sealed.yaml # Bitnami SealedSecret (runtime config)
│   └── kustomization.yaml
└── overlays/
    └── prod/
        └── kustomization.yaml          # pins image digests (CI-managed)
```

## Services (7 images, all on GHCR)

| Deployment | Image | Port | Exposed at |
|---|---|---|---|
| `bizpark-api` | `bizpark-api` | 3000 | `bizspark.online/api` |
| `bizpark-frontend` | `bizpark-frontend` | 3000 | `bizspark.online/` (SaaS dashboard) |
| `bizpark-admin` | `bizpark-admin` | 3002 | `admin.bizspark.online/` |
| `bizpark-commerce` | `bizpark-commerce` | 3003 | `commerce.bizspark.online/` |
| `bizpark-commerce-web` | `bizpark-commerce-web` | 3004 | `store.bizspark.online/` **and** `<slug>.bizspark.online/` |
| `bizpark-mcp` | `bizpark-mcp` | 3005 | `admin.bizspark.online/{sse,message,mcp,oauth,.well-known}` |
| `bizpark-runner` | `bizpark-runner` | — | worker (no ingress) |

## Networking / TLS

Two classes of hostname, served by **two ingresses + a fallback** with explicit Traefik router priorities (**exact host `50` > tenant wildcard `10` > host-less catch-all `1`**):

1. **Apex + named hosts** (`bizspark.online`, `store/commerce/admin.bizspark.online`) — Cloudflare-**proxied** (orange). Browsers get Cloudflare's free Universal SSL (`*.bizspark.online`, one label); Cloudflare→origin uses the **Cloudflare Origin cert** `cf-origin-cert` (SSL mode **Full**, so it isn't validated). Defined in `bizpark-master-ingress`.
2. **Per-tenant storefront subdomains** (`<slug>.bizspark.online`) — a single **DNS-only** wildcard `*.bizspark.online` → VM (Cloudflare can't proxy a wildcard on the free plan). They hit the VM directly, so Traefik serves a real **Let's Encrypt wildcard cert** (`bizspark-wildcard-tls`, issued by **cert-manager** via Cloudflare **DNS-01**). Defined in `bizpark-tenant-ingress`. A tenant claims a slug in the dashboard; the wildcard means **no per-record DNS is created** — subdomains resolve instantly, no NXDOMAIN/propagation delay.

> The commerce-web middleware resolves `<slug>` → tenant id via the Application API (`/api/storefront/resolve/:slug`), so the storefront knows which tenant to render.

## Cluster prerequisites

Created **once, out of band** (not in this repo):

| Secret / component | Type | Purpose |
|---|---|---|
| `ghcr-pull-secret` | docker-registry | pull images from GHCR |
| `cf-origin-cert` | TLS | Cloudflare Origin cert served by Traefik (proxied hosts) |
| sealed-secrets controller | — | Bitnami controller in `kube-system` decrypts SealedSecrets |
| **cert-manager** | — | issues/renews the LE wildcard cert (`kubectl apply -f cert-manager.yaml release`) |
| `cloudflare-api-token` (ns `cert-manager`) | generic, key `api-token` | Cloudflare DNS:Edit token for cert-manager DNS-01 |
| `cloudflare-api` (ns `bizpark`) | generic, key `token` | Cloudflare token the API used for DNS (now wildcard handles tenants) |
| `payhere` (ns `bizpark`) | generic, key `merchant-secret` | bizspark.online PayHere app merchant secret |

```bash
# recreate the out-of-band secrets if the cluster is rebuilt
kubectl -n cert-manager create secret generic cloudflare-api-token --from-literal=api-token='<DNS:Edit token>'
kubectl -n bizpark create secret generic cloudflare-api      --from-literal=token='<DNS:Edit token>'
kubectl -n bizpark create secret generic payhere             --from-literal=merchant-secret='<bizspark.online PayHere secret>'
```

## Secrets — Bitnami Sealed Secrets (+ env overrides)

Most runtime config lives **encrypted in Git** in `base/bizpark-runtime-env-sealed.yaml`; the controller decrypts it into `Secret bizpark-runtime-env`, consumed by pods via `envFrom`. **No plaintext secrets are committed.**

> ⚠️ **Known gotcha:** several values re-sealed during the `randitha.net → bizspark.online` migration **fail to decrypt** (their `kubeseal --raw` blobs don't match what the controller can unseal), so the Secret keeps stale randitha.net values for them. They are therefore **overridden on the `bizpark-api` deployment** in `workloads.yaml` (explicit `env` wins over `envFrom`):
> `PUBLIC_API_URL`, `FRONTEND_URL`, `COMMERCE_WEB_URL`, `FACEBOOK_REDIRECT_URI`, `INSTAGRAM_REDIRECT_URI` (plaintext) and `PAYHERE_MERCHANT_SECRET` (from the `payhere` Secret). `CLOUDFLARE_API_TOKEN` similarly comes from the `cloudflare-api` Secret. **TODO:** re-seal these properly and drop the overrides.

> `NEXT_PUBLIC_*` for the frontend/commerce-web are **build-time** values (baked from BE/FE repo *Variables* / workflow defaults), not runtime.

### Add or change a sealed key

```bash
echo -n "VALUE" | kubeseal --raw \
  --namespace bizpark --name bizpark-runtime-env \
  --controller-name sealed-secrets-controller --controller-namespace kube-system
# → paste the Ag… blob under spec.encryptedData.<KEY>, commit, push
```

> `envFrom` reads secrets **only at pod start** — after a change, `kubectl -n bizpark rollout restart deploy/bizpark-api`.

## Deploy / sync

ArgoCD auto-syncs on push. To force it:

```bash
kubectl -n argocd annotate application bizpark-prod argocd.argoproj.io/refresh=hard --overwrite
```

ArgoCD dashboard: `https://argocd.randitha.net` (not yet migrated — internal tool).

## Operational notes

- **Node disk** is the main fragility (image churn fills it → pod evictions). Disk was resized to 30 GB and k3s image-GC thresholds lowered (80/60). Prune occasionally: `sudo k3s crictl rmi --prune`.
- **Single small VM** (n2d-standard-2): commerce-web is resource-capped so a render spike can't starve the node; its SSR calls the commerce API **in-cluster** (`INTERNAL_COMMERCE_URL`), not through Cloudflare.

> Full cluster bring-up (K3s, ArgoCD, sealed secrets, Cloudflare, gotchas) is documented in `k3s-argocd-bizpark-deployment-guide.md` at the workspace root.
