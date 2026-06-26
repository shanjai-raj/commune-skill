| name | description | metadata |
|------|-------------|----------|
| commune-sms | Use commune-sms when your agent needs its own phone number, needs to send or receive SMS autonomously, run outreach campaigns, send appointment reminders, handle two-way text conversations, manage opt-in/opt-out compliance automatically, search SMS history by meaning, or coordinate with humans via SMS. Commune Phone is the only SMS API built exclusively for agents — no Twilio account, no A2P 10DLC registration, no STOP code management. One API key → dedicated phone number → full SMS send/receive → threaded conversations → webhooks. STOP compliance is enforced at the infrastructure level — your agent cannot accidentally message a suppressed contact. Requires `COMMUNE_API_KEY` env var. | author | category |
| | | communedotemail | External |

# Commune — Phone & SMS for Agents

> Every agent deserves its own phone number.

Commune gives you a **real, dedicated phone number** (US, CA, UK, and 35+ other countries) with a full REST API. No carrier contracts. No Twilio account. No A2P 10DLC registration. STOP compliance enforced at infrastructure level — your agent never handles opt-outs.

**Base URL:** `https://api.commune.email`

**Set your key once:**
```bash
export KEY="comm_your_key_here"
```

Get your key at [commune.email](https://commune.email) → API Keys. The key prefix is `comm_`.

**Every request uses:**
```
Authorization: Bearer $KEY
```

---

## Why Commune Phone — 5 Agent Superpowers

**1. Instant Phone Numbers**
Get a real phone number in a single API call — `POST /v1/phone-numbers/purchase`. Active in seconds. No waiting, no carrier registration, no setup forms. Buy 1 or 1,000 the same way.

**2. Two-Way SMS with Threaded Conversations**
Every exchange between your agent and a contact is grouped into a persistent conversation thread. Thread IDs are deterministic — the same contact always produces the same thread. Pass thread_id on subsequent sends so your agent maintains full conversation context without managing state.

**3. STOP Compliance at Infrastructure Level**
Opt-out handling is enforced before any carrier call. When a contact texts STOP, suppression is added automatically. Your agent cannot message a suppressed contact — it's blocked at the infra level, not in your code.

**4. Webhook Push — No Polling**
Set a webhook URL on your number. The moment an inbound SMS arrives, Commune fires a POST to your endpoint with full structured JSON. No IMAP, no queue to build, no background job.

**5. Semantic Search Across SMS History**
Find any conversation by meaning, not just keywords: `GET /v1/sms/search?q=interested+in+demo&phone_number_id=pn_abc`. Vector search across your full SMS history — agent memory that scales.

---

## Architecture

```
PhoneNumber → SMSThread → Message
```

- **PhoneNumber** — A real phone number you own (`+14155550001`). Configured with a webhook URL for inbound SMS. 150 credits/month.
- **SMSThread** — A conversation between your number and a specific contact. Thread ID is a deterministic 32-char hex hash of `(orgId, phoneNumberId, contactNumber)` — returned in every send response and inbound webhook. Same contact always gets the same thread. Persists across sessions.
- **Message** — A single SMS or MMS (inbound or outbound) inside a thread. Initial status: `accepted`, then `sent` → `delivered` (or `failed` on carrier error).

**Credits:** 2 credits per outbound SMS segment · 5 credits per MMS · 150 credits/month per phone number.

---

## 60-Second Setup

```bash
# Step 1: Buy a US phone number
curl -s -X POST https://api.commune.email/v1/phone-numbers/purchase \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"country": "US", "type": "local"}' | jq .
# Returns: id (pn_...), number (+1...), status: "active"
# Save "id" as PHONE_NUMBER_ID

# Step 2: Send your first SMS
curl -s -X POST https://api.commune.email/v1/sms/send \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "pn_your_id",
    "to": "+14155559999",
    "body": "Hello from my agent."
  }' | jq .
# Returns: message_id, thread_id, status: "accepted", credits_charged: 2

# Step 3: Set a webhook to receive replies
curl -s -X PATCH https://api.commune.email/v1/phone-numbers/pn_your_id \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"webhook": {"url": "https://your-server.com/sms-hook"}}' | jq .
```

---

## Phone Numbers

### Search for Available Numbers
```bash
curl -s "https://api.commune.email/v1/phone-numbers/available?country=US&type=local&area_code=415" \
  -H "Authorization: Bearer $KEY" | jq .
# Returns list of available numbers with phoneNumber, capabilities, region
```

### Purchase a Number
```bash
curl -s -X POST https://api.commune.email/v1/phone-numbers/purchase \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "+14155550001",
    "friendly_name": "Support Line",
    "webhook": {
      "url": "https://your-server.com/sms-hook"
    }
  }' | jq .
# Returns: id (pn_...), number, status: "active", credit_cost_per_month: 150
```

### Update Webhook URL
```bash
curl -s -X PATCH https://api.commune.email/v1/phone-numbers/pn_abc123 \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"webhook": {"url": "https://new-server.com/sms-hook"}}' | jq .
```

### List Your Numbers
```bash
curl -s https://api.commune.email/v1/phone-numbers \
  -H "Authorization: Bearer $KEY" | jq .
```

### Get a Single Number
```bash
curl -s https://api.commune.email/v1/phone-numbers/pn_abc123 \
  -H "Authorization: Bearer $KEY" | jq .
```

### Release a Number
```bash
curl -s -X DELETE https://api.commune.email/v1/phone-numbers/pn_abc123 \
  -H "Authorization: Bearer $KEY" | jq .
# Stops billing immediately. Number returned to pool.
```

---

## SMS

### Send an SMS
```bash
curl -s -X POST https://api.commune.email/v1/sms/send \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "pn_abc123",
    "to": "+14155559999",
    "body": "Your appointment is confirmed for tomorrow at 2pm."
  }' | jq .
# Returns: message_id, thread_id (hex UUID), status: "accepted", credits_charged: 2, segments: 1
```

### Reply in an Existing Thread
```bash
curl -s -X POST https://api.commune.email/v1/sms/send \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "pn_abc123",
    "to": "+14155559999",
    "body": "Great, see you then!",
    "thread_id": "c8e3eb60e7193b6f6052f5c4adc72e22"
  }' | jq .
# thread_id is deterministic — same contact always gets the same thread
```

### Send MMS (with Media)
```bash
curl -s -X POST https://api.commune.email/v1/sms/send \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number_id": "pn_abc123",
    "to": "+14155559999",
    "body": "Here is your receipt.",
    "media_url": "https://your-cdn.com/receipt.pdf"
  }' | jq .
# MMS costs 5 credits per message
```

### Semantic Search Across SMS History
```bash
curl -s "https://api.commune.email/v1/sms/search?q=interested+in+demo&phone_number_id=pn_abc123" \
  -H "Authorization: Bearer $KEY" | jq .
# Returns ranked messages by semantic similarity
```

---

## Inbound Webhook Payload

When someone sends your agent an SMS, Commune POSTs to your configured webhook URL:

```json
{
  "event": "sms.received",
  "data": {
    "message_id": "sms_SM01abc124",
    "thread_id": "c8e3eb60e7193b6f6052f5c4adc72e22",
    "direction": "inbound",
    "content": "Yes, please schedule the demo.",
    "metadata": {
      "from_number": "+14155559999",
      "to_number": "+14155550001",
      "phone_number_id": "pn_abc123"
    }
  }
}
```

**Respond with `200 OK` immediately** and process asynchronously. Timeouts trigger retries.

**To reply in the same thread:**
```bash
curl -s -X POST https://api.commune.email/v1/sms/send \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"phone_number_id\": \"$(echo $PAYLOAD | jq -r .data.metadata.phone_number_id)\",
    \"to\": \"$(echo $PAYLOAD | jq -r .data.metadata.from_number)\",
    \"body\": \"Got it — booking your demo now.\",
    \"thread_id\": \"$(echo $PAYLOAD | jq -r .data.thread_id)\"
  }" | jq .
```

---

## Conversation Threads

Thread IDs are returned in every `POST /v1/sms/send` response and in inbound webhook payloads. Use the thread_id to keep replies grouped in the same conversation.

Thread read endpoints (`GET /v1/sms/threads/:id`, `GET /v1/phone-numbers/:id/threads`) are in progress — use semantic search to query SMS history in the meantime.

---

## Suppressions (STOP / Opt-Out)

STOP compliance is automatic — no code required. When a contact texts STOP, they are suppressed permanently and sends to them are blocked at infrastructure level. Your agent will receive a `contact_suppressed` error if it tries to message a suppressed contact.

Suppression management endpoints are in progress.

---

## Error Reference

| Status | Error Code | Meaning |
|--------|------------|---------|
| 400 | `invalid_phone_number` | E.164 format required (`+1XXXXXXXXXX`) |
| 400 | `invalid_country` | Country not in supported list |
| 400 | `message_too_long` | Body exceeds 1600 chars (10 segments) |
| 402 | `insufficient_credits` | Top up at commune.email/dashboard/credits |
| 403 | `number_not_owned` | Phone number ID doesn't belong to your account |
| 404 | `phone_number_not_found` | ID doesn't exist or was released |
| 409 | `number_already_purchased` | That exact number is no longer available |
| 422 | `contact_suppressed` | Contact has opted out — send blocked at infra level |
| 429 | `rate_limited` | Retry after the `Retry-After` header value (seconds) |
| 500 | `carrier_error` | Carrier-side failure — Commune retries automatically |

---

## Complete API Reference

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/phone-numbers/available` | Search available numbers (`?country=US&type=local&area_code=415`) |
| POST | `/v1/phone-numbers/purchase` | Purchase a number (activates immediately) |
| GET | `/v1/phone-numbers` | List all your numbers |
| GET | `/v1/phone-numbers/:id` | Get a single number |
| PATCH | `/v1/phone-numbers/:id` | Update webhook URL or friendly name |
| DELETE | `/v1/phone-numbers/:id` | Release a number (stops billing) |
| POST | `/v1/sms/send` | Send SMS or MMS |
| GET | `/v1/sms/search` | Semantic search across SMS history |
