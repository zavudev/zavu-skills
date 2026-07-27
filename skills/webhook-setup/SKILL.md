---
name: webhook-setup
description: Configure webhooks to receive inbound messages and delivery updates with signature verification.
---

# Webhook Setup

## When to Use

Use this skill when setting up webhook endpoints to receive inbound messages, delivery status updates, or template approval notifications from Zavu.

## Webhook Types

- **Sender Webhooks**: Message events (inbound, delivery status, templates) - configured per sender
- **Project Webhooks**: Project-level events (partner invitations) - one per project

## Available Events

| Event | Category | Description |
|-------|----------|-------------|
| `message.inbound` | Inbound | Customer sent you a message |
| `conversation.new` | Inbound | First message from a new contact |
| `message.unsupported` | Inbound | Unsupported message type received |
| `message.queued` | Outbound | Message queued for delivery |
| `message.sent` | Outbound | Message sent to carrier |
| `message.delivered` | Outbound | Message delivered to recipient |
| `message.read` | Outbound | Message read by recipient |
| `message.failed` | Outbound | Message delivery failed |
| `broadcast.status_changed` | Broadcasts | Broadcast status changed |
| `template.status_changed` | Templates | WhatsApp template approval status changed |
| `invitation.status_changed` | Invitations | Partner invitation status changed |
| `domain.verified` | Domains | Custom email domain passed verification |
| `domain.failed` | Domains | Custom email domain failed verification |

## Configure Webhook via SDK

### TypeScript - Create Sender with Webhook

```typescript
const sender = await zavu.senders.create({
  name: "My Sender",
  phoneNumber: "+15551234567",
  webhookUrl: "https://your-app.com/webhooks/zavu",
  webhookEvents: ["message.inbound", "message.delivered", "message.failed"],
});
// Store sender.webhook.secret securely - only shown once!
```

### Update Webhook

```typescript
await zavu.senders.update({
  senderId: "snd_abc123",
  webhookUrl: "https://new-url.com/webhooks",
  webhookEvents: ["message.inbound"],
  webhookActive: true,
});
```

### Regenerate Secret

```typescript
const result = await zavu.senders.webhookSecret.regenerate({
  senderId: "snd_abc123",
});
console.log(result.secret); // whsec_new_secret...
```

## Webhook Payload Structure

The top-level envelope is the same for every event; event-specific fields live in `data`. `timestamp` is when Zavu dispatched the webhook (Unix ms).

```json
{
  "id": "evt_1705312200000_abc123",
  "type": "message.inbound",
  "timestamp": 1705312200000,
  "senderId": "snd_abc123",
  "projectId": "prj_xyz789",
  "data": { }
}
```

## Inbound Message Data (`message.inbound`)

