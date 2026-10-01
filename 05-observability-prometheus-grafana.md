# Task 5: Observability with Prometheus and Grafana

**Time budget: 30 minutes**

## Goal

Your `api` service already exposes a `/metrics` endpoint (built in Task 2). This task installs a real
Prometheus + Grafana stack locally, configures Prometheus to actually scrape that endpoint, and builds a
dashboard panel showing live task-submission data - covering Module 9's observability concepts (metrics
collection, visualization, health checks) entirely offline.

## Steps

### 5.1: Install kube-prometheus-stack

This single chart installs Prometheus, Grafana, and Alertmanager together - the standard, current way to
get all three without configuring each separately:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
```

**Use exactly this release name (`monitoring`)** - the next step depends on it matching precisely.

This takes a few minutes to fully start. Watch it:

```
kubectl get pods -n monitoring -w
```

Wait until every Pod shows `Running`, then Ctrl+C.

### 5.2: The single most important gotcha in this entire task

`kube-prometheus-stack` documents its default `serviceMonitorSelector` as `{}` (meaning "match every
ServiceMonitor in the cluster"). **In practice, this is not what actually happens** - due to a
Helm-templating quirk in the chart itself, the live Prometheus configuration ends up requiring any
ServiceMonitor to carry the label `release: monitoring` (matching whatever name you used in Step 5.1) or
Prometheus will never discover it - even though the ServiceMonitor object itself is created successfully
with no error at all. This is a real, well-documented, frequently-hit trap, not a hypothetical edge case
- the ServiceMonitor you write in the next step includes this label specifically because of this.

### 5.3: Create a ServiceMonitor telling Prometheus to scrape your api service

**In VS Code:** create `k8s/servicemonitor.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: capstone-api-monitor
  namespace: capstone
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: api
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

Two things to notice: `release: monitoring` is exactly the label from Step 5.2. `endpoints[].port: http`
references your api Service's port **by the name you gave it** (`name: http`, back in Task 4's Service
template) - not by the raw port number 3000. This is how the ServiceMonitor CRD works generally, and a
very common point of confusion for anyone using it for the first time.

```
kubectl apply -f k8s/servicemonitor.yaml
```

### 5.4: Verify Prometheus actually picked it up - don't just trust that `kubectl apply` succeeded

First, find the actual Prometheus Service name - chart versions vary slightly in exactly how they name
it, so confirm rather than guess:

```
kubectl get svc -n monitoring | grep prometheus
```

Look for the one with `9090` in its ports (not `-operated`, which is a headless variant) - port-forward
that one:

```
kubectl port-forward svc/PASTE-THE-REAL-NAME-HERE -n monitoring 9090:9090
```

Open **http://localhost:9090** in your browser, go to **Status > Targets**, and look for
`capstone/capstone-api-monitor` in the list, showing `UP`. **This is the real verification** - the
ServiceMonitor object existing in Kubernetes and Prometheus actually scraping it are two different
things, and Step 5.2's gotcha is exactly what causes them to diverge.

Generate some real traffic while you have this open:

```
for i in 1 2 3 4 5; do curl -X POST $(minikube service api --url)/tasks -H "Content-Type: application/json" -d "{\"text\":\"load test $i\"}"; done
```

Then query for your custom metric directly in Prometheus's UI (the search box on the Graph tab):

```
tasks_submitted_total
```

You should see a real number, reflecting everything you've submitted since the api Pods started -
including Task 4's testing, not just these 5 new requests.

### 5.5: Access Grafana

```
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

**Copy this password now.** The default Grafana admin password for this chart is **not** `admin` - it's
generated (commonly `prom-operator` unless a values override changes it), and multiple people have been
confused by this exact thing before you - always retrieve it with this command rather than guessing.

```
kubectl port-forward svc/monitoring-grafana -n monitoring 3001:80
```

Open **http://localhost:3001**, log in with username `admin` and the password from above.

### 5.6: Build a dashboard panel for your custom metric

In Grafana: **Dashboards > New > New Dashboard > Add visualization**. Choose the `Prometheus`
data source (already pre-configured by the chart - you don't need to add it manually). In the query
field, enter:

```
rate(tasks_submitted_total[5m])
```

This shows the RATE of task submissions per second, averaged over a 5-minute window - a more useful
metric than a raw ever-increasing counter, since it shows current activity rather than a total that only
ever goes up. Give the panel a title like "Task Submission Rate" and save the dashboard.

Generate a bit more traffic and watch the panel update:

```
for i in 1 2 3 4 5 6 7 8; do curl -X POST $(minikube service api --url)/tasks -H "Content-Type: application/json" -d "{\"text\":\"more load $i\"}"; sleep 1; done
```

## Verify Task 5

- Prometheus's **Status > Targets** page shows `capstone/capstone-api-monitor` as `UP`
- Querying `tasks_submitted_total` in Prometheus returns a real, non-zero number
- You successfully logged into Grafana using the password retrieved from the Kubernetes Secret (not a
  guessed default)
- Your custom dashboard panel shows the task submission rate changing as you generate more traffic

## Solution Reference

`solutions/05-observability/servicemonitor.yaml` contains the ServiceMonitor. Grafana dashboards aren't
version-controllable as a simple file in this setup, so there's no dashboard solution file - the exact
query in Step 5.6 is the "solution" for that part.

## Cleanup Note

Leave Prometheus and Grafana running - you don't need to uninstall them for Tasks 6-7.
