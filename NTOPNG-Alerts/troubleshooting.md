# Ntopng Security Pipeline — Troubleshooting Log

This document covers the full troubleshooting process to get the Ntopng → N8N → Luna (Claude AI) → Telegram pipeline working, including every error encountered and how it was resolved.

---

## Pipeline Overview

```
Ntopng Alerts → N8N Webhook → IF Node (severity filter) → Code Node (build Claude body) → HTTP Request (Claude API / Luna) → Telegram → Ntfy (escalation)
```

---

## Issue 1 — JSON parameter needs to be valid JSON

**Error:**
```
Problem in node 'HTTP Request'
JSON parameter needs to be valid JSON
```

**Cause:**
The HTTP Request node was set to **Body Content Type: JSON** with **Specify Body: Using JSON**. N8N's JSON mode does not support `{{ }}` expressions inside a multi-line JSON body. The expression `{{ JSON.stringify($json.body.alerts) }}` inside the content field caused N8N to reject the body as invalid JSON before sending it.

**Fix Attempted (Raw mode):**
Switched Body Content Type to **Raw** with Content Type `application/json` and placed the full JSON as a single-line expression using the `fx` button. This resolved the JSON validation error but introduced a new error.

---

## Issue 2 — Bad request: request body must be a JSON object, got str

**Error:**
```
Bad request - please check your parameters
The request body must be a JSON object, got str.
```

**Cause:**
Raw mode was sending the body as a plain string rather than a parsed JSON object. The Anthropic API rejected it because it expected a proper JSON object, not a string.

**Root Fix — Code Node:**
Instead of fighting with N8N's body modes, a **Code node (JavaScript)** was added between the Webhook and the HTTP Request node to build the Claude API body cleanly before it reaches the HTTP Request node.

**Code node contents:**
```javascript
const alerts = $input.first().json.body.alerts;

return [{
  json: {
    model: "claude-sonnet-4-5",
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

**HTTP Request node configuration after fix:**
- Body Content Type: `JSON`
- Specify Body: `Using JSON`
- JSON field: `{{ $json }}`

The Code node outputs a clean JSON object. The HTTP Request node passes it straight through without any expression parsing issues.

---

## Issue 3 — Code node placed in wrong position

**Symptom:**
The Code node input was showing Claude API response fields (`model`, `content`, `usage`) instead of raw webhook alert data (`alerts`, `score`, `cli_blacklisted`).

**Cause:**
The Code node was accidentally placed after the HTTP Request node instead of before it. The input data it was receiving was the Claude API response, not the Ntopng webhook payload.

**Fix:**
Rewired the canvas to the correct order:
```
Webhook → Code (Alerts) → HTTP Request (Claude.AI) → Telegram
```

---

## Issue 4 — API key exposed in screenshot

**Issue:**
During troubleshooting, a screenshot was shared that contained the Anthropic API key in plain text inside the N8N HTTP Request node header (`x-api-key`).

**Fix:**
- Deleted the exposed API key from the Anthropic console at `console.anthropic.com`
- Generated a new API key
- Updated the `x-api-key` header value in both the Ntopng and Wazuh N8N workflows with the new key

---

## Issue 5 — Git push returning 403 Permission Denied

**Error:**
```
remote: Permission to Scavenger503/devops-lab.git denied to Scavenger503.
fatal: unable to access 'https://github.com/Scavenger503/devops-lab.git/': The requested URL returned error: 403
```

**Cause:**
GitHub no longer accepts account passwords for Git operations. A Personal Access Token (PAT) is required. The credential helper was also caching a bad/expired token.

**Fix:**
1. Generated a new Personal Access Token (classic) at GitHub → Settings → Developer Settings → Personal Access Tokens with `repo` scope
2. Cleared the cached credentials:
```bash
git config --global --unset credential.helper
git config --global credential.helper store
```
3. Ran `git push origin main` again and entered the new PAT as the password when prompted
4. Credentials were saved and subsequent pushes no longer prompted for authentication

---

## Issue 6 — N8N attribution footer appearing in Telegram messages

**Symptom:**
Telegram messages sent by N8N included an unwanted footer:
```
This message was sent automatically with [n8n](https://n8n.io/...)
```

**Cause:**
A newer version of N8N added automatic branding attribution to outgoing messages.

**Fix:**
Added the following environment variable to the N8N Docker stack via Portainer:
```yaml
N8N_PERSONALIZATION_ENABLED=false
```
Restarted the N8N container for the change to take effect.

---

## Escalation Logic Added — IF Node

After the core pipeline was working, an IF node was added between the Webhook and the Code node to filter alerts by severity before sending them to Luna and Telegram.

**IF Node Position:**
```
Webhook → IF → true → Code (Alerts) → Claude.AI → Telegram → Ntfy
                false → (dropped, no action)
```

**Conditions (OR mode):**
| Field | Operator | Value |
|-------|----------|-------|
| `$json.body.alerts[0].score` | greater than or equal to | `50` |
| `$json.body.alerts[0].cli_blacklisted` | is true | — |
| `$json.body.alerts[0].srv_blacklisted` | is true | — |

**Convert types where required:** ON

**Reasoning:**
- Score ≥ 50 catches high-risk generic alerts
- Blacklisted client or server IP always escalates regardless of score
- Low severity alerts (score < 50, not blacklisted) are silently dropped to reduce noise

---

## Ntfy Escalation Node

An HTTP Request node was added after the Telegram node to send a loud push notification for critical alerts that bypass Do Not Disturb.

**Configuration:**
- Method: `POST`
- URL: `http://YOUR_NTFY_IP:2586/homelab-alerts`
- Headers:
  - `Priority: urgent`
  - `Tags: rotating_light`
  - `Title: CRITICAL NETWORK ALERT`
- Body Content Type: `Raw`
- Content Type: `text/plain`
- Body: `Ntopng detected a critical network event. Check Telegram for Luna's full analysis.`

---

## Final Working Pipeline

```
Ntopng → Webhook → IF (score ≥ 50 OR blacklisted) → Code (build Claude body) → HTTP Request (Claude API) → Telegram (Luna analysis) → Ntfy (urgent escalation)
```

Raw Ntopng Telegram notifications were disabled after Luna was confirmed working, as they provided no additional value over Luna's plain-English analysis.

---

*Pipeline built and documented by Scavenger — World of Hackers LLC*
