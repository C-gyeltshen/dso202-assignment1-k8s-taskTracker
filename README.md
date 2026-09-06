# DSO202 — Assignment 1: Three-Tier Application Deployment on Kubernetes

**Namespace:** `dso202-assignment-01`
**Cluster:** `kind` (1 control-plane node, 2 worker nodes)

This repository contains the Kubernetes manifests and supporting documentation for deploying a three-tier Task Tracker application (frontend, backend, database) to a local `kind` cluster, as specified in the DSO202 Assignment 1 brief.

---

## Repository Structure

```bash
assignment-1/
├── README.md
├── common-manifests/
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── quota.yaml
├── database/
│   ├── pvc.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── backend/
│   ├── deployment.yaml
│   └── service.yaml
├── frontend/
│   ├── deployment.yaml
│   └── service.yaml
├── cluster/
│   └── kind-cluster.yaml
├── app/                      # provided application source (frontend, backend, db) + docker-compose.yml
├── evidence/                 # screenshots supporting Task 7 verification
└── report/
    └── assignment1.md        # full step-by-step build log
```

---

## 1. Architecture Note (Task 1)

### Cluster Topology

The `kind` cluster consists of three Docker containers, each acting as a Kubernetes Node:

| Container | Node role | Components running |
| --- | --- | --- |
| `dso202-control-plane` | `control-plane` | `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, `kubelet`, `kube-proxy` |
| `dso202-worker` | `worker-node-1` | `kubelet`, `kube-proxy`, application Pods |
| `dso202-worker2` | `worker-node-2` | `kubelet`, `kube-proxy`, application Pods |

The host machine maps port `30080` to the same port on the control-plane container, which is what allows the frontend's NodePort Service to be reached directly from a browser at `http://localhost:30080`.

### What Happens on the Control Plane for Every Pod

Regardless of which tier a Pod belongs to, the same sequence occurs on the control plane:

1. **`kube-apiserver`** receives the `kubectl apply`, validates the manifest against the API schema, and writes it to `etcd`. Every read and write in the cluster passes through this component.
2. **`etcd`** persists the desired state (the manifest) as the cluster's single source of truth.
3. **`kube-scheduler`** watches for unscheduled Pods, evaluates the two worker nodes against resource requests/limits and any affinity rules, and binds the Pod to one of them.

The control plane's role is entirely about **deciding and recording** — it never runs application containers itself.

### What Happens on the Worker Node That Is Selected

Once the scheduler binds a Pod to `worker-node-1` or `worker-node-2`:

1. **`kubelet`** on that node watches the API server for Pods assigned to it, pulls the required image (from Docker Hub) via the container runtime (`containerd`), and starts the container(s).
2. **`kube-proxy`** programs the node's iptables/IPVS rules so traffic sent to a Service's ClusterIP is forwarded to the correct Pod IP, regardless of which node the Pod is actually running on.
3. **The CNI plugin** (`kindnet`, by default in `kind`) assigns the Pod an IP from the cluster's pod subnet (`10.244.0.0/16`) and wires up networking so Pods can reach each other across nodes.

### Why a Dedicated Namespace

A namespace is a logical partition within a single physical cluster — it is not tied to any node and does not get its own control plane or scheduler. All three tiers of this application live in one namespace (`dso202-assignment-01`) so that:

- Their Services can resolve each other by short DNS name (e.g. `db-svc`, `backend-svc`) without a fully qualified path.
- Resource governance (ResourceQuota, LimitRange) and RBAC can be scoped to the assignment as a single unit, isolated from `default` and other namespaces on the cluster.

### Object Choice per Tier

| Tier | Object Used | Why |
| --- | --- | --- |
| **Frontend** (nginx, static assets) | `Deployment` | Stateless and interchangeable — any replacement Pod is identical to the one it replaces. Exposed via a `NodePort` Service so it can be reached directly from the host browser. |
| **Backend** (REST API) | `Deployment` | Stateless as long as it holds no local session state; all persistence is delegated to Postgres. Exposed internally via a `ClusterIP` Service so only cluster-internal traffic (from the frontend, or `kubectl exec`/port-forward for testing) can reach it. |
| **Database** (PostgreSQL) | `Deployment` + `PersistentVolumeClaim` | A single-replica Deployment is used per the assignment's scope, with its data path backed by a PVC so that Pod lifecycle and data lifecycle are decoupled. Exposed via a **headless** Service (`clusterIP: None`) so the backend can address it by a stable DNS name rather than a load-balanced virtual IP — appropriate for a single-instance database. |

