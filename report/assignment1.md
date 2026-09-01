## Repository Structure
```bash
├── app
│   ├── backend
│   │   ├── Dockerfile
│   │   ├── package-lock.json
│   │   ├── package.json
│   │   └── src
│   │       ├── db.js
│   │       ├── routes
│   │       │   └── tasks.js
│   │       └── server.js
│   ├── db
│   │   ├── Dockerfile
│   │   └── init
│   │       └── 01-init.sql
│   ├── docker-compose.yml
│   └── frontend
│       ├── Dockerfile
│       ├── docker-entrypoint.sh
│       ├── nginx.conf
│       └── public
│           ├── app.js
│           ├── config.js.template
│           ├── index.html
│           └── styles.css
├── backend
├── cluster
│   └── kind-cluster.yaml
├── common-manifests
├── database
├── evidence
│   ├── 1.png
│   ├── 2.png
│   ├── 3.png
├── frontend
└── report
    └── assignment1.md
```
* **App folder**: 
    * Contains the three tier application (frontend, backend, database) and docker-compose.yml file to run the application.
* **Cluster folder**: 
    * Contains the manifests to deploy the application in kubernetes cluster using `kind`.

* **evidence**:
    * Contains the screenshots of the application running in local machine and in kubernetes cluster.

* **manifests**:
    * Contains the manifests to deploy the application in kubernetes cluster using `kind`.

* **report**:
    * Contains the report of the assignment.

## **`Stage 0:  Prerequisites`**
* `Docker Engine or Docker Desktop` installed in your local machine.
* `Kind` installed in your local machine.
* `Kubectl` installed in your local machine.

## **`Stage 1: Creating the Three Node Cluster using Kind`**
```
                     host machine (one laptop)
   ┌───────────────────────────────────────────────────────────────┐
   │  Docker                                                       │
   │                                                               │
   │   ┌───────────────────────┐                                   │
   │   │ container:            │  Kubernetes Node object name:     │
   │   │ dso202-control-plane  │  control-plane                    │
   │   │                       │  runs kube-apiserver, etcd,       │
   │   │                       │  kube-scheduler,                  │
   │   │                       │  kube-controller-manager,         │
   │   │                       │  kubelet, kube-proxy              │
   │   └───────────────────────┘                                   │
   │                                                               │
   │   ┌───────────────────────┐   ┌───────────────────────┐       │
   │   │ container:            │   │ container:            │       │
   │   │ dso202-worker         │   │ dso202-worker2        │       │
   │   │ Node object name:     │   │ Node object name:     │       │
   │   │ worker-node-1         │   │ worker-node-2         │       │
   │   │ runs kubelet,         │   │ runs kubelet,         │       │
   │   │ kube-proxy,           │   │ kube-proxy,           │       │
   │   │ application Pods      │   │ application Pods      │       │
   │   └───────────────────────┘   └───────────────────────┘       │
   │                                                               │
   │   host port 30080  ──►  control-plane container port 30080    │
   └───────────────────────────────────────────────────────────────┘

```
### Step 1: Copy Listing 1 from `DSO202_Practical1_Manifests.md` into `cluster/kind-cluster.yaml.`

```bash
cd cluster
touch kind-cluster.yaml
```
![2](../evidence/2.png)

### Step 2: Create the cluster using `kind` command
```bash
kind create cluster --config cluster/kind-cluster.yaml
```
The `--config` flag is used to specify the configuration file for the cluster. The `kind create cluster` command will create a new Kubernetes cluster with the specified configuration.
![3](../evidence/3.png)
![4](../evidence/4.png)

### Step 3: Confirm the clsuter exist, and list all the nodes in the cluster.

```bash
kind get clusters
kind get nodes --name <cluster name>
```
![5](../evidence/5.png)

### Step 4: Inspecting the Cluster and its Components.
```bash
kind cluster-info
```
![6](../evidence/6.png)

### Step 5: List all the nodes.
```bash 
kubectl get nodes -o wide
```
![7](../evidence/7.png)
There are three nodes (control-plan, worker-node-1 and worker-node-2)

### Step 6: List all the namespace.
```bash
kubectl get namespaces
```
![8](../evidence/8.png)

## **`Namespace and Architecture Note`**

* In kubernetes, a namespace provides a machinism for isolating groups of resources withen a single clauster.

* since in this assignment we are going to create deployment, replicasets, services and configmaps so keeping this all resources in the `default` namespace is not a good idea. So we will create a new namespace called `dso202` and keep all the resources in this namespace.

`What all will get into the namespace?`
All the three components (frontend, backend and database) will get into the namespace. So we will create a new namespace called `dso202` and keep all the resources in this namespace including
* **workloads** (Pods/ Deployments/ StatefulSets):
    * Frontend & Backend: Deployments (which manage your application Pods).

    * Database: Deployment or StatefulSet (which runs your database container).

