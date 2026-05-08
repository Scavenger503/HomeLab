# 🏠 Homelab Docker Stacks

A production-grade, self-hosted infrastructure built on Docker and managed via Portainer. This repository contains the Docker Compose configurations powering a multi-node homelab environment designed for security monitoring, knowledge management, workflow automation, and IT operations.

---

## 📐 Infrastructure Overview

| Host | Role |
|------|------|
| Primary Docker Host (always-on) | Runs all core services |
| DevOps & Security Machine | Hosts SIEM stack |
| NAS | Backup target & storage |
| Raspberry Pi 5 | Primary DNS resolver |
| Raspberry Pi 3B | Secondary DNS resolver |

All services are externally accessible via **Cloudflare Tunnel + Zero Trust**, eliminating the need for open inbound ports on the network perimeter. DNS is managed through **AdGuard Home** running redundantly across two Raspberry Pi nodes.

---

## 🐳 Services

### 📝 Ghost CMS
**Purpose:** Self-hosted content management and web platform

- Self-hosted CMS powering a public-facing website
- Configured with a custom domain via Cloudflare Tunnel
- MySQL backend with persistent volume storage
- Environment-variable driven configuration for portability

---

### ⚙️ N8N — Workflow Automation
**Purpose:** Security orchestration and IT automation

- Self-hosted workflow automation platform
- Integrated with **Wazuh SIEM** to receive, process, and triage security alerts
- Delivers formatted alert notifications to **Telegram**
- Escalation logic for critical unacknowledged alerts via Twilio voice call
- Claude API integration for intelligent alert correlation and natural language triage

---

### 🛡️ Wazuh SIEM
**Purpose:** Security Information and Event Management

- Full Wazuh stack (Manager, Indexer, Dashboard) deployed via Docker Compose
- Agents installed across the homelab fleet:
  - Primary Docker host
  - Proxmox nodes
  - Fedora desktop
  - Raspberry Pi DNS nodes
  - NAS (via syslog)
- Custom Python integration script routes alerts to N8N webhook for automated triage
- Provides real-time threat detection, log aggregation, and compliance monitoring

---

### 📚 BookStack — Knowledge Base
**Purpose:** Public-facing IT knowledge base and documentation platform

- Self-hosted wiki platform with a publicly accessible read-only interface
- No login required for public visitors — Guest role restricted to view-only
- Organized into Shelves → Books → Pages covering cybersecurity topics and vendor troubleshooting
- Login endpoint protected via Cloudflare Zero Trust email authentication
- MySQL backend with nightly rsync backups to NAS

---

### 🔖 Linkwarden — Bookmark Manager
**Purpose:** Self-hosted bookmark and link archiving

- Replaces cloud-based bookmark services with a fully self-hosted alternative
- Archives full page content locally, preserving resources even if original URLs go offline
- Organized into collections by category (DevOps, Homelab, Security, etc.)
- PostgreSQL backend

---

### 🔄 AdGuard Home Sync
**Purpose:** DNS configuration synchronization across redundant resolvers

- Keeps both AdGuard Home instances (Pi 5 and Pi 3B) in sync automatically
- Primary node acts as the source of truth; changes propagate to the replica automatically
- Ensures consistent blocklists, filtering rules, and DNS settings without manual maintenance
- Provides DNS redundancy — clients fail over seamlessly if the primary node becomes unavailable

---

## 🔐 Security Practices

- **No open inbound ports** — all external access routed through Cloudflare Tunnel
- **Zero Trust access policies** on sensitive endpoints (admin panels, login pages)
- **Secrets management** via `.env` files, never hardcoded in Compose files
- **Nightly backups** from primary Docker host to NAS via rsync daemon
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

> ⚠️ All credentials, API keys, internal IPs, and sensitive values have been removed and replaced with environment variable placeholders. See `.env.example` in each service folder for required variables. Never commit secrets to version control.

---

## 🛠️ Stack

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Portainer](https://img.shields.io/badge/Portainer-13BEF9?style=flat&logo=portainer&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-F38020?style=flat&logo=cloudflare&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat&logo=linux&logoColor=black)
![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-blue?style=flat)
