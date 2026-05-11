# Infra

GitOps manifests for Bizpark services.

ArgoCD should track:

```text
apps/bizpark/overlays/prod
```

The BE repository workflow updates `apps/bizpark/overlays/prod/kustomization.yaml`
with immutable `ghcr.io/codesprint-bizspark/*` image digests after every
successful build on `main`.

## Cluster prerequisites

- Namespace: created by this Kustomize app as `bizpark`
- GHCR pull secret: `ghcr-pull-secret`
- Runtime env secret: `bizpark-runtime-env`

`bizpark-runtime-env` should contain the production values needed by the services,
such as database URLs, Redis settings, JWT/internal keys, OpenAI keys, and any
Commerce Web server-side runtime values. `NEXT_PUBLIC_*` Commerce Web values are
build-time values supplied from BE repository variables.
