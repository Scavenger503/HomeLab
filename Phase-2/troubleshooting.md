# Troubleshooting — Wazuh → N8N → Claude → Telegram Pipeline

Issues encountered during the setup and operation of the security alerting pipeline. Entries are added as problems occur.

---

## 2026-05-14

### Cloudflare WAF blocking Wazuh webhook POST requests

**Symptoms**
Wazuh integration configured and restarted but no alerts arriving in N8N. No executions appearing in the N8N execution log despite confirmed level 7+ alerts in the Wazuh dashboard.

**Diagnosis**
The webhook URL in `ossec.conf` was pointing at the public Cloudflare Tunnel URL:
```xml
<hook_url>https://n8n.example.com/webhook/wazuh-alerts</hook_url>
```

Cloudflare's bot protection identified the automated POST requests from the Wazuh Manager as bot traffic and returned `403 Forbidden`, silently dropping all webhook deliveries. No error was visible in Wazuh logs — the integration appeared configured correctly.

Verified by testing the internal IP directly from the Wazuh host:
```bash
curl -X POST http://N8N-HOST-IP:5678/webhook/wazuh-alerts \
  -H "Content-Type: application/json" \
  -d '{"rule":{"level":7,"description":"test"},"agent":{"name":"test"}}'
```
Response: `{"message":"Workflow was started"}` — confirmed internal route works.

**Resolution**
Updated the hook URL in `ossec.conf` to use the internal LAN IP directly, bypassing Cloudflare entirely:

Since `nano` and `vi` are not available inside the Wazuh manager container, used `sed`:
```bash
docker exec -it single-node-wazuh.manager-1 bash
sed -i 's|https://n8n.example.com/webhook/wazuh-alerts|http://N8N-HOST-IP:5678/webhook/wazuh-alerts|g' /var/ossec/etc/ossec.conf
grep "hook_url" /var/ossec/etc/ossec.conf
exit
docker restart single-node-wazuh.manager-1
```

**Lesson**
Never route Wazuh webhook integrations through Cloudflare Tunnels. Cloudflare WAF treats automated POST requests as bot traffic. Always use the direct internal LAN IP and port for N8N webhook URLs in the Wazuh integration config.

---

### N8N workflow receiving alerts but completing in 7ms with no Telegram output

**Symptoms**
N8N execution log showed successful executions with 7ms runtime. No Telegram messages delivered. Wazuh dashboard confirmed level 7+ alerts were being generated.

**Cause**
The workflow only contained the Webhook trigger node — no downstream nodes (IF filter, Claude API, Telegram) had been built yet. The workflow was receiving the webhook and completing immediately with no further processing.

Additionally, the workflow was in test/draft mode rather than published production mode. In test mode, N8N only responds to the test URL (`/webhook-test/wazuh-alerts`), not the production URL (`/webhook/wazuh-alerts`) that Wazuh was posting to.

**Resolution**
Built out the complete workflow:
1. Added IF node to filter alerts at level 7+
2. Added HTTP Request node for Claude API triage
3. Added Telegram node for alert delivery
4. Clicked **Publish** to create a production version
5. Confirmed **Published** status in Overview → Workflows

**Lesson**
N8N webhooks have two URLs — test and production. Wazuh (and any external system) must post to the production URL. The workflow must be published, not just saved, for the production webhook to be active.

---

### IF node Execute step spinning indefinitely

**Symptoms**
Clicking Execute step on the IF node caused it to spin without producing output. The node appeared to be waiting indefinitely.

**Cause**
The IF node was waiting for live webhook data in test mode. The input panel showed stale data from a previous execution with a warning: "The fields below come from the last successful execution. Execute node to refresh them."

In test mode, N8N listens on the test URL. The curl commands being run were posting to the production URL, so the test listener never received fresh data.

