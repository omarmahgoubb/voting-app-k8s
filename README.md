# Legacy Voting App on Kubernetes (minikube)

This repo tracks my step-by-step journey to run and improve a legacy app on Kubernetes.

Stack:
- **vote** (frontend) → **redis**
- **worker** (moves votes) → **postgres**
- **result** (frontend) ← **postgres**

> ⚠️ The current Postgres is **ephemeral** (`emptyDir`). Deleting the DB pod wipes data (we’ll switch to a StatefulSet later).

---

## Prerequisites
- Linux with Docker running
- `kubectl` and `minikube` installed

---

## What’s working now
- App runs fully on **minikube**.
- Image pulls fixed using `imagePullPolicy: IfNotPresent` and pre-loaded images.
- **Health checks** (liveness/readiness) for vote/result/db/redis.
- **Resource requests/limits** set for all pods.
- **metrics-server** enabled → `kubectl top` works.
- **Ingress** (single host **vote.local**) with path `/result` for the results UI.

---

## Chronological Log (commands + why)

### 0) Namespace
```bash
kubectl create ns vote
```
**Why:** Isolate all app resources in a dedicated namespace.

---

### 1) First deploy (manifests)

Create a file called **`voting-app.yaml`** (contains Redis, Postgres, vote, worker, result, and Services), then:
```bash
kubectl -n vote apply -f voting-app.yaml
```
**Why:** Apply initial Kubernetes objects so pods and services come up.

---

### 2) Handle image pulls on minikube

Pulled images locally:
```bash
docker pull redis:7-alpine
docker pull postgres:16
docker pull dockersamples/examplevotingapp_vote:latest
docker pull dockersamples/examplevotingapp_worker:latest
docker pull dockersamples/examplevotingapp_result:latest
```

Loaded images into the **minikube** node:
```bash
minikube image load redis:7-alpine
minikube image load postgres:16
minikube image load dockersamples/examplevotingapp_vote:latest
minikube image load dockersamples/examplevotingapp_worker:latest
minikube image load dockersamples/examplevotingapp_result:latest
```
**Why:** Avoid `ImagePullBackOff` by ensuring the node has images locally.

Set **imagePullPolicy** to use local images if present:
```bash
kubectl -n vote patch deploy vote   -p '{"spec":{"template":{"spec":{"containers":[{"name":"vote","imagePullPolicy":"IfNotPresent"}]}}}}'
kubectl -n vote patch deploy result -p '{"spec":{"template":{"spec":{"containers":[{"name":"result","imagePullPolicy":"IfNotPresent"}]}}}}'
kubectl -n vote patch deploy worker -p '{"spec":{"template":{"spec":{"containers":[{"name":"worker","imagePullPolicy":"IfNotPresent"}]}}}}'
```

Restart and watch:
```bash
kubectl -n vote rollout restart deploy
kubectl -n vote get pods -w
```
**Why:** Ensure Kubernetes uses local copies and refresh pods with the new policy.

Optional port-forward to open the UIs directly:
```bash
kubectl -n vote port-forward svc/vote-svc   8080:80   # http://localhost:8080
kubectl -n vote port-forward svc/result-svc 8081:80   # http://localhost:8081
```

---

### 3) Update (1): Add Health Checks ✅

Applied updated `voting-app.yaml` with **readiness** and **liveness** probes for:
- **vote/result**: HTTP readiness on `/`; TCP liveness on port `80`
- **redis**: TCP `6379`
- **db (postgres)**: TCP `5432`

Rolled out and verified:
```bash
kubectl -n vote apply -f voting-app.yaml
kubectl -n vote rollout status deploy/redis
kubectl -n vote rollout status deploy/db
kubectl -n vote rollout status deploy/vote
kubectl -n vote rollout status deploy/result
kubectl -n vote rollout status deploy/worker
```

**Why:**
- **Readiness** gates traffic until the pod is actually ready.
- **Liveness** restarts containers that hang/crash.

#### How to verify health checks
Delete a `vote` pod → a new one is created; it only receives traffic after readiness passes:
```bash
kubectl -n vote delete pod -l app=vote --wait=false
kubectl -n vote get pods -l app=vote -w
```

