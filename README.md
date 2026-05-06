#  ArgoCD Demo — GitOps Continuous Delivery on Minikube

This project demonstrates **GitOps-based continuous delivery** using ArgoCD on a local Kubernetes cluster (Minikube). When you push a change to this GitHub repo, ArgoCD automatically detects and deploys it to your cluster.

---

##  Repository Structure

```
argocd-demo/
├── index.html          ← The web page served by nginx
└── k8s/
    ├── deployment.yaml ← Kubernetes Deployment (nginx + git-clone initContainer)
    └── service.yaml    ← Kubernetes Service (NodePort on port 30080)
```

---

##  Prerequisites

Make sure the following are installed on your machine before starting:

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) — running and active
- [Minikube](https://minikube.sigs.k8s.io/docs/start/) — v1.38+
- [kubectl](https://kubernetes.io/docs/tasks/tools/) — Kubernetes CLI
- Windows PowerShell (run as **Administrator**)

---

##  Step-by-Step Setup

### Step 1 — Start Minikube

```powershell
minikube start --driver=docker
```

This starts a local single-node Kubernetes cluster inside Docker.

Verify it is running:

```powershell
kubectl get nodes
```

You should see one node with status `Ready`.

---

### Step 2 — Install ArgoCD

Create a dedicated namespace for ArgoCD:

```powershell
kubectl create namespace argocd
```

Install ArgoCD using the official manifest:

```powershell
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

> ⚠️ You may see a warning about `applicationsets.argoproj.io` being too long. This is harmless — ignore it.

---

### Step 3 — Wait for ArgoCD Pods to be Ready

Watch the pods start up (takes about 2–3 minutes):

```powershell
kubectl get pods -n argocd -w
```

Wait until **all pods** show `1/1 Running`. Press `Ctrl+C` to stop watching once ready.

Expected output:

```
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          3m
argocd-applicationset-controller-57cb77786d-xxxxx   1/1     Running   0          3m
argocd-dex-server-846595988d-xxxxx                  1/1     Running   0          3m
argocd-notifications-controller-7dc46994fc-xxxxx    1/1     Running   0          3m
argocd-redis-56c8c46dc6-xxxxx                       1/1     Running   0          3m
argocd-repo-server-7bb7cdc9cd-xxxxx                 1/1     Running   0          3m
argocd-server-786979cb84-xxxxx                      1/1     Running   0          3m
```

---

### Step 4 — Get the ArgoCD Admin Password

```powershell
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}"
```

This returns a **base64 encoded** string. Decode it in PowerShell:

```powershell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("PASTE_BASE64_HERE"))
```

Save this password — you will need it to log into the ArgoCD UI.

---

### Step 5 — Access the ArgoCD UI

> ⚠️ Port 8080 may be blocked on Windows. Use port **9090** instead.

```powershell
kubectl port-forward svc/argocd-server -n argocd 9090:443
```

Keep this PowerShell window open. Open your browser and go to:

```
https://localhost:9090
```

Accept the SSL warning (click **Advanced → Proceed**). This is normal for local ArgoCD.

**Login credentials:**
- Username: `admin`
- Password: *(decoded password from Step 4)*

---

### Step 6 — Connect GitHub Repo to ArgoCD

In the ArgoCD dashboard, create a new application:

1. Click **"+ New App"** (top left)
2. Fill in the form:

| Field | Value |
|---|---|
| Application Name | `argocd-demo` |
| Project Name | `default` |
| Sync Policy | `Automatic` ✅ |
| Auto-Create Namespace | ✅ enable |
| Repository URL | `https://github.com/YOUR_USERNAME/argocd-demo` |
| Revision | `HEAD` |
| Path | `k8s` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

3. Click **"Create"**

ArgoCD will immediately sync and deploy your app. You will see the status change from `OutOfSync` → `Syncing` → **`Healthy ✅`**

---

### Step 7 — Open Your Deployed App

```powershell
minikube service argocd-demo-svc --url
```

Open the printed URL in your browser. You will see your `index.html` live! 🎉

---

##  How the Demo Works

```
Developer pushes to GitHub
          ↓
ArgoCD polls GitHub every ~3 minutes
          ↓
ArgoCD detects change → auto-syncs
          ↓
Kubernetes pod is updated
          ↓
Browser shows new content ✅
```

The deployment uses an **initContainer** that clones the GitHub repo on pod startup, then nginx serves the `index.html` from it.

---

##  Triggering a New Deployment (The "Wow" Moment)

### Option A — Edit directly on GitHub (easiest for demo)

1. Go to your repo on GitHub
2. Click `index.html` → click the **pencil ✏️ edit icon**
3. Change `Version 1.0` to `Version 2.0`
4. Click **"Commit changes"** with message: `Update to v2.0`
5. Watch the ArgoCD UI — within ~3 minutes the SYNC STATUS updates automatically

### Option B — Edit locally and push

```bash
# Edit index.html, then:
git add .
git commit -m "Update to v2.0"
git push
```

---

##  Restarting the Pod After a Sync

Because the app uses `initContainer git clone`, the HTML is only pulled **once at pod startup**. After ArgoCD syncs a new commit, restart the pod to pull the latest content:

