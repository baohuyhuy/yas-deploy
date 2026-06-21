# yas-deploy

GitOps **config repo** for the YAS DevOps project (Đồ án 2). This repo is the single source of
truth that **ArgoCD** reconciles into the cluster. Application source code, Dockerfiles and
image-build CI live in the separate app repo (`baohuyhuy/yas`); this repo only describes *what runs
where*.

## Layout

```
yas-deploy/
├── charts/                 # the 20 YAS service Helm charts (moved out of yas/k8s/charts)
│   ├── backend/            #   shared library subchart (file://../backend dependency)
│   ├── ui/                 #   shared library subchart for Next.js frontends
│   ├── tax/  cart/  ...     #   one chart per service
│   └── yas-configuration/  #   shared config/secrets
├── argocd/                 # ArgoCD AppProject + ApplicationSets (added in Phase 3)
│   ├── appproject.yaml
│   ├── applicationset-dev.yaml
│   └── applicationset-staging.yaml
└── envs/                   # per-environment value overrides ArgoCD layers on top of each chart
    ├── dev/values.yaml     #   namespace: dev,     image tag tracks latest main commit (SHA)
    └── staging/values.yaml #   namespace: staging, image tag pinned to a release (e.g. v1.2.3)
```

## Environments

| Env       | Namespace | Image tag           | Trigger                              |
|-----------|-----------|---------------------|--------------------------------------|
| `dev`     | `dev`     | latest `main` SHA   | any commit to `main` (auto-deploy)   |
| `staging` | `staging` | release tag `v1.2.3`| a `v*` git tag on the app repo       |

## How a deploy happens (GitOps flow)

1. App repo CI builds & pushes `baohuyhuy/yas-<svc>:<tag>` to Docker Hub.
2. App repo CI pushes a one-line "bump" commit **into this repo**, rewriting the image tag in
   `envs/<env>/values.yaml`.
3. ArgoCD detects the git change and syncs the matching namespace to the new desired state.

Rollback = `git revert` the bump commit here; ArgoCD re-syncs to the previous version.

> Status: Phase 0 (repo split). ArgoCD manifests and env values are added in later phases.
