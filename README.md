# Module 3 — GitOps with ArgoCD: Repository Setup & Walkthrough

## Repository Layout

```
.
├── .github/
│   ├── CODEOWNERS                       # Required reviewers per path (security objective)
│   ├── pull_request_template.md         # PR checklist (security objective)
│   └── workflows/
│       └── validate.yaml                # CI: validates manifests before merge
│
└── gitops/
    ├── applications/                    # Apply ONCE, by hand — bootstraps everything else
    │   ├── appproject.yaml              # AppProject: security boundary (allowed repos/namespaces)
    │   └── root-app.yaml                # Root "App of Apps" — points at gitops/platform
    │
    ├── platform/                        # The "App of Apps" — child Applications live here
    │   ├── monitoring-app.yaml          # → points to gitops/infrastructure/monitoring
    │   ├── logging-app.yaml             # → points to gitops/infrastructure/logging
    │   ├── ingress-app.yaml             # → points to gitops/infrastructure/ingress
    │   └── workloads-app.yaml           # → points to gitops/workloads/nginx/overlays/dev
    │
    ├── infrastructure/                  # Platform services, each wrapping a Helm chart
    │   ├── monitoring/                  # kube-prometheus-stack
    │   │   ├── kustomization.yaml
    │   │   ├── namespace.yaml
    │   │   └── values.yaml
    │   ├── logging/                     # loki-stack
    │   │   ├── kustomization.yaml
    │   │   ├── namespace.yaml
    │   │   └── values.yaml
    │   └── ingress/                     # ingress-nginx
    │       ├── kustomization.yaml
    │       ├── namespace.yaml
    │       └── values.yaml
    │
    └── workloads/                       # Application workloads (Lab 1 + Lab 2 material)
        └── nginx/
            ├── base/                    # Shared manifests
            │   ├── deployment.yaml
            │   ├── service.yaml
            │   └── kustomization.yaml
            └── overlays/                # Environment-specific patches
                ├── dev/kustomization.yaml
                ├── test/kustomization.yaml
                └── prod/kustomization.yaml
```

### Why this shape

- **`applications/` is the only thing you ever `kubectl apply` by hand.** Everything beneath `platform/` is reconciled by ArgoCD itself once the root app exists — that's the entire point of App of Apps: one root object, GitOps-managed from then on.
- **`platform/` is a level of indirection**, not real infrastructure. It exists purely so ArgoCD has something to sync that *creates more Applications*. This is what Lab 3 in the module is asking you to build.
- **`infrastructure/` uses Kustomize's native Helm chart inflation** (`helmCharts:` block) rather than raw manifests, since Prometheus/Loki/ingress-nginx are realistically deployed via their upstream charts. `values.yaml` next to each `kustomization.yaml` keeps overrides in Git, reviewable and diffable.
- **`workloads/` uses base + overlays** so the same nginx manifests power dev/test/prod with only the differences (replica count, resources) checked in per environment — this is what makes Lab 2's drift detection meaningful: you can see exactly which overlay is "live."

---

## Step-by-Step Walkthrough

### Step 0 — Create the Git repository

```bash
mkdir gitops-repo && cd gitops-repo
git init
git checkout -b main
# copy the gitops/ and .github/ folders from this template in
git add .
git commit -S -m "Initial GitOps repository structure"
git remote add origin https://github.com/<YOUR_ORG>/<YOUR_REPO>.git
git push -u origin main
```

Replace every `<YOUR_ORG>/<YOUR_REPO>` placeholder across the YAML files with your actual repo path before pushing — `root-app.yaml`, `appproject.yaml`, and all four files under `platform/` reference it.

### Step 1 — Enable branch protection (Security Objective: protected branches)

In your Git provider's repo settings:
1. Protect `main`.
2. Require a pull request before merging — no direct pushes.
3. Require approvals from Code Owners (this activates the `.github/CODEOWNERS` file).
4. Require status checks to pass — select the `validate` job from `.github/workflows/validate.yaml`.
5. Require signed commits (Security Objective: signed commits). This rejects any commit not signed with a GPG/SSH key.

At this point your repo enforces the three security objectives the module lists: signed commits, protected branches, PR approvals.

### Step 2 — Install ArgoCD on the cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl get pods -n argocd -w
```

Get the initial admin password and log in:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

kubectl port-forward svc/argocd-server -n argocd 8080:443
argocd login localhost:8080
```

### Step 3 — Bootstrap the AppProject and root Application

