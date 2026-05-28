# Troubleshooting — K3s Cluster

Real issues encountered during K3s cluster setup and operation. Entries document the symptom, diagnosis, resolution, and lesson learned.

---

## 2026-05-27 / 2026-05-28

### Docker install script fails on Ubuntu Server

**Symptoms**
```
curl -sfL https://get.k3s.io | sh -
```
Prompted for sudo password mid-install — this is expected behavior. K3s install script requires sudo.

**Note**
The K3s install script is different from the Docker install script. K3s uses `https://get.k3s.io` not `https://get.docker.com`. Do not confuse the two.

---

### Netplan config file not found at expected path

**Symptoms**
```
sudo nano /etc/netplan/00-installer-config.yaml
```
File not found.

**Cause**
The netplan config filename varies depending on how Ubuntu was installed. Cloud images and server installs use different default filenames.

**Resolution**
List the actual filename first:
```bash
ls /etc/netplan/
```
Then edit whichever file exists — commonly `50-cloud-init.yaml`.

**Lesson**
Always check the actual filename before attempting to edit netplan config.

---

### sudo reboot command failed with name resolution error

**Symptoms**
```
sudo -h resudo reboot
sudo: unable to resolve host resudo: Temporary failure in name resolution
```

**Cause**
Typo — `sudo -h resudo reboot` was typed instead of `sudo reboot`. The `-h` flag in sudo specifies a remote host.

**Resolution**
```bash
sudo reboot
```

**Lesson**
Always double-check commands before running them, especially ones involving reboots.

---

### Homepage Host Validation Failed

**Symptoms**
Accessing Homepage via browser returned:
```
Error
Host validation failed. See logs for more details.
```

**Cause**
Homepage v2 introduced host validation — it rejects requests from hostnames not explicitly listed in the `HOMEPAGE_ALLOWED_HOSTS` environment variable. This affects both IP:port access and domain-based access.

**Resolution**
Add the `HOMEPAGE_ALLOWED_HOSTS` environment variable to the deployment:

```yaml
env:
  - name: HOMEPAGE_ALLOWED_HOSTS
    value: "homepage.k3s.scavenger"
```

If accessing via both IP and domain, include both:
```yaml
value: "NODE-IP:PORT,homepage.k3s.scavenger"
```

Apply and restart:
```bash
sudo kubectl apply -f K3s/homepage/homepage-deployment.yaml
sudo kubectl rollout restart deployment homepage -n homepage
```

**Lesson**
Homepage v2 requires explicit host allowlisting. Always set `HOMEPAGE_ALLOWED_HOSTS` to match however you intend to access the service.

---

### NodePort assigned different port than specified

**Symptoms**
Deployment specified `nodePort: 30080` but `kubectl get svc` showed a different port assigned.

**Cause**
When a service is deleted and recreated, Kubernetes may assign a different NodePort if the requested port is outside the valid range or already in use.

**Resolution**
Always check the actual assigned port after applying:
```bash
sudo kubectl get svc -n homepage
```

Use the port shown in `PORT(S)` column, not the one in the manifest.

**Lesson**
Never hardcode a NodePort in your browser bookmark — always verify the actual assigned port after deployment.

---

### ArgoCD Sync Failed — live state dump in manifests

**Symptoms**
ArgoCD sync repeatedly failed with:
```
Operation cannot be fulfilled on deployments.apps "homepage": the object has been modified
Operation cannot be fulfilled on configmaps "kube-root-ca.crt": the object has been modified
```

**Cause**
Manifests in the GitHub repository were live state exports (from `kubectl get -o yaml`) rather than clean declarative manifests. Live state exports contain runtime fields (`resourceVersion`, `uid`, `creationTimestamp`, `status`) that conflict with ArgoCD's patching mechanism.

Additionally, one configmap manifest contained both `homepage-custom-css` AND the system-managed `kube-root-ca.crt` configmap — ArgoCD attempted to patch a system resource it has no business managing.

**Resolution**
Replace all manifests with clean declarative versions containing only:
- `apiVersion`
- `kind`
- `metadata` (name and namespace only — no uid, resourceVersion, annotations added by Kubernetes)
- `spec`

Remove any system resources (`kube-root-ca.crt`) from manifests entirely.

Example of a clean deployment manifest:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homepage
  namespace: homepage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: homepage
  template:
    metadata:
      labels:
        app: homepage
    spec:
      containers:
        - name: homepage
          image: ghcr.io/gethomepage/homepage:latest
```

**Lesson**
Never use `kubectl get <resource> -o yaml` output directly as an ArgoCD manifest. Always write manifests from scratch or use clean templates. ArgoCD manages the live state — the manifest only needs to define the desired state.

---

### ArgoCD Sync Failed — middleware annotation references non-existent CRD

**Symptoms**
ArgoCD showed Synced but `http://argocd.k3s.scavenger` returned 404.

**Cause**
The ArgoCD Ingress had a middleware annotation referencing a CRD resource that didn't exist in the cluster:
```yaml
annotations:
  traefik.ingress.kubernetes.io/router.middlewares: argocd-stripprefix@kubernetescrd
```

Traefik silently failed to process the routing rule because the referenced middleware didn't exist, resulting in 404 for all requests to that hostname.