Check Service endpoints shrink/expand with readiness:
```bash
kubectl -n vote get endpoints vote-svc -o=jsonpath='{range .subsets[*].addresses[*]}{.ip}{"\n"}{end}'
```

---

### 4) Update (2): Resource requests/limits (stability) ✅

Set sane requests/limits per container and persisted them in `voting-app.yaml`:

- vote/result/worker → **requests**: `100m / 128Mi`, **limits**: `500m / 512Mi`  
- redis → **requests**: `100m / 128Mi`, **limits**: `500m / 256Mi`  
- postgres → **requests**: `250m / 512Mi`, **limits**: `1 CPU / 1Gi`

Verify:
```bash
kubectl -n vote describe pod -l app=vote   | sed -n '/Containers:/,/QoS Class:/p'
kubectl -n vote describe pod -l app=result | sed -n '/Containers:/,/QoS Class:/p'
kubectl -n vote describe pod -l app=worker | sed -n '/Containers:/,/QoS Class:/p'
kubectl -n vote describe pod -l app=redis  | sed -n '/Containers:/,/QoS Class:/p'
kubectl -n vote describe pod -l app=db     | sed -n '/Containers:/,/QoS Class:/p'
```
Notes: CPU over limit → throttled; Memory over limit → OOMKilled.

---

### 5) Metrics (kubectl top) ✅
Enabled and verified pod metrics.

```bash
minikube addons enable metrics-server
kubectl -n vote top pods
```
If you see `Metrics API not available`, ensure the metrics-server pod is running and the `v1beta1.metrics.k8s.io` APIService is **Available**.

---

### 6) Ingress (single host with path for result) ✅

Using NGINX Ingress to expose:
- `http://vote.local/` → vote UI
- `http://vote.local/result` → result UI

**Manifest (`ingress.yaml`):**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vote-ing
  namespace: vote
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: vote.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vote-svc
            port:
              number: 80
      - path: /result
        pathType: Prefix
        backend:
          service:
            name: result-svc
            port:
              number: 80
```

**Enable & apply:**
```bash
minikube addons enable ingress
kubectl -n vote apply -f ingress.yaml
```

**Host mapping (VERY IMPORTANT):**
```bash
echo "$(minikube ip) vote.local" | sudo tee -a /etc/hosts
```

**Test:**
```bash
curl -I http://vote.local/
curl -I http://vote.local/result
# Assets should resolve under /result/ as well:
curl -I http://vote.local/result/stylesheets/style.css
curl -I http://vote.local/result/app.js
curl -I http://vote.local/result/socket.io/socket.io.js
```
> If `http://vote.local/result` looks plain or blank, hard-refresh (Ctrl+F5).  
> If CSS still fails, check which URL the page requests:  
> `curl -s http://vote.local/result | grep -i stylesheet` and ensure it points to `/result/stylesheets/style.css` (the file that exists in the container).

---

## Troubleshooting

- **ImagePullBackOff** → ensure images are loaded into minikube and `imagePullPolicy: IfNotPresent` is set (see step 2).
- **DB data vanished** → current DB uses `emptyDir`. This is expected; we’ll switch to a StatefulSet + PVC later.
- **Ingress 404/503** → check endpoints exist for services:
  ```bash
  kubectl -n vote get endpoints vote-svc result-svc
  ```
- **vote.local not resolving** → re-add the hosts entry with the latest `minikube ip`.

---

## Git tips (Conventional Commits)

Use short, typed messages:
- `feat`: new feature (e.g., `feat(ingress): expose vote.local and /result`)
- `fix`: bug fix
- `docs`: docs only
- `chore`: tooling/infra or refactors with no user-visible change
- `perf`, `ci`, `test`, `refactor`, `revert` as needed

Examples:
```bash
git commit -m "docs: add metrics and ingress sections"
git commit -m "feat(k8s): add liveness/readiness probes"
```

---

## Next plans
- Switch Postgres to **StatefulSet + PVC** (persistent data).
- Add **HPAs** for vote/result.
- Add **PDBs**, **Secrets** for DB creds, and **NetworkPolicies** when ready.
