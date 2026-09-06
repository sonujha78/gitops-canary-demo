# GitOps Progressive Delivery — Zero-Touch Canary Deployments

Production-style GitOps pipeline where **no human ever runs `kubectl apply` for a release**. Every deployment happens by pushing to Git, and every new version is rolled out gradually to real traffic with automatic rollback if metrics degrade — zero manual intervention.

**Stack:** ArgoCD + Flagger + Kubernetes (minikube) + Prometheus + Istio + GitHub

Repo: https://github.com/sonujha78/gitops-canary-demo

---

## 1. Architecture

```
 ┌─────────────┐        push        ┌──────────────┐
 │  Developer  │ ─────────────────► │  GitHub Repo │
 └─────────────┘                    │ (source of   │
                                     │   truth)     │
                                     └──────┬───────┘
                                            │ continuous poll / auto-sync
                                            ▼
                                     ┌──────────────┐
                                     │    ArgoCD    │  watches repo,
                                     │ (GitOps CD)  │  enforces selfHeal
                                     └──────┬───────┘
                                            │ applies Deployment/Service/
                                            │ SealedSecret manifests
                                            ▼
                                     ┌──────────────┐
                                     │  Kubernetes  │
                                     │   Cluster    │
                                     │  (minikube)  │
                                     └──────┬───────┘
                                            │ new Deployment revision detected
                                            ▼
                                     ┌──────────────┐
                                     │   Flagger    │  creates primary +
                                     │ (canary ctrl)│  canary, shifts traffic
                                     └──────┬───────┘
                                            │ configures
                                            ▼
                                     ┌──────────────┐
                                     │ Istio Virtual│  5% → 20% → 50% → 100%
                                     │   Service    │  traffic split
                                     └──────┬───────┘
                                            │ live traffic
                                            ▼
                              ┌─────────────┴─────────────┐
                              ▼                           ▼
                     ┌─────────────────┐        ┌──────────────────┐
                     │ podinfo-primary │        │  podinfo-canary   │
                     │  (stable, old)  │        │  (new version)    │
                     └─────────────────┘        └────────┬─────────┘
                                                          │ metrics scraped
                                                          ▼
                                                 ┌──────────────────┐
                                                 │    Prometheus     │
                                                 │ success-rate, p99 │
                                                 └────────┬─────────┘
                                                          │ query results
                                                          ▼
                                                 ┌──────────────────┐
                                                 │      Flagger       │
                                                 │ pass → advance %   │
                                                 │ fail → rollback 0% │
                                                 └────────────────────┘
```

**GitOps guarantee (drift correction):** Git is truth — if anyone manually
runs `kubectl scale` / `kubectl edit` on a resource ArgoCD manages, ArgoCD's
`selfHeal: true` reverts it back to the Git-defined state within seconds.

**Progressive delivery guarantee:** Flagger never replaces the old version
immediately. It creates a `-primary` (stable) and canary (new) deployment,
shifts a small % of real traffic to canary, watches Prometheus metrics, and
only promotes to 100% if metrics stay healthy — otherwise it rolls back to
0% automatically.

---

## 2. Repository Structure

```
gitops-canary-demo/
├── base/                          # Base Kustomize resources (shared)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── kustomization.yaml
│   └── secrets/
│       ├── kustomization.yaml
│       └── podinfo-sealed-secret.yaml   # encrypted, safe to commit
├── environments/
│   ├── dev/kustomization.yaml           # replicas: 1
│   ├── staging/kustomization.yaml       # replicas: 2
│   └── production/kustomization.yaml    # replicas: 3, sealed secret, image tag
├── argocd/
│   ├── dev-application.yaml
│   ├── staging-application.yaml
│   └── production-application.yaml      # ignoreDifferences for Flagger fields
├── flagger/
│   ├── canary.yaml                      # Canary resource (analysis config)
│   ├── loadtester-deployment.yaml
│   └── loadtester-service.yaml
├── docs/
│   └── evidence/                        # captured proof for every section
│       ├── drift-correction.txt
│       ├── canary-rollout-progress.txt
│       ├── successful-canary-rollout.txt
│       ├── failure-injection-rollback.txt
│       ├── sealed-secrets.txt
│       ├── multi-environment-applications.txt
│       ├── pr-promotion-dev.txt
│       ├── pr-promotion-staging.txt
│       └── full-promotion-trail.txt
└── README.md
```