* **Services**: 
    * Network endpoints that allow components to discover and communicate with each other inside the cluster (e.g., backend-service, db-service).

* **ConfigMaps & Secrets**:
    * ConfigMaps: For non-sensitive config environment variables (e.g., API base URLs, database port numbers).

    * Secrets: For sensitive data (e.g., database passwords, secret keys).

* **PersistentVolumeClaims (PVCs)**: 
    * Used by your database to request persistent storage on the worker nodes so data isn't lost if the DB container restarts.
![9](../evidence/9.png)

### Step 0: Writing the manifest file to create namespace called `dso202-assignment-01`.
```bash
cd common-manifests
touch namespace.yaml
# COnfigure the namespace in the namespace.yaml file.
```
![10](../evidence/10.png)
![11](../evidence/11.png)

```bash 
k apply -f common-manifests/namespace.yaml 
```
![12](../evidence/12.png)

**Verify:**
```bash
k get namespaces
k describe namespace dso202-assignment-01
```
![13](../evidence/13.png)

#### **About the Namespace**
A namespace is a logical partation or a virtual cluster inside a physical clauster and it does not get its own control plan or schedulaer.

All the three tire application (frontend, backend and database) lives in one namespace so their services can find each other by a short DNS name.

#### **Where does namespace resides? inside or out of node?**

Nodes are cluster scoped resources representing the physical or virtual machines in the cluster. Namespaces are also cluster scoped resources, but they are not tied to any specific node. Instead, namespaces provide a way to organize and manage resources across the entire cluster.

![14](../evidence/14.png)

#### **What happens on the control-plane node, for every Pod ?**
Regardless of tier, the same control-plane components act on every Pod we create:
1. **`kube-apiserver`**: receives your kubectl apply, validates the object against the schema, writes it to etcd. Every single read/write in the cluster goes through this.
2. **`etcd`**: persists the desired state (manifest) as the source of truth.
3. **`kube-scheduler`**: watches for Pods, evaluates the two worker nodes (worker-node-1, worker-node-2) against resource requests/limits, taints/tolerations, affinity rules, etc., and binds the Pod to one of them. 

So the control plane's job is entirely about ***deciding*** and ***recording***, it never actually runs containers.

#### **What happens on whichever worker node gets picked**

Once the scheduler assigns a Pod to worker-node-1 or worker-node-2:

1. **`kubelet`** on that node watches the API server for Pods bound to it, then pulls the image (your Docker Hub image) via the container runtime (containerd, inside kind's node image) and starts the container(s).

2. **`kube-proxy`** on that node programs iptables/IPVS rules so that traffic sent to a Service's ClusterIP gets forwarded to the right Pod IP, on whichever node it's actually running.

3. **`CNI plugin`** (kindnet by default in kind) assigns the Pod an IP from your podSubnet: 10.244.0.0/16 and wires up networking so Pods can reach each other across nodes.

`Object choice per tier:`
# Object Choice per Tier — DSO202 Assignment 01

| Tier | Object | Why |
|---|---|---|
| **Frontend (static HTML/CSS/JS)** | `Deployment` | Stateless, interchangeable replicas. If a Pod dies, any new Pod is identical — the RollingUpdate strategy lets you push new images with zero downtime. Exposed via a `Service` (ClusterIP or the NodePort already wired to host port 30080). |
| **Express backend (API)** | `Deployment` | Also stateless as long as it doesn't hold local session state — each replica handles any request identically, talking to Postgres for persistence. Exposed internally via a `ClusterIP` `Service` so the frontend/browser or an Ingress can reach it; scaling out is just increasing `replicas`. |
| **PostgreSQL** | `StatefulSet` (not Deployment) | Postgres has identity and disk state that must survive restarts and be stable across rescheduling: a StatefulSet gives it a stable network identity (`postgres-0`), a stable stored volume via `volumeClaimTemplates` (backed by a `PersistentVolumeClaim` → `PersistentVolume`), and ordered, predictable Pod creation/termination. A Deployment's Pods are fungible and its ephemeral storage model is wrong for a database — you'd lose data on rescheduling without a StatefulSet + PVC. Exposed via a **headless** `Service` (`clusterIP: None`) so the backend can address it by stable DNS name rather than a load-balanced VIP. |

## Supporting objects (independent of the three Pods)

| Object | Purpose |
|---|---|
| **Secret** | Postgres credentials (`POSTGRES_PASSWORD`, etc.) and the connection string the backend uses. Never plain env vars in a ConfigMap for these. |
| **ConfigMap** | Non-secret config, e.g. `DB_HOST`, `DB_PORT`, feature flags for the backend/frontend. |
| **PersistentVolumeClaim** | Provisioned via the StatefulSet's `volumeClaimTemplates`; backed in kind by the default `standard` StorageClass, which provisions `hostPath`-based volumes on the node. |















