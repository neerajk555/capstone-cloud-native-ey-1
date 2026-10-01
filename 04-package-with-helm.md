# Task 4: Package Everything with Helm

**Time budget: 35 minutes**

## Goal

Package the api, worker, and Redis into one Helm chart, deploy it manually (before handing control to
ArgoCD in Task 6), and finally test the NetworkPolicy from Task 3 against real, running Pods.

## Steps

### 4.1: Point your terminal's Docker at minikube's internal Docker daemon

```
eval $(minikube docker-env)
```

**Run this in every new terminal before building images** - minikube runs its own internal Docker
daemon, separate from your system's. This command redirects `docker build` in THIS terminal to build
directly inside minikube, so Kubernetes can immediately use the result with no registry or push step.

### 4.2: Build both images

```
cd api
docker build -t capstone-api:v1 .
cd ../worker
docker build -t capstone-worker:v1 .
cd ..
```

### 4.3: Scaffold the chart

```
helm create helm-chart
rm helm-chart/templates/*.yaml
rm -rf helm-chart/templates/tests
rm helm-chart/values.yaml
mkdir -p helm-chart/templates
```

### 4.4: Write `values.yaml`

**In VS Code:** create `helm-chart/values.yaml`:

```yaml
api:
  replicaCount: 2
  image:
    repository: capstone-api
    tag: v1
    pullPolicy: IfNotPresent
  service:
    type: NodePort
    port: 3000
  resources:
    requests:
      cpu: 20m
      memory: 64Mi

worker:
  replicaCount: 1
  image:
    repository: capstone-worker
    tag: v1
    pullPolicy: IfNotPresent

redis:
  image: redis:7-alpine
  port: 6379

serviceAccountName: capstone-sa
```

`serviceAccountName` points at the `capstone-sa` ServiceAccount you created in Task 3 - this chart
doesn't create its own ServiceAccount; it assumes the one from Task 3 already exists, the same
"security primitives are provisioned separately" pattern as the Secret. The api's `resources.requests`
looks like a small, easy-to-skip detail right now, but Task 7's autoscaling demo depends on it entirely
- Kubernetes can only calculate CPU **utilization percentage** relative to a requested baseline; without
this, a CPU-based HPA has nothing to divide by and simply won't work.

### 4.5: Write the Redis template

**In VS Code:** create `helm-chart/templates/redis.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: {{ .Values.redis.image }}
          ports:
            - containerPort: {{ .Values.redis.port }}
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
    - port: {{ .Values.redis.port }}
      targetPort: {{ .Values.redis.port }}
```

Notice the `app: redis` label on the Pod template - this is exactly the label Task 3's NetworkPolicy
selects on. If this label didn't match, the NetworkPolicy would silently apply to nothing.

### 4.6: Write the api template

**In VS Code:** create `helm-chart/templates/api.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: {{ .Values.api.replicaCount }}
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      serviceAccountName: {{ .Values.serviceAccountName }}
      containers:
        - name: api
          image: "{{ .Values.api.image.repository }}:{{ .Values.api.image.tag }}"
          imagePullPolicy: {{ .Values.api.image.pullPolicy }}
          resources:
            requests:
              cpu: {{ .Values.api.resources.requests.cpu }}
              memory: {{ .Values.api.resources.requests.memory }}
          ports:
            - containerPort: 3000
          env:
            - name: REDIS_URL
              value: "redis://redis:{{ .Values.redis.port }}"
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: API_KEY
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: {{ .Values.api.service.type }}
  selector:
    app: api
  ports:
    - name: http
      port: {{ .Values.api.service.port }}
      targetPort: 3000
```

Notice the port is explicitly **named** `http`, not just numbered - Task 5's Prometheus ServiceMonitor
will reference this port BY NAME, not by number, which is how the ServiceMonitor CRD's `endpoints[].port`
field actually works. Skipping the name here would make Task 5's metrics scraping silently fail to find
this Service at all.

Two things tie directly back to earlier tasks: `secretKeyRef` reads `API_KEY` from the `app-secret` you
created imperatively in Task 3 (this chart never creates that Secret itself), and the readiness probe
hits the SAME `/health` endpoint that genuinely checks Redis connectivity - a Pod won't show `Ready`
until it can actually reach Redis, not just until the process starts.

### 4.7: Write the worker template

**In VS Code:** create `helm-chart/templates/worker.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
spec:
  replicas: {{ .Values.worker.replicaCount }}
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      serviceAccountName: {{ .Values.serviceAccountName }}
      containers:
        - name: worker
          image: "{{ .Values.worker.image.repository }}:{{ .Values.worker.image.tag }}"
          imagePullPolicy: {{ .Values.worker.image.pullPolicy }}
          env:
            - name: REDIS_URL
              value: "redis://redis:{{ .Values.redis.port }}"
```

No Service needed here - nothing ever needs to initiate a connection TO the worker; it only ever
initiates connections OUT to Redis.

### 4.8: Render locally before deploying - catch mistakes early

```
helm template capstone-app ./helm-chart
```

Read through the output - confirm `replicas: 2` for api, `replicas: 1` for worker and redis, and that
`serviceAccountName: capstone-sa` appears in both api and worker's Pod specs.

### 4.9: Deploy it

```
helm install capstone-app ./helm-chart
```

### 4.10: Verify everything is running

```
kubectl get pods
kubectl get svc
```

Wait until all 4 Pods (2x api, 1x worker, 1x redis) show `Running` and `1/1` or `2/2` ready.

### 4.11: Test the full system through Kubernetes

```
minikube service api --url
```

```
curl $(minikube service api --url)/health

curl -X POST $(minikube service api --url)/tasks -H "Content-Type: application/json" -d '{"text":"running on kubernetes now"}'
```

Copy the returned `id`, wait 2 seconds, then:

```
curl $(minikube service api --url)/tasks/PASTE-THE-ID-HERE
```

You should see the completed, uppercased result - the exact same behavior as Task 2's Docker Compose
test, now running as real, separate Kubernetes Pods talking to each other through a Service, not
containers on one Docker network.

### 4.12: Now test Task 3's NetworkPolicy against real Pods

First, confirm normal traffic still works (api and worker ARE allowed to reach redis - you just proved
this indirectly in Step 4.11, since task processing worked at all). Now prove the DENY side - try to
reach Redis from a Pod that ISN'T labeled `app: api` or `app: worker`:

```
kubectl run netpolicy-test --image=redis:7-alpine --restart=Never --rm -it -- redis-cli -h redis ping
```

This starts a temporary, unrelated Pod (no matching label) and tries to ping Redis directly. **This
should hang and eventually time out** - proof the NetworkPolicy is actually blocking it, not just
present but inert. Press Ctrl+C if it hangs longer than 10-15 seconds, then confirm the Pod cleaned
itself up:

```
kubectl get pods | grep netpolicy-test
```

(Should show nothing - `--rm` cleans it up automatically once you exit.)

## Verify Task 4

- All 4 Pods (2x api, 1x worker, 1x redis) show `Running`
- The full task-submission flow works end-to-end through the real Kubernetes Service, matching Task 2's
  Compose behavior
- An unlabeled test Pod trying to reach Redis directly **times out** - proof the NetworkPolicy from
  Task 3 is genuinely enforced, not just declared

## Solution Reference

`solutions/04-package-with-helm/helm-chart/` contains the complete chart - `Chart.yaml`, `values.yaml`,
and all three templates.

## A Note Before Continuing

**Do not run `helm uninstall` yet** - Task 6 does this deliberately, as part of handing control over to
ArgoCD. Leave everything running as-is.
