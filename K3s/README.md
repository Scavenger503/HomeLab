# K3s — Kubernetes Cluster

A lightweight Kubernetes cluster running on a dedicated Lenovo ThinkCentre M715q, managed via GitOps using ArgoCD. This cluster serves as a hands-on learning environment for Kubernetes and a portfolio demonstration of production-grade container orchestration practices.

---

## Hardware

| Component | Detail |
|---|---|
| Device | Lenovo ThinkCentre M715q |
| CPU | AMD PRO A12-9800E (4 cores) |
| RAM | 8GB (maximum supported) |
| OS | Ubuntu Server 22.04.5 LTS |
| IP | Static — assigned via router DHCP reservation |
| Hostname | k3s-node-01 |
| K3s Version | v1.35.5+k3s1 |

---

## Architecture

```
GitHub Repository (source of truth)
    │
    ▼
ArgoCD (GitOps controller)
    │
    ├── Homepage (dashboard)
    ├── Prometheus + Grafana (monitoring)
    └── Traefik (ingress controller + dashboard)

Traefik (ingress)
    │
    ├── homepage.k3s.scavenger → Homepage
    ├── argocd.k3s.scavenger → ArgoCD
    ├── grafana.k3s.scavenger → Grafana
    └── traefik.k3s.scavenger → Traefik Dashboard

AdGuard Home (DNS)
    └── *.k3s.scavenger → 192.168.1.190 (K3s node)
```

All services are accessible via local DNS only — no public exposure. External access will be handled via Cloudflare Tunnel.

---

## Deployed Applications

| Application | Namespace | Local URL | Public URL | Managed By |
|---|---|---|---|---|
| Homepage | homepage | http://homepage.k3s.scavenger | https://k3s-homepage.scavenger.pro | ArgoCD |
| ArgoCD | argocd | http://argocd.k3s.scavenger | https://k3s-argocd.scavenger.pro | Manual |
| Grafana | monitoring | http://grafana.k3s.scavenger | https://k3s-grafana.scavenger.pro | ArgoCD |
| Prometheus | monitoring | Internal only | — | ArgoCD |
| Traefik Dashboard | kube-system | http://traefik.k3s.scavenger/dashboard/#/ | — | ArgoCD |

---

## GitOps Workflow

This cluster uses ArgoCD for GitOps-based deployments. The workflow is:

1. Make changes to manifests in the `K3s/` folder
2. Commit and push to GitHub
3. ArgoCD detects the change automatically
4. ArgoCD syncs the cluster to match the desired state in GitHub

No manual `kubectl apply` is needed for managed applications. The repository is the single source of truth.

---

## Monitoring

The cluster runs the `kube-prometheus-stack` Helm chart providing:

- **Prometheus** — metrics collection and storage (7 day retention)
- **Grafana** — visualization and dashboards
- **Alertmanager** — alert routing
- **Node Exporter** — hardware-level metrics
- **Kube State Metrics** — Kubernetes object metrics

Pre-built dashboards available:
- Kubernetes / Compute Resources / Cluster
- Node Exporter / Nodes
- Kubernetes / Persistent Volumes
- Prometheus / Overview

---

## DNS Configuration

All K3s services use local DNS rewrites configured in AdGuard Home. The pattern is:

```
<service>.k3s.scavenger → K3S-NODE-IP
```

Each service is then routed by Traefik based on the hostname to the correct backend service.

---

## Repository Structure

```
K3s/
├── README.md
├── argocd/          ← ArgoCD manifests
├── homepage/        ← Homepage deployment, service, ingress, configmap
├── prometheus-grafana/ ← Grafana ingress
├── traefik/         ← Traefik dashboard ingress
└── docs/            ← Setup guides and troubleshooting
```

---

## Certifications Context

This cluster is part of active preparation for:
- Certified Kubernetes Administrator (CKA)
- HashiCorp Terraform Associate
- AWS Cloud Practitioner