---

## 3. Full Setup — Commands By Phase

### Phase 1 — Base tools + cluster

```bash
docker --version && kubectl version --client && git --version

# minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# start cluster (tune memory/cpus to your machine)
minikube start --driver=docker --memory=6144 --cpus=3 --disk-size=30g
kubectl get nodes
```

### Phase 2 — Repo + base app manifests

```bash
mkdir -p ~/gitops-canary-demo && cd ~/gitops-canary-demo
git init
mkdir -p base environments/{dev,staging,production} flagger argocd docs/evidence
# base/deployment.yaml, base/service.yaml, base/kustomization.yaml
# (see Repository Structure above for content)
git add . && git commit -m "base podinfo manifests"
git branch -M main
git remote add origin https://github.com/<user>/gitops-canary-demo.git
git push -u origin main
```

### Phase 3 — Environment overlays (Kustomize)

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production
kubectl kustomize environments/dev
kubectl kustomize environments/staging
kubectl kustomize environments/production
```

### Phase 4 — Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts
kubectl get pods -n argocd

kubectl port-forward svc/argocd-server -n argocd 8081:443 &
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
# login at https://localhost:8081  (user: admin)
```

### Phase 5 — ArgoCD Application (GitOps sync + drift correction)

```yaml
# argocd/dev-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<user>/gitops-canary-demo.git
    targetRevision: main
    path: environments/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true          # <-- this is the GitOps guarantee
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f argocd/dev-application.yaml
kubectl get application -n argocd
```

**Prove drift correction:**
```bash
kubectl scale deployment podinfo -n dev --replicas=5
sleep 20
kubectl get pods -n dev     # ArgoCD reverts back to Git-defined replica count
```

### Phase 6 — Istio + Prometheus + Flagger

```bash
# Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*/ && export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y
cd ~/gitops-canary-demo

# Prometheus + Grafana (bundled Istio addons)
kubectl apply -f istio-*/samples/addons/prometheus.yaml
kubectl apply -f istio-*/samples/addons/grafana.yaml

# Flagger CRDs
kubectl apply -f https://raw.githubusercontent.com/fluxcd/flagger/main/artifacts/flagger/crd.yaml

# Flagger controller (Helm)
helm repo add flagger https://flagger.app && helm repo update
helm upgrade -i flagger flagger/flagger \
  --namespace=istio-system \
  --set crd.create=false \
  --set meshProvider=istio \
  --set metricsServer=http://prometheus.istio-system:9090

# Namespace needs Istio sidecar injection
kubectl label namespace production istio-injection=enabled
```

### Phase 7 — Flagger Canary resource

```yaml
# flagger/canary.yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  progressDeadlineSeconds: 60
  service:
    port: 9898
    targetPort: 9898
    gateways: ["mesh"]
    hosts: ["podinfo.production.svc.cluster.local"]   # no wildcards with mesh gateway
  analysis:
    interval: 15s
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
      - name: request-success-rate
        thresholdRange: { min: 99 }
        interval: 1m
      - name: request-duration
        thresholdRange: { max: 500 }
        interval: 1m
    webhooks:
      - name: load-test
        url: http://flagger-loadtester.production/
        timeout: 5s
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://podinfo-canary.production:9898/"
```

```bash
kubectl apply -f flagger/canary.yaml

# Load tester (needed — Flagger halts without traffic to measure)
curl -s https://raw.githubusercontent.com/fluxcd/flagger/main/kustomize/tester/deployment.yaml -o /tmp/lt-d.yaml
curl -s https://raw.githubusercontent.com/fluxcd/flagger/main/kustomize/tester/service.yaml -o /tmp/lt-s.yaml
kubectl apply -f /tmp/lt-d.yaml -n production
kubectl apply -f /tmp/lt-s.yaml -n production

kubectl get canary podinfo -n production -w
```