**Resolution**
Remove the middleware annotation entirely — it was not needed:

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

**Lesson**
Traefik does not warn when a referenced middleware doesn't exist — it silently drops the route. Always verify that any CRD resources referenced in annotations actually exist in the cluster before applying ingress manifests.

---

### Fedora DNS not resolving K3s local domains

**Symptoms**
`http://homepage.k3s.scavenger` returned "Server Not Found" in Firefox despite AdGuard having the correct DNS rewrite configured.

**Cause**
Fedora was using the router (`192.168.1.1`) as its DNS server rather than AdGuard Home. The router doesn't know about the `.k3s.scavenger` local domain, so it couldn't resolve it.

Confirmed with:
```bash
resolvectl status
```
Output showed router IP as the DNS server, not AdGuard.

**Resolution**

**Option 1 — Point Fedora DNS to AdGuard via NetworkManager:**
```bash
sudo nmcli con mod "CONNECTION-NAME" ipv4.dns "ADGUARD-PRIMARY-IP ADGUARD-SECONDARY-IP"
sudo nmcli con mod "CONNECTION-NAME" ipv4.ignore-auto-dns yes
sudo nmcli con up "CONNECTION-NAME"
```

**Option 2 — Add static entries to /etc/hosts (permanent fallback):**
```bash
sudo nano /etc/hosts
```
Add one line per K3s service:
```
K3S-NODE-IP homepage.k3s.scavenger
K3S-NODE-IP argocd.k3s.scavenger
K3S-NODE-IP grafana.k3s.scavenger
K3S-NODE-IP traefik.k3s.scavenger
```

Both options were implemented — NetworkManager for primary DNS resolution, `/etc/hosts` as a permanent fallback that survives network reconnections and sleep/wake cycles.

**Lesson**
Always verify the workstation is using AdGuard as its DNS resolver before troubleshooting domain resolution issues. `/etc/hosts` is the most reliable fallback for local lab domains.

---

### Traefik dashboard returning 404

**Symptoms**
`http://traefik.k3s.scavenger` and port-forward to 9000 both returned `404 page not found`.

**Cause 1 — Wrong port**
The Traefik dashboard runs on port 8080 (the `traefik` entrypoint), not port 9000. Port 9000 is not configured by default in K3s Traefik.

**Cause 2 — Dashboard not enabled**
K3s Traefik does not expose the dashboard by default. The `--api.insecure=true` flag must be explicitly added via HelmChartConfig.

**Resolution**

Step 1 — Create HelmChartConfig to enable the dashboard:
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

Step 2 — Restart Traefik:
```bash
sudo kubectl rollout restart deployment/traefik -n kube-system
```

Step 3 — Verify dashboard is responding:
```bash
curl http://localhost:8080/api/overview
```

Step 4 — Create ingress pointing to port 8080:
```yaml
backend:
  service:
    name: traefik
    port:
      number: 8080
```

Step 5 — Access at:
```
http://traefik.k3s.scavenger/dashboard/#/
```

The trailing `#/` is required.

**Lesson**
Always verify which port a service is actually listening on before creating an ingress. Use `kubectl describe pod` to check actual container ports, and `curl localhost:PORT` to confirm the service is responding before routing external traffic to it.

---

### Helm install fails with "repo not found" when using sudo

**Symptoms**
```
sudo helm install prometheus prometheus-community/kube-prometheus-stack
Error: INSTALLATION FAILED: repo prometheus-community not found
```

**Cause**
Helm repos are stored per-user. When `helm repo add` is run as a regular user, the repo is stored in that user's home directory. Running `helm install` with `sudo` switches to the root user which has no repos configured.

**Resolution**
Run Helm without sudo but pass the K3s kubeconfig explicitly:
```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --kubeconfig /etc/rancher/k3s/k3s.yaml
```

If the kubeconfig isn't readable:
```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

**Lesson**
Helm repos are user-scoped. Never mix sudo and non-sudo Helm commands. Either run everything as the same user, or copy the kubeconfig to a location accessible by your user.

---

### Grafana pod restarting with SQLite database locked error

**Symptoms**
Grafana pod showed `2/3` containers ready and restarted once. Logs showed:
```
database is locked (5) (SQLITE_BUSY)
```

**Cause**
During startup, multiple Grafana processes briefly attempted to access the SQLite database simultaneously, causing a lock contention. This is a known Grafana startup race condition.

**Resolution**
No action required. The lock resolves itself within 2-3 minutes as Grafana finishes initialization. The pod restarted once and came up healthy (`3/3 Running`) automatically.

**Lesson**
Not every pod restart requires intervention. Check if the pod eventually reaches healthy status before troubleshooting. A single restart during initial startup is often a transient race condition, not a persistent error.

---

## Known Issues

**ArgoCD ApplicationSet CRD errors in logs**
ArgoCD logs repeatedly show:
```
failed to list *v1alpha1.ApplicationSet: the server could not find the requested resource
```
This indicates the ApplicationSet CRD may not be fully installed. The UI and basic Application management work correctly despite this error. ApplicationSets (a more advanced ArgoCD feature) are not currently in use.

**Traefik dashboard requires authentication**
The dashboard is currently exposed without authentication (`--api.insecure=true`). For a production environment, add a BasicAuth middleware. For a LAN-only homelab this is acceptable.
