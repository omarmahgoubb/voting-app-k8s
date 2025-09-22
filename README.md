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

## Chronological Log (commands + why)

### 0) Namespace
```bash
kubectl create ns vote
```
**Why:** Isolate all app resources in a dedicated namespace.

---

### 1) First deploy (manifests)

Create a file called `voting-app.yaml` (contains Redis, Postgres, vote, worker, result, and Services), then:
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

Loaded images into the minikube node:
```bash
minikube image load redis:7-alpine
minikube image load postgres:16
minikube image load dockersamples/examplevotingapp_vote:latest
minikube image load dockersamples/examplevotingapp_worker:latest
minikube image load dockersamples/examplevotingapp_result:latest
```
**Why:** Avoid `ImagePullBackOff` by ensuring the node has images locally.

Set `imagePullPolicy` to use local images if present:
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

Port-forward UIs:
```bash
kubectl -n vote port-forward svc/vote-svc   8080:80   # http://localhost:8080
kubectl -n vote port-forward svc/result-svc 8081:80   # http://localhost:8081
```

---

### 3) Update (1): Add Health Checks ✅

Applied updated `voting-app.yaml` with readiness and liveness probes for:
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

### Troubleshooting

- **ImagePullBackOff** → ensure images are loaded into minikube and `imagePullPolicy: IfNotPresent` is set (see step 2).
- **DB data vanished** → current DB uses `emptyDir`. This is expected; we’ll switch to a StatefulSet + PVC later.