### Phase 8 — Successful rollout test

```bash
# bump image tag in environments/production/kustomization.yaml, then:
git add . && git commit -m "roll out v6.6.0" && git push
# ArgoCD auto-syncs -> Flagger auto-detects new revision -> progressive rollout
kubectl get canary podinfo -n production -w
# expect: 10 -> 20 -> 30 -> 40 -> 50 -> Promoting -> Finalising -> Succeeded
```

### Phase 9 — Failure injection test

```bash
# in environments/production/kustomization.yaml, patch container command to:
#   - ./podinfo
#   - --port=9898
#   - --level=info
#   - --random-error      # ~33% of requests return HTTP 500
git add . && git commit -m "inject failure for rollback test" && git push
kubectl get canary podinfo -n production -w
# expect: weight climbs to 10, success-rate metric drops below 99%,
# after threshold consecutive failures -> STATUS=Failed, WEIGHT=0 (auto rollback)
```

### Phase 10 — Sealed Secrets (GitOps-safe secrets)

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/controller.yaml
curl -OL https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/kubeseal-0.27.1-linux-amd64.tar.gz
tar -xvzf kubeseal-0.27.1-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

kubectl create secret generic podinfo-secret -n production \
  --from-literal=API_KEY='super-secret-value-12345' \
  --dry-run=client -o yaml > /tmp/secret-plain.yaml

kubeseal --controller-namespace kube-system --format yaml \
  < /tmp/secret-plain.yaml > base/secrets/podinfo-sealed-secret.yaml

