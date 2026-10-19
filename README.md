# microservices-gitops (CD)

Desired state for the cluster. Argo CD watches this repo and Argo Rollouts performs the progressive delivery.
Application code and CI live in **`microservices-apps`**; its pipeline updates the image tags here.

| Service | Strategy | Namespace | Tag updated by CI via |
|---|---|---|---|
| `catalog-service` | Canary: 20% → 50% → 80% → 100% with automated analysis | `catalog` | Direct commit to `main` |
| `payment-gateway` | Blue-green: preview stack, smoke test, manual promote | `payments` | Pull request |

```
microservices-apps CI ──(image tag commit / PR)──► this repo ──► Argo CD ──► Rollout CRDs ──► Argo Rollouts
```

## Layout

```
k8s/catalog-service/    Rollout (canary), Services, Ingress, AnalysisTemplate, kustomization
k8s/payment-gateway/    Rollout (blue-green), Services, Ingress, AnalysisTemplate, kustomization
argocd/                 Argo CD Application manifests (one per service)
kind-config.yaml        Local kind cluster with host ports 80/443 mapped to the ingress controller
.github/workflows/      validate.yaml: kustomize render + kubeconform schema check on every change
```

## Validation gate

`validate.yaml` renders each service with `kubectl kustomize` and validates it with kubeconform, including the
Argo Rollouts and Argo CD CRDs. It runs on pull requests and on pushes to `main`. For `payment-gateway`, the CI
pull request has to pass this check before you merge it. Turn on branch protection to enforce that.

## Bootstrap a local cluster

```bash
kind create cluster --config kind-config.yaml

# Ingress controller (needed for canary traffic weights)
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml

# Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Argo Rollouts controller + kubectl plugin
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
# plugin: https://argo-rollouts.readthedocs.io/en/stable/installation/#kubectl-plugin-installation
```

> The canary uses Argo Rollouts' nginx traffic router. ingress-nginx is being retired upstream, so if you
> use something else, Argo Rollouts also supports Gateway API and other routers
> (`strategy.canary.trafficRouting` in `k8s/catalog-service/rollout.yaml`).

## Connect Argo CD to this repo

If the repo is **public**, nothing to do. If it is **private**, register it first:

```bash
argocd repo add https://github.com/AKASHSHAKTHIRT/microservices-gitops.git \
  --username <github-user> --password <read-only-token>
```

Then create the two applications:

```bash
kubectl apply -f argocd/
```

Before the first CI run there is no image to pull. Either run the CI pipeline once (it will commit the first real
tag here), or push a `v1` image by hand using the Dockerfiles in the apps repo.

## Release flow

**catalog-service (canary).** Merge to `main` in the apps repo. CI commits the new tag here, Argo CD syncs, and:

```bash
kubectl argo rollouts get rollout catalog-service -n catalog -w
```

Traffic goes 20% → analysis → 50% → 80% → 100%. A failed analysis aborts and returns traffic to stable.
To see it, use **Run workflow** with `error_rate=1` on the catalog workflow in the apps repo.

**payment-gateway (blue-green).** Merge to `main` in the apps repo. CI opens a pull request here. Review it,
let `validate-manifests` pass, merge. Then:

```bash
kubectl argo rollouts get rollout payment-gateway -n payments -w
```

1. Green comes up behind `payment-gateway-preview`; live traffic still hits blue.
2. The pre-promotion smoke test (a Job: payment, idempotent replay, refund) runs against the preview Service.
3. The rollout waits; Argo CD shows it as *Suspended*. Check green: `curl http://payments-preview.localtest.me/version`
4. Promote: `kubectl argo rollouts promote payment-gateway -n payments`
5. The active Service flips to green, the post-promotion check runs, and blue is scaled down after 120s.

Instant rollback while blue is still up: `kubectl argo rollouts undo payment-gateway -n payments`.
For a permanent rollback, revert the tag commit in this repo as well, otherwise Argo CD will sync the bad tag again.

`*.localtest.me` resolves to 127.0.0.1, which kind maps to the ingress controller:
`catalog.localtest.me`, `payments.localtest.me`, `payments-preview.localtest.me`.

## Notes

- Manifests and workflows have not been applied to a live cluster or run on GitHub yet; expect small fixes on first run.
- The payment gateway is a sandbox mock: in-memory storage, tokens only, no real card data.
- The canary analysis uses a plain HTTP probe so it works without Prometheus. Swap in a Prometheus provider for real
  error-rate and latency gates.
