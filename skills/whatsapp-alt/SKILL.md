---
name: whatsapp-alt
description: Link a WhatsApp number by scanning a QR code (no Meta Business Account, no template approval) and send/receive on the gated whatsapp_alt channel. Covers session lifecycle, egress (Zavu proxy vs Android device), groups, and stories.
---

# WhatsApp Alternative

## When to Use

Use this skill when the user wants to connect a WhatsApp number by **scanning a QR code** (the way WhatsApp Web links a device) instead of running Meta's embedded signup. There is no Meta Business Account, no template approval, and no 24-hour window. You link a number in seconds and send on the `whatsapp_alt` channel.

Good fits:
- The number is not set up on the official WhatsApp Business Platform (Cloud API).
- You need a client's number live fast, or you want to automate an existing personal/business number without migrating to a WABA.
- Two-way, consented conversations (inbound + outbound text, media, reactions, groups).

For scale, marketing, and compliance, use the official `whatsapp` channel with approved templates instead (see the `whatsapp-templates` skill).

> **Gated feature.** WhatsApp Alternative must be enabled for the team before the `whatsapp_alt` channel and the `/v1/whatsapp-alt/*` endpoints are available. It is **not available with test-mode API keys** (`zv_test_...`). Contact Zavu to enable it.

## How it differs from official WhatsApp

| | Official WhatsApp (Cloud API) | WhatsApp Alternative |
|---|---|---|
| Connection | Meta embedded signup (WABA) | Scan a QR code (like WhatsApp Web) |
| Setup time | Minutes to hours | Seconds |
| Templates | Required outside the 24h window | Not used |
| 24-hour window | Enforced by Meta | Not enforced |
| Egress | Meta's servers | Zavu residential proxy or your own device |
| Groups | Not exposed | Supported (send + receive) |
| Best for | Scale, compliance, marketing | Quick links, existing numbers, conversations |

## Session lifecycle

A **session** is one linked-device connection to WhatsApp. You create it, poll for a QR code, scan the QR from the phone, then link it to a sender.

| Status | Meaning |
|--------|---------|
| `initializing` | Session created, the connection is starting. |
| `qr_ready` | `qrCode` is available. Scan it from the phone to link. |
| `authenticating` | QR scanned, handshaking. |
| `ready` | Linked and connected. You can send and receive. |
| `disconnected` | Unlinked or dropped (Zavu reconnects automatically when possible). |
| `failed` | Could not connect (see `lastError`). |

## Session workflow

