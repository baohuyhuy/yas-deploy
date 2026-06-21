# YAS GitOps with ArgoCD — setup & runbook

This is the step-by-step guide for the **ArgoCD bonus** (Đồ án 2, §6 + Nâng cao):
continuously deploy `main` to a `dev` namespace and release `v*` tags to a
`staging` namespace, all driven by ArgoCD pulling from this config repo.

## Architecture

```
  github.com/baohuyhuy/yas            github.com/baohuyhuy/yas-deploy (this repo)
  ─ app source, Dockerfiles            ─ charts/<svc>            (Helm charts)
  ─ *-image-ci.yaml  (build :main,:sha) ─ argocd/appproject.yaml
  ─ deploy-dev.yaml  ──bump──────────▶  ─ argocd/applicationset-dev.yaml
  ─ release-staging.yaml ──bump──────▶  ─ argocd/applicationset-staging.yaml
                                        ─ envs/dev/values.yaml    (tag/commit)
                                        ─ envs/staging/values.yaml(release ver)
                                                  │
                                          ArgoCD watches this repo
                                                  ▼
                                       dev namespace  +  staging namespace
                                       (apps share the cluster's existing
                                        Postgres/Kafka/Keycloak/ES/Redis)
```

ArgoCD runs **in-cluster** and reconciles the live state to match git. Nothing
`kubectl apply`s app workloads by hand — a git change in this repo is the only
trigger.

## One-time cluster setup

```bash
export KUBECONFIG=~/.kube/yas-config

# 1. Install ArgoCD (pinned). The ApplicationSet CRD is large -> server-side apply.
kubectl create namespace argocd
kubectl apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.4/manifests/install.yaml

# 2. Serve the UI over plain HTTP behind ingress-nginx (avoids TLS redirect loops)
kubectl -n argocd patch configmap argocd-cmd-params-cm --type merge \
  -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deployment argocd-server
kubectl apply -f argocd/ingress.yaml          # argocd.yas.local.com

# 3. Admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
# login: admin / <that password> at http://argocd.yas.local.com:30080
```

## One-time environment setup (dev + staging)

The apps reuse the cluster's existing shared infra namespaces; each app namespace
only needs YAS config/secrets and the media images PVC:

```bash
for ns in dev staging; do
  kubectl create namespace "$ns"
  helm upgrade --install yas-configuration ../charts/yas-configuration -n "$ns"
  kubectl -n "$ns" apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: media-images-pvc }
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: local-path
  resources: { requests: { storage: 1Gi } }
EOF
done
```

## Deploy the GitOps control plane

```bash
kubectl apply -f argocd/appproject.yaml
kubectl apply -f argocd/applicationset-dev.yaml
kubectl apply -f argocd/applicationset-staging.yaml
# -> ArgoCD generates 20 Applications per environment and auto-syncs them.
kubectl get applications -n argocd
```

## CI wiring (in the app repo, baohuyhuy/yas)

Create a repo secret **`DEPLOY_REPO_TOKEN`** = a PAT with `contents:write` on
`baohuyhuy/yas-deploy`, then the two workflows close the loop:

- **`deploy-dev.yaml`** — on push to `main`, bumps the commit annotation in
  `envs/dev/values.yaml`; ArgoCD rolls `dev` to re-pull the latest `:main` images.
- **`release-staging.yaml`** — on a `v*` tag, builds versioned images for every
  service then pins `envs/staging/values.yaml` to that version; ArgoCD deploys
  the immutable release to `staging`.

## Demo

```bash
# dev: any commit to main -> dev redeploys automatically
git commit --allow-empty -m "demo: dev redeploy" && git push origin main
#   watch the dev-* apps go OutOfSync -> Synced in the ArgoCD UI

# staging: cut a release -> versioned images -> staging deploys that version
git tag v1.0.0 && git push origin v1.0.0
#   watch release-staging build, then the staging-* apps move to v1.0.0
```

## Notes / gotchas

- **Shared Postgres memory.** Three YAS copies (yas + dev + staging) share one
  Postgres pod. The Zalando CR ships a tiny **500Mi** memory limit, which OOMKills
  the DB once enough services connect (each backend ~5-10MB). Two mitigations:
  1. Raise the limit (the node has 32GB free):
     ```bash
     kubectl patch postgresql postgresql -n postgres --type merge \
       -p '{"spec":{"resources":{"requests":{"memory":"1Gi"},"limits":{"memory":"3Gi"}}}}'
     ```
  2. Cap each service's Hikari pool in `envs/{dev,staging}/values.yaml`
     (`SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE=2`) to bound total connections.
- **`staging` is held at `replicaCount: 0`** in `envs/staging/values.yaml`: even at
  3Gi, running all three full copies at once is tight on the single worker. Its 20
  Applications still stay present and Synced in ArgoCD (the GitOps deliverable). To
  demo staging at runtime, scale `dev` to 0 first, then bump staging to 1.
- **Multi-source `$values`.** The ApplicationSets use ArgoCD's two-source pattern
  (a bare `ref: values` source + the chart source) so one env file can override
  every chart, even though it lives outside the chart directory.
- **Ingress disabled** in dev/staging to avoid host collisions with `yas`; reach
  an app with `kubectl port-forward -n dev svc/storefront-ui 3000:3000`.
