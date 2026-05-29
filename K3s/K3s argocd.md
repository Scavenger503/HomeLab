# ArgoCD Setup Guide

Step-by-step guide for deploying ArgoCD on K3s and configuring a GitOps workflow connected to a GitHub repository.

---

## What is ArgoCD?

ArgoCD is a declarative GitOps continuous delivery tool for Kubernetes. It watches a Git repository for changes to Kubernetes manifests and automatically syncs the cluster to match the desired state defined in the repo.

**The workflow:**
```
Developer pushes manifest changes to GitHub
    │
    ▼
ArgoCD detects the change (polls every 3 minutes or via webhook)
    │
    ▼
ArgoCD applies the changes to the cluster
    │
    ▼
Cluster state matches repository state
```

<img width="1624" height="1308" alt="image" src="https://github.com/user-attachments/assets/e3961371-398f-4c2f-a715-594e91e91175" />


---

## Prerequisites

- K3s cluster running and healthy
- `kubectl` access to the cluster
- GitHub repository containing Kubernetes manifests
- DNS rewrite configured in AdGuard for `argocd.k3s.scavenger`

---

## 1. Create ArgoCD Namespace

```bash
sudo kubectl create namespace argocd
```

---

## 2. Install ArgoCD

```bash
sudo kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This installs all ArgoCD components:
- `argocd-server` — API server and web UI
- `argocd-application-controller` — reconciliation controller
- `argocd-repo-server` — repository caching
- `argocd-dex-server` — SSO integration
- `argocd-redis` — caching layer
- `argocd-applicationset-controller` — ApplicationSet support
- `argocd-notifications-controller` — notification support

Wait for all pods to be ready:

```bash
sudo kubectl get pods -n argocd -w
```

All pods should show `Running` before proceeding.

---

## 3. Expose ArgoCD via Ingress

Create the Ingress manifest at `K3s/argocd/argocd-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: argocd.k3s.scavenger
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

Apply it:

```bash
sudo kubectl apply -f K3s/argocd/argocd-ingress.yaml
```

> **Important:** Do not add middleware annotations to the ArgoCD ingress. Extra annotations like `stripprefix` will cause routing failures.

---

## 4. Disable ArgoCD HTTPS Redirect

By default ArgoCD redirects HTTP to HTTPS. Since Traefik is handling TLS termination, patch ArgoCD server to run in insecure mode:

```bash
sudo kubectl patch deployment argocd-server -n argocd --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--insecure"}]'
```

---

## 5. Get Initial Admin Password

```bash
sudo kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Save this password — you'll need it for first login.

---

## 6. Access the ArgoCD UI

Navigate to `http://argocd.k3s.scavenger` in your browser.

- **Username:** `admin`
- **Password:** output from step 5

Change the password immediately after first login:
**User Info → Update Password**

---

## 7. Connect GitHub Repository

In the ArgoCD UI:

1. Go to **Settings → Repositories → Connect Repo**
2. Choose **HTTPS**
3. Enter your GitHub repository URL
4. If the repo is public, no credentials needed
5. Click **Connect**

The repository should show **Successful** connection status.

---

## 8. Create Your First Application

In the ArgoCD UI click **+ New App**:

**General:**
- Application Name: `homepage`
- Project: `default`
- Sync Policy: `Automatic`
- Check: **Prune Resources** and **Self Heal**

**Source:**
- Repository URL: your GitHub repo URL
- Revision: `HEAD`
- Path: `K3s/homepage`

**Destination:**
- Cluster URL: `https://kubernetes.default.svc`
- Namespace: `homepage`

Click **Create**.

ArgoCD will sync the application and show **Healthy + Synced** when complete.

---

## 9. GitOps Workflow in Practice

Once applications are set up, the workflow is:

```bash
# Make changes to a manifest locally
nano K3s/homepage/deployment.yaml

# Commit and push
git add K3s/homepage/deployment.yaml
git commit -m "feat: update homepage deployment"
git push
```

ArgoCD detects the push within 3 minutes and automatically applies the changes. No manual `kubectl apply` needed.

To force an immediate sync without waiting:
- Click **Refresh** then **Sync** in the ArgoCD UI

---

## 10. Verify Applications

All applications should show **Healthy + Synced** in the ArgoCD UI:

```
homepage          Healthy  Synced
prometheus-grafana Healthy  Synced
traefik           Healthy  Synced
```

---

## Important Notes

**Manifest cleanliness**
ArgoCD manifests must be clean declarative YAML — not live state exports. Never copy manifests directly from `kubectl get <resource> -o yaml` as these contain runtime fields (`resourceVersion`, `uid`, `creationTimestamp`, `status`) that cause sync conflicts.

A clean manifest contains only:
- `apiVersion`
- `kind`
- `metadata` (name and namespace only)
- `spec`

**System resources**
Never include system-managed resources like `kube-root-ca.crt` in your manifests. These are managed by Kubernetes itself and ArgoCD will fail trying to patch them.

**Namespace must exist**
ArgoCD does not create namespaces automatically. Always create the target namespace before creating an ArgoCD application:

```bash
sudo kubectl create namespace <namespace>
```

Or add `CreateNamespace=true` to the sync options in the application settings.
