# Ntopng Security Pipeline — N8N Workflow Documentation

This document describes the N8N automation workflow that processes Ntopng network alerts through Luna AI and delivers structured security analysis to Telegram with Ntfy escalation for critical threats.

---

## Pipeline Architecture

```
Ntopng → Webhook → IF (severity filter) → Code (build Claude body) → HTTP Request (Claude API) → Telegram → Ntfy (critical escalation)
```

---

## Node Configuration

### 1. Webhook
- **Type:** Webhook
- **Method:** POST
- **Path:** `/ntopng-alerts`
- **Authentication:** Shared secret header
- **Purpose:** Receives raw alert payloads from Ntopng when a network threat is detected

**Expected payload structure:**
```json
{
  "version": "0.2",
  "timestamp": 1234567890,
  "sharedsecret": "YOUR_SECRET",
  "alerts": [
    {
      "alert_category": 1,
      "cli_blacklisted": false,
      "srv_blacklisted": false,
      "cli_port": 12345,
      "srv_port": 80,
      "score": 30,
      "alert_type": "flow",
      "proto.ndpl_breed": "HTTP",
      "proto.l4": "TCP"
    }
  ]
}
```

---

### 2. IF Node — Severity Filter
- **Mode:** OR
- **Purpose:** Filters alerts by severity before passing to Luna. Low-severity alerts are silently dropped.

**Conditions:**

| Field | Operator | Value |
|-------|----------|-------|
| `$json.body.alerts[0].score` | Greater than or equal to | `50` |
| `$json.body.alerts[0].cli_blacklisted` | Is true | — |
| `$json.body.alerts[0].srv_blacklisted` | Is true | — |

- **Convert types where required:** ON
- **True branch:** Passes to Code node for Luna analysis
- **False branch:** Dropped — no action taken

**Escalation criteria reasoning:**
- Score ≥ 50 captures high-risk generic alerts
- Blacklisted client or server IP always escalates regardless of score, as blacklisted traffic represents known malicious infrastructure

---

### 3. Code Node — Build Claude API Body
- **Type:** Code (JavaScript)
- **Name:** Alerts
- **Purpose:** Transforms raw Ntopng webhook data into a properly structured Anthropic API request body

**Code:**
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

**Why a Code node:**
N8N's JSON body mode does not support `{{ }}` expressions inside multi-line JSON bodies. Raw mode sends the body as a string rather than a parsed object. The Code node builds a clean JSON object that the HTTP Request node passes through without parsing issues.

---

### 4. HTTP Request Node — Claude API (Luna)
- **Name:** Claude.AI
- **Method:** POST
- **URL:** `https://api.anthropic.com/v1/messages`
- **Body Content Type:** JSON
- **Specify Body:** Using JSON
- **Body:** `{{ $json }}`

**Headers:**
| Name | Value |
|------|-------|
| `x-api-key` | `YOUR_ANTHROPIC_API_KEY` |
| `anthropic-version` | `2023-06-01` |
| `content-type` | `application/json` |

---

### 5. Telegram Node — Luna Analysis Delivery
- **Type:** Telegram — Send a text message
- **Operation:** sendMessage
- **Chat ID:** `YOUR_TELEGRAM_CHAT_ID`
- **Message:** `{{ $json.content[0].text }}`
- **Parse Mode:** Markdown

**Purpose:** Delivers Luna's full SOC-style analysis including threat assessment, context, and recommended actions to the designated Telegram channel.

---

### 6. HTTP Request Node — Ntfy Escalation
- **Method:** POST
- **URL:** `http://YOUR_NTFY_IP:2586/homelab-alerts`
- **Body Content Type:** Raw
- **Content Type:** `text/plain`
- **Body:** `Ntopng detected a critical network event. Check Telegram for Luna's full analysis.`

**Headers:**
| Name | Value |
|------|-------|
| `Priority` | `urgent` |
| `Tags` | `rotating_light` |
| `Title` | `CRITICAL NETWORK ALERT` |

**Purpose:** Sends a loud push notification to the Ntfy mobile app that bypasses Do Not Disturb for critical alerts. Acts as a wake-up call pointing to the full Luna analysis in Telegram.

---

## Alert Flow Summary

```
Alert received
     │
     ▼
Score ≥ 50 OR cli_blacklisted OR srv_blacklisted?
     │
     ├── NO  → Dropped silently
     │
     └── YES
          │
          ▼
     Code node builds Claude API body
          │
          ▼
     Claude API (Luna) analyzes alert
          │
          ▼
     Telegram ← Full SOC-style analysis
          │
          ▼
     Ntfy ← Urgent push notification
```

---

## Environment Variables (N8N Docker)

```yaml
environment:
  - N8N_PERSONALIZATION_ENABLED=false
```

Disables N8N attribution footer from appearing in outgoing Telegram messages.

---

## Related Files

- [`troubleshooting.md`](../ntopng/troubleshooting.md) — Full troubleshooting log covering every error encountered during setup
- [`ntfy/docker-compose.yml`](../ntfy/docker-compose.yml) — Ntfy self-hosted notification server
- [`ntfy/server.yml`](../ntfy/server.yml) — Ntfy server configuration

---

*Pipeline built and documented by Scavenger — World of Hackers LLC*
