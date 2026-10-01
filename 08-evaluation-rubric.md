# Task 8: Evaluation Rubric

**Time budget: 15 minutes** (self-assessment) - or used by an instructor for grading

## How to Use This

Score each section honestly against what's actually running on your machine right now, not what you
intended to build. Where a check has a specific command, run it - don't estimate.

**Total: 100 points.** Bands: 90-100 Excellent · 75-89 Good · 60-74 Needs Rework · Below 60 Incomplete

---

## Section 1: Microservices Architecture (15 points)

| Check | Command / Evidence | Points |
|---|---|---|
| Both `api` and `worker` build and run without errors | `docker compose up --build` succeeds | 3 |
| Task submission is genuinely asynchronous | Immediately after submitting, status shows `"processing"`, not `"done"` | 4 |
| Worker correctly processes and stores results | Status eventually shows `"done"` with correct uppercased text | 4 |
| `/health` reflects a real dependency check | Stopping Redis causes `/health` to return `503`, not `200` (test this: `docker compose stop redis`, then curl `/health`) | 4 |

## Section 2: Kubernetes Security Primitives (15 points)

| Check | Command / Evidence | Points |
|---|---|---|
| Namespace, Secret, RBAC, NetworkPolicy all exist | `kubectl get namespace,secret,role,rolebinding,networkpolicy -n capstone` | 3 |
| Secret was created imperatively, not committed to Git | `git log --all --full-history -- "*secret*"` shows no Secret YAML ever committed | 3 |
| RBAC least-privilege is provably correct (both allow AND deny) | `kubectl auth can-i list pods --as=system:serviceaccount:capstone:capstone-sa` → yes; same for `delete pods` → no | 5 |
| NetworkPolicy is genuinely enforced, not just declared | An unlabeled test Pod cannot reach Redis (times out) | 4 |

## Section 3: Helm Packaging (15 points)

| Check | Command / Evidence | Points |
|---|---|---|
| Chart renders without error | `helm template ./helm-chart` succeeds | 3 |
| All 4 Pods reach Running/Ready via Helm | `kubectl get pods -n capstone` before any ArgoCD involvement | 4 |
| Pods run as the correct ServiceAccount, not default | `kubectl get pod <api-pod> -n capstone -o jsonpath='{.spec.serviceAccountName}'` returns `capstone-sa` | 4 |
| Full task flow works through the real Kubernetes Service | `curl` through `minikube service api --url` completes a task end to end | 4 |

## Section 4: Observability (15 points)

| Check | Command / Evidence | Points |
|---|---|---|
| Prometheus actually scrapes the custom ServiceMonitor | Prometheus UI Status > Targets shows `capstone/capstone-api-monitor` as `UP` | 5 |
| Custom metric returns real data | Querying `tasks_submitted_total` in Prometheus returns a non-zero number | 4 |
| Logged into Grafana using the retrieved Secret password | Not a guessed default - can explain where the password came from | 3 |
| Built a working dashboard panel | Panel shows `rate(tasks_submitted_total[5m])` and visibly changes under load | 3 |

## Section 5: GitOps with ArgoCD (20 points)

| Check | Command / Evidence | Points |
|---|---|---|
| ArgoCD shows the app Synced and Healthy | ArgoCD UI | 4 |
| A `values.yaml` change, committed and pushed, deployed with **zero** manual `kubectl`/`helm` commands | Can demonstrate the commit history and the resulting live change together | 8 |
| Manual scale-up to 5 replicas was automatically corrected back to Git's stated value | `kubectl get pods` before/after, roughly a minute apart | 8 |

## Section 6: Resilience and Scaling (10 points)

| Check | Command / Evidence | Points |
|---|---|---|
| HPA shows a real (not `<unknown>`) utilization value | `kubectl get hpa -n capstone` | 3 |
| Replica count genuinely increased under sustained load | `kubectl get hpa -n capstone -w` output during the load test | 4 |
| Replica count decreased again after load stopped | Same, a few minutes after | 3 |

## Section 7: Understanding Check - Explain, Don't Just Demonstrate (10 points)

Answer these in your own words - a working cluster with no understanding of why is worth partial credit
at most:

1. Why does the api return `202`, not `200`, when a task is submitted? (2 pts)
2. Why is the Secret never committed to Git, while the NetworkPolicy is? What's the actual difference
   between them that justifies this? (3 pts)
3. What specifically caused the manually-scaled replicas to revert back on their own? Name the exact
   mechanism, not just "ArgoCD did it." (3 pts)
4. Why did the HPA need `resources.requests.cpu` set on the api Deployment to work at all? (2 pts)

---

## Common Point Deductions (things that look done but aren't)

- **Helm chart "works" but was never actually rendered/reviewed first** (`helm template` skipped
  entirely) - deduct up to 3 points even if the end result happens to be correct, since this is a real
  habit worth grading, not just the outcome
- **NetworkPolicy exists but was never actually tested against a real unlabeled Pod** - deduct the full
  4 points from Section 2's last row; an unverified policy that happens to be correctly written is not
  the same as a proven one
- **ServiceMonitor created but never confirmed `UP` in Prometheus's own Targets page** - deduct the full
  5 points from Section 4's first row, even if metrics appear to be flowing by coincidence
- **GitOps change verified only by looking at the ArgoCD UI, never by an independent `kubectl get pods`
  check** - deduct up to 4 points from Section 5, since the point of Section 5 is proving the mechanism,
  not just trusting one dashboard

## Final Score

Add up all sections: **___ / 100**
