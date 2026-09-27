# Assignment 2 Implementation Guide: StatefulSet and Ingress

This guide builds **directly on top of Assignment 1**. It explains what currently exists, what exactly changes and why, and gives you every manifest and command you need to implement both features.

---

## Current State (Assignment 1)

### Cluster

The `kind` cluster `dso202` exists and all 3 nodes are `Ready`. However, after Docker Engine was stopped and restarted, `etcd` state was lost — the namespace `dso202-assignment-01` and every Kubernetes object inside it (Deployments, Services, ConfigMap, Secret, PVC, Pods) was wiped. See `verify.md` for a full explanation and evidence.

```bash
kind get clusters      # → dso202 (exists, 18 days old)
kubectl get nodes      # → control-plane, worker-node-1, worker-node-2 all Ready
kubectl get namespaces # → dso202-assignment-01 is absent
```

The cluster is healthy but completely empty. All YAML files on disk are intact.

### Files on disk (Assignment 1 — what was written, nothing deployed yet)

```
cluster/kind-cluster.yaml          ← kind cluster: 1 control-plane + 2 workers
                                      host port 30080 only (no 80/443)

common-manifests/
  namespace.yaml                   ← namespace: dso202-assignment-01
  configmap.yaml                   ← app-config (DB_HOST="db-svc", BACKEND_URL="http://backend-svc:8080")
  secret.yaml                      ← app-secret (DB credentials, base64-encoded)
  quota.yaml                       ← ResourceQuota + LimitRange

database/
  pvc.yaml                         ← PersistentVolumeClaim: db-pvc (no longer needed — StatefulSet replaces this)
  deployment.yaml                  ← Deployment: db-deployment (no longer needed — StatefulSet replaces this)
  service.yaml                     ← Headless Service: db-svc (kept as-is)

backend/
  deployment.yaml                  ← Deployment: backend-deployment
  service.yaml                     ← ClusterIP Service: backend-svc (port 8080)

frontend/
  deployment.yaml                  ← Deployment: frontend-deployment
  service.yaml                     ← NodePort Service: frontend-svc (nodePort: 30080)
```

### Known limitations in Assignment 1

| Limitation | Root cause |
|---|---|
| Database Pod gets a random name (`db-deployment-7fd9-xxxx`) on every restart | `Deployment` has no identity guarantee |
| Data storage is decoupled via a separately-created `PVC` (two separate objects to manage) | `Deployment` cannot own its own storage |
| Frontend is reachable at `http://localhost:30080` via a raw `NodePort` | No Ingress layer; only one port is mapped on the cluster |
| Browser JavaScript cannot reach `backend-svc:8080` | `BACKEND_URL` is a cluster-internal DNS name; browsers run outside the cluster network |
| Frontend and backend have no single, clean HTTP entry point | Each tier has its own exposure mechanism |

---

## Part 1 — StatefulSet for the Database

### Why This Change

The database tier is currently a `Deployment` + a separately created `PVC`. A `Deployment` treats pods as interchangeable — pods get random names, there is no ordering guarantee, and storage must be pre-created and referenced manually.

A **StatefulSet** is Kubernetes' purpose-built workload for stateful applications like databases. The key differences:

| Property | `Deployment` (current) | `StatefulSet` (new) |
|---|---|---|
| Pod naming | `db-deployment-7fd9b7dc94-d272b` (random) | `db-0`, `db-1`, ... (stable, ordinal) |
| Pod DNS | Changes every restart | `db-0.db-svc.dso202-assignment-01.svc.cluster.local` (permanent) |
| Storage management | Separate `PVC` object you create manually | `volumeClaimTemplates` inside the StatefulSet — Kubernetes creates and owns the PVC per pod |
| Startup order | All pods start in parallel, no ordering | Pods start in order: `db-0` must be `Running` before `db-1` starts |
| Deletion order | Pods deleted in any order | Pods deleted in reverse order: `db-1` before `db-0` |

