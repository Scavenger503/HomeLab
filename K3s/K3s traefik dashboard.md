# Traefik Dashboard Setup Guide

Step-by-step guide for enabling and exposing the Traefik dashboard on a K3s cluster. K3s ships with Traefik as the default ingress controller but the dashboard is disabled by default.

---

## What is Traefik?

Traefik is a modern reverse proxy and ingress controller that automatically discovers services and routes traffic to them. In K3s it acts as the entry point for all external traffic into the cluster.

**What Traefik handles:**
- HTTP/HTTPS routing based on hostname and path rules
- TLS termination
- Load balancing
- Middleware (authentication, rate limiting, redirects)

The Traefik dashboard provides a real-time view of all configured routes, services, and middlewares.

---

## Prerequisites

- K3s cluster running with Traefik installed (default in K3s)
- `kubectl` access to the cluster
- DNS rewrite configured for `traefik.k3s.scavenger`

---

## 1. Verify Traefik is Running

```bash
sudo kubectl get pods -n kube-system | grep traefik
sudo kubectl get svc -n kube-system | grep traefik
```

Expected service output:
```
traefik   LoadBalancer   CLUSTER-IP   NODE-IP   80:PORT/TCP,443:PORT/TCP
```

Traefik is bound to the node IP on ports 80 and 443.

---

## 2. Check Traefik Default Configuration

```bash
sudo kubectl describe pod -n kube-system $(sudo kubectl get pods -n kube-system | grep "traefik-" | grep -v "helm\|svclb" | awk '{print $1}') | grep -A15 "Args"
```

By default K3s Traefik runs with:
- `--entryPoints.traefik.address=:8080` — internal dashboard entrypoint
- `--entryPoints.web.address=:8000` — HTTP traffic
- `--entryPoints.websecure.address=:8443` — HTTPS traffic
- No `--api.insecure=true` — dashboard not accessible externally

---

## 3. Enable the Dashboard

K3s manages Traefik configuration via a `HelmChartConfig` resource. Create one to enable the dashboard:

```bash
cat > /tmp/traefik-dashboard.yaml << 'EOF'
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    dashboard:
      enabled: true
    additionalArguments:
      - "--api.insecure=true"
    ports:
      traefik:
        expose:
          default: true
EOF
sudo kubectl apply -f /tmp/traefik-dashboard.yaml
```

Wait for K3s to apply the Helm chart change (watch for a new `helm-install-traefik` pod to complete):

```bash
sudo kubectl get pods -n kube-system -w | grep helm-install-traefik
```

Then restart Traefik to apply the new configuration:

```bash
sudo kubectl rollout restart deployment/traefik -n kube-system
```

Wait for the new pod to be running:

```bash
sudo kubectl get pods -n kube-system | grep "traefik-"
```

---

## 4. Verify Dashboard is Running

Test the dashboard API locally on the K3s node:

```bash
curl http://localhost:8080/api/overview
```

Expected output (JSON with router and service counts):
```json
{"http":{"routers":{"total":7,"warnings":0,"errors":0},"services":{"total":8,"warnings":0,"errors":0},...}}
```

If this returns data the dashboard is working. If it returns `404 page not found` the configuration change hasn't applied yet — wait and retry.

---

## 5. Create Ingress for Dashboard

Create the Ingress manifest at `K3s/traefik/traefik-dashboard-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-dashboard
  namespace: kube-system
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: traefik.k3s.scavenger
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: traefik
                port:
                  number: 8080
```

Apply it:

```bash
sudo kubectl apply -f K3s/traefik/traefik-dashboard-ingress.yaml
```

---

## 6. Access the Dashboard

Navigate to:

```
http://traefik.k3s.scavenger/dashboard/#/
```

> **Important:** The trailing `#/` is required. Without it Traefik returns a redirect that may not resolve correctly.

The dashboard shows:
- **Entrypoints** — configured ports (metrics, traefik, web, websecure)
- **HTTP Routers** — all Ingress rules currently active
- **HTTP Services** — backend services Traefik is routing to
- **HTTP Middlewares** — active middleware configurations
- **Features** — tracing, metrics, and access log status
- **Providers** — KubernetesIngress, KubernetesCRD

---

## 7. Add to ArgoCD

Create an ArgoCD application to manage the Traefik ingress via GitOps:

- **Application Name:** `traefik`
- **Path:** `K3s/traefik`
- **Namespace:** `kube-system`

---

## Understanding the Dashboard

**HTTP Routers** — each Ingress rule you create appears here. Clicking a router shows the full routing rule including host, path, entrypoint, and backend service.

**HTTP Services** — the backend services Traefik routes traffic to. Each Kubernetes Service referenced by an Ingress appears here.

**Providers** — shows which Kubernetes providers Traefik is using to discover routes:
- `KubernetesIngress` — standard Kubernetes Ingress resources
- `KubernetesCRD` — Traefik-specific CRD resources (IngressRoute, Middleware, etc.)

---

## Notes

- The `--api.insecure=true` flag exposes the dashboard without authentication. For a production environment add authentication middleware. For a homelab this is acceptable since the dashboard is LAN-only.
- The Traefik dashboard is read-only — it shows configuration but cannot be used to make changes.
- Changes to Traefik configuration via `HelmChartConfig` trigger a Helm reconciliation and pod restart automatically — no manual restart needed after the initial setup.
- Never add middleware annotations referencing non-existent CRD resources to ingress objects. This silently breaks routing without obvious error messages.
