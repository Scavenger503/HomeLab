# Troubleshooting — Cloudflare Tunnel & External Access

Real issues encountered while setting up Cloudflare Tunnel for external access to K3s services. Every issue here was hit in production during the initial setup.

---

## 2026-05-29

### SSL_ERROR_NO_CYPHER_OVERLAP — TLS handshake failing at Cloudflare edge

**Symptoms**
```
error:0A000410:SSL routines::sslv3 alert handshake failure
```
Browser showed: `Firefox can't create a secure connection. Error Code: SSL_ERROR_NO_CYPHER_OVERLAP`
Phone showed: `A TLS error caused the secure connection to fail`

**Cause**
Cloudflare's universal SSL certificate (`*.scavenger.pro`) does not cover multi-level subdomains. The original domain pattern `homepage.k3s.scavenger.pro` has three levels of subdomains which are not covered by a single-level wildcard certificate.

Cloudflare's free universal certificate covers:
- `scavenger.pro` ✅
- `*.scavenger.pro` ✅ (one level — e.g. `homepage.scavenger.pro`)
- `*.k3s.scavenger.pro` ❌ (two levels — requires Advanced Certificate Manager at $10/month)

**Resolution**
Renamed all K3s public hostnames from multi-level to single-level subdomains:

| Before | After |
|---|---|
| `homepage.k3s.scavenger.pro` | `k3s-homepage.scavenger.pro` |
| `argocd.k3s.scavenger.pro` | `k3s-argocd.scavenger.pro` |
| `grafana.k3s.scavenger.pro` | `k3s-grafana.scavenger.pro` |

Updated:
1. Cloudflare DNS records
2. Cloudflare Tunnel public hostname entries
3. Cloudflare Configuration Rule expression

**Lesson**
Cloudflare's free wildcard cert only covers one level of subdomains. Plan your domain structure before setting up tunnels — use `service.domain.com` not `service.category.domain.com`.

---

### Cloudflare Configuration Rule not matching hostnames

**Symptoms**
Created a Configuration Rule with `*.k3s.scavenger.pro` to override SSL mode from Full (Strict) to Full, but TLS errors persisted.

**Cause**
Cloudflare wildcard rules in Configuration Rules only match one subdomain level. `*.k3s.scavenger.pro` does not match `homepage.k3s.scavenger.pro` because the wildcard only covers one level.

**Resolution**
Changed the rule expression to explicitly list each hostname:
```
(http.host eq "homepage.k3s.scavenger.pro") or (http.host eq "argocd.k3s.scavenger.pro") or (http.host eq "grafana.k3s.scavenger.pro")
```

This was ultimately unnecessary after switching to single-level subdomains, but the lesson applies to any multi-level domain situation.

**Lesson**
Cloudflare wildcard matching in rules has the same limitation as SSL certificates — one level only. Use explicit hostname matching or restructure your domain hierarchy.

---

### Cloudflare Full (Strict) SSL blocking tunnel connections

**Symptoms**
TLS handshake failure even after the tunnel was confirmed healthy and connected. No requests appearing in cloudflared logs when accessing the public URL.

**Cause**
The domain `scavenger.pro` had SSL/TLS mode set to **Full (Strict)**. This mode requires a valid, trusted certificate on the origin server. The Cloudflare Origin Certificate installed in Traefik is only trusted by Cloudflare — browsers and other clients don't trust it directly.

With Full (Strict), Cloudflare verifies the origin certificate before forwarding traffic. When the certificate validation failed, Cloudflare dropped the connection before it ever reached cloudflared.

**Resolution**
Two approaches were attempted:

**Attempt 1 — Configuration Rule (failed due to multi-level subdomain issue above)**

**Attempt 2 — Switch to single-level subdomains**
By moving to `k3s-homepage.scavenger.pro`, the standard `*.scavenger.pro` wildcard certificate applied, and Full (Strict) mode worked correctly since Cloudflare's own certificate was being used end-to-end.

**Lesson**
Full (Strict) SSL mode is the most secure option but requires careful planning. If using Cloudflare Tunnel, ensure your subdomain structure is covered by the available certificates before configuring the tunnel.

---

### No traffic appearing in cloudflared logs

**Symptoms**
Accessed public URL from phone but zero new lines appeared in `kubectl logs -n cloudflared deployment/cloudflared -f`. Connection failed with network error on the phone.

**Cause**
The TLS handshake was failing at Cloudflare's edge before traffic reached the tunnel. Cloudflared never received the request because Cloudflare dropped it during SSL negotiation.

**Diagnosis**
```bash
curl -v https://k3s-homepage.scavenger.pro 2>&1 | grep -E "TLS|SSL|error"
```
Output showed `TLS alert, handshake failure` — confirmed the failure was at Cloudflare's edge, not at the tunnel or origin.

**Resolution**
Fixed the underlying SSL certificate coverage issue (see above).

