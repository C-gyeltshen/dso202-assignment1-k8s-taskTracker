# Assignment 1 — Namespace Loss Explanation

## What Happened

After finishing Assignment 1, Docker Engine was stopped. When Docker stopped, all `kind` cluster containers (which are Docker containers) stopped too. On the next Docker start, the containers resumed but the `etcd` state inside the cluster — which stores every Kubernetes object (namespaces, pods, services, configmaps, secrets, PVCs) — was lost.

**The cluster itself survived. Every Kubernetes object inside it did not.**

This is expected behavior for a local `kind` cluster. `kind` is designed for development and testing, not for persisting state across Docker restarts.

---

## Evidence

### The cluster is still present and healthy (18 days old)

```bash
kind get clusters
```
```
dso202
```

```bash
kubectl get nodes
```
```
NAME            STATUS   ROLES           AGE   VERSION
control-plane   Ready    control-plane   18d   v1.36.1
worker-node-1   Ready    <none>          18d   v1.36.1
worker-node-2   Ready    <none>          18d   v1.36.1
```

### The namespace and all resources are gone

```bash
kubectl get namespaces
```
```
NAME                 STATUS   AGE
default              Active   18d
kube-node-lease      Active   18d
kube-public          Active   18d
kube-system          Active   18d
local-path-storage   Active   18d
```

`dso202-assignment-01` is absent. All objects that lived inside it (Deployments, Services, ConfigMap, Secret, PVC) are gone with it.

```bash
kubectl get all -n dso202-assignment-01
```
```
No resources found in dso202-assignment-01 namespace.
```

---

## Why the YAML Files Are Still Safe

All manifests were version-controlled in this repository. The loss only affected the **live cluster state**, not the source files on disk. Every object can be re-applied from the existing YAML files at any time — which is exactly the point of the declarative, file-based approach used throughout Assignment 1.
