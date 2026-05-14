# Homelab Infrastructure

A personal homelab running self-hosted services across multiple physical machines and virtual environments. This repository documents the architecture, tooling decisions, and operational lessons learned from running a production-like infrastructure at home.

This is a living project — services are added, moved, and replaced as I learn and as requirements change.

---

## Hardware

| Device | CPU | RAM | Role |
|---|---|---|---|
| Lenovo ThinkCentre M710q | Intel Core i5 | 16GB | Primary Docker host |
| Lenovo ThinkCentre M715q | AMD Ryzen | 16GB | Security & DevOps |
| Synology DS923+ | AMD Ryzen R1600 | 20GB | NAS + vDSM host |
| Raspberry Pi | ARM | 4GB | DNS & DHCP |
| Proxmox Node | — | — | Virtualization testing |
| Fedora 43 KDE | — | — | Primary workstation |

All devices run on a flat local network with static IP assignments.

---

## Network Architecture

External access is handled entirely through Cloudflare Tunnels. There are no open inbound ports on the router. All traffic enters through Cloudflare's edge network, providing WAF protection, DDoS mitigation, and SSL termination before reaching any service.

Each Cloudflare Tunnel runs as a dedicated container rather than a single shared tunnel. This decision came from a real incident — a WAF rule blocked a specific webhook endpoint mid-deployment, which would have taken down all external services if they shared a single tunnel. Isolating tunnels per service group contains the blast radius of any Cloudflare-side disruption.

DNS filtering is handled at the network level by AdGuard Home on the Raspberry Pi, which intercepts all DNS queries before they leave the network.

---

## Service Distribution

### M710q — Primary Docker Host

Runs all user-facing and automation services via Docker, managed through Portainer CE. Exposed externally via Cloudflare Tunnels.

**Content & Publishing**
- Ghost CMS — multiple independent instances (portfolio, podcast, music, quotes)
- Postiz — social media scheduling
- Linkstack — bio link pages

**Productivity & Knowledge**
- Outline — wiki and documentation
- Bookstack — structured knowledge base
- Trilium — personal note-taking
- Linkwarden — bookmark and link archiving
- Vikunja — task management
- Kimai — time tracking

**Infrastructure & DevOps**
- Portainer CE — container lifecycle management
- N8N — workflow automation and API orchestration
- Gatus — internal service health monitoring
- Dozzle — real-time container log viewer
- Diun — container image update notifications
- Homepage — unified service dashboard

**Utilities**
- IT-Tools — developer utility collection
- CyberChef — data encoding and transformation
- PrivateBin — encrypted pastebin
- Kutt — URL shortener
- Mattermost — self-hosted team messaging
- Immich — photo management (testing phase, ML disabled)

### M715q — Security & DevOps

Dedicated to security monitoring. Kept physically separate from production workloads so the SIEM remains operational even if the primary host has issues, and to avoid resource contention between monitoring and production services.

- Wazuh SIEM v4.12.0 (single-node Docker deployment)
- Wazuh agents deployed across all machines in the environment

### DS923+ — NAS

Primary storage and backup destination. Hosts a vDSM virtual machine for monitoring workloads rather than running them directly on DSM — this isolates containers from the NAS operating system so a broken container cannot affect file services or scheduled backups.

**Host DSM**
- Rsync backup target for M710q Docker volumes (nightly at 2AM via dedicated backup user)
- Jellyfin media server (migration in progress)

**vDSM (2 vCPU, 12GB RAM)**
- Uptime Kuma — service uptime monitoring
- Ntopng — network traffic analysis and intrusion detection

### Raspberry Pi
- AdGuard Home — network-wide DNS filtering and DHCP

---

## Security Stack

| Tool | Purpose |
|---|---|
| Wazuh SIEM | Centralized log aggregation, threat detection, file integrity monitoring |
| Wazuh Agents | Deployed on all nodes — covers file changes, auth events, process monitoring |
| Ntopng | Real-time network flow analysis with active threat intelligence feeds |
| AdGuard Home | DNS-level filtering, blocks malicious domains before connection |
| Cloudflare WAF | Edge protection for all externally exposed services |
| Uptime Kuma | Service availability monitoring with instant alerting |

**Threat intelligence feeds (Ntopng):** Abuse.ch URLhaus, Emerging Threats, IPsum, NoCoin Filter List — loaded at startup, updated automatically.

**Behavioral monitoring rules enabled:** Dangerous host detection, ICMP flood, flow flood, suspicious domain scan, unexpected DNS/DHCP/SMTP/NTP server detection.

### Alert Pipeline

```
Wazuh SIEM
    │
    ▼
N8N (Luna — Core Processor workflow)
    │
    ├── Claude API  ← alert enrichment and triage
    ├── Telegram    ← all severity levels
    └── Twilio SMS  ← critical severity, daytime hours only

Ntopng  ──► VoidWatch Bot ──► Telegram
Uptime Kuma ────────────────► Telegram
```

---

## Operational Lessons

**Resource contention is invisible until it isn't.**
The primary Docker host was running at a load average above 5.0 with swap nearly exhausted. The culprit turned out to be Netdata running as a system service — it was polling every subsystem continuously and consuming disproportionate CPU. Removing it and relocating monitoring to a dedicated vDSM environment dropped load average below 1.0. Lesson: always profile before assuming a workload is the problem.

**ML containers have a hidden cost.**
Immich's machine learning container loads models into memory continuously even when not actively processing. On a host already under memory pressure, this pushed the system into heavy swap usage. Disabling ML reduced swap consumption measurably without affecting core photo management functionality. ML features will be re-evaluated after migrating Immich to the vDSM where memory headroom is available.

**Orphan containers survive compose restarts.**
Commenting out a service in `docker-compose.yml` and running `docker compose up -d` does not remove containers that were previously created from that service. The container continues running as an orphan. The correct sequence is `docker compose down --remove-orphans` followed by `docker compose up -d`. This is a subtle but important distinction when managing compose stacks manually.

**vDSM is not a standard Linux distribution.**
Attempting to install Docker on vDSM using the standard `curl -fsSL https://get.docker.com | sh` script fails — vDSM uses a stripped Synology kernel with no `/etc/os-release`. The correct approach is installing Container Manager through the Synology Package Center GUI, which provides a properly integrated Docker environment for that platform.

**Portainer multi-environment management scales well.**
Rather than managing separate Portainer instances per host, deploying Portainer Agent on the vDSM and connecting it to the existing Portainer installation on the primary host provides a single control plane for both environments. Stack deployments, log access, and container lifecycle management work identically regardless of which physical host the container runs on.

---

## Roadmap

- [ ] K3s cluster (Oracle Cloud Free Tier — 4 ARM OCPUs, 24GB RAM)
- [ ] Migrate Homepage to K3s as a portfolio demonstration
- [ ] Complete Wazuh → N8N → Telegram alerting pipeline
- [ ] Migrate Immich to vDSM with NAS-native storage volumes
- [ ] Terraform provisioning for repeatable infrastructure
- [ ] CKA exam preparation lab environment

---

## Certifications in Progress

- HashiCorp Terraform Associate
- Certified Kubernetes Administrator (CKA)
- GitHub Actions
- AWS Cloud Practitioner

---

## Stack

`Docker` `Portainer` `Cloudflare Tunnels` `Wazuh` `N8N` `Ntopng` `AdGuard Home` `Synology DSM` `Proxmox` `Ghost CMS` `Fedora Linux` `Bash` `YAML` `Git`