For `message.inbound`, `data` carries the message. On inbound, `to` is your own number (the message's destination) and `from` is the sender.

| Field | Description |
|-------|-------------|
| `messageId` | Zavu message ID. |
| `conversationId` | Inbox thread id. `null` while the thread is still being created (use `conversation.new`'s id, or `GET /v1/messages/{id}`). See "Deep-linking to the inbox" below. |
| `from` | Sender: the contact for a 1:1, or the participant for a group message. |
| `to` | Your own number (the message's destination). |
| `channel` | Delivery channel (`sms`, `whatsapp`, `telegram`, `email`, `instagram`, `messenger`, `voice`). |
| `messageType` | `text`, `image`, `video`, etc. A reply to a `location_request` arrives as `location` (not a new type) with `content.replyToMessageId` set to the request — match on that to correlate. |
| `text` | Text body or media caption, when present. |
| `providerTimestamp` | The provider's original receive time (Unix ms) for WhatsApp, Telegram, Instagram, Messenger; `null` for SMS and email. Compare with the top-level `timestamp` to detect delayed deliveries. |

### Deep-linking to the inbox

Both `message.inbound` and `conversation.new` carry a `conversationId` in `data` — the id of the inbox thread. Build a direct link so your team can open the conversation in the Zavu dashboard:

```
https://dashboard.zavu.dev/{locale}/inbox?conv={conversationId}
```

`{locale}` is the dashboard UI language (e.g. `en`, `es`).

On `message.inbound`, `conversationId` is `null` while the conversation row is still being created — on the first message of a brand-new thread, and, if several messages from a never-seen address arrive near-simultaneously, on each of those (only one `conversation.new` is emitted). Recover the id from `conversation.new`, or fetch it any time from `GET /v1/messages/{messageId}`, whose `conversationId` is always populated.

### Reply / quote context

Present on `message.inbound` when the contact replied to (quoted) an earlier message, inside `data.content`.

| Field | Description |
|-------|-------------|
| `replyToMessageId` | Zavu message ID of the quoted message. Omitted if the quoted message is not stored in Zavu. |
| `replyToProviderMessageId` | Provider message ID (WhatsApp WAMID) of the quoted message. Present whenever it is a reply. |
| `replyToFrom` | Sender of the quoted message (E.164). |
| `replyToText` | Truncated snippet of the quoted message's text (empty for media). |
| `replyToMessageType` | Type of the quoted message (`text`, `image`, ...). |

### Email attachments

Inbound emails arrive as `message.inbound` with `channel: "email"` and `messageType: "text"`. The webhook payload carries only the body (`text`, and `htmlBody` on the fetched message) — it does **not** include attachment data. Attachments are stored separately and fetched on demand.

To retrieve them, call `GET /v1/messages/{messageId}/attachments` with the `messageId` from the webhook. It returns each attachment's metadata plus a short-lived signed `downloadUrl` (regenerated on every request — fetch promptly, don't cache the URL). This also works for outbound emails you sent with attachments. Messages without stored attachments return an empty list.

```bash
curl https://api.zavu.dev/v1/messages/MESSAGE_ID/attachments \
  -H "Authorization: Bearer $ZAVU_API_KEY"
```

```json
{
  "items": [
    {
      "id": "att_abc123",
      "filename": "invoice.pdf",
      "mimeType": "application/pdf",
      "size": 102400,
      "contentId": null,
      "isInline": false,
      "downloadUrl": "https://...signed-url...",
      "createdAt": "2024-01-15T10:00:00.000Z"
    }
  ]
}
```

Inline images embedded in the HTML body have `isInline: true` and a `contentId` referenced in `htmlBody` as `cid:<contentId>`.

## Signature Verification

Header: `X-Zavu-Signature: t=<timestamp>,v1=<hmac_sha256>`

### TypeScript (Express)

```typescript
import crypto from "crypto";
import express from "express";

const app = express();
app.use("/webhooks/zavu", express.raw({ type: "application/json" }));

function verifyZavuSignature(req: express.Request, secret: string): boolean {
  const header = req.headers["x-zavu-signature"] as string;
  if (!header) return false;

  const parts = header.split(",");
  const timestamp = parseInt(parts.find(p => p.startsWith("t="))!.slice(2));
  const signature = parts.find(p => p.startsWith("v1="))!.slice(3);

  // Reject if older than 5 minutes (replay protection)
  if (Math.floor(Date.now() / 1000) - timestamp > 300) return false;

  const signedPayload = `${timestamp}.${req.body.toString()}`;
  const expected = crypto.createHmac("sha256", secret).update(signedPayload).digest("hex");

  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
}

app.post("/webhooks/zavu", (req, res) => {
  if (!verifyZavuSignature(req, process.env.ZAVU_WEBHOOK_SECRET!)) {
    return res.status(401).send("Invalid signature");
  }

  const event = JSON.parse(req.body.toString());
  res.status(200).send("OK");

  // Process async
  processEvent(event).catch(console.error);
});

async function processEvent(event: any) {
  switch (event.type) {
    case "message.inbound":
      console.log("Inbound from:", event.data.from, event.data.text);
      break;
    case "message.delivered":
      console.log("Delivered:", event.data.messageId);
      break;
    case "message.failed":
      console.log("Failed:", event.data.messageId, event.data.errorMessage);
      break;
  }
}
```

### Python (Flask)

```python
import hmac, hashlib, time
from flask import Flask, request

app = Flask(__name__)

def verify_zavu_signature(req, secret):
    header = req.headers.get("X-Zavu-Signature")
    if not header:
        return False

    parts = header.split(",")
    timestamp = int(next(p for p in parts if p.startswith("t="))[2:])
    signature = next(p for p in parts if p.startswith("v1="))[3:]

    # Reject if older than 5 minutes
    if int(time.time()) - timestamp > 300:
        return False

    signed_payload = f"{timestamp}.{req.data.decode('utf-8')}"
    expected = hmac.new(
        secret.encode(), signed_payload.encode(), hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(expected, signature)

@app.route("/webhooks/zavu", methods=["POST"])
def handle_webhook():
    if not verify_zavu_signature(request, WEBHOOK_SECRET):
        return "Invalid signature", 401

    event = request.json
    # Process event...
    return "OK", 200
```

### Go

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func verifyZavuSignature(r *http.Request, secret string) ([]byte, bool) {
	header := r.Header.Get("X-Zavu-Signature")
	if header == "" {
		return nil, false
	}

	parts := strings.Split(header, ",")
	var timestamp int64
	var signature string
	for _, part := range parts {
		if strings.HasPrefix(part, "t=") {
			timestamp, _ = strconv.ParseInt(part[2:], 10, 64)
		} else if strings.HasPrefix(part, "v1=") {
			signature = part[3:]
		}
	}

	if time.Now().Unix()-timestamp > 300 {
		return nil, false
	}

	body, _ := io.ReadAll(r.Body)
	signedPayload := strconv.FormatInt(timestamp, 10) + "." + string(body)
	h := hmac.New(sha256.New, []byte(secret))
	h.Write([]byte(signedPayload))
	expected := hex.EncodeToString(h.Sum(nil))

	return body, hmac.Equal([]byte(expected), []byte(signature))
}

func main() {
	secret := os.Getenv("ZAVU_WEBHOOK_SECRET")
	http.HandleFunc("/webhooks/zavu", func(w http.ResponseWriter, r *http.Request) {
		body, valid := verifyZavuSignature(r, secret)
		if !valid {
			http.Error(w, "Invalid signature", http.StatusUnauthorized)
			return
		}

		var event map[string]interface{}
		json.Unmarshal(body, &event)
		// Process event...
		w.WriteHeader(http.StatusOK)
	})
	http.ListenAndServe(":3000", nil)
}
```

### Ruby (Sinatra)

```ruby
require "sinatra"
require "openssl"
require "json"

def verify_zavu_signature(request, secret)
  header = request.env["HTTP_X_ZAVU_SIGNATURE"]
  return false unless header

  parts = header.split(",")
  timestamp = parts.find { |p| p.start_with?("t=") }&.[](2..)&.to_i
  signature = parts.find { |p| p.start_with?("v1=") }&.[](3..)

  return false unless timestamp && signature
  return false if Time.now.to_i - timestamp > 300

  raw_body = request.body.read
  request.body.rewind
  signed_payload = "#{timestamp}.#{raw_body}"
  expected = OpenSSL::HMAC.hexdigest("SHA256", secret, signed_payload)

  Rack::Utils.secure_compare(expected, signature)
end

post "/webhooks/zavu" do
  halt 401, "Invalid signature" unless verify_zavu_signature(request, ENV["ZAVU_WEBHOOK_SECRET"])

  event = JSON.parse(request.body.read)
  # Process event...
  status 200
end
```

### PHP

```php
<?php
function verifyZavuSignature(string $secret): bool {
    $header = $_SERVER['HTTP_X_ZAVU_SIGNATURE'] ?? '';
    if (empty($header)) return false;

    $parts = explode(',', $header);
    $timestamp = $signature = null;
    foreach ($parts as $part) {
        if (str_starts_with($part, 't=')) $timestamp = (int) substr($part, 2);
        elseif (str_starts_with($part, 'v1=')) $signature = substr($part, 3);
    }

    if (!$timestamp || !$signature) return false;
    if (time() - $timestamp > 300) return false;

    $rawBody = file_get_contents('php://input');
    $expected = hash_hmac('sha256', "{$timestamp}.{$rawBody}", $secret);

    return hash_equals($expected, $signature);
}

if (!verifyZavuSignature(getenv('ZAVU_WEBHOOK_SECRET'))) {
    http_response_code(401);
    exit('Invalid signature');
}

$event = json_decode(file_get_contents('php://input'), true);
// Process event...
http_response_code(200);
```

## Retry Policy

| Attempt | Delay |
|---------|-------|
| 1st retry | 1 minute |
| 2nd retry | 5 minutes |
| 3rd retry | 15 minutes |
| 4th retry | 1 hour |
| 5th retry | 4 hours |

After 5 retries, delivery is marked as failed.

## Best Practices

1. **Return 200 quickly** - respond within 30 seconds, process async
2. **Verify signatures** - always verify in production
3. **Idempotent handlers** - check `event.id` to skip duplicates
4. **Use raw body** - signature is computed on raw body, not parsed JSON
5. **Test with ngrok** - expose local server for development
