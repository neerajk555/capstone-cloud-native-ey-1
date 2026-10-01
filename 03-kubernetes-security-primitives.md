# Task 3: Kubernetes Security Primitives

**Time budget: 30 minutes**

## Goal

Before Helm wraps everything into templates in Task 4, build the raw Kubernetes security objects by
hand once, so you understand exactly what Helm will be generating on your behalf later. This covers
Module 10's core Kubernetes security topics: least-privilege RBAC, Secrets, and network segmentation.

## Steps

### 3.1: Create the namespace

**In VS Code:** create `k8s/namespace.yaml` (a new top-level folder):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: capstone
  labels:
    project: capstone
```

```
kubectl apply -f k8s/namespace.yaml
kubectl config set-context --current --namespace=capstone
```

The second command makes `capstone` your default namespace for the rest of this session, so you stop
needing `-n capstone` on every command.

### 3.2: Create the Secret - imperatively, NOT as a committed file

This is deliberate, and worth understanding why: **Task 6 pushes this entire project to a public GitHub
repository.** A Kubernetes Secret's values are only base64-**encoded**, not encrypted - committing a
Secret's YAML to Git would put the "secret" in plain sight to anyone who looks, permanently in your
repo's history even if you delete it later. This is a real, common mistake, not a hypothetical one.

The correct pattern - and what you'll do here - is: **the Secret is created once, directly in the
cluster, outside of what Git/ArgoCD manages.** Later, your Helm chart will *reference* this Secret by
name, assuming it already exists, rather than trying to create it itself.

```
kubectl create secret generic app-secret \
  --from-literal=API_KEY=demo-key-not-a-real-secret \
  -n capstone
```

Verify it exists (without printing the value in plaintext, matching how you'd actually treat this):

```
kubectl get secret app-secret -n capstone
```

If you ever need to confirm the actual value for debugging:

```
kubectl get secret app-secret -n capstone -o jsonpath='{.data.API_KEY}' | base64 -d
```

### 3.3: Create least-privilege RBAC

**In VS Code:** create `k8s/rbac.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: capstone-sa
  namespace: capstone
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: capstone-pod-reader
  namespace: capstone
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: capstone-pod-reader-binding
  namespace: capstone
subjects:
  - kind: ServiceAccount
    name: capstone-sa
    namespace: capstone
roleRef:
  kind: Role
  name: capstone-pod-reader
  apiGroup: rbac.authorization.k8s.io
```

This grants the `capstone-sa` identity permission to **read** Pod information (get/list/watch) in this
namespace only - nothing else. It can't create, delete, or modify anything, and it has zero visibility
outside the `capstone` namespace. This is the actual meaning of least privilege: grant exactly what's
needed, nothing more - your `api` and `worker` Pods will run as this ServiceAccount starting in Task 4.

```
kubectl apply -f k8s/rbac.yaml
```

### 3.4: Prove the RBAC actually works - both the allow AND the deny

```
kubectl auth can-i list pods --as=system:serviceaccount:capstone:capstone-sa -n capstone
kubectl auth can-i delete pods --as=system:serviceaccount:capstone:capstone-sa -n capstone
kubectl auth can-i list secrets --as=system:serviceaccount:capstone:capstone-sa -n capstone
```

You should see `yes`, `no`, `no` in that order. **Checking the denials is just as important as checking
the allow** - confirming this ServiceAccount CAN'T delete Pods or read Secrets is the actual proof that
least-privilege is working, not just that something was granted.

### 3.5: Create a NetworkPolicy restricting who can reach Redis

**In VS Code:** create `k8s/networkpolicy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-restrict-ingress
  namespace: capstone
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
        - podSelector:
            matchLabels:
              app: worker
      ports:
        - protocol: TCP
          port: 6379
```

By default, Kubernetes allows every Pod to talk to every other Pod - a flat, fully-open network. This
policy changes that specifically for Pods labeled `app: redis`: once this policy exists, Redis **only**
accepts inbound connections from Pods labeled `app: api` or `app: worker`, on port 6379 - everything
else is denied, including a random Pod someone else might later deploy in this same namespace.

```
kubectl apply -f k8s/networkpolicy.yaml
```

**You can't fully test this policy's enforcement yet** - there's no Redis Pod running to test against
until Task 4 deploys one. Task 4 includes the actual test once real Pods exist to verify against.

## Verify Task 3

- `kubectl get namespace capstone` shows it exists
- `kubectl get secret app-secret -n capstone` shows it exists (created imperatively, not from a
  committed file)
- `kubectl auth can-i list pods --as=system:serviceaccount:capstone:capstone-sa -n capstone` returns
  `yes`
- `kubectl auth can-i delete pods --as=system:serviceaccount:capstone:capstone-sa -n capstone` returns
  `no`
- `kubectl get networkpolicy -n capstone` shows `redis-restrict-ingress`

## Solution Reference

`solutions/03-kubernetes-security-primitives/` contains `namespace.yaml`, `rbac.yaml`, and
`networkpolicy.yaml`. The Secret has no solution file, on purpose - re-read Step 3.2 if that seems like
an omission rather than the point.