```powershell
kubectl rollout restart deployment/argocd-demo -n default
```

Then refresh your browser to see the updated page.

> 💡 **Why?** The `initContainer` clones the repo when the pod starts. It does not watch for changes while running. A restart triggers a fresh clone with the latest code.

---

##  Useful Commands

```powershell
# Check all pods in default namespace
kubectl get pods -n default

# Check all services
kubectl get services -n default

# Check all resources
kubectl get all -n default

# Check ArgoCD pods
kubectl get pods -n argocd

# Restart the demo pod (after ArgoCD syncs a change)
kubectl rollout restart deployment/argocd-demo -n default

# Get the app URL
minikube service argocd-demo-svc --url

# Watch pod restart live
kubectl get pods -n default -w
```

---

##  After Every PC Restart

When you restart your PC, Docker and Minikube stop. You do **NOT** need to reinstall anything. Just run these commands:

**1. Open Docker Desktop** from Start menu and wait until it shows "Engine running" ✅

**2. Open PowerShell as Administrator:**

```powershell
# Start Minikube
minikube start --driver=docker

# Start ArgoCD UI access (keep this window open)
kubectl port-forward svc/argocd-server -n argocd 9090:443

# Get your app URL
minikube service argocd-demo-svc --url
```

### What you do NOT need to redo

- ✅ ArgoCD installation (already in cluster)
- ✅ Creating the app in ArgoCD UI (already saved)
- ✅ GitHub repo setup (stays as is)
- ✅ Admin password (same: `AsGYbXVWI7wcIDLb`)

---

## 🖥️ Understanding the ArgoCD UI

When you open `https://localhost:9090`, you will see the **Application Details** view showing a visual tree of your deployment:

### Top Status Bar

| Element | What It Shows |
|---|---|
| **APP HEALTH** | ✅ **Healthy** — all resources running correctly |
| **SYNC STATUS** | ✅ **Synced to HEAD** — cluster matches latest Git commit |
| **LAST SYNC** | ✅ **Sync OK** — timestamp and commit message of last successful sync |

### Visual Tree Explanation

The UI shows your Kubernetes resources as a tree diagram:

```
argocd-demo (root app)
    ├── argocd-demo-svc (Service)
    │   └── NodePort exposes the app externally
    │
    └── argocd-demo (Deployment)
        ├── argocd-demo-99bc85d55 (ReplicaSet - current version)
        │   └── argocd-demo-99bc85d55-7gdrj (Pod - running/1/1)
        │
        └── argocd-demo-b975f59b5 (ReplicaSet - old version, rev.1)
```

**What each icon means:**

| Icon | Resource Type | What It Does |
|---|---|---|
| 📦 Stack (argocd-demo) | Application | The root ArgoCD application tracking your GitHub repo |
| 🌐 Mesh (svc) | Service | Exposes your app via NodePort on port 30080 |
| 🔄 Circles (deploy) | Deployment | Manages your pods and rolling updates |
| 📋 Document (rs) | ReplicaSet | Maintains the desired number of pod replicas |
| 📦 Box (pod) | Pod | The actual running container (nginx + your HTML) |

**Green checkmarks** mean healthy and running. **"3 hours"** shows how long since the resource was created. **"rev:2"** means this is the 2nd revision (after your update).

When you push a change to GitHub and ArgoCD syncs, you'll see:
1. A new ReplicaSet appears (e.g., `rev:3`)
2. A new Pod spins up under it
3. The old ReplicaSet scales down to 0
4. Sync Status updates to show your new commit message

---

##  Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `port 8080 bind error` | Port 8080 blocked on Windows | Use port 9090 instead |
| `SVC_NOT_FOUND` | Service not deployed yet | Check `kubectl get services -n default` |
| Browser shows old content | initContainer only runs at startup | Run `kubectl rollout restart deployment/argocd-demo` |
| ArgoCD not syncing | Polls every 3 min by default | Wait or click "Sync" manually in UI |
| Pod stuck in `Init` state | `alpine/git` image pulling slowly | Wait 1–2 min, check with `kubectl describe pod` |
| SSL warning on localhost:9090 | Self-signed cert | Click Advanced → Proceed to localhost |

---

##  Key Concepts

| Term | Meaning |
|---|---|
| **GitOps** | Git is the single source of truth for infrastructure and app config |
| **ArgoCD** | Tool that watches Git and keeps your cluster in sync with it |
| **Sync** | Process of making the cluster match what is in Git |
| **Drift** | When live cluster state differs from what is declared in Git |
| **initContainer** | A container that runs before the main container starts — used here to clone the repo |
| **NodePort** | A Kubernetes service type that exposes your app on a port accessible from outside the cluster |
| **Auto-Sync** | ArgoCD automatically applies changes when it detects a new commit |

---

##  Notes

- ArgoCD polls GitHub every **3 minutes** by default. This is normal — for instant sync, webhooks can be configured.
- The `applicationsets.argoproj.io` CRD warning during install is **harmless**.
- Always run PowerShell as **Administrator** for Minikube and kubectl commands.
- Keep the `kubectl port-forward` window open while using the ArgoCD UI.

---

*Prepared as part of DevOps Tools & Practices course presentation.*