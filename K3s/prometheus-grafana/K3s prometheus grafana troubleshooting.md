# Troubleshooting — Prometheus & Grafana

Real issues encountered during Prometheus and Grafana deployment and configuration on K3s. Every issue here was hit in production.

---

## Helm & Deployment Issues

### Grafana password reset on every Helm upgrade

**Symptoms**
After running `helm upgrade`, Grafana rejects the previously set password. The admin password reverts to the default value stored in the Helm release.

**Cause**
Helm stores the `adminPassword` value in the release. If `--set grafana.adminPassword=` is not included on every upgrade, Helm uses whatever was set during the initial install. Additionally, changing the password through the Grafana UI doesn't update the Helm release value — so the next upgrade overwrites it.

**Resolution**
Always include `--set grafana.adminPassword=YOUR_PASSWORD` on every `helm upgrade` command. Without it, the password resets to the Helm-managed value.

The permanent fix is to store the password in a Kubernetes secret and reference it:
```bash
kubectl create secret generic grafana-admin-secret \
  --from-literal=admin-password=YOUR_PASSWORD \
  -n monitoring
```

Then reference it in the Helm values instead of passing it as a flag.

**Lesson**
Never change Grafana credentials only through the UI when deployed via Helm. Always update the Helm values to match, or use a Kubernetes secret as the source of truth.

---

### Helm repo not found when using sudo

**Symptoms**
```
Error: INSTALLATION FAILED: repo prometheus-community not found
```

**Cause**
Helm repos are stored per-user in `~/.config/helm/repositories.yaml`. Running `helm repo add` as a regular user stores the repo for that user. Running `helm install` or `helm upgrade` with `sudo` switches to root, which has no repos configured.

**Resolution**
Run all Helm commands without sudo, passing the K3s kubeconfig explicitly:
```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --kubeconfig /etc/rancher/k3s/k3s.yaml
```

If the kubeconfig isn't readable by your user:
```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

**Lesson**
Never mix sudo and non-sudo Helm commands. All Helm operations should run as the same user. Use `--kubeconfig` to point to the K3s config file instead of relying on sudo.

---

### Grafana pod CrashLoopBackOff — Permission denied on PVC

**Symptoms**
New Grafana pod shows `Init:CrashLoopBackOff`. Logs show:
```
chown: /var/lib/grafana/csv: Permission denied
chown: /var/lib/grafana/pdf: Permission denied
chown: /var/lib/grafana/png: Permission denied
```

**Cause**
The Grafana Helm chart includes an init container (`busybox`) that runs `chown` on the PVC data directory to ensure correct ownership. When the PVC already has data owned by a different UID (from a previous pod), the init container fails if it doesn't have sufficient permissions.

This commonly occurs after a Helm upgrade creates a new pod that tries to take ownership of a PVC previously used by another pod.

**Resolution**
Disable the init chown container:
```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.initChownData.enabled=false \
  --kubeconfig /etc/rancher/k3s/k3s.yaml
```

Adding `grafana.initChownData.enabled=false` skips the ownership check. Grafana will still function correctly — it just won't attempt to chown the data directory on startup.

**Lesson**
When using persistent storage with Grafana on K3s, disable `initChownData` to avoid permission conflicts. The init container is only needed in environments where the PVC ownership needs to be set on first run.

---

### Two Grafana pods stuck — ReadWriteOnce PVC conflict

**Symptoms**
After a Helm upgrade, two Grafana pods exist simultaneously. The new pod is in `Init:CrashLoopBackOff` while the old pod is still `Running`. The new pod logs show permission errors on the PVC.

**Cause**
The PVC uses `ReadWriteOnce` (RWO) access mode — only one pod can mount it at a time. During a rolling update, the new pod can't fully initialize because the old pod still has the PVC mounted. The init container fails trying to access the locked PVC.

**Resolution**
Manually delete the old pod to release the PVC:
```bash
kubectl delete pod POD_NAME -n monitoring
```

The new pod will then successfully mount the PVC and complete initialization.

**Lesson**
Grafana's RWO PVC means rolling updates don't work cleanly. When upgrading Grafana via Helm, be prepared to manually delete the old pod if the new one gets stuck. Brief downtime is unavoidable with RWO storage.

---

## Access & Authentication Issues

### Grafana anonymous access not persisting after Helm upgrade

**Symptoms**
Anonymous access was configured but after a Helm upgrade, Grafana redirects all visitors to the login page. The setting was lost.

**Cause**
The `grafana.ini` approach for anonymous access (`--set grafana.grafana\.ini.auth\.anonymous.enabled=true`) doesn't always persist correctly across upgrades. The ini file gets regenerated from the Helm values on each upgrade.

**Resolution**
Use environment variables instead of grafana.ini settings — they are more reliably applied:
```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.env.GF_AUTH_ANONYMOUS_ENABLED=true \
  --set grafana.env.GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer \
  --kubeconfig /etc/rancher/k3s/k3s.yaml