For PostgreSQL specifically:
- The pod name `db-0` never changes, even after crashes or rescheduling. The backend can address it as `db-0.db-svc` — a stable, predictable DNS name, not an IP.
- If you later add replicas, PostgreSQL primary (`db-0`) starts before replicas (`db-1`, `db-2`), which is the correct bootstrap order.

### What Changes

| Action | File |
|---|---|
| **No longer needed** | `database/deployment.yaml` — namespace is gone, nothing to delete from cluster; this file can be removed from the repo |
| **No longer needed** | `database/pvc.yaml` — StatefulSet manages its own PVC automatically; this file can be removed from the repo |
| **Create** | `database/statefulset.yaml` |
| **Update** | `common-manifests/configmap.yaml` — change `DB_HOST` from `db-svc` to `db-0.db-svc` |
| Keep as-is | `database/service.yaml` — the headless service stays; StatefulSet references it via `serviceName` |

### Step 1 — Update the ConfigMap

The StatefulSet pod is addressable by its stable DNS name `db-0.db-svc` (pod `db-0` behind the headless service `db-svc`). Update `DB_HOST` to use this explicit, stable identity rather than the service name alone.

**File: `common-manifests/configmap.yaml`**

Change:
```yaml
  DB_HOST: "db-svc"
```
To:
```yaml
  DB_HOST: "db-0.db-svc"
```

Full updated file:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dso202-assignment-01
  labels:
    app: task-tracker
data:
  DB_HOST: "db-0.db-svc"
  DB_PORT: "5432"
  DB_NAME: "taskdb"
  APP_PORT: "8080"
  CORS_ORIGIN: "*"
  POSTGRES_DB: "taskdb"
  BACKEND_URL: "http://backend-svc:8080"
```

> **Why `db-0.db-svc` instead of `db-svc`?**
> `db-svc` is the headless service and still resolves to the pod IP — it would still work with 1 replica. But using `db-0.db-svc` explicitly demonstrates the StatefulSet's core feature: a stable, per-pod DNS identity that never changes regardless of rescheduling. This is what makes StatefulSets meaningful for databases.

### Step 2 — Create the StatefulSet manifest

**Create file: `database/statefulset.yaml`**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  namespace: dso202-assignment-01
  labels:
    tier: database
spec:
  serviceName: "db-svc"
  replicas: 1
  selector:
    matchLabels:
      tier: database
  template:
    metadata:
      labels:
        tier: database
    spec:
      containers:
        - name: db
          image: sarojsanyasi/dso202-db:1.0
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: POSTGRES_DB
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: db-storage
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: db-storage
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

**Explanation of key fields:**

- `serviceName: "db-svc"` — must match the name of the existing headless service. This is what enables the stable DNS `db-0.db-svc`.
- `volumeClaimTemplates` — this replaces `database/pvc.yaml`. Kubernetes automatically creates a PVC named `db-storage-db-0` and binds it to the `db-0` pod. The PVC is owned by the StatefulSet and persists independently of the pod.
- The pod spec (container image, env vars, volumeMounts) is identical to the old `deployment.yaml` — only the wrapper object type changes.

### Step 3 — Apply the changes

The namespace and all previous objects were already wiped when Docker was restarted (see `verify.md`). There is nothing to delete from the cluster — apply directly.

First apply the namespace and supporting objects if not done yet:

```bash
kubectl apply -f common-manifests/namespace.yaml
kubectl apply -f common-manifests/secret.yaml
kubectl apply -f common-manifests/quota.yaml
```

Then apply the updated ConfigMap, headless service, and StatefulSet:

```bash
kubectl apply -f common-manifests/configmap.yaml
kubectl apply -f database/service.yaml
kubectl apply -f database/statefulset.yaml
```

### Step 4 — Verify

```bash
# Confirm the StatefulSet is created
kubectl get statefulset -n dso202-assignment-01

# Confirm the pod name is db-0 (stable identity)
kubectl get pods -n dso202-assignment-01 -l tier=database

