# Task 7: Prove GitOps, Self-Healing, and Autoscaling

**Time budget: 20 minutes**

## Goal

Three concrete proofs, each building on everything so far: a real GitOps deployment triggered by nothing
but a `git push`, a self-healing correction after a manual change, and CPU-based autoscaling actually
kicking in under load.

## Part A: Prove GitOps Works (7 min)

### 7.1: Make a real change

**In VS Code:** open `helm-chart/values.yaml` and change:
```yaml
worker:
  replicaCount: 1
```
to:
```yaml
worker:
  replicaCount: 2
```

### 7.2: Commit and push - do NOT touch kubectl or helm

```
git add helm-chart/values.yaml
git commit -m "Scale worker to 2 replicas"
git push
```

If you find yourself wanting to type `kubectl apply` or `helm upgrade` right now, stop - that would
defeat the entire point of this task.

### 7.3: Watch ArgoCD pick it up

In the ArgoCD UI, within about 3 minutes `capstone-app` will show **OutOfSync**, then sync
automatically. Confirm:

```
kubectl get pods -n capstone -l app=worker
```

You should see 2 worker Pods now, having never run a deployment command yourself.

## Part B: Prove Self-Healing (5 min)

### 7.4: Manually break something, bypassing Git entirely

```
kubectl scale deployment api -n capstone --replicas=5
kubectl get pods -n capstone -l app=api
```

You now have 5 api Pods - but Git still says 2 (from Task 4's `values.yaml`, untouched by Part A's
worker change).

### 7.5: Watch ArgoCD correct it

Wait about a minute, then check again:

```
kubectl get pods -n capstone -l app=api
```

ArgoCD's `selfHeal: true` should have already noticed live state (5 replicas) doesn't match Git (2
replicas) and corrected it back down, with no action from you.

## Part C: Prove Autoscaling Works (8 min)

### 7.6: Add a CPU-intensive test endpoint

**In VS Code:** open `api/server.js` and add this function and route (anywhere before
`async function main()`):

```javascript
function busyWork(durationMs) {
  const end = Date.now() + durationMs;
  let x = 0;
  while (Date.now() < end) {
    x += Math.sqrt(x + 1);
  }
  return x;
}

app.get('/stress', (req, res) => {
  busyWork(200);
  res.json({ stressed: true });
});
```

This deliberately burns CPU for 200ms per request via a busy loop - real work an autoscaler needs to
react to, not a `setTimeout` (which costs no real CPU at all while waiting).

### 7.7: Rebuild, redeploy through the normal GitOps path

```
eval $(minikube docker-env)
cd api
docker build -t capstone-api:v1 .
cd ..
```

Since the image tag didn't change (`v1` again), ArgoCD won't detect a difference on its own here -
force a refresh of the running Pods so they pick up the freshly-built image:

```
kubectl rollout restart deployment api -n capstone
```

### 7.8: Create the HPA

**In VS Code:** create `k8s/hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: capstone
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

```
kubectl apply -f k8s/hpa.yaml
```

Confirm metrics-server (enabled back in Task 1) is actually reporting real numbers, not `<unknown>`:

```
kubectl top pods -n capstone -l app=api
kubectl get hpa -n capstone
```

If `TARGETS` shows `<unknown>/50%` instead of a real percentage, wait another 30-60 seconds -
metrics-server needs a short warm-up period after Task 1 to start reporting.

### 7.9: Generate sustained load and watch it scale

```
for i in $(seq 1 200); do curl -s $(minikube service api --url)/stress > /dev/null & done
```

In a second terminal, watch the HPA react in real time:

```
kubectl get hpa -n capstone -w
```

Within a minute or two, you should see `REPLICAS` climb from 2 toward higher numbers (up to the
`maxReplicas: 6` ceiling) as `TARGETS` shows utilization well above 50%.

### 7.10: Stop the load and watch it scale back down

Let the background `curl` loop finish (or `Ctrl+C` it), then keep watching:

```
kubectl get hpa -n capstone -w
```

After a few minutes of low CPU usage, `REPLICAS` should drop back toward `minReplicas: 2`. This is
slower than scaling up on purpose - Kubernetes deliberately waits longer before scaling down, to avoid
rapidly flapping replica counts up and down from momentary quiet periods.

## Verify Task 7

- **Part A**: `values.yaml`'s worker replica change went live after nothing but `git push`
- **Part B**: manually scaling `api` to 5 replicas was automatically corrected back to 2 within about a
  minute
- **Part C**: `kubectl get hpa -n capstone` showed a real (not `<unknown>`) utilization percentage, and
  `REPLICAS` genuinely increased under load and decreased once load stopped

## A Note on Design Patterns (Module 5)

You've already built two resilience patterns without a dedicated task for them: the async task queue
from Task 2 is itself a form of the **Bulkhead pattern** - a failure or slowdown in `worker` can't
directly take down `api`, since they're only connected through Redis, not a direct call. And the
readiness probe from Task 4 checking real Redis connectivity is a simple form of a **Circuit
Breaker**'s core idea - stop sending traffic to something that can't currently do its job, rather than
letting requests fail one at a time. If you have extra time, consider how you'd extend `server.js` to
retry a failed Redis call with backoff before giving up - the next logical step in this same pattern
family.

## Solution Reference

`solutions/07-resilience-and-scaling/hpa.yaml` and `values-change-reference.yaml`
