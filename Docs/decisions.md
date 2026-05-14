# Architecture Decisions

A record of significant infrastructure decisions — what was chosen, what was considered, and why. Useful for understanding the reasoning behind the current setup and for revisiting decisions as requirements change.

---

## Cloudflare Tunnels over a self-hosted reverse proxy

**Decision:** Use Cloudflare Tunnels for all external access instead of a self-hosted reverse proxy such as Nginx Proxy Manager or Traefik.

**Considered alternatives:**
- Nginx Proxy Manager with port forwarding
- Traefik with Let's Encrypt
- Caddy with automatic HTTPS

**Reasoning:**
Cloudflare Tunnels require no open inbound ports on the router. The tunnel initiates an outbound connection from inside the network to Cloudflare's edge — no port forwarding rules, no exposure of the home IP address, no attack surface at the router level. SSL termination, WAF protection, and DDoS mitigation are included at no cost.

Self-hosted reverse proxies require at minimum ports 80 and 443 to be open, which exposes the home network's public IP and requires managing Let's Encrypt certificate renewal, firewall rules, and upstream security independently.

The trade-off is dependency on Cloudflare as an intermediary. Services that must remain reachable even if Cloudflare has an outage are kept LAN-only and not tunneled.

---

## One Cloudflare Tunnel container per service group

**Decision:** Run each Cloudflare Tunnel as a separate Docker container rather than consolidating all services into a single tunnel with multiple ingress rules.

**Considered alternatives:**
- Single cloudflared container with a multi-ingress config file
- Grouped tunnels (one per host machine)

**Reasoning:**
This decision came directly from a real incident. A Cloudflare WAF rule blocked POST requests to a specific webhook endpoint mid-deployment. If all services had shared a single tunnel, the WAF intervention could have affected all external services simultaneously. Running separate tunnel containers provides blast radius isolation — a blocked or misconfigured tunnel affects only the services it serves, not the entire external-facing infrastructure.

The trade-off is resource overhead from running multiple cloudflared containers. Each container uses approximately 25-50MB of RAM, which is acceptable given the isolation benefit.

---

## Dedicated physical machine for Wazuh SIEM

**Decision:** Run Wazuh SIEM on a separate physical machine (M715q) rather than as containers on the primary Docker host (M710q).

**Considered alternatives:**
- Wazuh containers on the M710q alongside other services
- Wazuh in a Proxmox VM

**Reasoning:**
A SIEM monitoring production workloads should not compete with those workloads for resources. Running Wazuh on the same host it monitors creates two problems: resource contention under load, and a single point of failure where a crashed production host also takes down the security monitoring.

Physical separation ensures the SIEM stays operational regardless of what happens on the production host. It also maintains a cleaner security boundary — the monitoring system is not co-located with the systems it watches.

The M715q is also used as the primary DevOps and certification study machine, which fits naturally alongside a security-focused role.

---

## vDSM for monitoring workloads instead of running directly on DSM

**Decision:** Run monitoring containers (Uptime Kuma, Ntopng) inside a vDSM virtual machine rather than directly on the Synology DSM host.

**Considered alternatives:**
- Running containers directly in Synology Container Manager on DSM
- Running monitoring on the M710q
- Installing community packages from the Synology Package Center

**Reasoning:**
Running workloads directly on DSM risks destabilizing the NAS operating system. A misbehaving container or a failed update cannot crash DSM or interrupt file services and backup jobs if it runs inside a VM. The vDSM provides a proper isolation boundary between monitoring workloads and NAS duties.

Community packages from the Synology Package Center were evaluated but rejected. Package maintenance is inconsistent — some packages are years behind the official upstream versions — and they install into the DSM root filesystem in ways that are difficult to cleanly reverse.

The vDSM approach also provides a standard Linux-like environment with full Docker support, consistent with how containers are managed everywhere else in the homelab.

---

## Portainer Agent over standalone Portainer per host

**Decision:** Deploy Portainer Agent on secondary hosts and connect them to a central Portainer CE instance rather than running a full Portainer installation on each host.

**Considered alternatives:**
- Separate Portainer instance per host
- SSH-based management only
- Portainer Edge Agent