> The `/v1/whatsapp-alt/*` endpoints (sessions, egress, proxy status, egress nodes) are REST-only today. They are not yet exposed as methods in `@zavudev/sdk`, so the examples below use `curl`. Sending on the channel uses the normal `zavu.messages.send` SDK call (see [Sending](#sending-on-the-whatsapp_alt-channel)).

### 1. Create a session

By default the session egresses through the managed **Zavu residential proxy**, geo-matched to the number's country. Pass an `egress` object to route through your own Android device instead (see [Egress](#egress)).

```bash
curl -X POST https://api.zavu.dev/v1/whatsapp-alt/sessions \
  -H "Authorization: Bearer $ZAVU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "displayName": "Support line" }'
```

The session starts in `initializing`. The QR is generated asynchronously.

### 2. Poll for the QR code

Poll every 2 to 3 seconds until `status` is `qr_ready`. The `qrCode` field is the QR payload. Render it as a QR image for the user to scan. It rotates until scanned, so keep polling and re-render when it changes.

```bash
curl https://api.zavu.dev/v1/whatsapp-alt/sessions/SESSION_ID \
  -H "Authorization: Bearer $ZAVU_API_KEY"
```

Poll this until `status` is `qr_ready`, then render `session.qrCode` as a QR image for the user to scan. It rotates until scanned, so re-render whenever it changes.

### 3. Scan from the phone

On the phone, open **WhatsApp -> Linked devices -> Link a device**, then scan the QR. The session moves through `authenticating` to `ready`, and `phoneNumber` and `pushName` are populated. A number can be linked to several devices at once, so linking to Zavu does not log it out of the phone. Do not scan the same session's QR from two places.

### 4. Link the session to a sender

Attach the ready session to a sender so its outbound `whatsapp_alt` messages route through this number and inbound messages are attributed to it.

```bash
curl -X POST https://api.zavu.dev/v1/whatsapp-alt/sessions/SESSION_ID/link \
  -H "Authorization: Bearer $ZAVU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "senderId": "sender_12345" }'
```

## Managing a session

| Action | Endpoint | Notes |
|--------|----------|-------|
| List | `GET /v1/whatsapp-alt/sessions` | Paginated (`limit`, `cursor`). |
| Get | `GET /v1/whatsapp-alt/sessions/{sessionId}` | Live `status` + `qrCode` while linking. |
| Set egress | `POST /v1/whatsapp-alt/sessions/{sessionId}/egress` | Change how the number reaches WhatsApp. Takes effect on next connect. |
| Reconnect | `POST /v1/whatsapp-alt/sessions/{sessionId}/reconnect` | Re-initialize after `failed`/`disconnected`. Reconnects without a new QR if credentials are still valid. |
| Log out | `POST /v1/whatsapp-alt/sessions/{sessionId}/logout` | Clears credentials; the device disappears from the phone's Linked devices. The session row is kept. Reconnect to link again with a fresh QR. |
| Link | `POST /v1/whatsapp-alt/sessions/{sessionId}/link` | Attach the session to a sender. |
| Unlink | `POST /v1/whatsapp-alt/sessions/{sessionId}/unlink` | Detach every sender pointing at this session. |
| Delete | `DELETE /v1/whatsapp-alt/sessions/{sessionId}` | Deletes the session, unlinks its senders, and logs the device out. |

**Reconnect vs logout:** reconnect keeps the link alive (no QR), while logout unlinks the device and requires scanning a new QR to reconnect.

```bash
# Reconnect a dropped session (no new QR if credentials are valid)
curl -X POST https://api.zavu.dev/v1/whatsapp-alt/sessions/SESSION_ID/reconnect \
  -H "Authorization: Bearer $ZAVU_API_KEY"

# Log out (device removed from the phone's Linked devices; row kept)
curl -X POST https://api.zavu.dev/v1/whatsapp-alt/sessions/SESSION_ID/logout \
  -H "Authorization: Bearer $ZAVU_API_KEY"

# Detach every sender from the session
curl -X POST https://api.zavu.dev/v1/whatsapp-alt/sessions/SESSION_ID/unlink \
  -H "Authorization: Bearer $ZAVU_API_KEY"

# Delete the session entirely (unlinks senders + logs the device out)
curl -X DELETE https://api.zavu.dev/v1/whatsapp-alt/sessions/SESSION_ID \
  -H "Authorization: Bearer $ZAVU_API_KEY"
```

## Egress

WhatsApp watches the network path a linked number connects from. To keep a number healthy, every session egresses through an **in-country residential IP** and always appears from the **same exit**, never a datacenter or server IP.

> **Direct egress (a server / datacenter IP) is never allowed** and can never be set through the API. It is a guaranteed block. Every session must use either the Zavu residential proxy or an Android egress device.

| Egress | `kind` | Description |
|--------|--------|-------------|
| Zavu Residential Proxy | `external` (default) | Zavu routes the number through a managed in-country residential IP, geo-matched to the number's country and pinned stable per session. Metered and billed per GB. |
| Android Egress Device | `android` | The number egresses through a phone you control running the Zavu Egress Node app. Uses the device's own connection, no per-GB charge. Requires `nodeId`. |

### Set the egress on a session

Pass `egress` when creating a session, or change it later. `country` is optional. When omitted, Zavu derives it from the number so the exit matches the number's own country.

```bash
# Zavu proxy (default)
curl -X POST https://api.zavu.dev/v1/whatsapp-alt/sessions \
  -H "Authorization: Bearer $ZAVU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "Support line",
    "egress": { "kind": "external", "country": "cl" }
  }'

# Android device
curl -X POST https://api.zavu.dev/v1/whatsapp-alt/sessions \
  -H "Authorization: Bearer $ZAVU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "Support line",
    "egress": { "kind": "android", "nodeId": "nd_abc123" }
  }'

# Change egress on an existing session (applies on next connect)
curl -X POST https://api.zavu.dev/v1/whatsapp-alt/sessions/SESSION_ID/egress \
  -H "Authorization: Bearer $ZAVU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "egress": { "kind": "android", "nodeId": "nd_abc123" } }'
```

### Is the managed proxy available?

Check whether the managed Zavu proxy is currently offered for your team before defaulting to it. When `zavuProxyAvailable` is `false`, pair an Android egress node instead.

```bash
curl https://api.zavu.dev/v1/whatsapp-alt/proxy-status \
  -H "Authorization: Bearer $ZAVU_API_KEY"
```

The response is `{ "zavuProxyAvailable": true | false }`.

### Pairing an Android egress node

An egress node is a phone running the Zavu Egress Node app that lends its connection to a sender's sessions.

1. **Create the node.** The response includes a short-lived pairing payload (`token`, `gatewayUrl`, `wsPath`).

   ```bash
   curl -X POST https://api.zavu.dev/v1/whatsapp-alt/egress-nodes \
     -H "Authorization: Bearer $ZAVU_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{ "senderId": "sender_12345", "label": "Store phone" }'
   ```

2. **Pair the device.** Encode `token`, `gatewayUrl`, and `wsPath` into a QR and scan it from the Zavu Egress Node Android app. The token is not stored. Re-issue it with `POST /v1/whatsapp-alt/egress-nodes/{nodeId}/pairing-token` to show the QR again.

3. **Use the node.** Once it is `online`, set a session's egress to `{ "kind": "android", "nodeId": "<nodeId>" }`.

Node status is `pending` (paired but never connected), `online` (tunnel live), or `offline` (last tunnel dropped). List a sender's nodes with `GET /v1/whatsapp-alt/egress-nodes?senderId=...`, get one with `GET /v1/whatsapp-alt/egress-nodes/{nodeId}`, and revoke one with `DELETE /v1/whatsapp-alt/egress-nodes/{nodeId}` (this also kills its live tunnel).

### Automatic fallback and per-GB billing

When a session runs on an Android node and the device goes offline, Zavu can **automatically fall back to the managed Zavu proxy** so the number stays connected, then switch back when the device returns.

- Traffic through the Zavu residential proxy (`external`, including active fallback) is metered and charged per GB to your team balance. The per-GB rate depends on your plan, starting at **$2.99/GB** and decreasing on higher plans (down to **$0.99/GB**).
- Traffic through an Android egress device is **not charged**. It uses the device's own bandwidth.
- If the device is offline and fallback is not enabled, the session waits for the device to return without dropping the link (no re-scan needed).

## Sending on the whatsapp_alt channel

Set `channel: "whatsapp_alt"` and target the sender that owns the linked session with the `Zavu-Sender` header (or omit it to use the project default).

```bash
curl -X POST https://api.zavu.dev/v1/messages \
  -H "Authorization: Bearer $ZAVU_API_KEY" \
  -H "Zavu-Sender: sender_12345" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "+14155551234",
    "channel": "whatsapp_alt",
    "text": "Hello from Zavu!"
  }'
```

```typescript
const { message } = await zavu.messages.send({
  to: "+14155551234",
  channel: "whatsapp_alt",
  text: "Hello from Zavu!",
  "Zavu-Sender": "sender_12345",
});
```

```python
resp = zavu.messages.send(
    to="+14155551234",
    channel="whatsapp_alt",
    text="Hello from Zavu!",
    extra_headers={"Zavu-Sender": "sender_12345"},
)
```

**Supported content:** text, media (`image`, `video`, `audio`, `document`, `sticker`), `location`, `contact`, and `reaction`. Media and reactions are sent exactly like the official WhatsApp channel (set `messageType` + `content`); see the `send-message` skill. This channel does **not** use templates or the 24-hour window.

```json
{
  "to": "+14155551234",
  "channel": "whatsapp_alt",
  "messageType": "image",
  "text": "Check this out",
  "content": { "mediaUrl": "https://example.com/photo.jpg" }
}
```

> The send is rejected if the team does not have WhatsApp Alternative enabled or the sender has no connected `whatsapp_alt` session. Keep to consented, conversational messaging. High-volume or unsolicited sending risks WhatsApp blocking the number.

## Receiving

Inbound messages arrive through your sender's webhook as `message.inbound` events with the **same shape as every other channel**, so your existing handler works without changes. Subscribe to `message.inbound` on the sender and verify the signature as documented in the `webhook-setup` skill. Delivery and read receipts arrive as `message.delivered` / `message.read` for outbound messages.

```json
{
  "id": "evt_1736850000000_abc123",
  "type": "message.inbound",
  "timestamp": 1736850000000,
  "senderId": "sender_12345",
  "projectId": "proj_abc",
  "data": {
    "messageId": "jx77...",
    "to": "+13125551212",
    "from": "+14155551234",
    "channel": "whatsapp_alt",
    "messageType": "text",
    "text": "Hi there",
    "status": "received",
    "providerTimestamp": 1736849999000
  }
}
```

## Groups

Group messages are delivered on the **same** `message.inbound` event, with extra group fields on `data`. There is no separate event type. The conversation is keyed on the **group**, while `from` is the **participant** who sent the message.

```json message.inbound (group)
{
  "type": "message.inbound",
  "data": {
    "messageId": "jx77...",
    "channel": "whatsapp_alt",
    "from": "+14155551234",
    "to": "+13125551212",
    "messageType": "text",
    "text": "Anyone free at 3pm?",
    "status": "delivered",
    "isGroup": true,
    "groupId": "120363000000000000@g.us",
    "groupAuthor": "+14155551234",
    "groupName": "Weekend Trip",
    "providerTimestamp": 1736849999000
  }
}
```

| Field | Present | Description |
|-------|---------|-------------|
| `isGroup` | groups only | `true` for a group message. Absent/falsy for a one-to-one message. |
| `groupId` | groups only | The group's JID (`<id>@g.us`). This is the conversation key. |
| `groupAuthor` | groups only | The participant who sent the message, in E.164. Same value as `from`. |
| `groupName` | when known | The group's display name (subject). |
| `providerTimestamp` | when known | When WhatsApp originally received the message (Unix ms). |

Branch on `isGroup` to tell a group message from a one-to-one:

```typescript
app.post("/webhooks/zavu", (req, res) => {
  const { type, data } = req.body;
  if (type === "message.inbound" && data.channel === "whatsapp_alt") {
    if (data.isGroup) {
      // Attribute to the participant, thread on the group.
      console.log(`[${data.groupName ?? data.groupId}] ${data.groupAuthor}: ${data.text}`);
    } else {
      console.log(`${data.from}: ${data.text}`);
    }
  }
  res.sendStatus(200);
});
```

### Sending to a group

Send to a group by passing its JID (`<id>@g.us`) as `to` on the `whatsapp_alt` channel. Use the `groupId` from an inbound group message. You must have received at least one message from the group so you know its JID. Media works the same way, with the group JID as `to`.

```bash
curl https://api.zavu.dev/v1/messages \
  -H "Authorization: Bearer $ZAVU_API_KEY" \
  -H "Zavu-Sender: sender_12345" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "120363000000000000@g.us",
    "channel": "whatsapp_alt",
    "text": "Reminder: we leave at 3pm."
  }'
```

```typescript
await zavu.messages.send({
  to: "120363000000000000@g.us",
  channel: "whatsapp_alt",
  text: "Reminder: we leave at 3pm.",
});
```

Group JIDs are only valid on the `whatsapp_alt` channel. Sending a `<id>@g.us` recipient on any other channel is rejected. Group membership-change events (joins, removals, subject changes) are not emitted today.

## Stories & Status

When a contact posts a WhatsApp **status** (a story), the linked number receives it as a dedicated **`message.status`** webhook event, **not** as an inbound message. It never enters the inbox and never creates a conversation. It is **opt-in** (currently emitted for `whatsapp_alt` only): add `message.status` to the sender's `webhookEvents`, or story updates are silently dropped. Media bytes are not downloaded, only metadata and any text/caption.

```bash
curl -X PATCH https://api.zavu.dev/v1/senders/sender_12345 \
  -H "Authorization: Bearer $ZAVU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "webhookUrl": "https://api.example.com/webhooks/zavu",
    "webhookEvents": ["message.inbound", "message.status"]
  }'
```

```typescript
await zavu.senders.update("sender_12345", {
  webhookUrl: "https://api.example.com/webhooks/zavu",
  webhookEvents: ["message.inbound", "message.status"],
});
```

The event payload:

```json message.status
{
  "type": "message.status",
  "data": {
    "messageId": "jx77...",
    "channel": "whatsapp_alt",
    "from": "+14155551234",
    "to": "+13125551212",
    "messageType": "image",
    "text": "At the beach",
    "mimetype": "image/jpeg",
    "providerTimestamp": 1736849999000
  }
}
```

| Field | Present | Description |
|-------|---------|-------------|
| `from` | always | The contact who posted the status, in E.164. |
| `to` | always | Your linked number. |
| `messageType` | always | The story type: `text`, `image`, `video`, or `audio`. |
| `text` | when present | The caption, or the text of a text story. |
| `mimetype` | media stories | MIME type of the story media (bytes are not included). |
| `providerTimestamp` | when known | When WhatsApp originally received the status (Unix ms). |

Your own status posts are not delivered. Only statuses from your contacts trigger the event.

## Connecting clients via invitations

If you are a partner onboarding clients, send a QR-link invitation instead of driving the session API yourself: set `connectionType: "whatsapp_alt"` when creating a partner invitation (`POST /v1/invitations`). The client scans the QR and the resulting sender lands in your project. This also requires the WhatsApp Alternative feature to be enabled (otherwise the request returns 400).

## Constraints

- Gated feature: must be enabled per team. **Not available with test-mode API keys** (all `/v1/whatsapp-alt/*` endpoints and the `whatsapp_alt` channel).
- Every session must egress via `external` (Zavu proxy) or `android` (egress node). `direct` egress is never allowed and cannot be set.
- The sender must have a connected `whatsapp_alt` session, or sends on the channel are rejected.
- No templates and no 24-hour window on this channel. Use the official `whatsapp` channel for those.
- Group JIDs (`<id>@g.us`) are valid only on `whatsapp_alt`. You must have received a message from a group to learn its JID before sending to it.
- `message.status` (stories) is opt-in and `whatsapp_alt`-only; media bytes are never downloaded.
- Poll the session every 2 to 3 seconds for the QR; it rotates until scanned. Do not scan the same session's QR from two places.
- Zavu proxy (`external`) traffic is billed per GB ($2.99/GB down to $0.99/GB by plan); Android egress reports no billable bytes.
- Keep to consented, conversational messaging. High-volume or unsolicited sending risks the number being blocked by WhatsApp.