**Resolution**
Changed the curl command to post to the test URL instead:
```bash
curl -X POST http://N8N-HOST-IP:5678/webhook-test/wazuh-alerts \
  -H "Content-Type: application/json" \
  -d '{"rule":{"level":7,"description":"Test"},"agent":{"name":"test"},"timestamp":"2026-01-01T00:00:00Z"}'
```

This sent data to the test listener while the IF node was waiting, allowing Execute step to complete and show output.

**Lesson**
When building and testing N8N workflows node by node, always post to the test URL (`/webhook-test/`) not the production URL (`/webhook/`). Switch to the production URL only after the workflow is published.

---

### Claude API returning "resource not found" error

**Symptoms**
HTTP Request node returned:
```
The resource you are requesting could not be found
model: claude-sonnet-4-20250514
```

**Cause**
The model string `claude-sonnet-4-20250514` was incorrect. Starting with the Claude 4.6 generation, Anthropic uses a dateless model ID format.

**Resolution**
Updated the model string in the HTTP Request JSON body:
```json
"model": "claude-sonnet-4-6"
```

**Lesson**
Always verify the current model string against the Anthropic documentation at `console.anthropic.com`. Model ID formats changed between generations — older dated formats (e.g. `claude-sonnet-4-20250514`) are not valid for newer models.

---

### Workflow published but not triggering on real Wazuh alerts

**Symptoms**
Manual curl tests worked correctly and delivered Telegram messages. Real Wazuh alerts visible in the dashboard at level 7+ but no automatic N8N executions triggered.

**Cause**
The workflow showed as saved but was not in **Published** state. N8N requires a workflow to be explicitly published for the production webhook to remain active across restarts and sessions. The toggle visible inside the workflow editor reflects active/inactive state for the current session, but **Publish** is required to persist production availability.

**Resolution**
1. Clicked **Publish** in the workflow editor top bar
2. Named the version `Initial Release`
3. Confirmed **Published** status in **Overview → Workflows** — green indicator visible

Real Wazuh alerts began flowing through automatically immediately after publishing.

**Lesson**
In N8N, saving a workflow and publishing a workflow are different operations. Save preserves your changes. Publish makes the production webhook permanently active. Always publish after building or modifying a workflow that uses a webhook trigger.

---

### No text editors available inside Wazuh manager container

**Symptoms**
Attempting to edit `ossec.conf` inside the Wazuh manager container failed:
```
bash: nano: command not found
bash: vi: command not found
```

**Cause**
The Wazuh manager Docker image is a minimal container that does not include standard text editors.

**Resolution**
Used `sed` for in-place string replacement:
```bash
sed -i 's|OLD-VALUE|NEW-VALUE|g' /var/ossec/etc/ossec.conf
```

For more complex edits, copy the file out of the container, edit it on the host, then copy it back:
```bash
docker cp single-node-wazuh.manager-1:/var/ossec/etc/ossec.conf ./ossec.conf
nano ossec.conf
docker cp ./ossec.conf single-node-wazuh.manager-1:/var/ossec/etc/ossec.conf
docker restart single-node-wazuh.manager-1
```

**Lesson**
Minimal Docker containers often omit standard utilities. Always have `sed` and `docker cp` as fallback editing strategies when working inside containers.

---

## Known Issues

**Alert volume from netstat rule 533**
Rule 533 (listened ports status changed) fires frequently on active Linux hosts — every time a port opens or closes. This generates significant alert volume through the pipeline. Consider raising the IF node threshold to level 8 or adding a second filter to exclude rule ID 533 if the volume becomes excessive.

To add a rule ID exclusion, add a second condition to the IF node:
- Value: `{{ $json.body.rule.id }}`
- Operator: `is not equal to`
- Value: `533`

**Anthropic API key exposure**
The API key is stored as a plain text value in the N8N HTTP Request node header. Rotate the key immediately if the N8N instance is ever compromised or if the key is accidentally exposed in a screenshot or log. Generate a new key at `console.anthropic.com` and update the header value in the HTTP Request node.
