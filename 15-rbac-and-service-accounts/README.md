# RBAC and ServiceAccounts: Who Can Do What in the Cluster

[← Kubernetes from scratch](../README.md)

> **In short:** Every request to the API server is checked twice: **who are you** (authentication) and **are you allowed** (authorization). On EKS people log in with **AWS IAM**, Pods use a **ServiceAccount**. **RBAC** decides what each one may do: a **Role** lists allowed actions and a **RoleBinding** gives it to someone.

## About this topic

| Part | Topic | What you learn |
|---|---|---|
| **Part 1: Concepts** | Authentication vs authorization, Role, ClusterRole, bindings, ServiceAccounts, Pod Identity / IRSA | How access works for people and Pods |
| **Part 2: Lab** | Read-only ServiceAccount | Test with `kubectl auth can-i`, call the API from a Pod, see a 403 |

## Table of Contents

**Part 1: Concepts**

1. [Authentication vs authorization](#1-authentication-vs-authorization)
2. [RBAC: Role, ClusterRole and bindings](#2-rbac-role-clusterrole-and-bindings)
3. [ServiceAccounts: identity for Pods](#3-serviceaccounts-identity-for-pods)

**Part 2: Lab**

4. [Lab: a read-only ServiceAccount](#4-lab-a-read-only-serviceaccount)

**Wrap-up**

5. [What I learned](#5-what-i-learned)
6. [Commands](#6-commands)
7. [Interview Questions](#7-interview-questions)

---

# Part 1: Concepts

## 1. Authentication vs authorization

**Definition:** **authentication** checks **who you are**. **Authorization** checks **what you're allowed to do**. In Kubernetes, every request to the API server goes through both, and **RBAC** (Role-Based Access Control) is the authorization part.

```
 kubectl / Pod ──► API server ──► 1. Authentication: who is this?
                                  2. Authorization (RBAC): may they do this?
                                  3. Allowed ──► do it   |   Not allowed ──► 403 Forbidden
```

| Who calls the API | How they prove who they are |
|---|---|
| **People** (`kubectl` on EKS) | **AWS IAM**: the IAM user / role is mapped to a Kubernetes user or group by an **EKS access entry** |
| **Pods** (apps, Argo CD, controllers) | A **ServiceAccount** token mounted into the Pod |

> We already used this: the bastion's IAM role is mapped by an access entry to the group `bastion-admin`, and a ClusterRoleBinding gives that group `cluster-admin` ([EKS with Terraform](../03-eks-cluster-with-terraform/README.md)).

---

## 2. RBAC: Role, ClusterRole and bindings

**Definition:** **RBAC** gives permissions with two kinds of objects: a **Role** lists **what** can be done (verbs on resources), and a **RoleBinding** says **who** gets that Role.

| Object | Scope | What it is |
|---|---|---|
| **Role** | One namespace | Permissions inside that namespace |
| **ClusterRole** | Whole cluster | Permissions everywhere, or on cluster-wide objects (nodes, PVs) |
| **RoleBinding** | One namespace | Gives a Role (or ClusterRole) to someone **in that namespace** |
| **ClusterRoleBinding** | Whole cluster | Gives a ClusterRole to someone **in every namespace** |

```
 Subject (User / Group / ServiceAccount) ──RoleBinding──► Role (verbs + resources)
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]                 # "" = core API group (pods, services, secrets…)
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: app-reader
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

| Field | Meaning |
|---|---|
| `apiGroups` | `""` for core objects, `apps` for Deployments, `networking.k8s.io` for Ingress |
| `resources` | `pods`, `deployments`, `secrets`… |
| `verbs` | `get`, `list`, `watch` (read) · `create`, `update`, `patch`, `delete` (write) |
| `subjects` | `User`, `Group` or `ServiceAccount` |

**Rules to remember:**
- Everything is **denied** until a binding allows it. Permissions only **add up**; there's no "deny" rule.
- **Built-in ClusterRoles:** `cluster-admin` (everything), `admin` (everything in a namespace), `edit` (change apps, not RBAC), `view` (read only, no Secrets).
- **Least privilege:** give a ClusterRole with a **RoleBinding** to limit it to one namespace, for example `edit` only in `roboshop-dev`.

> **Interview one-liner:** RBAC uses Roles (namespaced) and ClusterRoles (cluster-wide) to list allowed verbs on resources, and RoleBindings or ClusterRoleBindings to give them to users, groups or ServiceAccounts; everything is denied unless a binding allows it.

---

## 3. ServiceAccounts: identity for Pods

**Definition:** a **ServiceAccount** is an identity for **Pods** (not people). Kubernetes mounts a short-lived token for it into the Pod, and the app uses that token to call the API server. RBAC then decides what it may do.

| Fact | Detail |
|---|---|
| Default | Every namespace has a ServiceAccount called `default`; Pods use it if we don't set `serviceAccountName` |
| Token location | `/var/run/secrets/kubernetes.io/serviceaccount/token` (+ `ca.crt`, `namespace`) |
| Name in RBAC errors | `system:serviceaccount:<namespace>:<name>` |
| Don't need the API? | Set `automountServiceAccountToken: false` so no token is mounted |

**Who uses ServiceAccounts:** Argo CD (to create objects), the AWS Load Balancer Controller (to read Ingresses), Prometheus (to list Pods), Falco, CI/CD runners that deploy to the cluster.

**ServiceAccount → AWS permissions:** a Pod that needs AWS (S3, Secrets Manager, EBS) gets an **IAM role** linked to its ServiceAccount through **EKS Pod Identity** (newer, simpler) or **IRSA** (IAM Roles for Service Accounts, uses OIDC). No AWS keys are stored in the Pod.

```
 Pod ──uses──► ServiceAccount ──RBAC──► Kubernetes API permissions
                             └─Pod Identity / IRSA──► IAM role ──► AWS API permissions
```

> **Interview one-liner:** A ServiceAccount is the identity a Pod uses to call the Kubernetes API, with permissions from RBAC; on EKS we link it to an IAM role with Pod Identity or IRSA so the Pod gets AWS access without stored keys.

---

# Part 2: Lab

## 4. Lab: a read-only ServiceAccount

**Goal:** create a ServiceAccount that can only **read Pods** in the `dev` namespace, test it with `kubectl auth can-i`, then call the API from inside a Pod and see allowed and forbidden requests.

**Before you start:**

| Need | Why |
|---|---|
| EKS cluster + bastion with `kubectl` as cluster admin ([EKS with Terraform](../03-eks-cluster-with-terraform/README.md)) | Where we run all commands |

### Step 1: Namespace and ServiceAccount

```bash
kubectl create namespace dev
```

```bash
kubectl create serviceaccount app-reader -n dev
```

### Step 2: Role and RoleBinding

```bash
vim rbac.yaml
```

Paste the **Role** + **RoleBinding** YAML from [section 2](#2-rbac-role-clusterrole-and-bindings).

```bash
kubectl apply -f rbac.yaml
```

### Step 3: Test the permissions with `can-i`

```bash
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:app-reader
```

**Expected:** `yes`

```bash
kubectl auth can-i delete pods -n dev --as=system:serviceaccount:dev:app-reader
```

**Expected:** `no` (the Role has only read verbs)

```bash
kubectl auth can-i list secrets -n dev --as=system:serviceaccount:dev:app-reader
```

**Expected:** `no`

```bash
kubectl auth can-i list pods -n default --as=system:serviceaccount:dev:app-reader
```

**Expected:** `no` (a RoleBinding works only in its own namespace)

### Step 4: A Pod that uses the ServiceAccount

```bash
vim api-client.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-client
  namespace: dev
spec:
  serviceAccountName: app-reader
  containers:
    - name: curl
      image: curlimages/curl
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f api-client.yaml
```

```bash
kubectl exec -it api-client -n dev -- sh
```

**What it does:** opens a shell inside the Pod. The next commands run **inside** it.

```bash
SA=/var/run/secrets/kubernetes.io/serviceaccount
```

```bash
curl -s --cacert $SA/ca.crt -H "Authorization: Bearer $(cat $SA/token)" https://kubernetes.default.svc/api/v1/namespaces/dev/pods | head -5
```

**Expected:** JSON that starts with `"kind": "PodList"`. Allowed.

```bash
curl -s --cacert $SA/ca.crt -H "Authorization: Bearer $(cat $SA/token)" https://kubernetes.default.svc/api/v1/namespaces/dev/secrets
```

**Expected:** `"code": 403` and `secrets is forbidden: User "system:serviceaccount:dev:app-reader" cannot list resource "secrets" ... in the namespace "dev"`.

```bash
exit
```

### Step 5: Clean up

```bash
kubectl delete namespace dev
```

**What it does:** removes the namespace with the ServiceAccount, Role, RoleBinding and Pod.

| Problem | Cause | Fix |
|---|---|---|
| `Forbidden ... cannot list resource "pods"` | No Role / binding for this subject, or wrong namespace | `kubectl auth can-i ... --as=...`; check the RoleBinding's namespace and subject |
| Binding exists but still Forbidden | `roleRef` name typo or wrong `apiGroups` (`apps` for Deployments) | `kubectl describe rolebinding` / `role` |
| `kubectl` on EKS: `Unauthorized` | Your IAM role has no EKS access entry | Add an access entry for the role |

---

# Wrap-up

## 5. What I learned

- **Authentication** = who (IAM for people on EKS, ServiceAccount tokens for Pods). **Authorization** = what (RBAC).
- **Role / RoleBinding** = one namespace. **ClusterRole / ClusterRoleBinding** = whole cluster. Denied by default, permissions only add up.
- **ServiceAccount** = identity for Pods. Linked to IAM with **Pod Identity / IRSA** for AWS access.
- Test with `kubectl auth can-i ... --as=...`.

```
 Who? (IAM / ServiceAccount) ──► Binding ──► Role (verbs on resources) ──► allowed / 403
```

## 6. Commands

| Command | Use |
|---|---|
| `kubectl auth can-i <verb> <resource> -n <ns>` | Can **I** do this? |
| `kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa>` | Can this ServiceAccount do it? |
| `kubectl auth can-i --list -n <ns>` | Everything I can do in a namespace |
| `kubectl get roles,rolebindings -n <ns>` | RBAC in a namespace |
| `kubectl get clusterroles` / `kubectl get clusterrolebindings` | Cluster-wide RBAC |
| `kubectl create serviceaccount <name> -n <ns>` | Create a ServiceAccount |
| `kubectl create rolebinding <name> --clusterrole=view --serviceaccount=<ns>:<sa> -n <ns>` | Quick read-only access in one namespace |

## 7. Interview Questions

### Other Questions

**Q1. What is RBAC in Kubernetes?**
Role-Based Access Control: Roles and ClusterRoles list which verbs are allowed on which resources, and RoleBindings / ClusterRoleBindings give them to users, groups or ServiceAccounts.

**Q2. Role vs ClusterRole?**
A Role works inside one namespace. A ClusterRole is cluster-wide: it can cover cluster objects like nodes, or be reused in many namespaces through RoleBindings.

**Q3. RoleBinding vs ClusterRoleBinding?**
A RoleBinding gives permissions in one namespace (even for a ClusterRole). A ClusterRoleBinding gives a ClusterRole in all namespaces.

**Q4. What is a ServiceAccount?**
An identity for Pods. Its token is mounted into the Pod, and RBAC controls what the Pod can do with the Kubernetes API.

**Q5. How do people get access to an EKS cluster?**
They authenticate with AWS IAM. An EKS access entry maps their IAM user or role to a Kubernetes user / group (or an EKS access policy), and RBAC bindings give that group permissions.

**Q6. How does a Pod on EKS get AWS permissions without access keys?**
Link its ServiceAccount to an IAM role with EKS Pod Identity or IRSA. The Pod gets temporary AWS credentials for that role only.

**Q7. How do you check if a user or ServiceAccount can do something?**
`kubectl auth can-i <verb> <resource> -n <ns> --as=<user>` or `--as=system:serviceaccount:<ns>:<name>`.

**Q8. Can you write a "deny" rule in RBAC?**
No. RBAC is allow-only: everything is denied by default and bindings only add permissions. To take access away, remove or narrow the binding.

---

**Source:** [Using RBAC authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) · [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/) · [EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html) · [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
