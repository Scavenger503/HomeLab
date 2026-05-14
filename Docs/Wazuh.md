# Wazuh SIEM Setup Guide

Step-by-step guide for deploying Wazuh SIEM as a single-node Docker installation and enrolling agents across multiple hosts in the environment.

---

## Overview

Wazuh is an open-source security platform providing:
- Log aggregation and analysis
- File integrity monitoring (FIM)
- Vulnerability detection
- Active response
- Compliance monitoring (PCI-DSS, NIST, HIPAA)

This deployment uses the official Wazuh single-node Docker configuration, which runs the Wazuh Manager, Indexer (OpenSearch), and Dashboard as a unified stack.

---

## Prerequisites

- Dedicated host with minimum 4GB RAM (8GB+ recommended — OpenSearch is memory-intensive)
- Docker and Docker Compose installed
- At least 50GB available storage for log retention
- Static IP assigned to the Wazuh host
- SSH access to all hosts where agents will be deployed

---

## 1. Deploy Wazuh Single-Node Stack

Clone the official Wazuh Docker repository:

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.12.0
cd wazuh-docker/single-node
```

Generate SSL certificates for the stack:

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

Deploy the stack:

```bash
docker compose up -d
```

This starts three containers:
- `wazuh.manager` — core SIEM engine, receives agent data
- `wazuh.indexer` — OpenSearch instance, stores and indexes events
- `wazuh.dashboard` — Kibana-based web UI

Initial startup takes 3-5 minutes while OpenSearch initializes. Monitor progress:

```bash
docker compose logs -f wazuh.manager
```

Wait until you see:
```
INFO: Wazuh Manager is ready
```

---

## 2. Access the Dashboard

Navigate to `https://WAZUH-HOST-IP` in a browser.

Default credentials:
- **Username:** `admin`
- **Password:** `SecretPassword` (defined in `docker-compose.yml` — change before use)

> **Note:** The dashboard uses a self-signed certificate by default. Accept the browser warning on first access.

Change the default password immediately after first login via **Security → Internal Users → admin**.

---

## 3. Enroll Agents

Wazuh agents are lightweight processes installed on each monitored host. They collect logs, monitor file integrity, and report events to the Wazuh Manager.

### Linux Agent (Debian/Ubuntu)

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | tee -a /etc/apt/sources.list.d/wazuh.list

apt-get update
apt-get install wazuh-agent
```

Configure the agent to point at the Wazuh Manager:

```bash
WAZUH_MANAGER="WAZUH-HOST-IP" WAZUH_AGENT_NAME="hostname-here" apt-get install wazuh-agent
```

Or edit the config directly:

```bash
nano /var/ossec/etc/ossec.conf
```

Find the `<client>` section and set:

```xml
<client>
  <server>
    <address>WAZUH-HOST-IP</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

Start and enable the agent:

```bash
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

### Fedora / RHEL-based Agent

```bash
rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH

cat > /etc/yum.repos.d/wazuh.repo << EOF
[wazuh]
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
name=EL-\$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
protect=1
EOF

dnf install wazuh-agent
```

Configure manager address and start:

```bash
WAZUH_MANAGER="WAZUH-HOST-IP" WAZUH_AGENT_NAME="fedora-desktop" dnf install wazuh-agent
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

### Synology DSM (Syslog forwarding)

Synology DSM does not support native Wazuh agent installation. Use syslog forwarding instead to send DSM logs to the Wazuh Manager.

In DSM: **Control Panel → Log Center → Log Sending**

- **Server:** `WAZUH-HOST-IP`
- **Port:** `514`
- **Protocol:** `UDP`
- **Format:** `BSD (RFC 3164)`

In the Wazuh Manager, add a syslog listener. Edit `/var/ossec/etc/ossec.conf` inside the manager container:

```bash
docker exec -it single-node-wazuh.manager-1 bash
nano /var/ossec/etc/ossec.conf
```