# Confirm the PVC was automatically created by the StatefulSet
kubectl get pvc -n dso202-assignment-01

# Confirm the headless service still exists and has an endpoint
kubectl get svc db-svc -n dso202-assignment-01
kubectl get endpoints db-svc -n dso202-assignment-01
```

**Expected output:**

```
NAME   READY   AGE
db     1/1     30s

NAME     READY   STATUS    RESTARTS   AGE
db-0     1/1     Running   0          30s

NAME                STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
db-storage-db-0     Bound    pvc-xxxx   1Gi        RWO            standard       30s
```

**Verify the backend can reach the database through the stable DNS name:**

```bash
# Restart the backend pod so it picks up the updated DB_HOST from the ConfigMap
kubectl rollout restart deployment backend-deployment -n dso202-assignment-01

# Wait for it to come back
kubectl rollout status deployment backend-deployment -n dso202-assignment-01

# Check the backend logs — should see "db connected"
kubectl logs -n dso202-assignment-01 -l tier=backend --tail=10
```

**Verify from inside the cluster:**

```bash
kubectl exec -n dso202-assignment-01 -it db-0 -- psql -U postgres -d taskdb -c "\dt"
```

This connects directly to the database pod using its stable ordinal name `db-0` — not a random hash suffix.

---

## Part 2 — Ingress with NGINX Ingress Controller

### Why This Change

Currently:
- The frontend is exposed via `NodePort 30080` — a raw TCP port on the node, no path routing, no TLS.
- The browser cannot call `backend-svc:8080` because cluster-internal DNS names are not resolvable outside the cluster. This is why Assignment 1 Task 7a required `curl` through `kubectl port-forward` instead of using the browser.

An **Ingress** is an API object that defines HTTP routing rules (host/path → service). An **Ingress Controller** (NGINX in this case) is the component that reads those rules and actually proxies the traffic. Together they:

- Provide a **single entry point** at `http://localhost:80` for the entire application
- Route `/api` to the backend and `/` to the frontend — both under the same host
- Allow the browser's JavaScript to call `/api/tasks` as a relative URL, making CORS and DNS non-issues
- Are the foundation for adding TLS (`https://`) later

### What Changes

| Action | File |
|---|---|
| **Update** | `cluster/kind-cluster.yaml` — add port 80/443 mappings and `ingress-ready=true` label |
| **Update** | `common-manifests/configmap.yaml` — change `BACKEND_URL` to `""` (empty string) |
| **Update** | `frontend/service.yaml` — change type from `NodePort` to `ClusterIP` |
| **Create** | `ingress/ingress.yaml` — the routing rules |
| **One-time install** | NGINX Ingress Controller (applied to the cluster with `kubectl apply`) |
| **Cluster recreation** | Required to pick up the new port mappings in `kind-cluster.yaml` |

> **Important:** The `kind` cluster must be **deleted and recreated** because port mappings in `kind` cannot be changed after the cluster is created. All your YAML manifests are preserved — the cluster itself is stateless metadata; your app state lives in the YAML files. You will re-apply everything after the cluster is recreated.

### Step 1 — Update the kind cluster config

**File: `cluster/kind-cluster.yaml`**

Add port mappings for 80 and 443 to the control-plane node's `extraPortMappings`, and add the `ingress-ready=true` node label (required by the kind-specific NGINX Ingress manifest):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: dso202

networking:
  apiServerAddress: "127.0.0.1"
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/16"

