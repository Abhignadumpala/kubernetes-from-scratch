# Kubernetes Services: ClusterIP, NodePort, LoadBalancer, Ingress and Traffic Flow

[← Kubernetes from scratch](../README.md)

> **In short:** Pods are temporary and their IPs change, so a **Service** gives them **one fixed name + IP** and sends traffic only to **Ready** Pods. **ClusterIP** = inside the cluster only. **NodePort** = a port on every node. **LoadBalancer** = a cloud load balancer per Service. **Ingress** = one public door (an ALB) for all websites. In real projects: **every app is ClusterIP + one Ingress (ALB) is the public door**.

## About this topic

Here we learn, step by step, how Pods are reached inside the cluster, how users on the internet reach an app, and what real projects use.

| Part | Topic | What you learn |
|---|---|---|
| **Part 1: Concepts** | Service, ClusterIP, NodePort, LoadBalancer, Ingress, traffic flow | What each one is, when to use it, what production uses |
| **Part 2: Lab** | One app exposed 4 ways on EKS | Create each one, test it, see what it creates in AWS, clean up |

## Table of Contents

**Part 1: Concepts**

1. [The problem: Pod IPs keep changing](#1-the-problem-pod-ips-keep-changing)
2. [The fix: a Service](#2-the-fix-a-service)
3. [Types of Services](#3-types-of-services)
   - [3.1 ClusterIP: only inside the cluster](#31-clusterip-only-inside-the-cluster)
   - [3.2 NodePort: a door on every node](#32-nodeport-a-door-on-every-node)
   - [3.3 LoadBalancer: a cloud load balancer per Service](#33-loadbalancer-a-cloud-load-balancer-per-service)
4. [Ingress: one door for all websites](#4-ingress-one-door-for-all-websites)
5. [Which one to use (what real projects use)](#5-which-one-to-use-what-real-projects-use)
6. [How traffic flows in production](#6-how-traffic-flows-in-production)

**Part 2: Lab**

7. [Lab: expose one app 4 ways on EKS](#7-lab-expose-one-app-4-ways-on-eks)

**Wrap-up**

8. [What I learned](#8-what-i-learned)
9. [Commands](#9-commands)
10. [Interview Questions](#10-interview-questions)

---

# Part 1: Concepts

## 1. The problem: Pod IPs keep changing

**Definition:** a **Pod** is the smallest unit that runs our app's container in Kubernetes. Pods are **temporary**: Kubernetes deletes and recreates them whenever they crash, when we deploy a new version, or when it scales the app up or down. Every new Pod gets a **new IP address**, so a Pod's IP is never something another app can rely on.

| What happens | Result |
|---|---|
| A Pod crashes | A new Pod comes up with a **new IP** |
| We deploy a new version | **All Pods** get new IPs |
| We run 3 copies | **3 different IPs** |

So if the frontend wants to call the backend, **which IP should it use?** It can't know.

---

## 2. The fix: a Service

**Definition:** a **Service** is a Kubernetes object that gives a group of Pods **one fixed name and one fixed IP** that never change, even when the Pods behind it are replaced. It finds its Pods by **label** (for example `app: backend`), keeps a list of the ones that are **Ready**, and sends each request to one of them. So other apps always call the **Service name**, never a Pod IP.

```
 frontend ──► "backend" (Service, never changes) ──┬──► Pod 1 (Ready ✅)
                                                   ├──► Pod 2 (Ready ✅)
                                                   └──✖ Pod 3 (Not Ready, skipped)
```

> 📞 **Example:** a company's **phone extension**. You dial "backend" and whoever is free picks up. You don't care which employee it is.

**How it works:**

| Who | Job |
|---|---|
| **`selector`** in the Service | Picks the Pods by label (e.g. `app: backend`) |
| **Readiness probe** | Decides if each Pod is Ready |
| **EndpointSlices** | The list of **Ready** Pod IPs behind the Service (updated automatically) |
| **kube-proxy** | On every node, sends traffic for the Service IP to one Ready Pod |
| **CoreDNS** | Turns the name `backend` into the Service IP |

> **Interview one-liner:** Pods are temporary and their IPs change, so a Service gives a stable name and IP and load-balances across the Ready Pods it selects by label.

---

## 3. Types of Services

**Definition:** every Service has a **type**, set with `type:` in its YAML. The type decides **who can reach the app** behind the Service: only other Pods inside the cluster, or also users from outside. Kubernetes has **3 types**: **ClusterIP**, **NodePort** and **LoadBalancer**. If we don't write a type, Kubernetes uses **ClusterIP**.

| Type | Who can reach it | Why it exists |
|---|---|---|
| **ClusterIP** | Only Pods **inside** the cluster | So apps can call each other by a fixed name (frontend → backend) |
| **NodePort** | Also people **outside**, via `node-ip:30080` | The simplest way to open an app to the outside, without any cloud load balancer |
| **LoadBalancer** | Also the **internet**, via a cloud load balancer | Gives the app a proper public address, with health checks |

Each type adds one more way in on top of the one before:

```
 ClusterIP     →  inside the cluster only
 NodePort      →  ClusterIP + a port on every node
 LoadBalancer  →  NodePort + a cloud load balancer in front
```

Let's look at each one.

### 3.1 ClusterIP: only inside the cluster

**Definition:** **ClusterIP** is the **default** Service type. Kubernetes gives the Service an **internal IP** (for example `10.96.45.12`) and a name (for example `backend`) that work **only inside the cluster**. Other Pods can call it, but nobody outside the cluster (no user on the internet) can reach it. We use it for apps that only other apps need to talk to.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP       # default, can be left out
  selector:
    app: backend        # send traffic to Pods with this label
  ports:
  - port: 80            # Service port
    targetPort: 80      # container port
```

```
 frontend Pod ──► backend:80 (10.96.45.12) ──► Ready backend Pod
```

> 📞 **Example:** an **internal phone extension**. It works only inside the office.

| Who can reach it | Creates in AWS | Use for |
|---|---|---|
| Only Pods inside the cluster | Nothing | ✅ **Every app**: frontend, backend, workers, internal databases |

### 3.2 NodePort: a door on every node

**Definition:** **NodePort** is a Service type that opens the **same port number** (between **30000 and 32767**) on **every node** (server) in the cluster. Anyone who can reach a node can open the app at `<node-ip>:<nodePort>`, and Kubernetes forwards that request to one of the Ready Pods. It still has a ClusterIP too, so Pods inside the cluster can call it by name as usual.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080     # opened on every node (30000-32767)
```

```
 User ──► <node-ip>:30080 ──► Ready Pod
```

> 🚪 **Example:** a **side door with a weird number on every building**. It works, but you never give this address to customers.

**Why it's rare in production:**

| Problem | Why it matters |
|---|---|
| **Weird address** | Users would have to type `http://54.12.34.56:30080`. Nobody gives customers an IP and a port. Real users type a **domain name** like `shop.com` |
| **Node IPs change** | Nodes are replaced, scaled or upgraded, so the IP you shared stops working |
| **One node dies = address breaks** | The user is pointed at one node; if it goes down, the app looks down |
| **No HTTPS, no health checks** | There's no single stable entry point to attach a certificate or skip unhealthy targets |

| Who can reach it | Creates in AWS | Use for |
|---|---|---|
| Outside, via `node-ip:3xxxx` | Nothing | ❌ Quick testing, on-prem clusters |

### 3.3 LoadBalancer: a cloud load balancer per Service

**Definition:** **LoadBalancer** is a Service type that asks the **cloud** (AWS) to create a **real load balancer** with a **public address** for this one Service. Users on the internet send requests to that address, and the load balancer forwards them to the Ready Pods. On EKS this load balancer is an **NLB** (Network Load Balancer), and **each** LoadBalancer Service gets **its own** NLB.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external           # handled by the AWS Load Balancer Controller
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing  # public
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip      # NLB → Pod IPs directly
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

```
 User ──► NLB (a1b2c3.elb.amazonaws.com) ──► Ready Pod
```

**We only write this YAML.** The **AWS Load Balancer Controller** (installed once in the cluster) creates the NLB in AWS automatically, and its address appears in `EXTERNAL-IP`. Deleting the Service deletes the NLB.

> 🏢 **Example:** giving **every department its own reception desk** on the street. It works, but it's expensive.

**Why not for every app?** 10 apps = **10 load balancers** = 10 bills (about $16/month each, plus traffic) and 10 HTTPS setups. It also can't route by URL path.

| Who can reach it | Creates in AWS | Use for |
|---|---|---|
| Internet, via its own NLB | **1 NLB per Service** | ⚠️ Non-HTTP apps (TCP/UDP: game server, MQTT) |

---

## 4. Ingress: one door for all websites

**Definition:** an **Ingress** is a Kubernetes object that holds **HTTP/HTTPS routing rules** (by host and path) for many ClusterIP Services, so they can **share one public load balancer** instead of one LoadBalancer Service each. An **Ingress controller** makes the rules work; on EKS that's the **AWS Load Balancer Controller**, which creates one **ALB**.

```
 shop.com/api  → backend
 shop.com/     → frontend
```

| Who can reach it | Creates in AWS | Use for |
|---|---|---|
| Internet, via one ALB | **1 ALB for many apps** | ✅ **All websites and HTTP APIs** |

> 📘 **Full guide:** why we need Ingress, Ingress controllers, how a request reaches the Pod, a sample, all the rule types and a lab: [**Kubernetes Ingress**](../10-ingress/README.md).

> **Interview one-liner:** An Ingress is HTTP routing rules for many ClusterIP Services; on EKS the AWS Load Balancer Controller turns it into one ALB that routes by path or host straight to the Ready Pods.

---

## 5. Which one to use (what real projects use)

**Definition:** choosing how to expose an app means answering two questions: **who must reach it** (only other apps inside the cluster, or users on the internet?) and **what it speaks** (a website/API over **HTTP**, or plain **TCP/UDP** like a game server). The answers decide whether we use ClusterIP, an Ingress, or a LoadBalancer Service.

![Who can reach your app: Service types and Ingress](images/service-types.png)

| Type | Who can reach it | Creates in AWS | When to use |
|---|---|---|---|
| **ClusterIP** | Only Pods inside the cluster | Nothing | ✅ **Every app** (default) |
| **NodePort** | Outside, via `node-ip:3xxxx` | Nothing | ❌ Testing / on-prem only |
| **LoadBalancer** | Internet, via its own NLB | **1 NLB per Service** | ⚠️ Non-HTTP apps only |
| **Ingress** (not a Service type) | Internet, via one ALB | **1 ALB for many apps** | ✅ **All websites and APIs** |

**What we write in a real project:**

| Thing | What we write |
|---|---|
| frontend Service | `type: ClusterIP` |
| backend Service | `type: ClusterIP` |
| Ingress | Rules `/api` and `/` → creates **one ALB** |
| AWS Load Balancer Controller | Installed **once** per cluster with Helm |
| NodePort / LoadBalancer Services | **None** |

**Why learn NodePort and LoadBalancer if we rarely write them?**

| Reason | Why |
|---|---|
| **They're the basics** | ClusterIP → NodePort → LoadBalancer → Ingress: each one solves a problem of the one before |
| **Some real cases need them** | Non-HTTP apps → **LoadBalancer**. On-prem clusters → **NodePort** behind a hardware load balancer |
| **Testing** | A NodePort opens an app quickly without an Ingress |
| **Interviews** | "ClusterIP vs NodePort vs LoadBalancer" is one of the most asked Kubernetes questions |

> 🎯 **Interview answer:** "ClusterIP is internal, NodePort opens a port on every node, and LoadBalancer creates a cloud load balancer per Service. In production we keep all apps as ClusterIP and expose web traffic through one Ingress, which on EKS becomes an ALB. We use LoadBalancer only for non-HTTP apps or for the NGINX ingress controller, and NodePort mostly for testing or on-prem."

---

## 6. How traffic flows in production

**Definition:** the **traffic flow** is every stop a user's request passes through, from the moment they type `shop.com` in the browser until a **Ready Pod** sends back the answer: first **DNS** finds the address, then the **load balancer** receives the request, then the **Ingress rules** pick the app, and finally one **Pod** of that app's Service handles it.

![Production traffic flow](images/production-traffic.png)

```
 User types https://shop.com/api/products
    │
    ▼
 ① Route 53 (DNS) ..... "shop.com = this ALB"
    │
    ▼
 ② ALB (public) ....... created from the Ingress, ends HTTPS
    │                   rule: /api → backend
    ▼
 ③ backend Pod ........ ALB sends straight to a Ready Pod IP
    │                   (the list comes from the backend ClusterIP Service)
    ▼
 ④ backend calls "db" . by name, through its ClusterIP Service
```

| Step | What happens | Who does it |
|---|---|---|
| 1 | `shop.com` → the ALB's address | **Route 53** |
| 2 | HTTPS ends, the **path** picks the app | **ALB** (made from the Ingress) |
| 3 | Request goes to **one Ready Pod** | ALB, using the Service's Ready Pod list |
| 4 | Pods call each other **by name** | **ClusterIP Service** + CoreDNS |

For `https://shop.com/` the rule `/` sends the request to a **frontend Pod**, and the nginx in that Pod sends back the web page.

> ⭐ **Remember in one line:** **Route 53 → ALB (Ingress rules) → Ready Pods of a ClusterIP Service.**

> **Interview one-liner:** Route 53 points the domain to an ALB created from an Ingress; the ALB ends HTTPS and routes by path to the Ready Pods of the right ClusterIP Service, and those Pods call other Services by name.

---

# Part 2: Lab

## 7. Lab: expose one app 4 ways on EKS

**Goal:** run one nginx app and expose it as **ClusterIP**, **NodePort**, **LoadBalancer (NLB)** and **Ingress (ALB)**, then see what each one creates.

**Before you start:**

| Need | Why |
|---|---|
| EKS cluster + bastion with `kubectl` ([EKS with Terraform](../03-eks-cluster-with-terraform/README.md)) | Where we run all commands |
| **AWS Load Balancer Controller** installed ([install guide](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/)) | Creates the NLB (step 5) and the ALB (step 6) |
| Public subnets tagged `kubernetes.io/role/elb = 1` | The controller puts internet-facing load balancers there |

> ⚠️ Steps 5 and 6 create **real AWS load balancers that cost money**. Do the clean-up (step 8) when you finish.

### Step 1: Create the app

```bash
vim web.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web               # every Service below finds Pods by this label
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        readinessProbe:        # only Ready Pods get traffic
          httpGet:
            path: /
            port: 80
          periodSeconds: 5
```

```bash
kubectl apply -f web.yaml
```

```bash
kubectl get pods -o wide
```

**Expected:** 2 `web` Pods, `1/1 Running`, each with a VPC IP (e.g. `10.0.3.25`).

### Step 2: ClusterIP (inside only)

```bash
kubectl expose deployment web --port=80
```

**What it does:** creates a **ClusterIP** Service named `web` (the default type).

```bash
kubectl get svc web
```

**Expected:** `TYPE ClusterIP`, a `CLUSTER-IP` like `172.20.x.x`, `EXTERNAL-IP <none>`.

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web
```

**Expected:** the 2 Pod IPs from step 1. This is the list of **Ready** Pods the Service sends to.

```bash
kubectl run tmp --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- web
```

**What it does:** starts a temporary Pod **inside** the cluster and calls the Service **by name**.

**Expected:** the nginx page `<title>Welcome to nginx!</title>` ✅. From outside the cluster, the ClusterIP can't be reached.

### Step 3: NodePort (a port on every node)

```bash
kubectl expose deployment web --name=web-nodeport --type=NodePort --port=80
```

```bash
kubectl get svc web-nodeport
```

**Expected:** `PORT(S) 80:3xxxx/TCP`. The `3xxxx` number is the **NodePort**, opened on **every** node.

```bash
kubectl get nodes -o wide
```

**What it does:** shows each node's `INTERNAL-IP`.

```bash
curl http://<node-internal-ip>:<nodePort>
```

**Expected:** the nginx page ✅ (the bastion is in the same VPC). If it **hangs**, the node security group doesn't allow that port from the bastion: allow TCP 30000–32767 from the bastion's security group.

> This is why NodePort is rare in production: real users type a **domain name** (`shop.com`), not a node IP and port 3xxxx.

### Step 4: Check what's in AWS so far

```bash
aws elbv2 describe-load-balancers --query "LoadBalancers[].{Name:LoadBalancerName,Type:Type}" --output table
```

**Expected:** no new load balancers. **ClusterIP and NodePort create nothing in AWS.**

### Step 5: LoadBalancer (NLB per Service)

```bash
vim web-nlb.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f web-nlb.yaml
```

```bash
kubectl get svc web-nlb -w
```

**Expected:** `EXTERNAL-IP` changes from `<pending>` to `k8s-default-webnlb-xxxx.elb.us-east-1.amazonaws.com`. Press `Ctrl+C`.

```bash
curl http://<EXTERNAL-IP>
```

**Expected:** the nginx page ✅ from the internet. The first 2–3 minutes may fail while the NLB starts and its DNS spreads.

### Step 6: Ingress (ALB for many apps)

```bash
vim web-ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web          # the ClusterIP Service from step 2
            port:
              number: 80
```

```bash
kubectl apply -f web-ingress.yaml
```

```bash
kubectl get ingress web -w
```

**Expected:** `ADDRESS` fills in with `k8s-default-web-xxxx.us-east-1.elb.amazonaws.com`. Press `Ctrl+C`.

```bash
curl http://<ADDRESS>
```

**Expected:** the nginx page ✅. The Ingress points at the **ClusterIP** Service `web`. No NodePort or LoadBalancer Service is needed.

### Step 7: See the result in AWS

```bash
aws elbv2 describe-load-balancers --query "LoadBalancers[].{Name:LoadBalancerName,Type:Type}" --output table
```

**Expected:** 2 load balancers:

| Name starts with | Type | Created by |
|---|---|---|
| `k8s-default-webnlb-...` | `network` | `Service type: LoadBalancer` (step 5) |
| `k8s-default-web-...` | `application` | `Ingress` (step 6) |

> With 10 apps, step 5 would mean **10 NLBs**. Step 6 can serve all 10 apps with **one ALB** (one Ingress rule per path or host).

**Troubleshooting:**

| Problem | Cause | Fix |
|---|---|---|
| `EXTERNAL-IP` stays `<pending>` | AWS Load Balancer Controller not installed / not running | `kubectl get deploy -n kube-system aws-load-balancer-controller` |
| Ingress `ADDRESS` stays empty | Wrong `ingressClassName`, or no tagged public subnets | `kubectl describe ingress web` (Events), `kubectl logs -n kube-system deploy/aws-load-balancer-controller` |
| ALB returns **503** | No Ready Pods behind the Service | `kubectl get endpointslices -l kubernetes.io/service-name=web`, check readiness |
| NodePort `curl` hangs | Node security group blocks 30000–32767 | Allow it from the bastion's security group |
| curl to NLB/ALB fails right after creation | Load balancer still provisioning, DNS not ready | Wait 2–3 minutes and retry |

### Step 8: Clean up (stops the AWS bill)

Delete the Ingress and the LoadBalancer Service **first**, so the controller deletes the ALB and NLB in AWS.

```bash
kubectl delete -f web-ingress.yaml
```

```bash
kubectl delete -f web-nlb.yaml
```

```bash
kubectl delete svc web web-nodeport
```

```bash
kubectl delete -f web.yaml
```

```bash
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerName" --output table
```

**Expected:** no `k8s-default-...` load balancers left ✅.

> ⚠️ Destroying the whole cluster with Terraform **before** deleting Ingresses and LoadBalancer Services can leave the ALB/NLB behind (Terraform didn't create them) and block VPC deletion. Delete them with `kubectl` first.

---

# Wrap-up

## 8. What I learned

**Part 1: Concepts**
- Pod IPs keep changing, so a **Service** gives a fixed name + IP and sends only to **Ready** Pods.
- **ClusterIP** = inside only (every app). **NodePort** = port on every node (testing). **LoadBalancer** = one NLB per Service (non-HTTP apps).
- **Ingress** = rules; the **AWS Load Balancer Controller** turns them into **one ALB**, which sends straight to Ready Pods.
- **Production:** every app is ClusterIP + one Ingress (ALB) is the public door.

**Part 2: Lab**
- ClusterIP and NodePort create nothing in AWS. A LoadBalancer Service created an **NLB**; an Ingress created an **ALB**. Delete them with `kubectl` before destroying the cluster.

```
 User ──► Route 53 ──► ALB (Ingress rules) ──► Ready Pods (ClusterIP Service)
```

---

## 9. Commands

| Command | Use |
|---|---|
| `kubectl expose deployment web --port=80` | ClusterIP Service (default) |
| `kubectl expose deployment web --name=web-nodeport --type=NodePort --port=80` | NodePort Service |
| `kubectl get svc` | `TYPE`, `CLUSTER-IP`, `EXTERNAL-IP`, `PORT(S)` (shows the NodePort) |
| `kubectl get endpointslices -l kubernetes.io/service-name=web` | Ready Pod IPs behind a Service |
| `kubectl get ingress` | Ingress rules and the ALB `ADDRESS` |
| `kubectl describe ingress <name>` | Events: why the ALB wasn't created |
| `kubectl port-forward svc/web 8080:80` | Reach a Service from your machine for debugging |
| `aws elbv2 describe-load-balancers` | See the NLB / ALB created in AWS |

---

## 10. Interview Questions

### Other Questions

**Q1. Why do we need Services if Pods already have IPs?**
Pods are temporary: restarts, deploys and scaling give new IPs, and replicas have many IPs. A Service gives one stable name and IP, load-balances across the Pods it selects by label, and skips Pods that aren't Ready.

**Q2. What's the difference between ClusterIP, NodePort and LoadBalancer?**
ClusterIP is internal, NodePort opens a port on every node, and LoadBalancer creates a cloud load balancer per Service. In production we keep all apps as ClusterIP and expose web traffic through one Ingress, which on EKS becomes an ALB. We use LoadBalancer only for non-HTTP apps or for the NGINX ingress controller, and NodePort mostly for testing or on-prem.

**Q3. How do you expose a web app to users in production?**
With an Ingress. On EKS the AWS Load Balancer Controller creates one ALB that ends HTTPS and routes by host or path to ClusterIP Services. All app Services stay ClusterIP.

**Q4. If every app Service is ClusterIP, how does internet traffic get in?**
Through the ALB created from the Ingress, the only public piece. On EKS Pods have VPC IPs, so the ALB sends traffic straight to the Ready Pod IPs it gets from the Service.

**Q5. Why not use a LoadBalancer Service for every app?**
Each one creates its own billed load balancer, so 10 apps means 10 load balancers and 10 HTTPS setups, with no path routing. One Ingress shares a single ALB. LoadBalancer is still right for non-HTTP (TCP/UDP) apps.

**Q6. Why is NodePort rarely used in production?**
Users would need a node IP and a high port instead of a domain name. Node IPs change when nodes are replaced, the address breaks if that node dies, and there's no HTTPS or health checks.

**Q7. What is an Ingress controller?**
The component that makes Ingress rules work. On EKS the AWS Load Balancer Controller reads the Ingress and creates an ALB. Without a controller, an Ingress does nothing.

**Q8. Does a Service check which Pod is healthy?**
No. The readiness probe decides if a Pod is Ready. Only Ready Pods stay in the Service's EndpointSlices, so traffic goes only to them.

**Q9. Walk me through what happens when a user opens your app.**
Route 53 resolves the domain to the ALB; the ALB ends HTTPS and matches the Ingress path rule; it sends the request to a Ready Pod of that ClusterIP Service; the Pod calls other Services by name.

---

**Source:** [Kubernetes docs: Service](https://kubernetes.io/docs/concepts/services-networking/service/) · [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) · [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) · [EKS: network load balancing](https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html) · [EKS: application load balancing](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)


---

**Source:** [Kubernetes docs: Service](https://kubernetes.io/docs/concepts/services-networking/service/) · [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) · [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) · [EKS: network load balancing](https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html) · [EKS: application load balancing](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)
