# Prometheus + Grafana Setup Guide

Step-by-step guide for deploying the `kube-prometheus-stack` Helm chart on K3s, providing full cluster monitoring with Prometheus, Grafana, Alertmanager, Node Exporter, and Kube State Metrics.

---

## What is the kube-prometheus-stack?

The `kube-prometheus-stack` is a Helm chart that bundles the complete Kubernetes monitoring stack into a single deployment:

| Component | Purpose |
|---|---|
| Prometheus | Metrics collection and storage engine |
| Grafana | Visualization and dashboards |
| Alertmanager | Alert routing and notification delivery |
| Node Exporter | Hardware-level metrics (CPU, RAM, disk, network) |
| Kube State Metrics | Kubernetes object metrics (pod counts, deployment status) |
| Prometheus Operator | Manages Prometheus instances declaratively |

---

<img width="1624" height="1308" alt="image" src="https://github.com/user-attachments/assets/b7b34fdd-4c3b-428a-834d-7e695a5d20d8" />


## Prerequisites

- K3s cluster running and healthy
- Helm installed
- `kubectl` access to the cluster
- DNS rewrite configured for `grafana.k3s.scavenger`

---

## 1. Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

---

## 2. Add Prometheus Community Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## 3. Create Monitoring Namespace

```bash
sudo kubectl create namespace monitoring
```

---

## 4. Deploy kube-prometheus-stack

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=YOUR_PASSWORD \
  --set prometheus.prometheusSpec.retention=7d \
  --kubeconfig /etc/rancher/k3s/k3s.yaml
```

> **Note:** The `--kubeconfig` flag is required when running Helm without sudo on K3s. The default kubeconfig at `/etc/rancher/k3s/k3s.yaml` must be readable by your user. Fix permissions if needed:
> ```bash
> sudo chmod 644 /etc/rancher/k3s/k3s.yaml
> ```

Replace `YOUR_PASSWORD` with a strong Grafana admin password.

---

## 5. Verify Deployment

Watch pods come up:

```bash
sudo kubectl get pods -n monitoring -w
```

All pods should reach `Running` status:

```
NAME                                                     READY   STATUS
alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running
prometheus-grafana-xxx                                   3/3     Running
prometheus-kube-prometheus-operator-xxx                  1/1     Running
prometheus-kube-state-metrics-xxx                        1/1     Running
prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running
prometheus-prometheus-node-exporter-xxx                  1/1     Running
```

> **Note:** Grafana may show `ImagePullBackOff` initially while pulling the image, and may restart once due to a SQLite database lock during startup. This resolves automatically within 2-3 minutes — it is not an error requiring intervention.

---

## 6. Expose Grafana via Ingress

Create the Ingress manifest at `K3s/prometheus-grafana/grafana-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: grafana.k3s.scavenger
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-grafana
                port:
                  number: 80
```

Apply it:

```bash
sudo kubectl apply -f K3s/prometheus-grafana/grafana-ingress.yaml
```

---

## 7. Access Grafana

Navigate to `http://grafana.k3s.scavenger` in your browser.

- **Username:** `admin`
- **Password:** the password set during Helm install

Change the password immediately after first login.

---

## 8. Explore Pre-built Dashboards

The kube-prometheus-stack comes with pre-built dashboards. Navigate to **Dashboards → Browse** to find:

**Recommended for cluster overview:**
- `Kubernetes / Compute Resources / Cluster` — CPU, memory, network per namespace
- `Node Exporter / Nodes` — physical hardware metrics
- `Kubernetes / Persistent Volumes` — storage health
- `Prometheus / Overview` — Prometheus health and metrics ingestion rate

---

## 9. Create a Demo Playlist

For employer demonstrations, create a Grafana playlist that cycles through key dashboards automatically:

1. Go to **Dashboards → Playlists → New Playlist**
2. Name: `K3s Cluster Overview`
3. Interval: `30s`
4. Add dashboards:
   - Kubernetes / Compute Resources / Cluster
   - Node Exporter / Nodes
   - Kubernetes / Persistent Volumes
   - Prometheus / Overview
5. Click **Save**

Start the playlist before a demo — it cycles through all dashboards full screen automatically.

---

## 10. Add to ArgoCD

Create an ArgoCD application to manage the Grafana ingress via GitOps:

- **Application Name:** `prometheus-grafana`
- **Path:** `K3s/prometheus-grafana`
- **Namespace:** `monitoring`

> **Note:** The full Prometheus stack was deployed via Helm directly, not via ArgoCD manifests. Only the Grafana Ingress is managed by ArgoCD. Managing the full Helm release via ArgoCD is a future enhancement.

---

## Resource Usage

Typical memory consumption on an 8GB node:

| Component | Memory |
|---|---|
| Prometheus | ~300-500MB |
| Grafana | ~100-200MB |
| Alertmanager | ~50MB |
| Node Exporter | ~20-50MB |
| Kube State Metrics | ~50MB |
| **Total** | ~500-800MB |

At 38% total cluster memory utilization (including all other workloads), the monitoring stack fits comfortably on an 8GB node.

---

## Notes

- Prometheus retention is set to 7 days — adjust based on available storage
- Grafana uses SQLite by default for its internal database — sufficient for single-node homelab use
- The `Memory Requests %` showing over 100% in dashboards is expected — it means actual usage exceeds the requested amount set in the Helm chart, not that the node is out of memory
- To upgrade the stack: `helm upgrade prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --kubeconfig /etc/rancher/k3s/k3s.yaml`