This is the **only manual `kubectl apply`** in the entire workflow:

```bash
kubectl apply -f gitops/applications/appproject.yaml
kubectl apply -f gitops/applications/root-app.yaml
```

What happens next, automatically:
1. ArgoCD syncs the `platform` Application, which points at `gitops/platform/`.
2. ArgoCD discovers `monitoring-app.yaml`, `logging-app.yaml`, `ingress-app.yaml`, `workloads-app.yaml` inside that path and creates each as its own Application object.
3. Each of those then syncs its own target path (`infrastructure/monitoring`, `infrastructure/logging`, etc.), deploying the actual Helm-rendered resources.

Verify the cascade:
```bash
argocd app list
# Expect to see: platform, monitoring, logging, ingress, workloads
```

The `argocd.argoproj.io/sync-wave` annotations on the child apps control ordering — ingress syncs first (wave 0), monitoring/logging next (wave 1), workloads last (wave 2), so the nginx app doesn't try to deploy before its namespace and dependencies exist.

### Step 4 — Lab 1: Application Deployment (nginx)

The `workloads-app.yaml` Application already targets `gitops/workloads/nginx/overlays/dev`, which Kustomize-builds the base nginx deployment with the `dev` overlay's single-replica patch.

```bash
argocd app get workloads
argocd app sync workloads      # Usually unnecessary — selfHeal: true syncs automatically
kubectl get pods -n demo
```

To deploy into test or prod instead, change `workloads-app.yaml`'s `source.path` to the corresponding overlay and push — ArgoCD picks up the change and re-syncs to the new target.

### Step 5 — Lab 2: Drift Detection

```bash
# Manually break the desired state
kubectl scale deployment nginx -n demo --replicas=5
```

In the ArgoCD UI or CLI you'll see the app flip to `OutOfSync`:
```bash
argocd app get workloads
```

Because `syncPolicy.automated.selfHeal: true` is set on `workloads-app.yaml`, ArgoCD reverts the live replica count back to Git's value within seconds — no manual sync needed. To observe the transition live:
```bash
argocd app get workloads --watch
```

If you want to practice a *manual* sync instead, temporarily set `selfHeal: false` on the Application, repeat the drift, then run:
```bash
argocd app sync workloads
```

### Step 6 — Lab 3: App of Apps (already built)

You've already exercised this pattern in Steps 3–4: `platform-app.yaml` → `monitoring-app.yaml`/`logging-app.yaml`/`ingress-app.yaml`/`workloads-app.yaml`. To prove it to yourself, delete one child Application object directly and watch ArgoCD recreate it from Git:

```bash
kubectl delete application monitoring -n argocd
# Within one reconciliation loop of "platform", it reappears:
argocd app list
```

This demonstrates the core GitOps guarantee — Git is the source of truth, not the live cluster state.

### Step 7 — Make a workload change through the proper PR flow

```bash
git checkout -b feature/scale-nginx-prod
# edit gitops/workloads/nginx/overlays/prod/kustomization.yaml — bump replicas
git add gitops/workloads/nginx/overlays/prod/kustomization.yaml
git commit -S -m "Scale prod nginx to 4 replicas"
git push origin feature/scale-nginx-prod
# open a PR — CODEOWNERS requires platform-admins to approve changes under overlays/prod
```

Once merged, if a prod Application exists pointing at that overlay, ArgoCD syncs the change automatically — no one ever ran `kubectl apply` against the cluster.

---

## Validating Locally Before Pushing

```bash
# Render any kustomization to check it's valid
kubectl kustomize gitops/workloads/nginx/overlays/prod
kubectl kustomize gitops/infrastructure/monitoring   # requires Helm available to kustomize

# Or run the same check CI runs
yamllint gitops/
```

## Placeholders to Replace

| Placeholder | Replace with |
|---|---|
| `<YOUR_ORG>/<YOUR_REPO>` | Your actual Git org and repository name (in `root-app.yaml`, `appproject.yaml`, all `platform/*.yaml`) |
| `@<YOUR_ORG>/platform-admins`, `@<YOUR_ORG>/app-developers` | Real GitHub teams in `.github/CODEOWNERS` |
| `changeme-in-vault` in `infrastructure/monitoring/values.yaml` | Pull from Vault/External Secrets instead (see Module 7) |
| Helm chart `version:` pins | Bump as needed; pinning is intentional so drift is only ever caused by your own changes, not upstream chart updates |
