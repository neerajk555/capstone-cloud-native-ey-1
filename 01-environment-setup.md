# Task 1: Environment Setup

**Time budget: 10 minutes**

## Goal

Confirm every tool is installed and working, and get minikube running, before writing any code -
catching a missing tool now is far less frustrating than discovering it mid-task.

## Steps

### 1.1: Verify every tool

```
docker --version
minikube version
kubectl version --client
helm version
git --version
node --version
```

Every command should print a real version number. If any is missing, install it before continuing.

### 1.2: Start minikube with enough resources - and Calico for real NetworkPolicy enforcement

This capstone runs more components than a minimal exercise (two app services, Redis, Prometheus,
Grafana, ArgoCD, all at once) - give it real headroom. It also needs a CNI that actually enforces
`NetworkPolicy` - minikube's **default** CNI (`kindnet`) accepts NetworkPolicy objects without error but
**silently does nothing with them** (confirmed directly in minikube's own documentation) - Task 3's
NetworkPolicy would look correctly applied but have zero real effect without this flag:

```
minikube start --driver=docker --cpus=4 --memory=8192 --cni=calico
```

This can take a few minutes on first run - Calico's own pods need to start before the cluster is fully
ready, in addition to minikube's usual startup.

### 1.3: Confirm the cluster is actually up

```
kubectl get nodes
kubectl get pods -A
```

You should see one node in `Ready` status, and a handful of system Pods (`kube-system` namespace) all
`Running` - including several `calico-*` Pods, confirming the NetworkPolicy-enforcing CNI actually
started correctly, not just the cluster itself.

### 1.4: Enable the metrics-server addon now (needed later for autoscaling in Task 7)

```
minikube addons enable metrics-server
```

Doing this now, rather than in Task 7, means it has time to fully start while you work through the
earlier tasks.

## Verify Task 1

- All six version commands succeeded
- `kubectl get nodes` shows one `Ready` node
- `kubectl get pods -A` shows several `calico-*` Pods `Running` (confirms NetworkPolicy enforcement is
  actually active, not just the cluster itself)
- No Pods are in `CrashLoopBackOff` or `Error` state