nodes:
  - role: control-plane
    image: kindest/node:v1.36.1@sha256:3489c7674813ba5d8b1a9977baea8a6e553784dab7b84759d1014dbd78f7ebd5
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          name: control-plane
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
        listenAddress: "127.0.0.1"
        protocol: TCP
      - containerPort: 80
        hostPort: 80
        listenAddress: "127.0.0.1"
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        listenAddress: "127.0.0.1"
        protocol: TCP
  - role: worker
    image: kindest/node:v1.36.1@sha256:3489c7674813ba5d8b1a9977baea8a6e553784dab7b84759d1014dbd78f7ebd5
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          name: worker-node-1
          kubeletExtraArgs:
            node-labels: "dso202/node-role=worker,dso202/node-index=1"
  - role: worker
    image: kindest/node:v1.36.1@sha256:3489c7674813ba5d8b1a9977baea8a6e553784dab7b84759d1014dbd78f7ebd5
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          name: worker-node-2
          kubeletExtraArgs:
            node-labels: "dso202/node-role=worker,dso202/node-index=2"
```

**What changed from Assignment 1:**
- Added `kubeletExtraArgs: node-labels: "ingress-ready=true"` to the control-plane node. The kind-specific NGINX Ingress manifest targets nodes with this label so it knows which node to bind ports 80/443 on.
- Added two new `extraPortMappings`: host port 80 → container port 80, host port 443 → container port 443.
- Port 30080 is kept for backward compatibility, though after Ingress is working you won't need it.

### Step 2 — Update the ConfigMap

The frontend's `app.js` constructs API calls as:
```javascript
const API = `${BACKEND_URL}/api/tasks`;
```

With Ingress, both the frontend (`/`) and backend (`/api`) are served under the same host (`http://localhost`). Setting `BACKEND_URL` to `http://localhost` means API calls resolve to `http://localhost/api/tasks`, which the Ingress routes to the backend.

**File: `common-manifests/configmap.yaml`** (builds on the StatefulSet update from Part 1):

Change:
```yaml
  BACKEND_URL: "http://backend-svc:8080"
```
To:
```yaml
  BACKEND_URL: "http://localhost"
```

> **Why not `""`?** The frontend container's `docker-entrypoint.sh` uses `${BACKEND_URL:=http://localhost:8080}` — the `:=` shell syntax replaces the variable if it is empty OR unset. An empty string from the ConfigMap gets silently overwritten by the fallback before `envsubst` runs. Setting it to `http://localhost` is non-empty, bypasses that default, and works correctly with the Ingress.

This is the fix for the long-standing limitation where the browser could not reach the backend. By routing everything through one Ingress host, the DNS problem disappears entirely.

### Step 3 — Update the frontend Service

The frontend no longer needs to be reachable directly via `NodePort`. The Ingress controller will receive all external traffic on port 80 and route it to the frontend service. Change the service type to `ClusterIP`:

**File: `frontend/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: dso202-assignment-01
  labels:
    tier: frontend
spec:
  type: ClusterIP
  selector:
    tier: frontend
  ports:
    - port: 8080
      targetPort: 8080
```

> The `nodePort: 30080` field is removed because `ClusterIP` services are not exposed at the node level — all external traffic comes in through the Ingress.

### Step 4 — Create the Ingress resource

**Create directory and file: `ingress/ingress.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: task-tracker-ingress
  namespace: dso202-assignment-01
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-svc
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 8080
```

**Explanation of key fields:**

- `ingressClassName: nginx` — tells Kubernetes which Ingress Controller is responsible for this Ingress object. This must match the class name that the NGINX controller registers.
- **Path ordering:** `/api` is listed before `/`. NGINX evaluates paths in order of specificity — the more specific `/api` rule is matched first, so backend API calls never accidentally hit the frontend.
- `path: /api` with `pathType: Prefix` — any request whose URL path starts with `/api` (e.g. `/api/tasks`, `/api/status`) is forwarded to `backend-svc:8080`. The path is forwarded as-is, which is correct because the Express backend handles routes at `/api/tasks` (not `/tasks`).
- `path: /` with `pathType: Prefix` — everything else (including `/`, `/index.html`, `/styles.css`, `/app.js`) goes to `frontend-svc:8080`.

### Step 5 — Recreate the cluster and re-apply everything

**Delete the existing cluster:**

```bash
kind delete cluster --name dso202
```

**Create the cluster with the updated config:**

```bash
kind create cluster --config cluster/kind-cluster.yaml
```