**Reasoning:**
Multiple Portainer instances mean multiple logins, multiple dashboards, and no unified view of the environment. The Agent approach allows a single Portainer installation to manage containers across all connected hosts from one interface. Environment switching in the Portainer UI is instant — stack deployments, log access, and container lifecycle management work identically regardless of which physical host a container runs on.

The resource footprint of the Agent is minimal compared to a full Portainer installation, which matters on lower-powered hosts.

---

## Separate cloudflared container per tunnel rather than host networking

**Decision:** Run cloudflared containers in bridge networking mode with explicit port mappings rather than host networking.

**Considered alternatives:**
- Host networking mode for cloudflared containers

**Reasoning:**
Cloudflared does not need host networking to function. It initiates outbound connections to Cloudflare's edge and proxies requests to upstream services by container name or local IP. Bridge networking keeps each tunnel container isolated in its own network namespace, which is consistent with the isolation-first approach applied throughout the stack.

---

## Diun over Watchtower for container update notifications

**Decision:** Use Diun for container image update notifications rather than Watchtower for automatic updates.

**Considered alternatives:**
- Watchtower in auto-update mode
- Watchtower in monitor-only mode
- Manual checking via `docker pull`

**Reasoning:**
Watchtower in auto-update mode silently pulls and restarts containers when new images are available. In a homelab running production-like services, an automatic update that introduces a breaking change can cause unplanned downtime with no immediate visibility into what changed. This happened implicitly — Watchtower was found crash-looping, which itself was causing unnecessary resource consumption.

Diun sends a Telegram notification when a new image is available and does nothing else. Updates are applied manually and intentionally, with the opportunity to review release notes before pulling. This is a deliberate trade-off of convenience for control.

---

## Ntopng on vDSM over Netdata on the primary host

**Decision:** Replace Netdata (running as a system service on the M710q) with Ntopng on the vDSM for infrastructure monitoring.

**Considered alternatives:**
- Keeping Netdata and tuning its polling intervals
- Running Netdata in a Docker container with resource limits
- Prometheus + Grafana stack

**Reasoning:**
Netdata was identified as the primary cause of sustained high CPU load on the M710q — load average peaked above 5.0 on a 4-core machine, with swap nearly exhausted. Netdata polls every subsystem continuously and its overhead was disproportionate to the value it provided alongside the existing monitoring stack (Gatus for service health, Dozzle for container logs).

Ntopng on the vDSM provides network-level visibility — traffic flows, protocol breakdown, threat intelligence, behavioral anomaly detection — which Netdata did not offer. Moving monitoring off the primary host entirely removes the resource contention and provides a more capable tool.

Prometheus + Grafana was considered but deemed excessive for the current scale. It remains a future option as the environment grows.

---

## Ghost CMS for multiple independent sites

**Decision:** Run multiple independent Ghost instances for different content properties rather than a single Ghost instance with multiple sites.

**Considered alternatives:**
- Single Ghost instance with subdirectory routing
- WordPress multisite
- Static site generators (Hugo, Jekyll)

**Reasoning:**
Ghost does not natively support multiple independent sites from a single installation in the community edition. Subdirectory routing would require complex reverse proxy configuration and creates a dependency where one site's issue affects all sites.

Separate instances are fully isolated — a database migration, theme update, or misconfiguration on one site has no effect on others. Each instance has its own MySQL database, its own Cloudflare Tunnel, and its own container lifecycle. The trade-off is higher resource consumption from running multiple Node.js processes and MySQL instances, which is acceptable on the current hardware.

---

## Immich ML disabled on primary host

**Decision:** Disable the Immich machine learning container on the primary Docker host while testing the photo management workflow.

**Considered alternatives:**
- Running the full Immich stack including ML
- Migrating Immich to vDSM immediately

**Reasoning:**
The `immich-machine-learning` container loads computer vision models into memory on startup and keeps them resident continuously. On a host already under memory pressure, this contributed significantly to swap usage. The core photo backup and viewing workflow functions without ML — face recognition and smart search are disabled but browsing, uploading, and album management work normally.

ML will be re-evaluated after Immich migrates to the vDSM, where 12GB of dedicated RAM provides sufficient headroom for model loading without impacting other services.