**Lesson**
If cloudflared logs show zero activity when accessing a public URL, the problem is upstream of the tunnel — at Cloudflare's edge. Use `curl -v` against the public URL to diagnose where the TLS handshake fails.

---

### Traefik returning 404 after adding websecure entrypoint

**Symptoms**
After updating ingress annotations to include `websecure` entrypoint:
```yaml
traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
traefik.ingress.kubernetes.io/router.tls: "true"
```
HTTP requests to Traefik started returning 404. Local access via `http://service.k3s.scavenger` stopped working.

**Cause**
Adding TLS to the ingress caused Traefik to route HTTPS traffic correctly but HTTP requests no longer matched the route. Traefik's behavior with `router.tls: "true"` varies — in some configurations it redirects HTTP to HTTPS, in others it simply stops serving HTTP for that route.

**Resolution**
Use HTTPS for all internal access after enabling websecure:
```
https://service.k3s.scavenger
```

For the Cloudflare tunnel, set the service URL to `https://TRAEFIK-CLUSTER-IP` with **No TLS Verify** enabled in the tunnel public hostname settings. This allows cloudflared to connect to Traefik over HTTPS without validating the self-signed certificate.

**Lesson**
Once TLS is enabled on a Traefik ingress, all access (local and external) should use HTTPS. Update browser bookmarks, `/etc/hosts` references, and tunnel configurations accordingly.

---

### ArgoCD reverting manual kubectl changes

**Symptoms**
Updated ingress annotations with `kubectl apply` but Traefik kept showing the old configuration. ArgoCD was silently reverting the changes back to whatever was in the GitHub repository.

**Cause**
ArgoCD is configured with **Self Heal** enabled — it continuously reconciles the cluster state with the GitHub repository. Any manual `kubectl apply` that differs from the repo is automatically reverted within minutes.

**Resolution**
Always make changes to manifests in the GitHub repository, not directly with kubectl. The correct workflow:

1. Edit the file locally
2. `git add`, `git commit`, `git push`
3. ArgoCD detects the change and syncs automatically

If an immediate change is needed, apply with kubectl AND update the repo simultaneously to prevent reversion.

**Lesson**
When using GitOps, the repository is the single source of truth. Manual kubectl changes are temporary — ArgoCD will revert them. Never troubleshoot by making kubectl changes without also updating the repo.

---

### Tunnel token exposed in plaintext manifest

**Symptoms**
After creating the cloudflared deployment manifest, the Cloudflare Tunnel token was stored in plaintext in the YAML file. Committing this to a public GitHub repository would expose the token.

**Resolution**
Before committing, replace the token with a placeholder:
```yaml
args:
  - tunnel
  - --no-autoupdate
  - run
  - --token
  - YOUR_TUNNEL_TOKEN_HERE
```

**Proper solution** — store the token as a Kubernetes secret and reference it in the deployment:
```bash
kubectl create secret generic cloudflare-tunnel-token \
  --from-literal=token=YOUR_ACTUAL_TOKEN \
  -n cloudflared
```

Then reference in the deployment:
```yaml
env:
  - name: TUNNEL_TOKEN
    valueFrom:
      secretKeyRef:
        name: cloudflare-tunnel-token
        key: token
```

**Lesson**
Never commit secrets, tokens, or credentials to a public repository. Always use Kubernetes secrets or environment variable injection for sensitive values. Check every manifest for plaintext credentials before pushing.

---

### 403 bot challenge when testing with curl

**Symptoms**
After fixing all TLS issues, `curl https://k3s-homepage.scavenger.pro` returned 403 with Cloudflare's "Just a moment... Enable JavaScript and cookies" challenge page.

**Cause**
Cloudflare's bot protection identified curl as automated traffic and served a managed challenge. This is expected behavior — curl doesn't have JavaScript or cookies so it cannot pass the challenge.

**Resolution**
No action required. The 403 from curl confirmed the tunnel and TLS were working correctly. Real browsers (Chrome, Firefox, Safari) automatically pass the challenge and receive the actual page content.

**Lesson**
A 403 with `cf-mitigated: challenge` header from curl does not indicate a problem. Test with a real browser to confirm the service is accessible to actual users.

---

## Summary of Working Configuration

**Cloudflare Tunnel public hostname settings:**
- **Type:** HTTP
- **URL:** `TRAEFIK-CLUSTER-IP` (internal cluster IP, port 80 omitted)
- **No TLS Verify:** Enabled
- **HTTP Host Header:** `service.k3s.scavenger` (internal LAN hostname)
- **Origin Server Name:** Not set

**Domain structure:**
- Use single-level subdomains: `k3s-service.domain.com`
- Avoid multi-level: `service.k3s.domain.com` (not covered by wildcard cert)

**SSL/TLS mode:** Full (Strict) — works correctly with single-level subdomains covered by the universal wildcard certificate.