```

Verify the env var is actually set in the running pod:
```bash
kubectl exec -n monitoring GRAFANA_POD -- env | grep GF_AUTH
```

**Lesson**
For Grafana configuration in Kubernetes, environment variables (`GF_*`) are more reliable than `grafana.ini` settings when deployed via Helm. Always verify configuration is applied by checking the pod's actual environment variables.

---

### Grafana "origin not allowed" error when saving settings

**Symptoms**
When trying to save playlists, preferences, or other settings in Grafana:
```
Unable to update playlist
origin not allowed
```

**Cause**
Grafana 13 introduced stricter CSRF (Cross-Site Request Forgery) protection. When accessed via a public domain (`k3s-grafana.scavenger.pro`) but Grafana doesn't know its own public URL, it rejects requests whose origin doesn't match its configured root URL.

**Resolution**
Set the root URL and trusted origins via environment variables:
```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.env.GF_SERVER_ROOT_URL=https://k3s-grafana.scavenger.pro \
  --set grafana.env.GF_SECURITY_CSRF_TRUSTED_ORIGINS=k3s-grafana.scavenger.pro \
  --kubeconfig /etc/rancher/k3s/k3s.yaml
```

**Lesson**
When exposing Grafana via a public domain, always set `GF_SERVER_ROOT_URL` to match the public URL. Without this, Grafana's CSRF protection rejects requests from the public domain as potentially malicious.

---

### Grafana dashboards showing "No data"

**Symptoms**
Pre-built Kubernetes dashboards load but all panels show "No data".

**Cause — Option 1: Missing namespace/pod filter**
Pod and namespace-level dashboards require selecting a specific namespace or pod from the dropdown filters at the top of the dashboard. Without a selection, no data is returned.

**Cause — Option 2: Prometheus targets down**
If Prometheus can't reach its scrape targets, no metrics are collected and dashboards show no data.

**Diagnosis**
Check Prometheus targets at `http://NODE-IP:9090/targets` (requires port-forward):
```bash
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090 --address 0.0.0.0
```

All targets should show `1/1 up`.

**Resolution**
- For filter issues: select the appropriate namespace/node from the dashboard dropdowns
- For target issues: investigate why specific targets are down using the Prometheus targets page

**Lesson**
The `Kubernetes / Compute Resources / Cluster` dashboard works without filters and shows cluster-wide data. Use this as the default demo dashboard. Pod and namespace-level dashboards require filter selection.

---

### Grafana not accessible after port-forward is running

**Symptoms**
After running `kubectl port-forward` for Prometheus (port 9090), Grafana becomes inaccessible externally.

**Cause**
Port-forward binds to the specified address and can interfere with other network operations on the node. Running with `--address 0.0.0.0` exposes the port on all interfaces which can affect routing.

**Resolution**
Stop the port-forward with `Ctrl+C` after finishing diagnosis. Grafana access restores immediately after the port-forward is terminated.

**Lesson**
Use port-forward only for temporary diagnostic access. Always terminate it when done — don't leave it running in the background.

---

## Complete Working Helm Upgrade Command

The following command includes all required settings for a stable Grafana deployment:

```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.env.GF_SERVER_ROOT_URL=https://k3s-grafana.scavenger.pro \
  --set grafana.env.GF_SECURITY_CSRF_TRUSTED_ORIGINS=k3s-grafana.scavenger.pro \
  --set grafana.env.GF_AUTH_ANONYMOUS_ENABLED=true \
  --set grafana.env.GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.size=1Gi \
  --set grafana.adminPassword=YOUR_PASSWORD \
  --set grafana.initChownData.enabled=false \
  --kubeconfig /etc/rancher/k3s/k3s.yaml
```

Save this command somewhere safe and use it for all future upgrades. Replace `YOUR_PASSWORD` with your actual admin password.