rm /tmp/secret-plain.yaml          # never commit the plaintext
git add base/secrets/ && git commit -m "add sealed secret" && git push
```

### Phase 11 — Multi-environment promotion via PR

```bash
git checkout -b promote-podinfo-v2-to-dev
# edit environments/dev/kustomization.yaml
git add . && git commit -m "Promote: bump dev replicas"
git push -u origin promote-podinfo-v2-to-dev
# open GitHub -> Compare & pull request -> Merge
git checkout main && git pull
kubectl get pods -n dev     # ArgoCD auto-deploys after merge, no kubectl apply used
```
Repeat the same pattern for `promote-podinfo-v2-to-staging` targeting
`environments/staging/kustomization.yaml`. This gives a clean PR-based audit
trail: dev → staging → production, each promotion gated by a pull request
merge instead of a manual deployment step.

---

## 4. Evidence Index

| Section | File |
|---|---|
| Drift correction (Git is truth) | `docs/evidence/drift-correction.txt` |
| Successful canary rollout | `docs/evidence/successful-canary-rollout.txt` |
| Failure injection + auto-rollback | `docs/evidence/failure-injection-rollback.txt` |
| Sealed Secrets encrypt/decrypt | `docs/evidence/sealed-secrets.txt` |
| Multi-env ArgoCD Applications | `docs/evidence/multi-environment-applications.txt` |
| PR promotion — dev | `docs/evidence/pr-promotion-dev.txt` |
| PR promotion — staging | `docs/evidence/pr-promotion-staging.txt` |
| Full dev→staging→production trail | `docs/evidence/full-promotion-trail.txt` |

---

## 5. Troubleshooting Log (issues actually hit while building this)

| Symptom | Root Cause | Fix |
|---|---|---|
| `kubectl get nodes` → `connection to localhost:8080 refused` | `KUBECONFIG` env var pointed at a stale, unrelated project's kubeconfig file (set in `~/.bashrc`) | `grep KUBECONFIG ~/.bashrc`, remove the export line, `unset KUBECONFIG`, then `minikube update-context` |
| ArgoCD install: `CustomResourceDefinition "applicationsets.argoproj.io" is invalid: metadata.annotations: Too long` | `kubectl apply` stores a `last-applied-configuration` annotation that exceeds 262144 bytes for large CRDs | Re-apply with `kubectl apply --server-side --force-conflicts` |
| `kubectl apply -f .../flagger/kustomize/crd/crd.yaml` → 404 | Flagger repo restructured; that path no longer exists | Use `https://raw.githubusercontent.com/fluxcd/flagger/main/artifacts/flagger/crd.yaml` |
| No Prometheus pod after Istio `profile=demo` install | This Istio version's demo profile doesn't bundle Prometheus by default | `kubectl apply -f istio-*/samples/addons/prometheus.yaml` (and `grafana.yaml`) |
| Canary stuck `Initializing`, event: `wildcard host * is not allowed for virtual services bound to the mesh gateway` | Canary spec used `gateways: [public-gateway..., mesh]` + `hosts: ["*"]`; wildcard hosts are invalid when bound to the internal `mesh` gateway, and `public-gateway` didn't exist in-cluster | Use only `gateways: ["mesh"]` with an explicit host: `hosts: ["podinfo.production.svc.cluster.local"]` |
| ArgoCD keeps flipping `podinfo` Deployment back to Git's replica count while Flagger scales it to 0/back up | ArgoCD's `selfHeal: true` and Flagger both "own" the same Deployment fields (replicas, image, command) and fight each other | Add `ignoreDifferences` in the ArgoCD `Application` for `spec/replicas`, `spec/template/spec/containers/0/image`, `spec/template/spec/containers/0/command` |
| ArgoCD `Application` flaps `OutOfSync` even after the Deployment fix | Flagger also mutates the `podinfo` **Service** (selector/ports) to route traffic to primary/canary | Extend `ignoreDifferences` to also cover the Service's `/spec/selector` and `/spec/ports` |
| Injected failure test with `--error-rate=30` / `--delay=2s` → pod `CrashLoopBackOff` | Those flags don't exist in podinfo's CLI (binary printed `--help` and exited); `--delay` also blocks the readiness probe, hanging the rollout entirely | Use the real supported flag: `--random-error` (~1/3 requests return an error); avoid `--delay` unless probe timeouts are also raised |
| Canary halts every step: `load-test failed ... dial tcp: lookup flagger-loadtester.production: no such host` | Flagger's canary webhook calls a load-tester service that was never deployed | Deploy `kubectl apply -k github.com/fluxcd/flagger//kustomize/tester` (note: its default manifests are hardcoded to `namespace: test`; download and `sed` the namespace to your target namespace, or apply with `-n <namespace>`) |
| SealedSecret shows `ErrUnsealFailed: no key could decrypt secret` in `dev`/`staging` | A SealedSecret's ciphertext is encrypted **for a specific namespace + name**; it was accidentally included in the shared `base/kustomization.yaml`, so it tried (and failed) to decrypt in every environment | Move the SealedSecret out of `base/` into an overlay-only path (e.g. `base/secrets/` with its own nested `kustomization.yaml`) referenced only from the `production` overlay |
| `git push` → `Could not resolve host: github.com` | Flaky local network / DNS (confirmed via `ping` showing packet loss), not a code issue | Retry the push (`for i in 1 2 3; do git push && break; sleep 5; done`); if persistent, check `ping 8.8.8.8` vs `ping github.com` to isolate DNS vs connectivity, and consider adding `nameserver 8.8.8.8` to `/etc/resolv.conf` |
| GitHub compare page: "There isn't anything to compare" for a promotion branch | The intended file change had already been committed earlier on `main` (e.g., during an unrelated fix), so the new branch was identical to `main` | Confirm with `git show HEAD:<file>` before opening a PR; if already up to date, no PR is needed — that environment's promotion is already reflected in Git |

---

## 6. Key Takeaway

Across every phase, the only human action was `git add / commit / push` or
merging a pull request. ArgoCD handled *what* runs in the cluster, and
Flagger handled *how safely* a new version reaches production — including
deciding, on its own, to roll a bad release back to zero traffic. That
decision loop (Prometheus metrics → Flagger → traffic weight) is the actual
skill this task is built to prove out, not just the happy-path rollout.
