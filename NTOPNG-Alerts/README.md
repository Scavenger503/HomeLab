# NTOPNG-Alerts

An automated network security alerting pipeline that takes Ntopng threat detections, enriches them with Claude AI analysis via Luna, and delivers plain-English SOC-style reports to Telegram in real time.

---

## Overview

Ntopng monitors all network traffic on the LAN and fires alerts when it detects behavioral anomalies, blacklisted hosts, suspicious flows, or policy violations. This pipeline catches those alerts, passes them to Luna (Claude AI) for triage, and delivers actionable security reports directly to Telegram — automatically, 24/7.

---

## Architecture

```
Ntopng (network monitor)
    │
    │  HTTP POST (webhook)
    ▼
N8N Webhook (alert intake)
    │
    ▼
Code Node (alert formatting)
    │
    ▼
HTTP Request (Claude API — Luna triage)
    │
    ▼
Telegram (VoidWatch Bot — formatted alert delivery)
```

---

## Components

### Ntopng
Deployed on vDSM (Synology DS923+) via Docker. Monitors the `eth0` interface with the local network defined as `10.0.0.0/24`.

Key configuration:
- Interface: `eth0`
- Local network: your LAN CIDR
- Threat intelligence feeds: Abuse.ch URLhaus, Emerging Threats, IPsum, NoCoin Filter List
- Behavioral checks enabled: Dangerous Host, ICMP Flood, Flow Flood, Suspicious Domain Scan, Countries Contacts, Unexpected DNS/DHCP/SMTP/NTP server

Alerts are sent via Ntopng's built-in Telegram notification system to the VoidWatch Bot for raw network-level alerts, and via webhook to N8N for AI-enriched analysis.

### N8N Workflow
Four-node workflow named **Ntopng Alerts**:

1. **Webhook** — receives POST requests from Ntopng containing raw alert data
2. **Code node** — extracts and formats the alerts array into a clean Anthropic API request body
3. **HTTP Request** — calls the Claude API with Luna's security analyst prompt
4. **Telegram** — delivers the formatted triage report to the Home Lab Alerts group

### Luna (Claude AI)
Luna acts as an automated Tier 1 SOC analyst. For each Ntopng alert she:
- Identifies the affected device and alert type
- Assesses whether it represents a real threat or normal behavior
- Provides a severity assessment
- Delivers specific recommended actions
- Flags false positives (e.g. Ntopng monitoring its own traffic)

### VoidWatch Bot
A dedicated Telegram bot for network monitoring alerts. Receives both:
- Raw Ntopng behavioral alerts directly via Ntopng's built-in Telegram integration
- Luna-enriched analysis via the N8N pipeline

---

## N8N Workflow Setup

### Node 1 — Webhook

- **HTTP Method:** POST
- **Path:** `ntopng-alerts` (or your preferred path)
- **Authentication:** None
- **Respond:** Immediately

### Node 2 — Code Node

```javascript
const alerts = $input.first().json.body.alerts;

return [{
  json: {
    model: "claude-sonnet-4-6",
    max_tokens: 1000,
    messages: [
      {
        role: "user",
        content: "You are Luna, a cybersecurity analyst. Analyze this Ntopng network alert and explain in plain English what it means, whether it's a real threat or normal behavior, and what action if any should be taken:\n\n" + JSON.stringify(alerts)
      }
    ]
  }
}];
```

### Node 3 — HTTP Request (Claude API)

- **Method:** POST
- **URL:** `https://api.anthropic.com/v1/messages`
- **Authentication:** None

**Headers:**

| Name | Value |
|---|---|
| `x-api-key` | Your Anthropic API key |
| `anthropic-version` | `2023-06-01` |
| `content-type` | `application/json` |

- **Body Content Type:** JSON
- **Specify Body:** Using JSON
- **JSON field:** `{{ $json }}`

The Code node builds the complete request body. The HTTP Request node passes it through directly — no additional formatting needed.

### Node 4 — Telegram

- **Credential:** VoidWatch Bot token
- **Chat ID:** Your Telegram group ID
- **Text:**

```
🌐 *Ntopng Network Alert*

*🤖 Luna Analysis:*
{{ $json.content[0].text }}
```

- **Parse Mode:** Markdown (Legacy)

---

## Ntopng Webhook Configuration

In Ntopng: **Settings → Notifications → Add Recipient**

- **Type:** HTTP/Webhook
- **URL:** `http://N8N-HOST-IP:5678/webhook/ntopng-alerts`

Use the internal LAN IP directly — do not route through Cloudflare Tunnel. Cloudflare WAF blocks automated webhook POST requests from Ntopng.

---

## Pipeline in Action

Luna correctly identifies and triages real network events including:
- Port changes on monitored hosts
- Ntopng self-monitoring traffic (correctly flagged as false positive)
- Blacklisted host contacts
- Behavioral anomalies from LAN devices

Example output from Luna on a real alert:

> "This alert indicates Ntopng detected its own webhook activity calling back to itself — this is expected behavior from the monitoring stack and represents LOW RISK. No action required. Consider whitelisting this traffic pattern to reduce alert noise."

---

## Related Projects

- [HomeLab](https://github.com/Scavenger503/HomeLab) — Full homelab infrastructure documentation
- [Wazuh N8N Security Pipeline](https://github.com/Scavenger503/HomeLab/tree/main/Phase-2) — SIEM alerting pipeline (Wazuh → N8N → Luna → Telegram)

---

## Notes

- Ntopng community edition includes native Telegram notification support — VoidWatch Bot receives raw alerts directly without N8N for immediate notification
- The N8N pipeline adds the AI enrichment layer on top of the raw alerts
- Both alert streams (raw + enriched) deliver to the same Telegram group for a complete picture
- The Code node approach was chosen over N8N's built-in expression editor for cleaner handling of Ntopng's nested alerts array structure
