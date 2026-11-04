# Namespaces, Resource Requests and Limits, QoS, ResourceQuota and LimitRange

[← Kubernetes from scratch](../README.md)

> **In short:** A **namespace** is a virtual folder that splits one cluster between apps or teams. **Requests** tell the scheduler how much CPU and memory a container needs, **limits** are the most it may use (too much memory → **OOMKilled**). A **ResourceQuota** caps a whole namespace, and a **LimitRange** gives containers default values.

## About this topic

| Part | Topic | What you learn |
|---|---|---|
| **Part 1: Concepts** | Namespaces, requests and limits, QoS, ResourceQuota, LimitRange | How to split a cluster and how much CPU / memory each app gets |
| **Part 2: Lab** | Namespace on EKS | Requests and limits, an OOMKilled Pod, a quota and default limits |

## Table of Contents

**Part 1: Concepts**

1. [Namespaces](#1-namespaces)
2. [Requests and limits](#2-requests-and-limits)
3. [QoS classes: who gets evicted first](#3-qos-classes-who-gets-evicted-first)
4. [ResourceQuota: limits for a whole namespace](#4-resourcequota-limits-for-a-whole-namespace)
5. [LimitRange: default resources per container](#5-limitrange-default-resources-per-container)

**Part 2: Lab**

6. [Lab: namespace, resources, OOMKilled, quota](#6-lab-namespace-resources-oomkilled-quota)

**Wrap-up**

7. [What I learned](#7-what-i-learned)
8. [Commands](#8-commands)
9. [Interview Questions](#9-interview-questions)

---

# Part 1: Concepts

## 1. Namespaces

**Definition:** a **namespace** is a virtual folder inside one cluster. It groups related objects (Pods, Services, Deployments) and keeps their names separate, so two teams can each have a Service called `frontend` in their own namespace.

Every cluster starts with these:

| Namespace | What's in it |
|---|---|
| `default` | Where our objects go if we don't say a namespace |
| `kube-system` | Kubernetes' own Pods: CoreDNS, kube-proxy, aws-node (VPC CNI) |
| `kube-public` | Public info readable by everyone (rarely used) |
| `kube-node-lease` | Node heartbeats, so the control plane knows each node is alive |

**Why we use them:**

| Use | Example |
|---|---|
| One namespace per app | `roboshop`, `argocd`, `monitoring` |
| One namespace per environment (small clusters) | `roboshop-dev`, `roboshop-qa` |
| Limits per team | A **ResourceQuota** caps CPU / memory per namespace ([section 4](#4-resourcequota-limits-for-a-whole-namespace)) |
| Access per team | **RBAC** gives a team rights only in its namespace ([RBAC](../15-rbac-and-service-accounts/README.md)) |

**Namespaced vs cluster-wide objects:**

| Namespaced (live inside one namespace) | Cluster-wide (belong to the whole cluster) |
|---|---|
| Pod, Deployment, Service, ConfigMap, Secret, PVC, Role, Ingress | Node, Namespace, PersistentVolume, StorageClass, ClusterRole |

**Talking across namespaces:** a Pod reaches a Service in another namespace with `<service>.<namespace>`, for example `mongodb.roboshop` (full name: `mongodb.roboshop.svc.cluster.local`).

> ⚠️ A namespace is **not** a network wall. By default a Pod in one namespace can still talk to Pods in every other namespace. To block it, use a [Network Policy](../09-dns-and-network-policy/README.md).

> **Interview one-liner:** A namespace is a virtual folder in the cluster that groups objects, keeps names separate and lets us set quotas and RBAC per team or app; it doesn't block network traffic by itself.

---

## 2. Requests and limits

**Definition:** **requests** are the CPU and memory a container is **guaranteed**: the scheduler only puts the Pod on a node that has this much free. **Limits** are the **most** the container may use.

```yaml
      containers:
        - name: cart
          image: nginx
          resources:
            requests:
              cpu: "250m"        # 0.25 of a CPU core
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

| Unit | Meaning |
|---|---|
| `1` CPU / `1000m` | One full CPU core (one vCPU on EC2) |
| `250m` | 250 millicores = a quarter of a core |
| `128Mi`, `1Gi` | Memory in mebibytes / gibibytes |

**What happens when a container uses too much:**

| Resource | Goes over the limit → |
|---|---|
| **CPU** | **Throttled** (slowed down). It keeps running. |
| **Memory** | **Killed** with `OOMKilled` (exit code **137**) and restarted |

| Who uses what | |
|---|---|
| **Scheduler** | Looks only at **requests** to pick a node. Sum of requests on a node can't go over the node's size. |
| **Kubelet / Linux** | Enforces **limits** while the container runs |
| **HPA** | Measures CPU % **of the request**, so HPA doesn't work without requests ([Autoscaling](../14-autoscaling/README.md)) |

> **Real projects:** always set **requests** (the scheduler and HPA need them). Set a **memory limit** (often equal to the request) so a leaking app can't eat the node. Many teams leave out the CPU limit to avoid throttling.

> **Interview one-liner:** Requests are what the scheduler reserves for a container; limits are the maximum it may use. Over the CPU limit it's throttled, over the memory limit it's OOMKilled with exit code 137.

---

## 3. QoS classes: who gets evicted first

**Definition:** Kubernetes gives each Pod a **Quality of Service (QoS) class** based on its requests and limits. When a node runs out of memory, the kubelet **evicts** (removes) Pods in QoS order.

| QoS class | When | Evicted |
|---|---|---|
| **Guaranteed** | Every container has CPU **and** memory requests = limits | **Last** |
| **Burstable** | At least one request or limit set, but not Guaranteed | Middle |
| **BestEffort** | No requests or limits at all | **First** |

```
 Node memory full ──► evict BestEffort ──► then Burstable (using more than requested) ──► Guaranteed last
```

> **Interview one-liner:** QoS classes decide eviction order under memory pressure: BestEffort (no resources set) goes first, then Burstable, and Guaranteed (requests equal limits) goes last.

---

## 4. ResourceQuota: limits for a whole namespace

**Definition:** a **ResourceQuota** caps the **total** resources and object counts in one namespace, for example "this namespace may use at most 4 CPUs and 10 Pods".

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: roboshop-quota
  namespace: roboshop
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
```

| Behaviour | Result |
|---|---|
| A new Pod would go over the quota | Rejected: `exceeded quota` (a Deployment shows fewer Pods than replicas) |
| Quota sets `requests.cpu` / `limits.memory`… and the Pod has no resources | Rejected: `must specify limits / requests`, unless a **LimitRange** adds defaults |

---

## 5. LimitRange: default resources per container

**Definition:** a **LimitRange** sets **default** requests and limits for containers in a namespace (when the YAML has none) and can set a min / max per container.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-resources
  namespace: roboshop
spec:
  limits:
    - type: Container
      defaultRequest:      # used when a container has no requests
        cpu: "100m"
        memory: "128Mi"
      default:             # used when a container has no limits
        cpu: "500m"
        memory: "256Mi"
```

| | ResourceQuota | LimitRange |
|---|---|---|
| Controls | **Total** for the whole namespace | **Each** container / Pod |
| Example | Max 4 CPUs in `roboshop` | Every container gets 100m CPU if it says nothing |

> **Interview one-liner:** A ResourceQuota caps the total CPU, memory and object count of a namespace; a LimitRange gives each container default requests and limits and enforces a min and max per container.

---

# Part 2: Lab

## 6. Lab: namespace, resources, OOMKilled, quota

**Goal:** create a namespace, run Pods with requests and limits, see an OOMKilled container, then add a quota and a LimitRange.

**Before you start:**

| Need | Why |
|---|---|
| EKS cluster + bastion with `kubectl` ([EKS with Terraform](../03-eks-cluster-with-terraform/README.md)) | Where we run all commands |

### Step 1: Create a namespace

```bash
kubectl create namespace roboshop
```

**What it does:** creates an empty namespace called `roboshop`.

```bash
kubectl get namespaces
```

**Expected:** `roboshop` is listed with `default`, `kube-system`, `kube-public` and `kube-node-lease`.

### Step 2: Make it the default for kubectl

```bash
kubectl config set-context --current --namespace=roboshop
```

**What it does:** every next `kubectl` command uses `roboshop` without typing `-n roboshop`.

### Step 3: A Pod with requests and limits

```bash
vim web.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
    - name: nginx
      image: nginx
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "64Mi"
```

```bash
kubectl apply -f web.yaml
```

```bash
kubectl get pod web -o jsonpath='{.status.qosClass}{"\n"}'
```

**Expected:** `Guaranteed`, because requests = limits for both CPU and memory.

### Step 4: Go over the memory limit (OOMKilled)

```bash
vim memory-hog.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-hog
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]   # tries to use 150 MB
      resources:
        requests:
          memory: "50Mi"
        limits:
          memory: "100Mi"                                         # but may use only 100 Mi
```

```bash
kubectl apply -f memory-hog.yaml
```

```bash
kubectl get pod memory-hog -w
```

**Expected:** the status goes `OOMKilled` → `CrashLoopBackOff` and `RESTARTS` goes up. Press `Ctrl+C` to stop watching.

```bash
kubectl describe pod memory-hog | grep -A3 "Last State"
```

**Expected:** `Reason: OOMKilled` and `Exit Code: 137`.

### Step 5: Add a ResourceQuota

```bash
vim quota.yaml
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: roboshop-quota
spec:
  hard:
    pods: "3"
    requests.memory: 512Mi
    limits.memory: 1Gi
```

```bash
kubectl apply -f quota.yaml
```

```bash
kubectl run no-resources --image=nginx
```

**Expected:** `Error from server (Forbidden): ... must specify limits.memory for: no-resources; requests.memory for: no-resources`. The quota tracks memory, so every new Pod must say how much it needs.

```bash
kubectl describe resourcequota roboshop-quota
```

**Expected:** a `Used` / `Hard` table, for example `pods 2 / 3`.

### Step 6: Add a LimitRange so Pods get defaults

```bash
vim limitrange.yaml
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-resources
spec:
  limits:
    - type: Container
      defaultRequest:
        memory: "64Mi"
      default:
        memory: "128Mi"
```

```bash
kubectl apply -f limitrange.yaml
```

```bash
kubectl run no-resources --image=nginx
```

**Expected:** `pod/no-resources created`. The LimitRange filled in the memory request and limit.

```bash
kubectl get pod no-resources -o jsonpath='{.spec.containers[0].resources}{"\n"}'
```

**Expected:** `{"limits":{"memory":"128Mi"},"requests":{"memory":"64Mi"}}`

### Step 7: Clean up

```bash
kubectl config set-context --current --namespace=default
```

```bash
kubectl delete namespace roboshop
```

**What it does:** deleting a namespace deletes **everything inside it** (Pods, quota, LimitRange).

| Problem | Cause | Fix |
|---|---|---|
| Pod `Pending`, Events: `Insufficient cpu` | Requests are bigger than the free space on every node | Lower the requests or add nodes |
| `exceeded quota` | The namespace is at its quota | Raise the quota or delete unused Pods |
| `must specify limits.memory` | Quota on memory, no resources in YAML | Add resources or a LimitRange |

---

# Wrap-up

## 7. What I learned

- **Namespace** = virtual folder: separate names, quotas and RBAC per app or team. Not a network wall.
- **Requests** = reserved (scheduler, HPA). **Limits** = maximum. Over CPU → throttled, over memory → **OOMKilled (137)**.
- **QoS:** BestEffort evicted first, Guaranteed last.
- **ResourceQuota** = total per namespace. **LimitRange** = defaults and min / max per container.

```
 Namespace ──► ResourceQuota (total)  ──► LimitRange (defaults per container) ──► Pod requests / limits
```

## 8. Commands

| Command | Use |
|---|---|
| `kubectl create namespace <ns>` | Create a namespace |
| `kubectl get all -n <ns>` | Everything in a namespace |
| `kubectl config set-context --current --namespace=<ns>` | Change kubectl's default namespace |
| `kubectl api-resources --namespaced=false` | List cluster-wide object types |
| `kubectl top pods` / `kubectl top nodes` | Live CPU / memory use (needs Metrics Server) |
| `kubectl describe node <node>` | `Allocated resources`: requests and limits on the node |
| `kubectl describe resourcequota -n <ns>` | Quota used vs hard |
| `kubectl delete namespace <ns>` | Delete the namespace and everything in it |

## 9. Interview Questions

### Other Questions

**Q1. What is a namespace and why use it?**
A virtual folder in one cluster that groups objects and keeps names separate. We use it to split apps, teams or environments and to set ResourceQuotas and RBAC per namespace.

**Q2. Does a namespace isolate network traffic?**
No. Pods in different namespaces can talk to each other by default (`<service>.<namespace>`). A Network Policy is needed to block it.

**Q3. What's the difference between requests and limits?**
Requests are reserved for the container and used by the scheduler to pick a node. Limits are the maximum the container may use while running.

**Q4. What happens when a container goes over its CPU limit and its memory limit?**
CPU: it's throttled and keeps running. Memory: it's killed with OOMKilled (exit 137) and restarted.

**Q5. A Pod is Pending with `Insufficient memory`. What do you check?**
`kubectl describe node` → Allocated resources. The sum of requests on every node is too high for the new Pod; lower the request, remove unused Pods or add nodes.

**Q6. What are the QoS classes?**
Guaranteed (requests = limits for CPU and memory), Burstable (some resources set) and BestEffort (none). Under memory pressure BestEffort Pods are evicted first and Guaranteed last.

**Q7. ResourceQuota vs LimitRange?**
ResourceQuota caps the total for a namespace. LimitRange sets default and min / max resources for each container in the namespace.

**Q8. Why does HPA need requests?**
HPA measures CPU use as a percentage of the request. With no request there's nothing to compare against, so HPA shows `<unknown>`.

---

**Source:** [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) · [Resource management for Pods and containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) · [Pod QoS classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/) · [Resource quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) · [Limit ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
