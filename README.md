# 🚀 GitOps on Kubernetes with ArgoCD: Declarative Continuous Delivery

> A hands-on implementation of the GitOps workflow using **ArgoCD v3.6.1** on Kubernetes: a non-HA installation, CLI setup, application management through the UI and CLI, automated reconciliation, and a two-repository (application code and configuration) layout.

<img width="1678" height="1043" alt="Screenshot 2026-11-17 002324" src="https://github.com/user-attachments/assets/0c1c73db-e4b1-48bc-94f1-f65be8f6123a" />

<img width="1897" height="775" alt="Screenshot 2026-11-17 013407" src="https://github.com/user-attachments/assets/c1f4199f-5e25-440e-a643-937b2feeecbf" />
<img width="1272" height="686" alt="Screenshot 2026-11-17 012916" src="https://github.com/user-attachments/assets/d48f3f55-6fba-440d-917e-4c21c5c2bd34" />




---



## 📑 Table of Contents

1. [Project Overview](#-project-overview)
2. [Architecture & GitOps Workflow](#-architecture--gitops-workflow)
3. [Repository Structure](#-repository-structure)
4. [Prerequisites](#-prerequisites)
5. [Part 1: Install ArgoCD](#-part-1-install-argocd)
6. [Part 2: Explore the ArgoCD Configuration](#-part-2-explore-the-argocd-configuration)
7. [Part 3: Expose the ArgoCD UI via NodePort](#-part-3-expose-the-argocd-ui-via-nodeport)
8. [Part 4: Retrieve the Admin Password](#-part-4-retrieve-the-admin-password)
9. [Part 5: Install the ArgoCD CLI](#-part-5-install-the-argocd-cli)
10. [Part 6: ArgoCD Applications & Projects](#-part-6-argocd-applications--projects)
11. [Part 7: Create an Application from the UI](#-part-7-create-an-application-from-the-ui)
12. [Part 8: Create an Application with the CLI](#-part-8-create-an-application-with-the-cli)
13. [Part 9: Automatic Reconciliation (Timer & Webhook)](#-part-9-automatic-reconciliation-timer--webhook)
14. [Part 10: Demonstrating Drift (Out of Sync → Synced)](#-part-10-demonstrating-drift-out-of-sync--synced)
15. [CLI Cheat Sheet](#-argocd-cli-cheat-sheet)
16. [Troubleshooting](#-troubleshooting)
17. [Key Learnings](#-key-learnings)
18. [Security Notes](#-security-notes)
19. [References](#-references)

---

## 📖 Project Overview

**GitOps** is an operating model where **Git is the single source of truth** for the desired state of your infrastructure and applications. A GitOps controller running in the cluster continuously compares the *live state* with the *desired state* stored in Git and reconciles any difference.

This project covers:

- ✅ Installing a **non-HA ArgoCD v3.6.1** in the `argocd` namespace
- ✅ Inspecting ArgoCD internals (`argocd-cm` ConfigMap, `argocd-server` ports)
- ✅ Exposing the ArgoCD UI by converting the `argocd-server` Service from `ClusterIP` to `NodePort` (HTTPS port **30334**)
- ✅ Installing the **ArgoCD CLI v2.14.9**
- ✅ Creating and syncing ArgoCD **Applications** via the UI and CLI
- ✅ Following a **two-repository GitOps pattern**: one repo for application code, one for Kubernetes configuration
- ✅ Tuning the **reconciliation interval** and configuring a **Git webhook**
- ✅ Demonstrating **drift detection** (Out of Sync → Synced)

---

## 🏗 Architecture & GitOps Workflow

```
 ┌──────────────────┐        ┌────────────────────────┐
 │  App Repository  │        │  Config Repository     │
 │  (source code,   │        │  (Kubernetes manifests │
 │   Dockerfile)    │        │   / GitOps repo)       │
 └────────┬─────────┘        └───────────┬────────────┘
          │ CI builds & pushes image     │ image tag / manifest
          ▼                              │ updated via commit
   Container Registry ─────────────────► │
                                         │  ① Webhook (push event)
                                         │  ② or poll every N minutes
                                         ▼
                              ┌─────────────────────┐
                              │   ArgoCD Server     │
                              │ (argocd namespace)  │
                              └──────────┬──────────┘
                                         │ ③ Compare desired vs live state
                                         │ ④ Sync (manual or automated)
                                         ▼
                              ┌─────────────────────┐
                              │ Kubernetes Cluster  │
                              │ (solar-system ns)   │
                              └─────────────────────┘
```

### Why two repositories?

| Repository | Purpose | Contents |
|---|---|---|
| **Application repo** | Developers' source of truth for code | Source code, Dockerfile, tests, CI pipeline |
| **Config (GitOps) repo** | Source of truth for *deployment state* | Kubernetes manifests (Deployments, Services, etc.) |

<img width="1906" height="782" alt="Screenshot 2026-11-17 001345" src="https://github.com/user-attachments/assets/5901e9d3-7d31-46bf-ae71-247a0a753166" />
<img width="1290" height="501" alt="Screenshot 2026-11-17 012421" src="https://github.com/user-attachments/assets/06baca8e-7a59-4a65-93c4-664bf59af05f" />
<img width="1760" height="600" alt="Screenshot 2026-11-17 012348" src="https://github.com/user-attachments/assets/7ad88c9a-9621-4804-8836-27d6c81ccab7" />




**Benefits of separating them:**

- 🔒 **Access control**: different people/teams can have different permissions on code vs. deployment config.
- 🧹 **Clean history**: config changes (e.g., replica counts) don't pollute the application's commit log.
- 🔁 **No CI loops**: a manifest change doesn't re-trigger the application's build pipeline.
- 📜 **Clear audit trail**: every deployment change is a Git commit in the config repo.

---

## 📁 Repository Structure

```
.
├── README.md
├── screenshots/              # Evidence of each step
└── ...                       # (add your repo files here)
```

**Config repo layout used by ArgoCD** (`gitops-argocd`):

```
gitops-argocd/
└── solar-system/
    ├── deployment.yaml
    └── service.yaml
```

> 🔗 **Application repo:** `<link-to-app-repo>`
> 🔗 **Config repo:** `<link-to-config-repo>`

---

## ✅ Prerequisites

| Requirement | Notes |
|---|---|
| Kubernetes cluster | Any cluster you have admin access to |
| `kubectl` | Configured to talk to the cluster |
| `jq` | Used for extracting the admin password |
| `curl` | Used to download the CLI |
| Git server | GitHub / Gitea hosting the config repo |

---

## ⚙️ Part 1: Install ArgoCD

**Task:** Install a **non-HA ArgoCD v3.6.1** in the `argocd` namespace.

Release page: <https://github.com/argoproj/argo-cd/releases>

```bash
# Create the namespace
kubectl create namespace argocd

# Install the non-HA manifest for v3.6.1
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.1/manifests/install.yaml
```



> 💡 `manifests/install.yaml` is the **non-HA** install. The HA variant is `manifests/ha/install.yaml`.

Verify that everything is running:

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

📸 **Screenshot: ArgoCD pods running**

![ArgoCD pods](screenshots/01-argocd-pods.png)
<img width="817" height="447" alt="Screenshot 2026-11-17 013107" src="https://github.com/user-attachments/assets/bd31289c-3dc7-4f66-ae66-313f41b39f24" />

---

## 🔍 Part 2: Explore the ArgoCD Configuration

### 2.1 How many data entries does the `argocd-cm` ConfigMap have?

```bash
# Describe the ConfigMap
kubectl -n argocd describe configmap argocd-cm

# Count the data entries
kubectl -n argocd get configmap argocd-cm -o json | jq '.data | length'
```


![argocd-cm data entries](screenshots/02-argocd-cm.png)

### 2.2 Which port does `argocd-server` listen on?

```bash
kubectl -n argocd get svc argocd-server
kubectl -n argocd get deploy argocd-server -o jsonpath='{.spec.template.spec.containers[0].ports}'
```




---

## 🌐 Part 3: Expose the ArgoCD UI via NodePort

By default `argocd-server` is a `ClusterIP` Service, reachable only inside the cluster. To reach the UI from outside, convert it to `NodePort` and pin the HTTPS node port to **30334**.

```bash
kubectl -n argocd edit svc argocd-server
```

Change the following:

```yaml
spec:
  type: NodePort          # was: ClusterIP
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
    - name: https
      port: 443
      protocol: TCP
      targetPort: 8080
      nodePort: 30334     
```

Or do it non-interactively:

```bash
kubectl -n argocd patch svc argocd-server --type merge -p '{
  "spec": {
    "type": "NodePort",
    "ports": [
      {"name":"http",  "port":80,  "targetPort":8080, "protocol":"TCP"},
      {"name":"https", "port":443, "targetPort":8080, "protocol":"TCP", "nodePort":30334}
    ]
  }
}'
```

Verify:

```bash
kubectl -n argocd get svc argocd-server
```

📸 **Screenshot: Service now of type NodePort**
<img width="691" height="497" alt="Screenshot 2026-11-17 012327" src="https://github.com/user-attachments/assets/1da1d161-5590-48ef-bcdb-cb6a50b52e6a" />
![NodePort service](screenshots/04-nodeport-service.png)




📸 **Screenshot: ArgoCD login page**

![ArgoCD login](screenshots/05-argocd-login.png)

---

## 🔑 Part 4: Retrieve the Admin Password

ArgoCD generates an initial admin password and stores it in the `argocd-initial-admin-secret` Secret.

```bash
kubectl -n argocd get secrets argocd-initial-admin-secret -o json \
  | jq .data.password -r \
  | tr -d '\n' \
  | base64 -d
```

| Field | Value |
|---|---|
| **Username** | `admin` |
| **Password** | *output of the command above* |

> ⚠️ Never commit real passwords to a repository. Change the admin password after the first login and delete the initial secret (see [Security Notes](#-security-notes)).

📸 **Screenshot: ArgoCD dashboard after login**

![ArgoCD dashboard](screenshots/06-argocd-dashboard.png)

---

## 💻 Part 5: Install the ArgoCD CLI

**Task:** Install **ArgoCD CLI v2.14.9**.

```bash
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/download/v2.14.9/argocd-linux-amd64

chmod +x /usr/local/bin/argocd

# Verify
argocd version --client
```

Log in to the server (use the node IP and the NodePort):

```bash
argocd login <NODE_IP>:32766 --username admin --password <PASSWORD> --insecure
```

> ℹ️ **Version note:** the server is **v3.6.1** while the CLI is **v2.14.9**. The CLI works for the commands used here, but for production use keep the CLI and server on matching (or at least close) versions.

📸 **Screenshot: CLI installed and logged in**
<img width="1206" height="132" alt="Screenshot 2026-11-16 151156" src="https://github.com/user-attachments/assets/b924be79-4445-409b-9877-4787eb82fc08" />

![ArgoCD CLI](screenshots/07-argocd-cli.png)

---

## 📦 Part 6: ArgoCD Applications & Projects

### Application

An ArgoCD **Application** is a Kubernetes custom resource that connects a Git source to a cluster destination. Every Application always has:

| Component | Description | Example |
|---|---|---|
| **Source** | Git repo, path and revision containing the manifests | `<gitea-url>/bob/gitops-argocd.git` → `./solar-system` |
| **Destination** | Target cluster (server) and namespace | `https://kubernetes.default.svc` → `solar-system` |
| **Sync policy** | Manual or automated (prune, self-heal) | Manual |

### Project

An ArgoCD **Project** (`AppProject`) groups Applications and defines guardrails: which repos are allowed as sources, which clusters/namespaces are allowed as destinations, and which resource kinds can be deployed. Applications that don't specify a project belong to the built-in `default` project.

---

## 🖱 Part 7: Create an Application from the UI

**Application settings:**

| Field | Value |
|---|---|
| Application Name | `solar-system-app-1` |
| Project Name | `default` |
| Sync Policy | `Manual` |
| Sync Options | ✅ Auto-Create Namespace |
| Repository URL | `<Gitea URL>/bob/gitops-argocd.git` |
| Path | `./solar-system` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `solar-system` |

Steps:

1. Open the ArgoCD UI → **+ NEW APP**
2. Fill in the fields above
3. Click **CREATE**
4. Click **SYNC** → **SYNCHRONIZE** (manual sync policy means nothing deploys until you sync)

📸 **Screenshot: Application creation form**

![Create app via UI](screenshots/08-create-app-ui.png)

📸 **Screenshot: Application synced and healthy**

![solar-system-app-1 synced](screenshots/09-app1-synced.png)

---

## ⌨️ Part 8: Create an Application with the CLI

### 8.1 Create the application

```bash
argocd app create solar-system-app-2 \
  --repo <gitea-url>/bob/gitops-argocd.git \
  --path ./solar-system \
  --dest-namespace solar-system \
  --dest-server https://kubernetes.default.svc
```

### 8.2 Synchronize it

```bash
argocd app sync solar-system-app-2
```

### 8.3 List the applications

```bash
argocd app list
```

📸 **Screenshot: CLI app creation, sync and list**
<img width="886" height="651" alt="Screenshot 2026-11-17 002833" src="https://github.com/user-attachments/assets/177ce1b3-8634-4187-b31c-2145117c9d16" />


![CLI app create and sync](screenshots/10-app2-cli.png)

---

## 🔄 Part 9: Automatic Reconciliation (Timer & Webhook)

ArgoCD can learn about Git changes in two ways:

| Method | How it works | Latency |
|---|---|---|
| **Polling** (default) | ArgoCD checks the Git repo on a timer | Up to 3 minutes |
| **Webhook** | Git server notifies ArgoCD on every push | Near-instant |

### 9.1 Change the reconciliation timer (3 min → 1 min)

The default polling interval is **3 minutes**. I reduced it to **1 minute** by editing the `argocd-cm` ConfigMap:

```bash
kubectl -n argocd edit configmap argocd-cm
```

```yaml
data:
  timeout.reconciliation: 60s
```

Restart the application controller so the change takes effect:

```bash
kubectl -n argocd rollout restart statefulset argocd-application-controller
```

> ℹ️ In v3.x the application controller runs as a **StatefulSet**.

📸 **Screenshot: updated `argocd-cm`**
<img width="917" height="398" alt="Screenshot 2026-11-17 012257" src="https://github.com/user-attachments/assets/61729e9a-2c5b-48b5-941f-1cb5f4e5f912" />


![Reconciliation timeout](screenshots/11-reconciliation-timeout.png)

### 9.2 Configure a Git webhook

Add a webhook on the **config repo** that sends push events to ArgoCD so reconciliation happens immediately.

| Setting | Value |
|---|---|
| Payload URL | `https://<NODE_IP>:32766/api/webhook` |
| Content type | `application/json` |
| Trigger | Push events |
| Secret (optional but recommended) | Same value stored in `argocd-secret` |

Optional: store the webhook secret in ArgoCD (use the key that matches your Git provider, e.g. `webhook.github.secret`, `webhook.gogs.secret` for Gitea/Gogs).

```bash
kubectl -n argocd patch secret argocd-secret --type merge \
  -p '{"stringData": {"webhook.gogs.secret": "<your-secret>"}}'
```

📸 **Screenshot: webhook configured in the Git repo**

![Webhook](screenshots/12-webhook.png)

---

## 🔁 Part 10: Demonstrating Drift (Out of Sync → Synced)

This experiment proves ArgoCD detects the difference between **live cluster state** and **Git (desired) state**.

| Step | Action | Result |
|---|---|---|
| 1 | Cluster runs **1** replica, Git manifest declares **3** replicas | 🟡 **OutOfSync** (live 1 ≠ desired 3) |
| 2 | In the Git repo, change `replicas` back to **1** and commit | Git now matches the cluster |
| 3 | ArgoCD refreshes (timer or webhook) | 🟢 **Synced** |

Example change in the config repo:

```yaml
spec:
  replicas: 1    # was 3
```

```bash
git add solar-system/deployment.yaml
git commit -m "Set replicas back to 1"
git push
```

📸 **Screenshot: OutOfSync (1 pod in cluster vs 3 in Git)**
<img width="1272" height="686" alt="Screenshot 2026-11-17 012916" src="https://github.com/user-attachments/assets/41f9bcba-0a2d-48f3-9d39-c9ea77be09fa" />
<img width="1341" height="828" alt="Screenshot 2026-11-17 012949" src="https://github.com/user-attachments/assets/5b777749-c085-4814-be73-7d3f1a5940f0" />
<img width="592" height="136" alt="Screenshot 2026-11-17 012821" src="https://github.com/user-attachments/assets/5bb2dc0b-ca1f-4638-86c7-47679edc5f39" />




![Out of sync](screenshots/13-out-of-sync.png)

📸 **Screenshot: Synced after fixing Git**

![Synced](screenshots/14-synced.png)
<img width="837" height="526" alt="Screenshot 2026-11-17 013518" src="https://github.com/user-attachments/assets/4369e351-fbdd-4236-9e27-ba73fd3b2e97" />
<img width="767" height="558" alt="Screenshot 2026-11-17 013832" src="https://github.com/user-attachments/assets/08c8ca8c-0e25-46c8-85a1-3a0bd8433d6b" />




> 💡 With **automated sync + self-heal** enabled, ArgoCD would not just report drift, it would also revert manual cluster changes to match Git.

---

## 🧰 ArgoCD CLI Cheat Sheet

| Task | Command |
|---|---|
| Log in | `argocd login <host>:<port> --username admin --insecure` |
| List apps | `argocd app list` |
| Create app | `argocd app create <name> --repo <url> --path <path> --dest-server <server> --dest-namespace <ns>` |
| Get app details | `argocd app get <name>` |
| Sync app | `argocd app sync <name>` |
| Diff live vs Git | `argocd app diff <name>` |
| Set auto-sync | `argocd app set <name> --sync-policy automated --self-heal --auto-prune` |
| Delete app | `argocd app delete <name>` |
| List projects | `argocd proj list` |
| Create project | `argocd proj create <name> -d https://kubernetes.default.svc,<ns> -s <repo-url>` |
| Change password | `argocd account update-password` |

---

## 🛠 Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| UI not reachable | Service still `ClusterIP` or firewall blocks the port | Confirm `NodePort` and port **32766**; check node firewall |
| Browser certificate warning | ArgoCD uses a self-signed cert | Accept the warning, or use `--insecure` in the CLI |
| `argocd login` fails | Wrong host/port or password | Use `<NODE_IP>:32766` and re-fetch the password from the secret |
| App stuck `OutOfSync` | Manual sync policy | Run `argocd app sync <name>` or enable auto-sync |
| Namespace not found on sync | Namespace doesn't exist | Enable **Auto-Create Namespace** sync option |
| Reconciliation change ignored | Controller not restarted | Restart `argocd-application-controller` |
| Webhook not triggering | Wrong URL / content type / secret | Check the Git server's webhook delivery log |

---

## 🎓 Key Learnings

- **Git is the source of truth.** The cluster is made to match the repo, not the other way around.
- **Pull-based delivery** is more secure than push-based CI/CD: the cluster pulls its own state, so CI doesn't need cluster credentials.
- **Separating app and config repos** gives cleaner history, better access control, and avoids CI trigger loops.
- **Drift is visible and fixable.** `OutOfSync` shows exactly where live state differs from Git.
- **Polling vs. webhooks:** polling is simple (3 min default, tunable); webhooks give near-instant feedback.
- **Applications + Projects** provide structure and guardrails for multi-team usage.

---

## 🔐 Security Notes

- ❌ Do **not** commit passwords, tokens, or webhook secrets to Git.
- 🔄 Change the default `admin` password after first login:
  ```bash
  argocd account update-password
  ```
- 🗑 After setting your own password, delete the initial secret:
  ```bash
  kubectl -n argocd delete secret argocd-initial-admin-secret
  ```
- 🔒 For production: use TLS with a real certificate (Ingress), SSO/RBAC, and restricted Projects instead of `default`.
- 🛡 Use `--insecure` and `NodePort` only for lab/test environments.

---

## 📚 Helpful References
- https://kodekloud.com/
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD Releases](https://github.com/argoproj/argo-cd/releases)
- [ArgoCD Webhook Configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/)
- [OpenGitOps Principles](https://opengitops.dev/)

---

## 👤 Author

**Segun Ofe**
🔗 [GitHub](https://github.com/segunofe) · [LinkedIn](https://www.linkedin.com/in/segunofe/)
