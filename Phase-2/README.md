# Wazuh → N8N → Claude → Telegram Alert Pipeline

A security alerting pipeline that takes Wazuh SIEM alerts, enriches them with Claude AI triage analysis, and delivers formatted SOC-style notifications to Telegram.

---

## Architecture

```
Wazuh SIEM (alert generation)
    │
    │  HTTP POST (internal LAN)
    ▼
N8N Webhook (alert intake)
    │
    ▼
IF node (level ≥ 7 filter)
    │
    ▼
HTTP Request (Claude API triage)
    │
    ▼
Telegram (Luna AI — formatted alert delivery)
```
<img width="522" height="1073" alt="image" src="https://github.com/user-attachments/assets/3aa4daba-203d-4fcc-b46c-9e10883645cb" />
<img width="624" height="1259" alt="image" src="https://github.com/user-attachments/assets/cbbd6e08-607c-42c3-9e48-9050715aa076" />


---

## Prerequisites

- Wazuh SIEM deployed and running with at least one active agent
- N8N instance running and accessible on the local network
- Anthropic API key (console.anthropic.com)
- Telegram bot token and chat/group ID
- Both Wazuh and N8N on the same local network or reachable via LAN IP

---

## 1. Create the N8N Workflow

In N8N, create a new workflow named `Wazuh Security` and add the following nodes in order.

### Node 1 — Webhook (Trigger)

- **Node type:** Webhook
- **HTTP Method:** POST
- **Path:** `wazuh-alerts`
- **Authentication:** None
- **Respond:** Immediately

Note the **Production URL** — it will be in the format:
```
http://N8N-HOST-IP:5678/webhook/wazuh-alerts
```

This is the URL you will configure in Wazuh. Do not use the Test URL.

---

### Node 2 — IF (Alert Level Filter)

Filters out low-level noise. Only alerts at level 7 or above are processed.

- **Node type:** IF
- **Condition:**
  - Value 1: `{{ $json.body.rule.level }}`
  - Operator: `is greater than or equal to`
  - Value 2: `7`
- **Convert types where required:** Enabled

Alerts below level 7 exit via the False branch and are discarded. Connect the True branch to the next node.

**Level reference:**

| Level | Meaning |
|---|---|
| 3 | System notifications |
| 5 | User-generated errors |
| 7 | Attack attempts — recommended minimum |
| 10 | Multiple attack attempts |
| 12 | High importance events |
| 15 | Severe attack |

---

### Node 3 — HTTP Request (Claude API Triage)

Sends the alert data to Claude for analysis and returns a structured triage summary.

- **Node type:** HTTP Request
- **Method:** POST
- **URL:** `https://api.anthropic.com/v1/messages`
- **Authentication:** None
- **Send Headers:** Enabled

**Headers:**

| Name | Value |
|---|---|
| `x-api-key` | Your Anthropic API key |
| `anthropic-version` | `2023-06-01` |
| `content-type` | `application/json` |

- **Send Body:** Enabled
- **Body Content Type:** JSON
- **Specify Body:** Using JSON

**JSON Body:**

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "You are a security analyst. Analyze this Wazuh security alert and provide a brief triage summary in 2-3 sentences. Include severity assessment and recommended action.\n\nAlert:\nRule: {{ $('If').item.json.body.rule.description }}\nLevel: {{ $('If').item.json.body.rule.level }}\nAgent: {{ $('If').item.json.body.agent.name }}\nTimestamp: {{ $('If').item.json.body.timestamp }}"
    }
  ]
}
```

---

### Node 4 — Send a Text Message (Telegram)

Delivers the formatted alert and Claude triage to Telegram.

- **Node type:** Telegram → Send a text message
- **Credential:** Telegram bot token
- **Chat ID:** Your Telegram group or chat ID
- **Text:**

```
🚨 *Wazuh Security Alert*

*Agent:* {{ $('If').item.json.body.agent.name }}
*Rule:* {{ $('If').item.json.body.rule.description }}
*Level:* {{ $('If').item.json.body.rule.level }}
*Time:* {{ $('If').item.json.body.timestamp }}

