# Helm: The Package Manager for Kubernetes (Charts, Releases, Values, Rollback)

[← Kubernetes from scratch](../README.md)

> **In short:** **Helm** is the package manager for Kubernetes. Instead of applying 20+ YAML files by hand, `helm install` installs a whole app (a **chart**) with one command, lets us change settings with **values**, and can **upgrade** and **roll back** each install (a **release**).

## Table of Contents

1. [Introduction to Helm](#1-introduction-to-helm)
2. [Commands Cheat Sheet](#2-commands-cheat-sheet)
3. [Interview Questions](#3-interview-questions)

---

## 1. Introduction to Helm

**Definition:** **Helm** is the **package manager for Kubernetes**. Like `apt` / `yum` installs software on Linux, `helm install` installs a full app (all its YAML files) into a cluster with one command.

**Why Helm?** An app like Argo CD needs many YAML files: Deployments, Services, ConfigMaps, Secrets, RBAC, CRDs. Writing and applying them by hand is slow and easy to get wrong.

| Without Helm | With Helm |
|---|---|
| `kubectl apply` 20+ YAML files one by one | `helm install argocd argo/argo-cd` |
| Copy YAML for dev, test, prod, change values by hand | Same chart, different `values.yaml` |
| Upgrade = edit many files | `helm upgrade` |
| Rollback = remember old YAML | `helm rollback <release> <revision>` |
| Delete = find every object | `helm uninstall` |

### Helm words

| Word | Simple meaning | Linux example |
|---|---|---|
| **Chart** | A package: a folder of K8s YAML templates | `.rpm` / `.deb` package |
| **Repository** | A place that stores charts | yum / apt repo |
| **Release** | One installed copy of a chart in the cluster | Installed software |
| **values.yaml** | Settings you can change (replicas, image, service type) | Config file |
| **Template** | YAML with placeholders, filled from values | |
| **Revision** | Version number of a release; goes up on every upgrade | |

### Chart structure

```
mychart/
├── Chart.yaml        # name, version, description
├── values.yaml       # default settings
└── templates/        # K8s YAML with placeholders
    ├── deployment.yaml
    └── service.yaml
```

```yaml
# templates/deployment.yaml (part)
spec:
  replicas: {{ .Values.replicaCount }}
  containers:
    - image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

```yaml
# values.yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.27"
```

```
values.yaml ──┐
              ├──► helm ──► final K8s YAML ──► cluster (release)
templates/ ───┘
```

> **Interview one-liner:** Helm is the package manager for Kubernetes. A **chart** is a package of templated YAML, **values.yaml** holds the settings, and a **release** is one installed copy. It gives one-command install, upgrade and rollback.

### Helm use cases (2 main ones)

| Use case | What | Example |
|---|---|---|
| **1. Manage releases** | Install, check, upgrade and roll back an app with one command | `helm list`, `helm upgrade`, `helm rollback argocd 1` (see [Argo CD lab](../16-argocd-gitops/README.md#check-a-release-and-roll-back-helm-list-kubectl-get-pods-helm-rollback)) |
| **2. Install ready-made tools** | Every popular tool already has a chart. Find it on **Artifact Hub**, copy the commands, install | Argo CD, Jenkins, Prometheus, Grafana, ingress-nginx |

### Use case 2: Artifact Hub (the "app store" for Helm charts)

**Definition:** **Artifact Hub** ([artifacthub.io](https://artifacthub.io)) is a public website where tool publishers list their Helm charts. Whatever tool you want **inside the K8s cluster**, search it there and you get the ready-made install commands.

> Artifact Hub is the **catalog** (search page). The chart itself is stored in the **publisher's repo**. `helm repo add` points Helm to that repo, and `helm install` downloads the chart from it and installs it.

**Steps:**

| Step | Where | What to do |
|---|---|---|
| 1 | 💻 Laptop browser | Open [artifacthub.io](https://artifacthub.io), search the tool (e.g. `argo-cd`) |
| 2 | 💻 Laptop browser | Pick the chart from the **official / Verified Publisher** (see below) |
| 3 | 💻 Laptop browser | Click **INSTALL** → it shows the `helm repo add` + `helm install` commands. Copy them |
| 4 | 🖥️ Bastion | Paste and run: add repo → update → install |
| 5 | 🖥️ Bastion | Check: `helm list -n <ns>` and `kubectl get pods -n <ns>` |

**How to pick the right chart (many people publish charts with the same name):**

| Badge / check | Meaning |
|---|---|
| ✅ **Verified Publisher** | Artifact Hub confirmed the publisher really owns that repo |
| 🏅 **Official** | Published by the tool's own team (e.g. Argo project for Argo CD) |
| ⭐ Stars + recent updates | Popular and maintained |

> Always use the **official / Verified Publisher** chart. A random chart could be old, broken or unsafe.

**Official charts for common tools:**

| Tool | Repo add (name + URL) | Install chart |
|---|---|---|
| Argo CD | `argo https://argoproj.github.io/argo-helm` | `argo/argo-cd` |
| Jenkins | `jenkins https://charts.jenkins.io` | `jenkins/jenkins` |
| Prometheus + Grafana | `prometheus-community https://prometheus-community.github.io/helm-charts` | `prometheus-community/kube-prometheus-stack` |
| Grafana only | `grafana https://grafana.github.io/helm-charts` | `grafana/grafana` |

**Example: install Jenkins the same way as Argo CD**

```bash
# 🖥️ BASTION (commands copied from Artifact Hub → INSTALL)
helm repo add jenkins https://charts.jenkins.io   # 1. tell Helm where the chart lives
helm repo update                                  # 2. get the latest chart list
helm install jenkins jenkins/jenkins -n jenkins --create-namespace   # 3. install
helm list -n jenkins                              # 4. check the release
kubectl get pods -n jenkins                       # 5. check the pods
```

**Useful extra commands:**

```bash
# 🖥️ BASTION
helm search repo jenkins                          # find charts in repos you added
helm show values jenkins/jenkins > values.yaml    # see all settings you can change
helm pull jenkins/jenkins --untar                 # download the chart folder to look inside
```

> **Interview one-liner:** Helm has two main uses: managing releases (install, upgrade, rollback) and installing ready-made tools. Every popular tool (Argo CD, Jenkins, Prometheus) publishes an official chart on Artifact Hub; I pick the Verified Publisher chart, copy the `helm repo add` and `helm install` commands and run them on the bastion.

### Helm vs Ansible: which one in real projects?

**Definition:** **Ansible** is a **configuration management** tool: it logs in to **servers** (VMs, EC2) over SSH and sets them up by installing packages, creating users, copying files and starting services. **Helm** is the **package manager for Kubernetes**: it turns chart templates into Kubernetes YAML and **installs, upgrades and rolls back apps inside a cluster** through the Kubernetes API. They solve **different problems**, so they don't replace each other.

| | **Ansible** | **Helm** |
|---|---|---|
| Manages | **Servers** (VMs, EC2, on-prem, network devices) | **Apps on Kubernetes** |
| Talks to | The server's OS, over **SSH** | The **Kubernetes API** |
| Files | Playbooks, roles, inventory | Charts: `Chart.yaml`, `values.yaml`, `templates/` |
| Example | Install nginx and Java on 50 EC2 servers | Install Prometheus, or upgrade my app v1.2 → v1.3, on EKS |

**Why Kubernetes companies use Ansible less:** on a VM, the server lives for months and we keep **changing** it with Ansible. With containers we **don't log in and change** anything: the software is **baked into the Docker image**, and a new version means a **new image**, deployed with Helm.

**When every app is built by CI/CD, containerised and deployed to Kubernetes, Ansible has little or no part in deploying apps.** This is normal for cloud-native teams.

| Job | Old way (VMs) | Kubernetes company |
|---|---|---|
| Install software for the app (Java, Node, nginx) | Ansible on each server | **Dockerfile** (baked into the image) |
| Build and ship the app | Ansible copies files to servers | **CI/CD** builds the image and pushes it to ECR |
| Deploy / upgrade | Ansible playbook | **Helm** chart (often through **Argo CD**) |
| App config | Ansible templates | **ConfigMaps / Secrets** (values in Helm) |
| Create servers / infra | Manual or Ansible | **Terraform** (EKS, VPC, RDS) |
| Patch and update nodes | Ansible | **EKS managed node groups** (AWS updates the node images) |

So in a Kubernetes company the main daily tools are **Docker, CI/CD, Kubernetes, Helm, Argo CD and Terraform**.

**Where Ansible still appears, even in a Kubernetes company:**

- Servers **outside** Kubernetes: Jenkins and its agents, bastion hosts, VPN, monitoring VMs
- Databases running on **EC2** instead of RDS
- **Self-managed** Kubernetes nodes (kubeadm), unlike EKS
- Building **golden AMIs** (Packer + Ansible)
- On-prem servers or network devices

**A typical modern stack:**

```
 Terraform  → builds the infra (VPC, EKS, RDS)
 Ansible    → configures servers / VMs / AMIs (outside Kubernetes)
 Helm       → packages the apps for Kubernetes
 Argo CD    → keeps the cluster in sync with Git (deploys the Helm charts)
```

> **Interview one-liner:** Ansible is configuration management for servers: it connects over SSH and installs and configures software on VMs. Helm is a package manager for Kubernetes: it templates and versions Kubernetes manifests and installs, upgrades and rolls back apps in a cluster. In a company where every app is containerised and deployed to EKS through CI/CD, Helm and Argo CD do the deployment, and Ansible is used only for servers outside Kubernetes.

---

## 2. Commands Cheat Sheet

| Command | Use |
|---|---|
| `helm repo add <name> <url>` | Add a chart repo |
| `helm repo update` | Refresh repo list |
| `helm search repo <word>` | Find charts |
| `helm install <release> <repo/chart> -n <ns>` | Install a chart |
| `helm list -A` | List releases |
| `helm upgrade <release> <chart>` | Upgrade |
| `helm history <release>` | See revisions |
| `helm rollback <release> <rev>` | Go back to a revision |
| `helm uninstall <release>` | Remove |
| `helm show values <repo/chart>` | See all values you can change |

---

## 3. Interview Questions

### ⭐ Very Important

**Q1. How do you check which Helm releases are installed and their version?**
`helm list -n <ns>` (or `helm list -A` for all namespaces). It shows release name, revision, status, chart version and app version. It does not list pods; use `kubectl get pods -n <ns>` for that.

**Q2. How do you roll back a Helm release?**
`helm history <release> -n <ns>` to see revisions, then `helm rollback <release> <revision> -n <ns>`, e.g. `helm rollback argocd 1 -n argocd`. The rollback is saved as a new revision (rev 3 = copy of rev 1).

**Q3. What is the difference between chart version and app version?**
Chart version (`version` in Chart.yaml) is the version of the Helm package. App version (`appVersion`) is the version of the app inside it, e.g. Argo CD v3.

**Q4. How do you install a tool like Jenkins or Prometheus in a K8s cluster?**
With Helm. Search the tool on Artifact Hub, choose the official / Verified Publisher chart, then run its `helm repo add`, `helm repo update` and `helm install` commands from the admin machine (bastion).

**Q5. What is Artifact Hub?**
A public catalog of Helm charts. It lists each chart's repo, install commands, values and badges (Verified Publisher, Official). The chart is downloaded from the publisher's repo, not from Artifact Hub.

### Other Questions

**Q1. What is Helm?**
The package manager for Kubernetes. It installs, upgrades and rolls back apps using charts.

**Q2. Chart vs release?**
Chart = the package (templates + values). Release = one installed copy of a chart in the cluster.

**Q3. How do you change settings when installing a chart?**
`--set key=value` for a few values, or `-f my-values.yaml` for many.

**Q4. Do you install Helm in the cluster?**
No. Helm 3 is only a client tool on the machine that runs it (the bastion). It sends YAML to the API server like kubectl. (Old Helm 2 had a server part called Tiller; it was removed.)

**Q5. What's the difference between Helm and Ansible? Which do you use?**
Ansible configures servers over SSH (packages, users, files, services); Helm installs, upgrades and rolls back apps on Kubernetes. In our project every app is containerised and deployed to EKS through CI/CD with Helm and Argo CD, so Ansible isn't used for app deployment; it fits servers outside Kubernetes, like Jenkins agents, EC2 databases or golden AMIs.

---

**Source:** [Helm docs](https://helm.sh/docs/) · [Artifact Hub](https://artifacthub.io)
