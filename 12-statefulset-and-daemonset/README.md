# StatefulSet and DaemonSet: Stateful Apps and One Pod per Node

[← Kubernetes from scratch](../README.md)

> **In short:** A **Deployment** runs identical, replaceable Pods for stateless apps. A **StatefulSet** is for apps with data, like databases: each Pod gets a **fixed name** (`mysql-0`), its **own disk** and its **own DNS name**. A **DaemonSet** runs **one Pod on every node**, for node agents like the CNI, log shippers and Falco.

## About this topic

| Part | Topic | What you learn |
|---|---|---|
| **Part 1: Concepts** | Stateless vs stateful, StatefulSet, headless Service, DaemonSet | When to use each workload type |
| **Part 2: Lab** | StatefulSet with EBS + DaemonSet on EKS | Fixed names, per-Pod disks and DNS, one Pod per node |

## Table of Contents

**Part 1: Concepts**

1. [Stateless vs stateful apps](#1-stateless-vs-stateful-apps)
   - [RoboShop: which component uses which workload](#roboshop-which-component-uses-which-workload)
2. [StatefulSet](#2-statefulset)
   - [2.1 How each Pod gets its own volume automatically](#21-how-each-pod-gets-its-own-volume-automatically)
   - [2.2 Why the data is safe: what survives what](#22-why-the-data-is-safe-what-survives-what)
3. [DaemonSet](#3-daemonset)
4. [Which workload to use](#4-which-workload-to-use)

**Part 2: Lab**

5. [Lab: StatefulSet with its own disks, and a DaemonSet](#5-lab-statefulset-with-its-own-disks-and-a-daemonset)

**Wrap-up**

6. [What I learned](#6-what-i-learned)
7. [Commands](#7-commands)
8. [Interview Questions](#8-interview-questions)

---

# Part 1: Concepts

## 1. Stateless vs stateful apps

**Definition:** a **stateless** app keeps no data of its own (any copy can answer any request). A **stateful** app keeps data that must not be lost or mixed up (a database). Kubernetes runs stateless apps with a **Deployment** and stateful apps with a **StatefulSet**.

| | Stateless | Stateful |
|---|---|---|
| RoboShop examples | `frontend`, `catalogue`, `cart`, `user` | `mongodb`, `mysql`, `redis`, `rabbitmq` |
| Can any copy be replaced by any other? | ✅ Yes | ❌ No: each copy has its own data and role |
| Workload | **Deployment** | **StatefulSet** |

**The simple test:** "If I delete this Pod and a fresh copy starts, is anything lost?"
- **No** → stateless → **Deployment**
- **Yes** → stateful → **StatefulSet** with a PVC

### RoboShop: which component uses which workload

Every RoboShop **microservice** is stateless, because its data lives in a database. Only the **4 data stores** are stateful.

| RoboShop component | Tech | Stateless or stateful | Workload | Why |
|---|---|---|---|---|
| **frontend** | Nginx | Stateless | **Deployment** | Serves the UI and forwards `/api/*`; keeps no data |
| **catalogue** | Node.js | Stateless | **Deployment** | Reads products from MongoDB |
| **user** | Node.js | Stateless | **Deployment** | Users live in MongoDB, sessions in Redis |
| **cart** | Node.js | Stateless | **Deployment** | Cart items live in Redis, not in the Pod |
| **shipping** | Java | Stateless | **Deployment** | Cities and addresses live in MySQL |
| **payment** | Python | Stateless | **Deployment** | Sends orders to RabbitMQ |
| **dispatch** | Go | Stateless | **Deployment** | Reads orders from RabbitMQ |
| **mongodb** | Database | **Stateful** | **StatefulSet** | Stores products and users on disk |
| **mysql** | Database | **Stateful** | **StatefulSet** | Stores shipping data on disk |
| **redis** | Cache / store | **Stateful** | **StatefulSet** | Holds cart and session data |
| **rabbitmq** | Message queue | **Stateful** | **StatefulSet** | Holds queued messages until dispatch reads them |

```
 Stateless (Deployment, many replicas, any Pod can answer):
   frontend · catalogue · user · cart · shipping · payment · dispatch

 Stateful (StatefulSet + PVC / EBS, the data lives here):
   mongodb ◄── catalogue, user        redis    ◄── cart, user
   mysql   ◄── shipping               rabbitmq ◄── payment ──► dispatch
```

**Example:** delete a `cart` Pod and nothing is lost: the cart items are in Redis, so the new `cart` Pod works straight away. Delete `mongodb` without its disk and every product and user is gone. That's why only the data stores need a StatefulSet.

> 💡 Loading the MongoDB catalogue data or the MySQL schema is a **one-time task**, so it fits a **Job** ([Workloads](../02-pods-deployments-and-jobs/README.md)), not a Deployment.

**In real projects on AWS** the 4 stateful parts usually move out of the cluster to **managed services**, and EKS runs only the stateless Deployments:

| In-cluster StatefulSet | Managed AWS service |
|---|---|
| mongodb | Amazon DocumentDB |
| mysql | Amazon RDS for MySQL |
| redis | Amazon ElastiCache |
| rabbitmq | Amazon MQ |

> **Interview one-liner:** In RoboShop all 7 microservices are stateless Deployments because their data lives in the databases; MongoDB, MySQL, Redis and RabbitMQ are stateful, so they run as StatefulSets with PVCs, or as managed AWS services in production.

---

## 2. StatefulSet

**Definition:** a **StatefulSet** runs Pods that each have a **fixed name**, their **own disk** and a **fixed DNS name**, and starts and stops them **in order**. If `mysql-1` dies, the new Pod is again called `mysql-1` and gets back the same disk.

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random: `web-7d9f8c-x2kq` | Fixed, numbered: `mysql-0`, `mysql-1`, `mysql-2` |
| Start / stop order | All at once | **In order**: 0 → 1 → 2 up, 2 → 1 → 0 down |
| Storage | All Pods share one PVC (or none) | **One PVC per Pod** (`volumeClaimTemplates`) |
| DNS | One name for the Service | **One name per Pod**: `mysql-0.mysql` |
| Service | Normal ClusterIP | **Headless** Service (`clusterIP: None`) |

**Headless Service:** a Service with `clusterIP: None`. It has no single virtual IP; instead CoreDNS gives every Pod its own DNS name, so an app can talk to one exact Pod (for example "always write to `mysql-0`, the primary").

```
 Headless Service "mysql" ──► mysql-0.mysql ──► PVC data-mysql-0 ──► EBS vol A
                          ├─► mysql-1.mysql ──► PVC data-mysql-1 ──► EBS vol B
                          └─► mysql-2.mysql ──► PVC data-mysql-2 ──► EBS vol C
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql              # the headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:           # one PVC per Pod: data-mysql-0, data-mysql-1…
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 10Gi
```

> ⚠️ Deleting or scaling down a StatefulSet does **not** delete its PVCs. That protects the data; delete them by hand when you really want the data gone.

**What real projects do:** on AWS, most teams run databases as **managed services** (RDS for MySQL, DocumentDB for MongoDB, ElastiCache for Redis, Amazon MQ for RabbitMQ) and keep EKS for stateless apps. StatefulSets are used when a team runs its own Kafka, Elasticsearch, MongoDB or similar inside the cluster.

> **Interview one-liner:** A StatefulSet gives each Pod a fixed name, its own PVC and its own DNS name through a headless Service, and starts and stops Pods in order; we use it for databases and clustered apps, while Deployments are for stateless apps.

### 2.1 How each Pod gets its own volume automatically

**Definition:** we never create a PV or a PVC for each Pod by hand. We write **one `volumeClaimTemplates`** in the StatefulSet. The **StatefulSet** creates **one PVC per Pod** from it, and **dynamic provisioning** (the StorageClass + EBS CSI driver) creates **one EBS volume (PV) per PVC**. Each volume is then mounted only into its own Pod.

```
 1. We write:      StatefulSet mysql, replicas: 3, ONE volumeClaimTemplate (name: data, 10Gi, gp3)
 2. StatefulSet:   creates Pods mysql-0, mysql-1, mysql-2
                   + one PVC each: data-mysql-0, data-mysql-1, data-mysql-2
 3. Dynamic provisioning: StorageClass gp3 sees each PVC ──► creates one EBS volume (PV) per PVC
 4. Bind:          data-mysql-0 ⇄ EBS vol A   data-mysql-1 ⇄ EBS vol B   data-mysql-2 ⇄ EBS vol C
 5. Mount:         vol A is attached to mysql-0's node and mounted at /var/lib/mysql inside mysql-0
```

| Step | Who does it |
|---|---|
| Write the template (size, StorageClass, access mode) | **We**, once in the YAML |
| Create a PVC for each Pod (`<template>-<pod>`) | **StatefulSet** |
| Create an EBS volume (PV) for each PVC | **Dynamic provisioning**: StorageClass + EBS CSI driver ([Storage](../11-storage-volumes-pv-pvc/README.md)) |
| Attach the volume to the node and mount it into the Pod | **Kubelet + EBS CSI driver**, at the `mountPath` we set |

> 💡 **Dynamic provisioning is not a StatefulSet.** Dynamic provisioning is a **storage** feature: it creates a disk whenever **any** PVC asks (a plain Pod or Deployment too). A StatefulSet is a **workload** that creates **one PVC per Pod**; those PVCs are then filled by dynamic provisioning.

### 2.2 Why the data is safe: what survives what

The data lives on the **EBS volume**, not inside the container, the Pod or the node. So when any of those die, the volume is simply **attached again** to the new Pod.

| What dies / is deleted | What happens to the data |
|---|---|
| **Container** crashes and restarts | ✅ Safe. Same Pod, same volume, still mounted |
| **Pod** is deleted (`mysql-1`) | ✅ Safe. The new Pod is again called `mysql-1`, gets the **same PVC** `data-mysql-1` and the **same EBS volume** |
| **Node** dies | ✅ Safe. The volume is detached; `mysql-1` is recreated on another node **in the same AZ** and the volume is attached there |
| **StatefulSet** deleted or scaled down | ✅ Safe. PVCs and volumes are **kept**; scaling up or recreating reattaches them |
| **PVC** deleted (reclaim policy `Delete`) | ❌ **Lost**. The PV and the EBS volume are deleted too |

```
 mysql-1 Pod dies ──► new Pod "mysql-1" (same name) ──► same PVC data-mysql-1 ──► same EBS vol B ──► old data is back ✅
```

> ⚠️ An EBS volume lives in **one AZ**. If no node with free space is left in that AZ, the new Pod stays `Pending` until one is. For extra safety, take **EBS snapshots** (backups), and use `reclaimPolicy: Retain` for production data so deleting a PVC doesn't delete the disk.

> **Interview one-liner:** In a StatefulSet we write one volumeClaimTemplates; it creates a PVC per Pod, dynamic provisioning creates an EBS volume per PVC, and each volume is mounted only into its own Pod. The data lives on the EBS volume, so it survives container, Pod and node failures: the recreated Pod gets the same name, the same PVC and the same volume back.

---

## 3. DaemonSet

**Definition:** a **DaemonSet** runs **one copy of a Pod on every node** (or on every node that matches a selector). When a new node joins, the DaemonSet adds a Pod to it; when a node leaves, that Pod goes too.

**Where we've already seen DaemonSets:**

| DaemonSet | Namespace | Job on each node |
|---|---|---|
| `aws-node` | `kube-system` | VPC CNI: gives Pods their IPs ([CNI](../09-dns-and-network-policy/README.md)) |
| `kube-proxy` | `kube-system` | Service networking rules |
| `ebs-csi-node` | `kube-system` | Mounts EBS volumes ([Storage](../11-storage-volumes-pv-pvc/README.md)) |
| `falco` | `falco` | Runtime security alerts ([Falco](../17-falco-runtime-security/README.md)) |
| Fluent Bit, node-exporter | `logging`, `monitoring` | Ship node logs / node metrics |

**Running on tainted nodes:** a node agent must run on **every** node, including tainted ones (like a dedicated AI node), so DaemonSets usually carry a toleration that tolerates everything:

```yaml
      tolerations:
        - operator: Exists        # tolerate every taint
```

To run on **some** nodes only, add a `nodeSelector` ([Scheduling](../13-scheduling-taints-and-affinity/README.md)).

> **Interview one-liner:** A DaemonSet runs one Pod on every node and adds it automatically to new nodes; we use it for node agents like the CNI, kube-proxy, log shippers, monitoring and security tools.

---

## 4. Which workload to use

| Workload | Runs | Use for |
|---|---|---|
| **Deployment** | N identical, replaceable Pods | Web apps, APIs (most apps) |
| **StatefulSet** | N Pods with fixed names and own disks | Databases, Kafka, Elasticsearch |
| **DaemonSet** | 1 Pod per node | Node agents (logs, metrics, CNI, security) |
| **Job / CronJob** | Pods that finish | Migrations, backups ([Workloads](../02-pods-deployments-and-jobs/README.md)) |

---

# Part 2: Lab

## 5. Lab: StatefulSet with its own disks, and a DaemonSet

**Goal:** run a 2-Pod StatefulSet, see fixed names, per-Pod disks and per-Pod DNS, then run a DaemonSet and see one Pod per node.

**Before you start:**

| Need | Why |
|---|---|
| EKS cluster + bastion with `kubectl` ([EKS with Terraform](../03-eks-cluster-with-terraform/README.md)) | Where we run all commands |
| EBS CSI driver + `gp3` StorageClass ([Storage lab](../11-storage-volumes-pv-pvc/README.md#4-lab-data-that-survives-a-pod-restart-ebs-on-eks), steps 1–4) | Each StatefulSet Pod gets an EBS volume |

### Step 1: Headless Service + StatefulSet

```bash
vim web-sts.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  clusterIP: None              # headless
  selector:
    app: web
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 1Gi
```

```bash
kubectl apply -f web-sts.yaml
```

```bash
kubectl get pods -l app=web -w
```

**Expected:** `web-0` is created and becomes `Running` **first**, then `web-1`. Press `Ctrl+C`.

### Step 2: One PVC per Pod

```bash
kubectl get pvc
```

**Expected:** `www-web-0` and `www-web-1`, both `Bound`. Name = `<template name>-<pod name>`.

### Step 3: Each Pod keeps its own data

```bash
kubectl exec web-0 -- sh -c 'echo "I am web-0" > /usr/share/nginx/html/index.html'
```

```bash
kubectl exec web-1 -- sh -c 'echo "I am web-1" > /usr/share/nginx/html/index.html'
```

```bash
kubectl delete pod web-0
```

```bash
kubectl get pods -l app=web
```

**Expected:** a new Pod with the **same name** `web-0` is created.

```bash
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
```

**Expected:** `I am web-0`. The new Pod got the same PVC back.

### Step 4: Per-Pod DNS names

```bash
kubectl run dns-test --rm -it --image=busybox --restart=Never -- nslookup web-0.web
```

**Expected:** an answer with web-0's Pod IP. `web` alone returns **both** Pod IPs (headless Service, no single ClusterIP).

```bash
kubectl run web-test --rm -it --image=busybox --restart=Never -- wget -qO- http://web-1.web
```

**Expected:** `I am web-1`.

### Step 5: See the DaemonSets already running

```bash
kubectl get daemonsets -n kube-system
```

**Expected:** `aws-node`, `kube-proxy` (and `ebs-csi-node` if installed), each with `DESIRED` = number of nodes.

### Step 6: Create our own DaemonSet

```bash
vim node-agent.yaml
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      tolerations:
        - operator: Exists
      containers:
        - name: agent
          image: busybox
          command: ["sh", "-c", "while true; do echo agent on $NODE_NAME; sleep 60; done"]
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName      # the node this Pod runs on
```

```bash
kubectl apply -f node-agent.yaml
```

```bash
kubectl get pods -l app=node-agent -o wide
```

**Expected:** exactly **one Pod per node**; the `NODE` column shows each node once. Add a node to the node group ([Autoscaling lab](../14-autoscaling/README.md)) and a new `node-agent` Pod appears on it by itself.

### Step 7: Clean up

```bash
kubectl delete -f web-sts.yaml
```

```bash
kubectl delete pvc www-web-0 www-web-1
```

**What it does:** the StatefulSet left its PVCs behind on purpose; deleting them also deletes the EBS volumes.

```bash
kubectl delete -f node-agent.yaml
```

| Problem | Cause | Fix |
|---|---|---|
| `web-1` never starts | `web-0` isn't Ready (ordered start) | Fix `web-0` first: `kubectl describe pod web-0` |
| StatefulSet Pod `Pending`, PVC `Pending` | No StorageClass / EBS CSI driver | Do steps 1–4 of the Storage lab |
| DaemonSet `DESIRED` < number of nodes | Nodes are tainted and the Pod has no toleration | Add `tolerations: - operator: Exists` |

---

# Wrap-up

## 6. What I learned

- **Automatic volumes:** one `volumeClaimTemplates` → StatefulSet makes **one PVC per Pod** → dynamic provisioning makes **one EBS volume per PVC** → mounted into its own Pod. Data survives container, Pod and node failures; only deleting the PVC deletes it.
- **Stateless** → Deployment. **Stateful** → StatefulSet: fixed names (`web-0`), **one PVC per Pod**, ordered start / stop, per-Pod DNS via a **headless Service**. PVCs stay after delete.
- On AWS, databases usually run on **RDS / DocumentDB / ElastiCache**, not in StatefulSets.
- **DaemonSet** = one Pod per node, added to new nodes automatically: CNI, kube-proxy, log and metric agents, Falco. Usually tolerates all taints.

```
 Deployment  = many identical Pods        StatefulSet = numbered Pods + own disks
 DaemonSet   = one Pod on every node      Job/CronJob = Pods that finish
```

## 7. Commands

| Command | Use |
|---|---|
| `kubectl get statefulsets` / `kubectl get sts` | List StatefulSets |
| `kubectl scale sts <name> --replicas=3` | Scale (adds `<name>-2`) |
| `kubectl get daemonsets -A` / `kubectl get ds -A` | All DaemonSets in all namespaces |
| `kubectl rollout status sts/<name>` / `ds/<name>` | Watch a rolling update |
| `kubectl get pvc` | PVCs left by StatefulSets |

## 8. Interview Questions

### Other Questions

**Q1. Deployment vs StatefulSet?**
A Deployment runs identical, replaceable Pods with random names and shared or no storage. A StatefulSet gives each Pod a fixed name, its own PVC and DNS name, and starts / stops them in order.

**Q2. What is a headless Service and why does a StatefulSet need one?**
A Service with `clusterIP: None`. Instead of one virtual IP, DNS returns each Pod's IP and gives every Pod its own name (`mysql-0.mysql`), so clients can reach a specific Pod such as the primary database.

**Q3. What happens to PVCs when you delete or scale down a StatefulSet?**
They're kept, so the data isn't lost. Scaling back up reattaches them. Delete them by hand to remove the data.

**Q4. Would you run a database in Kubernetes?**
On AWS I'd normally use a managed service like RDS, which handles backups, failover and patching. If we must run it in the cluster, it goes in a StatefulSet with EBS-backed PVCs and a backup plan.

**Q5. What is a DaemonSet? Give examples.**
It runs one Pod on every (or every selected) node and adds one to new nodes automatically. Examples: aws-node (VPC CNI), kube-proxy, Fluent Bit, node-exporter, Falco.

**Q6. Why isn't a DaemonSet Pod running on one of the nodes?**
That node has a taint the Pod doesn't tolerate, or the DaemonSet has a nodeSelector that the node doesn't match, or the node lacks resources. Check `kubectl describe node` and the DaemonSet's tolerations.

**Q7. How does each StatefulSet Pod get its own persistent volume?**
We write one `volumeClaimTemplates`. The StatefulSet creates a PVC per Pod (`data-mysql-0`, `data-mysql-1`…), dynamic provisioning creates an EBS volume for each PVC, and each volume is mounted only into its own Pod.

**Q8. A StatefulSet Pod's node dies. Is the data lost?**
No. The data is on the EBS volume, not the node. The Pod is recreated with the same name on another node in the same AZ, gets the same PVC and the same volume attached, so the data is back. It's lost only if the PVC is deleted with reclaim policy Delete.

---

**Source:** [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) · [Headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) · [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