*🤖 Luna Triage:*
{{ $json.content[0].text }}
```

- **Additional Fields → Parse Mode:** `Markdown (Legacy)`

---

## 2. Publish and Activate the Workflow

After building all four nodes:

1. Click **Publish** in the top bar and name the version (e.g. `Initial Release`)
2. Navigate to **Overview → Workflows**
3. Confirm the workflow shows **Published** status with a green indicator

The workflow is now active and listening on the production webhook URL.

---

## 3. Configure Wazuh Integration

On the Wazuh host, edit the manager configuration inside the container:

```bash
docker exec -it single-node-wazuh.manager-1 bash
```

Since standard text editors are not available inside the container, use `sed` to update the webhook URL:

```bash
sed -i 's|EXISTING-HOOK-URL|http://N8N-HOST-IP:5678/webhook/wazuh-alerts|g' /var/ossec/etc/ossec.conf
```

Verify the change:

```bash
grep "hook_url" /var/ossec/etc/ossec.conf
```

Expected output:
```xml
<hook_url>http://N8N-HOST-IP:5678/webhook/wazuh-alerts</hook_url>
```

Exit the container and restart the manager:

```bash
exit
docker restart single-node-wazuh.manager-1
```

If the integration block does not exist yet, add it manually. Exit the container and copy the config file out, edit it, then copy it back:

```bash
docker cp single-node-wazuh.manager-1:/var/ossec/etc/ossec.conf ./ossec.conf
nano ossec.conf
```

Add inside the `<ossec_config>` block:

```xml
<integration>
  <name>custom-n8n</name>
  <hook_url>http://N8N-HOST-IP:5678/webhook/wazuh-alerts</hook_url>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```

Copy it back:

```bash
docker cp ./ossec.conf single-node-wazuh.manager-1:/var/ossec/etc/ossec.conf
docker restart single-node-wazuh.manager-1
```

---

## 4. Verify the Pipeline

**Step 1 — Test N8N webhook reachability from Wazuh host:**

```bash
curl -X POST http://N8N-HOST-IP:5678/webhook/wazuh-alerts \
  -H "Content-Type: application/json" \
  -d '{"rule":{"level":7,"description":"Test alert"},"agent":{"name":"test-agent"},"timestamp":"2026-01-01T00:00:00Z"}'
```

Expected response: `{"message":"Workflow was started"}`

**Step 2 — Check N8N executions:**

Navigate to N8N → **Overview → Executions** and confirm a new execution appeared with **Success** status.

**Step 3 — Confirm Telegram delivery:**

A formatted message from the Luna AI bot should appear in the configured Telegram group within a few seconds.

**Step 4 — Trigger a real Wazuh event:**

From any monitored host, make multiple failed SSH login attempts to another host:

```bash
ssh wronguser@MONITORED-HOST-IP
```

Repeat 5 times. Wazuh should detect the failed authentication attempts and fire a level 7+ alert that flows through the full pipeline automatically.

---

## 5. Operational Notes

**Use internal LAN IP, not the public Cloudflare URL.**
Wazuh POST requests to a Cloudflare-tunneled URL will be blocked by Cloudflare's WAF bot protection rules. The Wazuh integration sends automated HTTP requests that Cloudflare identifies as bot traffic and returns 403 Forbidden. Always use the direct internal IP and port for the N8N webhook URL.

**Alert volume.**
With the IF filter set to level 7+, typical homelab alert volume is 20-100 alerts per day depending on agent activity. Each alert consumes approximately 600 tokens via the Claude API. At this volume, API costs are under $5/month.

**Model selection.**
The pipeline uses `claude-sonnet-4-6`. This can be swapped for any Anthropic model by updating the `model` field in the HTTP Request node body. A cheaper alternative for high-volume environments is `claude-haiku-4-5-20251001`.

**Disconnected agents.**
Agents showing as Disconnected in Wazuh (`/var/ossec/bin/agent_control -l`) will not generate alerts. Reconnect them by restarting the `wazuh-agent` service on the affected host.

**Real alerts.**
Once the pipeline is live, real Wazuh detections from active agents will flow through automatically without any manual intervention. Common real-world alerts include netstat port changes, file integrity monitoring events, failed authentication attempts, and suspicious process activity.

![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-blue?style=flat)
![N8N](https://img.shields.io/badge/N8N-Automation-orange?style=flat)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-F38020?style=flat&logo=cloudflare&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-2CA5E0?style=flat&logo=telegram&logoColor=white)