### Supporting Objects (Not Tied to a Single Pod)

| Object | Purpose |
| --- | --- |
| **ConfigMap** (`app-config`) | Non-sensitive configuration shared across tiers: `DB_HOST`, `DB_PORT`, `DB_NAME`, `APP_PORT`, `CORS_ORIGIN`, `POSTGRES_DB`, `BACKEND_URL`. |
| **Secret** (`app-secret`) | Credential values: `DB_USER`, `DB_PASSWORD`, `POSTGRES_USER`, `POSTGRES_PASSWORD`. |
| **PersistentVolumeClaim** | Requests storage from `kind`'s default `standard` StorageClass (backed by `local-path`/`hostPath` provisioning on the node), used exclusively by the database tier. |

---

## 2. Configuration and Secrets (Task 2)

All non-sensitive configuration lives in a single ConfigMap (`app-config`); all credentials live in a single Secret (`app-secret`). The two objects are kept strictly separate — no credential value appears in the ConfigMap, and no non-sensitive value is duplicated unnecessarily in the Secret.

A deliberate naming mismatch exists between the two naming conventions and is preserved intentionally, per the assignment contract:

| Backend expects | Official Postgres image expects | Value |
| --- | --- | --- |
| `DB_NAME` | `POSTGRES_DB` | Same value (e.g. `taskdb`) |
| `DB_USER` | `POSTGRES_USER` | Same value |
| `DB_PASSWORD` | `POSTGRES_PASSWORD` | Same value |

### ⚠️ Secret Encoding Caveat

**Kubernetes Secrets are base64-encoded, not encrypted, by default.** Base64 is a reversible encoding, not a cryptographic protection — anyone with API access to the namespace, or direct access to `etcd`, can trivially decode a Secret's contents (`kubectl get secret ... -o jsonpath='{.data.KEY}' | base64 -d`). This is documented here as a known limitation of the assignment's scope (Unit I), not something remediated within these manifests. A production deployment would additionally require:

