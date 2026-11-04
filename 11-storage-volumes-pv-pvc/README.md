# Kubernetes Storage: Volumes, PersistentVolume, PersistentVolumeClaim, StorageClass and EBS on EKS

[← Kubernetes from scratch](../README.md)

> **In short:** A container loses its files when it restarts. **Volumes** keep data outside the container. For data that must survive (databases), a Pod asks for storage with a **PVC**, a **StorageClass** creates the disk (**PV**) automatically, and on EKS the **EBS CSI driver** makes it a real **EBS volume** in the same AZ as the node.

## About this topic

| Part | Topic | What you learn |
|---|---|---|
| **Part 1: Concepts** | Volume types, PV, PVC, StorageClass, access modes, EBS CSI driver | How Pods get disks that outlive them, EBS vs EFS |
| **Part 2: Lab** | EBS volume on EKS | Install the driver, create a PVC, write data, delete the Pod, data is still there |

## Table of Contents

**Part 1: Concepts**

1. [Why containers need volumes](#1-why-containers-need-volumes)
2. [PV, PVC and StorageClass](#2-pv-pvc-and-storageclass)
3. [Storage on EKS: the EBS CSI driver](#3-storage-on-eks-the-ebs-csi-driver)

**Part 2: Lab**

4. [Lab: data that survives a Pod restart (EBS on EKS)](#4-lab-data-that-survives-a-pod-restart-ebs-on-eks)

**Wrap-up**

5. [What I learned](#5-what-i-learned)
6. [Commands](#6-commands)
7. [Interview Questions](#7-interview-questions)

---

# Part 1: Concepts

## 1. Why containers need volumes

**Definition:** a **volume** is a folder that a container can read and write which lives **outside** the container's own filesystem. A container's filesystem is wiped every time the container restarts, so anything that must survive (database files, uploads) goes on a volume.

```
 Container restarts ──► its own files are gone ❌
 Data on a volume   ──► still there ✅
```

| Volume type | Lives as long as | Used for |
|---|---|---|
| `emptyDir` | The **Pod** | Sharing files between containers in one Pod (app + log sidecar) |
| `configMap` / `secret` | The ConfigMap / Secret | Config files ([ConfigMaps and Secrets](../06-configmaps-and-secrets/README.md)) |
| `hostPath` | The **node** | Agents that read node files (log collectors). Avoid for apps: the data is stuck on one node |
| `persistentVolumeClaim` | **Until we delete it** (Pod and node can die) | Databases and anything that must not be lost |

> **Interview one-liner:** Container files are lost on restart; emptyDir lives as long as the Pod, while a PersistentVolumeClaim keeps data even if the Pod or node is replaced.

---

## 2. PV, PVC and StorageClass

**Definitions:**
- **PersistentVolume (PV):** a real piece of storage in the cluster, for example one **EBS volume**. Cluster-wide.
- **PersistentVolumeClaim (PVC):** a Pod's **request** for storage: "I need 5 Gi that one node can write". Lives in a namespace.
- **StorageClass:** a **template** that says how to create a PV on demand (which provisioner, which disk type).

```
 Pod ──uses──► PVC (request: 5Gi, RWO) ──bound to──► PV ──is──► EBS volume (gp3)
                    ▲
                    └── StorageClass gp3 creates the PV + EBS volume automatically
```

| | Static provisioning | Dynamic provisioning ⭐ |
|---|---|---|
| Who creates the PV | An admin, by hand | The **StorageClass**, when a PVC asks |
| Real projects | Rare (old disks, NFS) | **Almost always** |

> 💡 **Dynamic provisioning is not a StatefulSet.** It's how a StorageClass creates a disk for **any** PVC (a Pod or Deployment too). A **StatefulSet** uses it to give **each Pod its own disk**: see [how each Pod gets its own volume](../12-statefulset-and-daemonset/README.md#21-how-each-pod-gets-its-own-volume-automatically).

**Access modes:**

| Mode | Meaning | AWS storage |
|---|---|---|
| `ReadWriteOnce` (RWO) | Read-write by Pods on **one node** | **EBS** |
| `ReadOnlyMany` (ROX) | Read-only by many nodes | EFS |
| `ReadWriteMany` (RWX) | Read-write by **many nodes** | **EFS** |
| `ReadWriteOncePod` (RWOP) | Read-write by **one Pod** only | EBS |

**Reclaim policy** (what happens to the disk when the PVC is deleted):

| Policy | Result |
|---|---|
| `Delete` (default for dynamic) | PV **and** the EBS volume are deleted |
| `Retain` | The EBS volume is kept; an admin cleans it up. Safer for production data |

> **Interview one-liner:** A PVC is a Pod's request for storage, a PV is the actual disk, and a StorageClass creates the PV automatically when a PVC asks for it (dynamic provisioning).

---

## 3. Storage on EKS: the EBS CSI driver

**Definition:** a **CSI driver** (Container Storage Interface) is the plugin that lets Kubernetes create and attach a cloud provider's disks. On EKS, the **Amazon EBS CSI driver** (an EKS add-on) creates EBS volumes and attaches them to the right node.

| Need | Why |
|---|---|
| **EBS CSI driver add-on** | Without it, PVCs stay `Pending` forever |
| **IAM permission** (`AmazonEBSCSIDriverPolicy`) | The driver calls the EC2 API to create and attach volumes. Real projects give it through **EKS Pod Identity** or **IRSA**; for a lab, attaching it to the node role is the quick way |
| **A StorageClass** | Newer EKS clusters don't mark any StorageClass as default, so we create a `gp3` one |

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
```

**Why `WaitForFirstConsumer`?** An EBS volume lives in **one Availability Zone** and can attach only to a node in that AZ. This setting waits until the Pod is scheduled, then creates the volume in **that node's AZ**. Without it the volume could land in `us-east-1a` while the Pod runs in `us-east-1b`, and the Pod would be stuck.

| EBS | EFS |
|---|---|
| One node at a time (RWO), one AZ | Many nodes at once (RWX), all AZs |
| Databases (MySQL, MongoDB) | Shared files (uploads, CMS content) |

> **Interview one-liner:** On EKS the EBS CSI driver creates EBS volumes for PVCs; we use a gp3 StorageClass with WaitForFirstConsumer so the volume is created in the same AZ as the Pod's node, and EFS when many Pods on different nodes must share files.

---

# Part 2: Lab

## 4. Lab: data that survives a Pod restart (EBS on EKS)

**Goal:** install the EBS CSI driver, create a gp3 StorageClass and a PVC, write a file, delete the Pod, and see that the data is still there.

**Before you start:**

| Need | Why |
|---|---|
| EKS cluster + bastion with `kubectl` and AWS CLI ([EKS with Terraform](../03-eks-cluster-with-terraform/README.md)) | Where we run all commands |

### Step 1: Find the node group's IAM role

```bash
aws eks list-nodegroups --cluster-name abhigna-cluster
```

**Expected:** the node group name, for example `abhigna-cluster-ng-1`.

```bash
aws eks describe-nodegroup --cluster-name abhigna-cluster --nodegroup-name abhigna-cluster-ng-1 --query 'nodegroup.nodeRole' --output text
```

**Expected:** a role ARN. The role name is the part after `role/`.

### Step 2: Give the nodes EBS permissions (lab shortcut)

```bash
aws iam attach-role-policy --role-name <node-role-name> --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

**What it does:** lets the EBS CSI driver (running on the nodes) create and attach EBS volumes. In real projects we give this policy only to the driver's service account with Pod Identity or IRSA, not to every node.

### Step 3: Install the EBS CSI driver add-on

```bash
aws eks create-addon --cluster-name abhigna-cluster --addon-name aws-ebs-csi-driver
```

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

**Expected:** after a minute, `ebs-csi-controller-…` Pods (2) and one `ebs-csi-node-…` Pod per node, all `Running`.

### Step 4: Create the gp3 StorageClass

```bash
vim gp3.yaml
```

Paste the StorageClass YAML from [section 3](#3-storage-on-eks-the-ebs-csi-driver).

```bash
kubectl apply -f gp3.yaml
```

```bash
kubectl get storageclass
```

**Expected:** `gp3 (default)` with provisioner `ebs.csi.aws.com`.

### Step 5: Create a PVC

```bash
vim data-pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: gp3
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f data-pvc.yaml
```

```bash
kubectl get pvc data-pvc
```

**Expected:** `STATUS: Pending`. That's correct: with `WaitForFirstConsumer` no disk is made until a Pod uses the PVC.

### Step 6: A Pod that writes to the volume

```bash
vim writer.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: writer
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

```bash
kubectl apply -f writer.yaml
```

```bash
kubectl get pvc data-pvc
```

**Expected:** `STATUS: Bound`, with a `VOLUME` name like `pvc-1a2b…`.

```bash
kubectl exec writer -- sh -c 'echo "order-1001 saved" > /data/orders.txt'
```

### Step 7: See the EBS volume in AWS

```bash
aws ec2 describe-volumes --filters Name=tag:kubernetes.io/created-for/pvc/name,Values=data-pvc --query 'Volumes[].{ID:VolumeId,AZ:AvailabilityZone,Size:Size,Type:VolumeType,State:State}' --output table
```

**Expected:** one `gp3` volume, size `1`, state `in-use`, in the same AZ as the node running `writer`.

### Step 8: Delete the Pod and check the data

```bash
kubectl delete pod writer
```

```bash
kubectl apply -f writer.yaml
```

```bash
kubectl exec writer -- cat /data/orders.txt
```

**Expected:** `order-1001 saved`. The Pod is new, but the EBS volume (and our file) stayed.

### Step 9: Clean up

```bash
kubectl delete pod writer
```

```bash
kubectl delete pvc data-pvc
```

**What it does:** the reclaim policy is `Delete`, so the PV and the **EBS volume** are deleted too. Run Step 7 again: the list is empty.

> ⚠️ Before `terraform destroy`, delete every PVC. EBS volumes made by Kubernetes aren't in Terraform's state, so they'd be left behind and keep costing money.

| Problem | Cause | Fix |
|---|---|---|
| PVC `Pending` **after** a Pod uses it | EBS CSI driver not installed or no IAM permission | Check `ebs-csi-controller` Pods and their logs |
| PVC `Pending`, no StorageClass | No default StorageClass | Set `storageClassName` or mark one as default |
| Pod `ContainerCreating`, `Multi-Attach error` | RWO EBS volume still attached to another node | Wait for the old Pod to stop; use EFS if many nodes need it |
| Pod `Pending`, `volume node affinity conflict` | EBS volume is in another AZ than every free node | Use `WaitForFirstConsumer`; keep a node in that AZ |

---

# Wrap-up

## 5. What I learned

- Container files are lost on restart. **emptyDir** lives with the Pod, a **PVC** keeps data after the Pod is gone.
- **PVC** = request, **PV** = the disk, **StorageClass** = creates PVs on demand (dynamic provisioning).
- On EKS: **EBS CSI driver** + IAM + a **gp3 StorageClass** with **WaitForFirstConsumer** (EBS is one AZ, RWO). **EFS** for RWX.
- Delete PVCs before destroying the cluster.

```
 Pod ──► PVC ──► StorageClass (gp3) ──► EBS CSI driver ──► EBS volume (same AZ as the node)
```

## 6. Commands

| Command | Use |
|---|---|
| `kubectl get storageclass` | StorageClasses and which one is default |
| `kubectl get pvc` / `kubectl get pv` | Claims and volumes, their status |
| `kubectl describe pvc <name>` | Events: why a PVC is Pending |
| `aws eks create-addon --cluster-name <c> --addon-name aws-ebs-csi-driver` | Install the EBS CSI driver |
| `aws ec2 describe-volumes --filters Name=tag:kubernetes.io/created-for/pvc/name,Values=<pvc>` | Find the EBS volume of a PVC |

## 7. Interview Questions

### Other Questions

**Q1. What's the difference between a PV and a PVC?**
A PV is the actual storage (an EBS volume) and is cluster-wide. A PVC is a Pod's request for storage in a namespace. Kubernetes binds a PVC to a matching PV.

**Q2. What is a StorageClass?**
A template for dynamic provisioning: when a PVC asks for storage, the StorageClass's provisioner (like `ebs.csi.aws.com`) creates a new PV and disk.

**Q3. What are the access modes?**
RWO (one node read-write), ROX (many nodes read-only), RWX (many nodes read-write), RWOP (one Pod read-write). EBS supports RWO / RWOP, EFS supports RWX.

**Q4. Why use `volumeBindingMode: WaitForFirstConsumer` with EBS?**
EBS volumes live in one AZ. Waiting for the Pod to be scheduled makes sure the volume is created in the same AZ as the Pod's node.

**Q5. What happens to the data when you delete a PVC?**
It depends on the reclaim policy. `Delete` removes the PV and the EBS volume. `Retain` keeps the disk so the data can be recovered.

**Q6. A PVC is stuck in Pending on EKS. What do you check?**
`kubectl describe pvc` events; whether a StorageClass exists or is default; whether the EBS CSI driver add-on is running and has IAM permissions; with WaitForFirstConsumer, whether a Pod is using the PVC yet.

**Q7. EBS vs EFS for Kubernetes?**
EBS is block storage for one node in one AZ (databases). EFS is a shared file system that many Pods on many nodes across AZs can read and write (RWX).

---

**Source:** [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) · [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) · [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/) · [Amazon EBS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
