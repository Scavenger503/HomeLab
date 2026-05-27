# 🚀 K3s Homelab Cluster

A production-style Kubernetes cluster running on a Lenovo M900 Tiny, managed with K3s.

## Cluster Info
- **Node:** k3s-node-01
- **K3s Version:** v1.32+
- **Architecture:** Single-node cluster (expanding)

## Deployed Projects

| Project | Namespace | Status |
|---------|-----------|--------|
| Homepage | homepage | ✅ Running |
| Traefik Ingress | kube-system | 🔄 Configuring |
| ArgoCD | argocd | 📋 Planned |
| Prometheus + Grafana | monitoring | 📋 Planned |

## Structure
- `homepage/` - K3s DevOps Lab dashboard
- `traefik/` - Ingress controller configuration
- `argocd/` - GitOps continuous deployment
- `prometheus-grafana/` - Monitoring stack