- Encryption at rest for Secrets in `etcd`.
- RBAC rules restricting Secret access to only the workloads and users that need it (see Task 8, optional bonus, for a minimal read-only RBAC example — note that Secrets are deliberately **excluded** from that Role's permissions).
- Consideration of an external secret store (e.g. HashiCorp Vault, cloud provider secret manager) rather than native Kubernetes Secrets.

---

## 3. ResourceQuota and LimitRange Justification (Task 6)

### Context

The namespace runs three single-replica Deployments (frontend, backend, database) — three Pods under normal, steady-state operation. The values below were chosen with that baseline in mind, plus headroom for temporary duplication during rollouts (e.g. an old Pod terminating while a new one starts, as happens during an image update or the Task 7c Pod-deletion test).

### ResourceQuota

```yaml
hard:
  requests.cpu: "1500m"
  requests.memory: "1.5Gi"
  limits.cpu: "3000m"
  limits.memory: "3Gi"
  pods: "10"
```

| Value | Reasoning |
| --- | --- |
| `requests.cpu: 1500m` | ~500m CPU per tier at request level — sufficient for steady-state operation of an nginx frontend, a lightweight REST backend, and a single Postgres instance, none of which are compute-heavy at this scale. |
| `requests.memory: 1.5Gi` | ~512Mi per tier at request level, sized for Postgres (the heaviest of the three), with frontend and backend needing considerably less. |
| `limits.cpu: 3000m` | Double the total request, allowing each tier to burst under load without any single tier consuming an entire node's CPU. |
| `limits.memory: 3Gi` | Same doubling logic as CPU — burst headroom without unbounded consumption. |
| `pods: "10"` | Covers the 3 steady-state Pods plus temporary duplicates during rollouts or Pod recreation, while still preventing runaway or accidental mass Pod creation. |

### LimitRange

```yaml
limits:
  - type: Container
    default:
      cpu: "250m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "500m"
      memory: "512Mi"
    min:
      cpu: "50m"
      memory: "64Mi"
```

| Value | Reasoning |
| --- | --- |
| `default` (limit) | Auto-applied to any container that doesn't declare its own `resources.limits`. None of the three Deployments specify container-level limits, so without this default every container would have no enforced ceiling — making the namespace ResourceQuota effectively meaningless. |
| `defaultRequest` | Same reasoning, for `resources.requests` — ensures every Pod contributes a measurable, predictable amount toward the ResourceQuota's request totals. |
| `max: 500m CPU / 512Mi memory` | Hard per-container ceiling so a single misbehaving container cannot consume the entire namespace's quota and starve the other two tiers. |
| `min: 50m CPU / 64Mi memory` | Floor preventing a container from being scheduled with an unrealistically small allocation that would cause immediate CPU throttling or an OOM kill unrelated to actual application bugs. |

### Why the LimitRange Is Necessary Alongside the ResourceQuota

A ResourceQuota only enforces a *ceiling* on the sum of requests/limits across the namespace — it does not require individual Pods to declare any resources at all. Since the provided container images' Deployments don't specify `resources:` blocks, applying a ResourceQuota alone (with `requests.cpu`/`requests.memory` set) would actually **block Pod creation entirely**, because Kubernetes refuses to schedule a Pod with no defined resource requests into a namespace governed by such a quota. The LimitRange resolves this by transparently injecting sensible default requests and limits onto every container, so Pods remain schedulable while staying properly bounded and accounted for.

---

## 4. Verification Evidence (Task 7)

Full command transcripts and screenshots are in `evidence/` and `report/assignment1.md`. Summary of what was demonstrated:

### 7a — Full CRUD Cycle
Verified via `curl` through a port-forwarded backend Service (`kubectl port-forward svc/backend-svc 8080:8080`), since the frontend's browser-side JavaScript cannot resolve the cluster-internal DNS name `backend-svc` from outside the cluster network. All five operations succeeded: create (`POST /api/tasks`), list (`GET /api/tasks`), retrieve, update (`PUT /api/tasks/{id}`), and delete (`DELETE /api/tasks/{id}`).

### 7b — Service DNS Resolution
From inside the running frontend Pod (`kubectl exec`), `curl http://backend-svc:8080/api/status` returned a successful response (`{"status":"ok","db":"connected"}`), confirming cluster DNS correctly resolves the backend Service by name from another Pod.

### 7c — Self-Healing and Data Persistence
1. A task was created and its ID recorded.
2. `kubectl get pods --watch` was left running in a separate terminal.
3. The backend Pod was deleted manually (`kubectl delete pod -l tier=backend`).
4. The watch output showed the old Pod terminate and a new Pod (different ReplicaSet hash) reach `Running`.
5. The previously created task was retrieved successfully via the new Pod, confirming that the database's PersistentVolumeClaim — not the backend container's ephemeral filesystem — is what preserves data across Pod recreation.

### 7d — Declarative vs. Imperative Comparison
A ConfigMap was created two ways for comparison:
- **Declaratively:** `kubectl apply -f common-manifests/configmap.yaml` (the method used for every real resource in this project).
- **Imperatively:** `kubectl create configmap demo-config --from-literal=DEMO_KEY=demo-value` (created and torn down as a throwaway, so it never touched the real `app-config`).

**Comparison:** the declarative approach is repeatable and idempotent — reapplying the same file produces no unwanted side effects, and the YAML itself is a reviewable, version-controlled source of truth (which is why every real resource in this repository is managed this way). The imperative approach is faster for one-off or exploratory changes but leaves no record in version control and is harder to reproduce exactly later. In practice, declarative management is preferred for anything intended to persist or be audited, while imperative commands are best reserved for quick, disposable debugging.

---

## 5. Known Constraints and Troubleshooting Notes

- **Browser-side `BACKEND_URL` limitation:** `BACKEND_URL` resolves correctly *inside* the cluster (Pod-to-Pod), but a browser running on the host machine cannot resolve cluster-internal DNS names like `backend-svc`. This is a structural characteristic of the assignment's networking design, not a misconfiguration — the brief explicitly permits `curl` through a port-forwarded backend as an alternative verification path for Task 7a, which is what this submission uses.
- **Image tag placeholders:** during initial deployment, both the backend and database Deployments were briefly left with a literal `<tag>` placeholder instead of the actual image tag (`1.0`), producing an `InvalidImageName` error. Resolved by confirming the exact published tag on Docker Hub for each of the three images and updating the manifests accordingly.
- **Minimal runtime images:** the provided images intentionally omit debugging tools such as `curl`; `wget` and `kubectl port-forward` were used as alternatives, consistent with the assignment's build standard of minimal, hardened runtime images.