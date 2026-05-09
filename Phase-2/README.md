# 🛡️ Wazuh + N8N Security Alerting Pipeline

An automated security alerting and orchestration pipeline built on top of Wazuh SIEM and N8N workflow automation. This project transforms raw SIEM alerts into intelligent, actionable notifications with multi-channel delivery and escalation logic.

> ⚠️ **Status: In Progress** — Wazuh SIEM and N8N are fully deployed with agents reporting across the homelab fleet. The custom Python integration and N8N workflow are built and configured. End-to-end alert delivery is pending resolution of a Cloudflare WAF issue blocking webhook POSTs from the Wazuh Manager to N8N. Planned fix is routing Wazuh directly to N8N's internal LAN address, bypassing the WAF entirely.

---

## 📐 Architecture Overview

```
Wazuh SIEM
    │
    │  (Custom Python Integration)
    ▼
N8N Webhook
    │
    ├──► Alert Triage & Enrichment (Claude API)
    │
    ├──► Telegram Notification
    │
    └──► Twilio Voice Escalation (Critical/Unacknowledged)
```

---

## 🧩 Components

### 🔍 Wazuh SIEM
- Full Wazuh stack (Manager, Indexer, Dashboard) deployed via Docker Compose
- Agents installed across the homelab fleet:
  - Primary Docker host
  - Proxmox nodes
  - Fedora desktop
  - Raspberry Pi DNS nodes
  - NAS (via syslog)
- Real-time threat detection, log aggregation, and compliance monitoring

### ⚙️ Custom Python Integration
- Wazuh custom integration script intercepts alerts at the manager level
- Filters alerts by severity level before forwarding
- POSTs structured alert payload to N8N webhook endpoint
- Runs natively on the Wazuh Manager container

### 🔄 N8N Workflow Automation
- Receives alert webhook from Wazuh
- Parses and enriches alert data
- Routes alerts based on severity level:
  - **Low/Medium** → Formatted Telegram notification
  - **High/Critical** → Telegram notification + Twilio voice escalation if unacknowledged
- Claude API integration for natural language alert summarization and triage recommendations

### 📲 Notification Channels
- **Telegram** — Formatted alert messages with severity, rule description, affected host, and timestamp
- **Twilio** — Automated voice call escalation for critical alerts that go unacknowledged

---

## 🔐 Security Practices

- All credentials managed via `.env` files — never hardcoded
- Wazuh Manager isolated on dedicated security machine
- Webhook endpoint protected behind Cloudflare Tunnel
- Alert forwarding uses internal LAN routing to avoid WAF interference

---

## 📁 Repository Structure

```
wazuh-n8n-security-pipeline/
├── wazuh/
│   ├── docker-compose.yml
│   └── .env.example
├── n8n/
│   ├── docker-compose.yml
│   └── .env.example
├── integration/
│   └── custom-n8n.py        # Wazuh custom integration script
├── workflows/
│   └── wazuh-alert-workflow.json  # N8N workflow export
└── README.md
```

---

## 🚀 Deployment Overview

### 1. Deploy Wazuh Stack
```bash
cd wazuh
cp .env.example .env
# Fill in your values
docker compose up -d
```

### 2. Deploy N8N
```bash
cd n8n
cp .env.example .env
# Fill in your values
docker compose up -d
```

### 3. Install Custom Integration
Copy `integration/custom-n8n.py` to the Wazuh Manager container:
```bash
docker cp custom-n8n.py wazuh-manager:/var/ossec/integrations/custom-n8n
docker exec wazuh-manager chmod +x /var/ossec/integrations/custom-n8n
```

### 4. Configure Wazuh
Add the following to your `ossec.conf`:
```xml
<integration>
  <name>custom-n8n</name>
  <hook_url>http://your-n8n-host:5678/webhook/wazuh-alerts</hook_url>
  <level>3</level>
  <alert_format>json</alert_format>
</integration>
```

### 5. Import N8N Workflow
Import `workflows/wazuh-alert-workflow.json` into your N8N instance and configure your Telegram and Twilio credentials.

---

## 🛠️ Stack

![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-blue?style=flat)
![N8N](https://img.shields.io/badge/N8N-Automation-orange?style=flat)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-F38020?style=flat&logo=cloudflare&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-2CA5E0?style=flat&logo=telegram&logoColor=white)