**Verify the new cluster has all three port mappings:**

```bash
kubectl get nodes -o wide
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep dso202
```

You should see ports `80`, `443`, and `30080` all mapped on the `dso202-control-plane` container.

**Re-apply the namespace and all supporting objects first:**

```bash
kubectl apply -f common-manifests/namespace.yaml
kubectl apply -f common-manifests/configmap.yaml
kubectl apply -f common-manifests/secret.yaml
kubectl apply -f common-manifests/quota.yaml
```

**Re-apply the database tier (headless service + StatefulSet from Part 1):**

```bash
kubectl apply -f database/service.yaml
kubectl apply -f database/statefulset.yaml
```

**Re-apply the backend tier:**

```bash
kubectl apply -f backend/deployment.yaml
kubectl apply -f backend/service.yaml
```

**Re-apply the frontend tier (now with ClusterIP service):**

```bash
kubectl apply -f frontend/deployment.yaml
kubectl apply -f frontend/service.yaml
```

**Redeploy frontend and backend once to pick up the updated ConfigMap:**

`BACKEND_URL` is not read at runtime — it is baked into `config.js` by the container's entrypoint script (`envsubst`) at the moment the pod starts. If a pod was already running before the ConfigMap was updated, it will still have the old value. A rollout restart forces both pods to restart and re-read the current ConfigMap.

```bash
kubectl rollout restart deployment frontend-deployment -n dso202-assignment-01
kubectl rollout restart deployment backend-deployment -n dso202-assignment-01
```

Wait for both to finish before applying the Ingress:

```bash
kubectl rollout status deployment frontend-deployment -n dso202-assignment-01
kubectl rollout status deployment backend-deployment -n dso202-assignment-01
```

Confirm the frontend picked up `BACKEND_URL: ""`:

```bash
kubectl exec -n dso202-assignment-01 \
  $(kubectl get pod -n dso202-assignment-01 -l tier=frontend -o jsonpath='{.items[0].metadata.name}') \
  -- cat /usr/share/nginx/html/config.js
```

Expected: `BACKEND_URL: ""`

**Install the NGINX Ingress Controller (kind-specific manifest):**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Wait for the Ingress Controller pod to be fully ready before continuing:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

**Apply the Ingress resource:**

```bash
kubectl apply -f ingress/ingress.yaml
```

### Step 6 — Verify

**Check all objects are in place:**

```bash
kubectl get all -n dso202-assignment-01
kubectl get ingress -n dso202-assignment-01
```

**Check the Ingress has been assigned an address:**

```bash
kubectl describe ingress task-tracker-ingress -n dso202-assignment-01
```

Expected output includes:
```
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /api   backend-svc:8080
              /      frontend-svc:8080
```

**Test the backend API through the Ingress:**

```bash
curl -s http://localhost/api/status
```

Expected: `{"status":"ok","db":"connected"}`

**Test a full CRUD cycle through the Ingress:**

```bash
# Create a task
curl -s -X POST http://localhost/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Ingress test","description":"routed via nginx ingress"}'

# List tasks
curl -s http://localhost/api/tasks
```

**Test the frontend through the Ingress:**

Open `http://localhost` in a browser. The Task Tracker UI should load, and the status pill should show `backend + db online` — this confirms that browser JavaScript can now call the backend API, because both are served under the same origin (`http://localhost`).

---

## Full Picture: Before and After

### Repository structure after both changes

```
cluster/kind-cluster.yaml          ← UPDATED: added port 80/443 mappings + ingress-ready label

common-manifests/
  namespace.yaml                   ← unchanged
  configmap.yaml                   ← UPDATED: DB_HOST="db-0.db-svc", BACKEND_URL=""
  secret.yaml                      ← unchanged
  quota.yaml                       ← unchanged

database/
  pvc.yaml                         ← DELETED (StatefulSet manages PVC automatically)
  deployment.yaml                  ← DELETED (replaced by StatefulSet)
  statefulset.yaml                 ← NEW
  service.yaml                     ← unchanged (headless service, db-svc)

backend/
  deployment.yaml                  ← unchanged
  service.yaml                     ← unchanged (ClusterIP, port 8080)

frontend/
  deployment.yaml                  ← unchanged
  service.yaml                     ← UPDATED: NodePort → ClusterIP (no nodePort field)

ingress/
  ingress.yaml                     ← NEW
```