Add inside the `<ossec_config>` block:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.0.0.0/24</allowed-ips>
</remote>
```

Restart the manager:

```bash
docker restart single-node-wazuh.manager-1
```

---

## 4. Verify Agent Enrollment

In the Wazuh Dashboard:

1. Go to **Agents** in the left sidebar
2. All enrolled agents should appear with status **Active**
3. Click any agent to view its events, alerts, and file integrity status

From the Wazuh Manager container, list connected agents:

```bash
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/agent_control -l
```

Expected output per agent:
```
ID: 001, Name: m710q, IP: 10.0.0.x, Status: Active
ID: 002, Name: fedora-desktop, IP: 10.0.0.x, Status: Active
```

---

## 5. Alerting Pipeline

Wazuh generates alerts based on decoded log events and rule matches. Alerts are routed through N8N for enrichment and delivery.

### Architecture

```
Wazuh Manager (alert generation)
    │
    ▼
N8N Webhook (alert intake)
    │
    ├── Claude API  ← alert triage and enrichment
    ├── Telegram    ← all severity levels
    └── Twilio SMS  ← critical severity, daytime hours only
```

### N8N Webhook Configuration

In the Wazuh Manager container, configure the integration:

```bash
docker exec -it single-node-wazuh.manager-1 bash
nano /var/ossec/etc/ossec.conf
```

Add inside `<ossec_config>`:

```xml
<integration>
  <name>custom-webhook</name>
  <hook_url>http://N8N-HOST-IP:5678/webhook/wazuh-alerts</hook_url>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```

The `<level>` value filters alerts by severity — only alerts at level 7 or above are forwarded. Adjust based on desired verbosity:

| Level | Description |
|---|---|
| 3 | System notifications |
| 5 | User-generated errors |
| 7 | Attack attempts (recommended minimum) |
| 10 | Multiple attack attempts |
| 12 | High importance events |
| 15 | Severe attack |

> **Known issue:** If the N8N webhook is exposed via Cloudflare Tunnel, Cloudflare's WAF may block POST requests from the Wazuh Manager. Use the internal LAN IP for the webhook URL to bypass this: `http://N8N-HOST-IP:5678/webhook/wazuh-alerts`

Restart the manager after config changes:

```bash
docker restart single-node-wazuh.manager-1
```

---

## 6. File Integrity Monitoring

File integrity monitoring (FIM) is enabled by default on all agents. Wazuh monitors configured directories for changes to files, permissions, and ownership.

Default monitored paths (Linux):
- `/etc`
- `/usr/bin`
- `/usr/sbin`
- `/bin`
- `/sbin`

Add custom paths in the agent's `ossec.conf`:

```xml
<syscheck>
  <directories check_all="yes">/opt/docker</directories>
  <directories check_all="yes" report_changes="yes">/etc/nginx</directories>
</syscheck>
```

`report_changes="yes"` includes a diff of what changed inside text files, not just that they changed.

---

## 7. Useful Commands

**Check manager status:**
```bash
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/wazuh-control status
```

**View active alerts in real time:**
```bash
docker exec -it single-node-wazuh.manager-1 tail -f /var/ossec/logs/alerts/alerts.json
```

**Restart all Wazuh components:**
```bash
docker compose restart
```

**Check agent connection from the agent host:**
```bash
systemctl status wazuh-agent
tail -f /var/ossec/logs/ossec.log
```

**List rules by level:**
```bash
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/ossec-logtest
```

---

## Notes

- OpenSearch (the indexer) is the primary memory consumer in this stack. On a host with less than 8GB RAM, consider reducing the OpenSearch heap size by setting `ES_JAVA_OPTS="-Xms1g -Xmx1g"` in the `docker-compose.yml` environment variables.
- Wazuh log retention defaults to 90 days. Adjust in **Settings → Index Management** in the dashboard based on available storage.
- The Wazuh Dashboard is not exposed externally via Cloudflare Tunnel. Access is LAN-only — security tooling should not have a public attack surface.
- Agent version must match the Manager version. If upgrading Wazuh, upgrade the Manager first and then all agents.
