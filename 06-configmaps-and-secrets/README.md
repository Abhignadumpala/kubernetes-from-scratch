# ConfigMaps and Secrets: Keeping Config and Passwords Out of the Image

[← Kubernetes from scratch](../README.md)

> **In short:** Settings like database addresses change between dev and prod, so we keep them **outside the image**. A **ConfigMap** holds normal settings, a **Secret** holds passwords and keys (only **base64**, not encrypted). Pods read them as **environment variables** or **files**. In real projects the secret values come from **AWS Secrets Manager**.

## About this topic

| Part | Topic | What you learn |
|---|---|---|
| **Part 1: Concepts** | ConfigMap, Secret, 3 ways to use them | What each is, how Pods read them, how real projects keep secrets safe |
| **Part 2: Lab** | RoboShop-style `catalogue` Pod | Env vars and files, decode a Secret, update config, fix CreateContainerConfigError |

## Table of Contents

**Part 1: Concepts**

1. [Why we keep config outside the image](#1-why-we-keep-config-outside-the-image)
2. [ConfigMap](#2-configmap)
3. [Secret](#3-secret)
4. [Three ways a Pod uses them](#4-three-ways-a-pod-uses-them)

**Part 2: Lab**

5. [Lab: ConfigMap and Secret for a RoboShop-style service](#5-lab-configmap-and-secret-for-a-roboshop-style-service)

**Wrap-up**

6. [What I learned](#6-what-i-learned)
7. [Commands](#7-commands)
8. [Interview Questions](#8-interview-questions)

---

# Part 1: Concepts

## 1. Why we keep config outside the image

**Definition:** **configuration** is anything that changes between environments: database address, feature flags, ports, passwords. Kubernetes keeps it in two objects, **ConfigMap** (normal settings) and **Secret** (sensitive values), so the **same image** runs in dev, QA and prod.

| Config inside the image | Config in a ConfigMap / Secret |
|---|---|
| New image for every config change | Same image everywhere, only the ConfigMap changes |
| Passwords baked into the image (anyone who pulls it sees them) | Passwords kept in a Secret, given only to the Pods that need them |

**RoboShop example:** the `catalogue` service needs the MongoDB address, the `cart` service needs the Redis and catalogue addresses.

```
 catalogue Pod ──reads──► ConfigMap: MONGO_URL=mongodb://mongodb:27017/catalogue
 shipping Pod  ──reads──► Secret:    DB_PASSWORD=******
```

---

## 2. ConfigMap

**Definition:** a **ConfigMap** stores non-secret settings as **key-value pairs** (or whole files) that Pods read as **environment variables** or **files**.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalogue-config
data:
  MONGO: "true"
  MONGO_URL: "mongodb://mongodb:27017/catalogue"
```

> A ConfigMap holds at most **1 MiB** of data. It's for settings, not big files.

---

## 3. Secret

**Definition:** a **Secret** stores sensitive values (passwords, tokens, keys, TLS certificates). It works like a ConfigMap, but the values are **base64-encoded** and Kubernetes handles it more carefully (kept in memory on the node, can be encrypted in etcd, separate RBAC).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:                  # plain text here; Kubernetes stores it base64-encoded
  DB_USER: "shipping"
  DB_PASSWORD: "<your-password>"
```

| Secret type | Used for |
|---|---|
| `Opaque` | Any key-value data (default) |
| `kubernetes.io/dockerconfigjson` | Login for a private image registry (`imagePullSecrets`) |
| `kubernetes.io/tls` | TLS certificate + key (for example for an Ingress) |

> ⚠️ **base64 is encoding, not encryption.** Anyone who can read the Secret can decode it with `base64 -d`. Protect Secrets with **RBAC**, turn on **KMS envelope encryption** for etcd on EKS, and **never commit Secret YAML with real values to Git**.

**What real projects use:** the real values live in **AWS Secrets Manager** (or SSM Parameter Store). The **External Secrets Operator** or the **Secrets Store CSI Driver** copies them into the cluster, so Git only has a reference, not the password.

```
 AWS Secrets Manager ──► External Secrets Operator ──► Kubernetes Secret ──► Pod
```

> **Interview one-liner:** A ConfigMap holds normal settings and a Secret holds sensitive ones; both are injected into Pods as env vars or files. Secrets are only base64-encoded, so we protect them with RBAC and KMS encryption, and in real projects sync them from AWS Secrets Manager.

---

## 4. Three ways a Pod uses them

| Way | YAML | Good for |
|---|---|---|
| **One env var** | `env` + `valueFrom.configMapKeyRef` / `secretKeyRef` | Picking a few keys |
| **All keys as env vars** | `envFrom` + `configMapRef` / `secretRef` | Many settings at once |
| **Files in a folder** | `volumes` + `volumeMounts` | Config files (`nginx.conf`, certificates) |

```yaml
      containers:
        - name: catalogue
          image: nginx
          envFrom:                                # every key → env var
            - configMapRef:
                name: catalogue-config
          env:
            - name: DB_PASSWORD                   # one key from a Secret
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
          volumeMounts:
            - name: config
              mountPath: /etc/config              # each key becomes a file here
      volumes:
        - name: config
          configMap:
            name: catalogue-config
```

**What happens when the ConfigMap changes:**

| Used as | Updated in a running Pod? |
|---|---|
| **Env var** | ❌ No. Env vars are set only when the container starts → `kubectl rollout restart` |
| **Volume (files)** | ✅ Yes, after a short delay (about a minute) |

**When the ConfigMap or Secret doesn't exist:** the container never starts and the Pod shows **`CreateContainerConfigError`**.

> **Interview one-liner:** Pods read ConfigMaps and Secrets as single env vars, all keys with envFrom, or files through a volume; env vars only change after a restart, mounted files update by themselves.

---

# Part 2: Lab

## 5. Lab: ConfigMap and Secret for a RoboShop-style service

**Goal:** create a ConfigMap and a Secret, give them to a Pod as env vars and files, decode the Secret, and see what happens when config is missing or changes.

**Before you start:**

| Need | Why |
|---|---|
| EKS cluster + bastion with `kubectl` ([EKS with Terraform](../03-eks-cluster-with-terraform/README.md)) | Where we run all commands |

### Step 1: Create a ConfigMap from the command line

```bash
kubectl create configmap catalogue-config --from-literal=MONGO=true --from-literal=MONGO_URL=mongodb://mongodb:27017/catalogue
```

**What it does:** creates a ConfigMap with 2 keys.

```bash
kubectl get configmap catalogue-config -o yaml
```

**Expected:** both keys under `data:` in plain text.

### Step 2: Create a Secret

```bash
kubectl create secret generic db-secret --from-literal=DB_USER=shipping --from-literal=DB_PASSWORD=MyDemoPass123
```

```bash
kubectl get secret db-secret -o yaml
```

**Expected:** under `data:` the values look scrambled, for example `DB_USER: c2hpcHBpbmc=`. That's only base64.

```bash
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
```

**Expected:** `MyDemoPass123`. Anyone allowed to read the Secret can do this, which is why RBAC matters.

### Step 3: A Pod that uses both

```bash
vim catalogue.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: catalogue
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      envFrom:
        - configMapRef:
            name: catalogue-config
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
      volumeMounts:
        - name: config
          mountPath: /etc/config
  volumes:
    - name: config
      configMap:
        name: catalogue-config
```

```bash
kubectl apply -f catalogue.yaml
```

```bash
kubectl exec catalogue -- env | grep -E "MONGO|DB_PASSWORD"
```

**Expected:** `MONGO=true`, `MONGO_URL=mongodb://mongodb:27017/catalogue` and `DB_PASSWORD=MyDemoPass123`.

```bash
kubectl exec catalogue -- ls /etc/config
```

**Expected:** one file per key: `MONGO` and `MONGO_URL`.

### Step 4: Change the ConfigMap

```bash
kubectl patch configmap catalogue-config -p '{"data":{"MONGO_URL":"mongodb://mongodb-new:27017/catalogue"}}'
```

Wait about a minute, then compare the file and the env var:

```bash
kubectl exec catalogue -- cat /etc/config/MONGO_URL; echo
```

**Expected:** the **new** value (`mongodb-new`).

```bash
kubectl exec catalogue -- printenv MONGO_URL
```

**Expected:** still the **old** value. Env vars only change when the container restarts. For a Deployment we run `kubectl rollout restart deployment/<name>`.

### Step 5: Missing Secret → CreateContainerConfigError

```bash
vim broken.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      envFrom:
        - secretRef:
            name: no-such-secret        # this Secret doesn't exist
```

```bash
kubectl apply -f broken.yaml
```

```bash
kubectl get pod broken
```

**Expected:** `STATUS: CreateContainerConfigError`.

```bash
kubectl describe pod broken | grep -i "not found"
```

**Expected:** `Error: secret "no-such-secret" not found`. Create the Secret and the Pod starts by itself.

### Step 6: Clean up

```bash
kubectl delete pod catalogue broken
```

```bash
kubectl delete configmap catalogue-config
```

```bash
kubectl delete secret db-secret
```

| Problem | Cause | Fix |
|---|---|---|
| `CreateContainerConfigError` | ConfigMap / Secret / key name doesn't exist | `kubectl describe pod`, fix the name or create the object |
| App still uses the old value | Config given as env var | `kubectl rollout restart deployment/<name>` |
| Secret value is wrong after `base64` | Encoded with a newline (`echo` without `-n`) | Use `echo -n` or `stringData` / `--from-literal` |

---

# Wrap-up

## 6. What I learned

- Config lives **outside** the image so one image runs everywhere.
- **ConfigMap** = normal settings. **Secret** = sensitive values, **only base64**, so protect it with RBAC and KMS, and sync from **AWS Secrets Manager** in real projects.
- Pods use them as **env**, **envFrom** or **files**. Files update by themselves, env vars need a restart.
- Missing config → **CreateContainerConfigError**.

```
 ConfigMap / Secret ──► env / envFrom (read at start) ──► container
                    └─► volume (files, auto-update) ──► container
```

## 7. Commands

| Command | Use |
|---|---|
| `kubectl create configmap <name> --from-literal=K=V` | ConfigMap from values |
| `kubectl create configmap <name> --from-file=app.conf` | ConfigMap from a file |
| `kubectl create secret generic <name> --from-literal=K=V` | Secret from values |
| `kubectl get secret <name> -o jsonpath='{.data.K}' \| base64 -d` | Decode one Secret value |
| `kubectl exec <pod> -- env` | See a container's env vars |
| `kubectl rollout restart deployment/<name>` | Restart Pods to pick up new env values |

## 8. Interview Questions

### Other Questions

**Q1. What's the difference between a ConfigMap and a Secret?**
Both store config for Pods. A ConfigMap is for normal settings in plain text. A Secret is for sensitive data: base64-encoded, kept in memory on nodes, can be encrypted in etcd and restricted with RBAC.

**Q2. Are Kubernetes Secrets secure?**
Not by themselves: base64 is only encoding. We make them safer with RBAC, KMS envelope encryption on EKS, and by keeping real values in AWS Secrets Manager synced by the External Secrets Operator, never in Git.

**Q3. How can a Pod use a ConfigMap?**
As one env var (`valueFrom.configMapKeyRef`), as all keys (`envFrom`), or as files through a volume mount.

**Q4. You updated a ConfigMap but the app still uses the old value. Why?**
The value was given as an env var, which is read only at container start. Restart the Pods (`kubectl rollout restart`), or mount it as a volume so the files update.

**Q5. What does `CreateContainerConfigError` mean?**
The Pod refers to a ConfigMap, Secret or key that doesn't exist, so the container can't be created. `kubectl describe pod` shows which one.

**Q6. How do you pull an image from a private registry?**
Create a `kubernetes.io/dockerconfigjson` Secret with the registry login and list it under `imagePullSecrets` in the Pod spec. On EKS with ECR, the node IAM role gives pull access instead.

---

**Source:** [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) · [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) · [Good practices for Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/) · [External Secrets Operator](https://external-secrets.io/)
