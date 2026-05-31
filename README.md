# Infra — Bizpark GitOps

Kubernetes (K3s) manifests for the **BizSpark AI** platform, delivered via **ArgoCD** GitOps. Everything that runs in the cluster is declared here; ArgoCD continuously reconciles the cluster to match this repo.

ArgoCD tracks:

```text
apps/bizpark/overlays/prod
```

The BE repository's CI updates `apps/bizpark/overlays/prod/kustomization.yaml` with immutable `ghcr.io/codesprint-bizspark/*` image **digests** after every successful build on `main`.

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
   ┌──────────▼───────────┐   Universal SSL (*.randitha.net), SSL mode: Full
   │  Cloudflare (proxy)  │   edge → origin via cf-origin-cert
   └──────────┬───────────┘
              │ :443
   ┌──────────▼───────────────────────────────────────────────────────────┐
   │                         K3s VM (Debian, GCP)                          │
   │  ┌────────────────────┐                                               │
   │  │  Traefik (ingress) │  host-based routing (base/ingress.yaml)       │
   │  └─────────┬──────────┘                                               │
   │   ┌────────┼─────────────┬───────────────┬────────────────────────┐   │
   │   ▼        ▼             ▼               ▼                        ▼   │
   │ frontend  api         commerce       commerce-web    admin + mcp      │
   │ :3000   :3000/api      :3003            :3004      :3002 / :3005       │
   │ (bizspark.r.n)      (commerce.r.n)   (store.r.n)   (admin.r.n)         │
   │                                                                       │
   │   api ──► Redis (in-cluster) ◄── runner (BullMQ worker)               │
   │   ▲                                                                   │
   │   │  envFrom  ┌───────────────────────────────┐                       │
   │   └──────────│ SealedSecret bizpark-runtime-env│ (sealed-secrets ctrl) │
   │              └───────────────────────────────┘                       │
   │   ArgoCD ── argocd.randitha.net                                       │
   └──────────────────────────┬────────────────────────────────────────────┘
                              │
            ┌─────────────────┼──────────────────────────────┐
            ▼                 ▼                              ▼
     Neon Postgres      Neon Postgres (Commerce)     External services
     (api/admin/runner) (tenant_<id> schemas)        OpenAI · Gemini · MiniMax
                                                     Meta (FB/IG) · PayHere
                                                     Claude (MCP, inbound)
```

- **Outbound** integrations: the platform calls OpenAI/Gemini/MiniMax (AI), Meta (social publishing), PayHere (billing notify/return).
- **Inbound** integration: Claude (Desktop/Web) connects **into** `bizpark-mcp` to read tenant store data — see the BE repo's *AI Connect (MCP)*.

---

## Repository layout

```
apps/bizpark/
├── base/
│   ├── namespace.yaml
│   ├── workloads.yaml                 # Deployments (7 services)
│   ├── services.yaml                  # ClusterIP services
│   ├── ingress.yaml                   # host-based Traefik ingress (randitha.net)
│   ├── bizpark-runtime-env-sealed.yaml# Bitnami SealedSecret (all runtime config)
│   └── kustomization.yaml
└── overlays/
    └── prod/
        └── kustomization.yaml         # pins image digests (CI-managed)
```

## Services (7 images, all on GHCR)

| Deployment | Image | Port | Exposed at |
|---|---|---|---|
| `bizpark-api` | `bizpark-api` | 3000 | `bizspark.randitha.net/api` |
| `bizpark-frontend` | `bizpark-frontend` | 3000 | `bizspark.randitha.net/` (SaaS dashboard) |
| `bizpark-admin` | `bizpark-admin` | 3002 | `admin.randitha.net/` |
| `bizpark-commerce` | `bizpark-commerce` | 3003 | `commerce.randitha.net/` |
| `bizpark-commerce-web` | `bizpark-commerce-web` | 3004 | `store.randitha.net/` |
| `bizpark-mcp` | `bizpark-mcp` | 3005 | `admin.randitha.net/{sse,message,mcp,oauth,.well-known}` |
| `bizpark-runner` | `bizpark-runner` | — | worker (no ingress) |

## Networking / TLS

- **Traefik** (bundled with K3s) does host-based routing — see `base/ingress.yaml`.
- **Cloudflare** sits in front (DNS proxied/orange, SSL mode **Full**). Browsers get Cloudflare's free Universal SSL (`*.randitha.net`); Cloudflare→origin uses a **Cloudflare Origin cert** stored as the TLS secret `cf-origin-cert`.
- Free SSL covers **one** subdomain label only, so all hosts are single-level: `bizspark / store / commerce / admin / argocd .randitha.net`.

## Cluster prerequisites

These are created **once, out of band** (not in this repo):

| Secret | Type | Purpose |
|---|---|---|
| `ghcr-pull-secret` | docker-registry | pull images from GHCR |
| `cf-origin-cert` | TLS | Cloudflare Origin cert served by Traefik |
| sealed-secrets controller | — | Bitnami controller in `kube-system` decrypts SealedSecrets |

## Secrets — Bitnami Sealed Secrets

All runtime config lives **encrypted in Git** in `base/bizpark-runtime-env-sealed.yaml`. The in-cluster controller decrypts it into a normal `Secret` named `bizpark-runtime-env`, which every pod consumes via `envFrom`. **No plaintext secrets are committed.**

It holds DB URLs (Neon), Redis host/port, JWT/internal keys, AI keys (OpenAI/Gemini/MiniMax), social OAuth (Facebook/Instagram), PayHere billing, token/OAuth-state keys, and the public URLs (`PUBLIC_API_URL`, `FRONTEND_URL`, `COMMERCE_WEB_URL`, `NEXT_PUBLIC_*`).

> `NEXT_PUBLIC_*` for the frontend/commerce-web are **build-time** values (baked into the image from BE/FE repo *Variables*), not runtime — the sealed copy is only for server-side reads.

### Add or change a sealed key

```bash
# seal a single value against the cluster's controller
echo -n "VALUE" | kubeseal --raw \
  --namespace bizpark --name bizpark-runtime-env \
  --controller-name sealed-secrets-controller --controller-namespace kube-system
# → paste the Ag… blob under spec.encryptedData.<KEY> in the sealed file, commit, push
```

> ⚠️ `envFrom` reads secrets **only at pod start**. After changing the secret, **restart** the affected deployment so it picks up the new value:
> `kubectl -n bizpark rollout restart deploy/bizpark-api`

## Deploy / sync

ArgoCD auto-syncs on push. To force it:

```bash
kubectl -n argocd annotate application bizpark-prod argocd.argoproj.io/refresh=hard --overwrite
```

ArgoCD dashboard: `https://argocd.randitha.net`.

> Full cluster bring-up (K3s, ArgoCD, sealed secrets, Cloudflare, gotchas) is documented in `k3s-argocd-bizpark-deployment-guide.md` at the workspace root.