### Traffic flow comparison

**Assignment 1:**
```
Browser → localhost:30080 → [NodePort] → frontend pod (nginx)
Browser → localhost:8080  → [port-forward] → backend pod  (curl only, not browser)
Backend → db-svc:5432     → [Headless Service] → db-deployment-7fd9-xxxx pod
```

**Assignment 2:**
```
Browser → localhost:80 → [NGINX Ingress] → /     → frontend pod (nginx)
                                          → /api  → backend pod (Express)
Backend → db-0.db-svc:5432 → [Headless Service] → db-0 pod (stable identity)
```

### Summary of what each change demonstrates

| Change | Unit 2 topic demonstrated |
|---|---|
| `Deployment` → `StatefulSet` for the database | 2.1 StatefulSets — stable pod identity, ordered lifecycle |
| `volumeClaimTemplates` replaces separate `pvc.yaml` | 2.1.4 Volume claim templates |
| Pod named `db-0`, addressable as `db-0.db-svc` | 2.1.2.1 Stable network identities |
| `serviceName: "db-svc"` references existing headless service | 2.1.3 Headless services for StatefulSets |
| NGINX Ingress Controller installed on kind cluster | 2.2.2.1 NGINX Ingress Controller |
| `Ingress` resource with `/api` and `/` routing rules | 2.2.1.1 Basic routing rules |
| `ingressClassName: nginx` annotation | 2.2.3 Ingress annotations for controller-specific features |
| `BACKEND_URL=""` so browser JS works without port-forward | Solves the Assignment 1 Task 7a browser limitation |

---

## Troubleshooting

### StatefulSet pod stuck in `Pending`

```bash
kubectl describe pod db-0 -n dso202-assignment-01
```

Common cause: the quota from `quota.yaml` was applied but the new pod has no resource requests. Confirm `quota.yaml` is applied and the LimitRange is active — it auto-injects default requests onto containers that don't declare them.

### Backend shows `ENOTFOUND db-0.db-svc`

The StatefulSet pod must be `Running` before the backend can resolve `db-0.db-svc`. Check:

```bash
kubectl get pod db-0 -n dso202-assignment-01
```

If the pod is not yet `Running`, wait and then restart the backend:

```bash
kubectl rollout restart deployment backend-deployment -n dso202-assignment-01
```

### Ingress returns 404 for `/api`

Check that the Ingress controller is running and the Ingress object has an address:

```bash
kubectl get pods -n ingress-nginx
kubectl get ingress task-tracker-ingress -n dso202-assignment-01
```

If `ADDRESS` is blank, the controller pod is not yet ready. Re-run the `kubectl wait` command from Step 5.

### `curl http://localhost/api/status` returns `connection refused`

The cluster was not recreated with the new port mappings. Verify:

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep dso202
```

You must see `:80->80/tcp` in the control-plane container's port column. If not, the cluster needs to be deleted and recreated with the updated `kind-cluster.yaml`.

### Browser shows `backend unreachable` in the status pill

Check that the frontend pod picked up the new `BACKEND_URL=""` from the ConfigMap:

```bash
kubectl exec -n dso202-assignment-01 -it \
  $(kubectl get pod -n dso202-assignment-01 -l tier=frontend -o jsonpath='{.items[0].metadata.name}') \
  -- cat /usr/share/nginx/html/config.js
```

Expected: `BACKEND_URL: ""`

If it shows the old value, restart the frontend deployment to force the ConfigMap to be re-read:

```bash
kubectl rollout restart deployment frontend-deployment -n dso202-assignment-01
```
