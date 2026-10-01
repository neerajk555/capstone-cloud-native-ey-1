# Task 6: GitOps with ArgoCD

**Time budget: 25 minutes**

## Goal

Push your project to GitHub, install ArgoCD, and hand off deployment control from manual `helm install`
commands to ArgoCD watching your repository - the same GitOps mechanism Module 8 covers, running
entirely locally.

## What ArgoCD Will and Won't Manage

Your repo has two kinds of content: the `helm-chart/` (application - api, worker, redis) and the `k8s/`
folder (Namespace, RBAC, NetworkPolicy, ServiceMonitor - security and platform primitives from Tasks 3
and 5). **ArgoCD will only watch `helm-chart/`** - the `k8s/` objects stay as one-time, manually-applied
resources, exactly like the Secret from Task 3. This mirrors a common real-world split: platform/security
setup and application deployment are often genuinely separate concerns, owned by different pipelines.

## Steps

### 6.1: Create a GitHub repository and push everything

```
git init
git add .
git commit -m "Capstone: microservices, k8s security primitives, Helm chart"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPO-NAME.git
git push -u origin main
```

If prompted for credentials, use a GitHub Personal Access Token as the password (create one under GitHub
Settings > Developer Settings > Personal Access Tokens if needed).

### 6.2: Install ArgoCD

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=180s
```

### 6.3: Get the initial admin password and log in

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open **https://localhost:8080**, accept the self-signed certificate warning, log in with `admin` and the
password above.

### 6.4: Create the Application, pointing at your repo

**In VS Code:** create `argocd/application.yaml` (a new top-level folder):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: capstone-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR-USERNAME/YOUR-REPO.git
    targetRevision: HEAD
    path: helm-chart
  destination:
    server: https://kubernetes.default.svc
    namespace: capstone
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

**Replace `YOUR-USERNAME/YOUR-REPO` with your actual repository** - the single most common mistake here.
`CreateNamespace=false` is deliberate, not a placeholder left unset - the `capstone` namespace already
exists from Task 3, and ArgoCD doesn't need to (and shouldn't) manage its lifecycle.

### 6.5: Remove your manual Helm release - let ArgoCD take over

```
helm uninstall capstone-app -n capstone
```

This is a clean, deliberate handoff. Confirm the Pods actually terminated before continuing:

```
kubectl get pods -n capstone
```

(Should briefly show nothing, or Pods `Terminating`.)

### 6.6: Apply the Application and watch ArgoCD redeploy everything

```
kubectl apply -f argocd/application.yaml
```

Go back to the ArgoCD UI and refresh - within a minute or two, `capstone-app` should show **Synced** and
**Healthy**, with all 4 Pods (2x api, 1x worker, 1x redis) back and running - created by ArgoCD this
time, not your earlier `helm install`.

### 6.7: Confirm the app still works exactly as before

```
curl $(minikube service api --url)/health
curl -X POST $(minikube service api --url)/tasks -H "Content-Type: application/json" -d '{"text":"deployed via argocd"}'
```

## Verify Task 6

- Your GitHub repo contains `api/`, `worker/`, `k8s/`, `helm-chart/`, and `argocd/`, fully committed
- The ArgoCD UI shows `capstone-app` as **Synced** and **Healthy**
- The app responds correctly to the same health/task-submission test as every previous task
- `kubectl get pods -n capstone` shows the same 4 Pods, but you can confirm via `kubectl describe` that
  they're now owned by the ArgoCD-managed release, not a lingering manual one

## Solution Reference

`solutions/06-gitops-with-argocd/application.yaml`
