# 🏠 Homelab Docker Stacks

A production-grade, self-hosted infrastructure built on Docker and managed via Portainer. This repository contains the Docker Compose configurations powering the World of Hackers LLC homelab environment — a multi-node setup designed for security monitoring, knowledge management, automation, and business operations.

---

## 📐 Infrastructure Overview

| Host | Role | IP |
|------|------|----|
| Lenovo M710q | Primary Docker host (always-on) | 192.168.1.213 |
| Lenovo M715q | DevOps & Security machine | 192.168.1.222 |
| Synology DS923+ | NAS / backup target | Static |
| Raspberry Pi 5 | Primary DNS (AdGuard Home) | Static |
| Raspberry Pi 3B | Secondary DNS (AdGuard Home) | Static |

All services are externally accessible via **Cloudflare Tunnel + Zero Trust**, eliminating the need for open inbound ports on the network perimeter. DNS is managed through **AdGuard Home** running redundantly across two Raspberry Pi nodes.

---

## 🐳 Services

### 📝 Ghost CMS
**Host:** M710q | **Purpose:** Business website & content platform for World of Hackers LLC

- Self-hosted CMS powering `worldofhackers.io`
- Configured with custom domain via Cloudflare Tunnel
- MySQL backend with persistent volume storage
- Environment-variable driven configuration for portability

---

### ⚙️ N8N — Workflow Automation
**Host:** M710q | **Purpose:** Security orchestration and business automation

- Self-hosted workflow automation platform
- Integrated with **Wazuh SIEM** to receive, process, and triage security alerts
- Delivers formatted alert notifications to **Telegram**
- Escalation logic built in for critical unacknowledged alerts (Twilio voice call)
- Houses the "Luna - Core Processor" workflow for intelligent alert correlation via Claude API

---

### 🛡️ Wazuh SIEM
**Host:** M715q | **Purpose:** Security Information and Event Management

- Deployed as a full Wazuh stack (Manager, Indexer, Dashboard) via Docker Compose
- Agents installed across the homelab fleet:
  - M710q (Docker host)
  - Proxmox nodes
  - Fedora desktop / streaming rig
  - Raspberry Pi / AdGuard node
  - Synology NAS (via syslog, port 514)
- Custom Python integration script routes alerts to N8N webhook for automated triage
- Provides real-time threat detection, log aggregation, and compliance monitoring

---

### 📚 BookStack — Knowledge Base
**Host:** M710q | **Purpose:** Public-facing IT knowledge base

- Self-hosted wiki platform powering `kb.worldofhackers.io`
- Publicly accessible with read-only Guest permissions (no login required)
- Organized into Shelves → Books → Pages hierarchy covering:
  - Cybersecurity Essentials
  - Security Operations & Models
  - Threat Awareness & Prevention
  - Partner vendor troubleshooting (Acronis, ESET, Aura, DigiCert, and more)
- Login endpoint protected via Cloudflare Zero Trust email authentication
- MySQL backend with nightly rsync backups to Synology NAS

---

### 🔖 Linkwarden — Bookmark Manager
**Host:** M710q | **Purpose:** Self-hosted bookmark and link archiving

- Replaces cloud-based bookmark services with a fully self-hosted alternative
- Archives full page content locally, preserving resources even if original URLs go offline
- Organized into collections by category (DevOps, Homelab, Security, etc.)
- PostgreSQL backend

---

### 🔄 AdGuard Home Sync
**Host:** M710q | **Purpose:** DNS configuration synchronization

- Keeps both AdGuard Home instances (Pi 5 and Pi 3B) in sync automatically
- Primary node (Pi 5) acts as the source of truth; changes propagate to the replica (Pi 3B)
- Ensures consistent blocklists, filtering rules, and DNS settings across both resolvers without manual maintenance
- Provides DNS redundancy — clients fail over seamlessly if the primary node becomes unavailable

---

## 🔐 Security Practices

- **No open inbound ports** — all external access routed through Cloudflare Tunnel
- **Zero Trust access policies** on sensitive endpoints (admin panels, login pages)
- **Secrets management** via `.env` files, never hardcoded in Compose files
- **Nightly backups** from M710q to Synology NAS via rsync daemon
- **Active SIEM monitoring** across all nodes with automated alerting pipeline

---

## 📁 Repository Structure

```
homelab-docker-stacks/
├── ghost/
│   └── docker-compose.yml
├── n8n/
│   └── docker-compose.yml
├── wazuh/
│   └── docker-compose.yml
├── bookstack/
│   └── docker-compose.yml
├── linkwarden/
│   └── docker-compose.yml
├── adguardhome-sync/
│   └── docker-compose.yml
└── README.md
```

> ⚠️ All credentials, API keys, and sensitive values have been removed and replaced with environment variable placeholders. Never commit secrets to version control.

---

## 🛠️ Stack

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Portainer](https://img.shields.io/badge/Portainer-13BEF9?style=flat&logo=portainer&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-F38020?style=flat&logo=cloudflare&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat&logo=linux&logoColor=black)
![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-blue?style=flat)

---

## 👤 Author

**World of Hackers LLC**
- 🌐 [worldofhackers.io](https://worldofhackers.io)
- 📚 [kb.worldofhackers.io](https://kb.worldofhackers.io)
