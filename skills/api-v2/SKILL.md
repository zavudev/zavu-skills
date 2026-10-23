---
name: api-v2
description: Use Zavu API v2 from the TypeScript SDK (client.v2) or over HTTP — send from an address instead of a sender, connect channels, reply by conversationId, configure the one project webhook, and act on flat errors and details.connect.
---

# Zavu API v2

## When to Use

Use this skill when code calls `https://api.zavu.dev/v2`, or when the user asks for v2. v2 is a smaller version of the same API: no sender object, no `Zavu-Sender` header, one project webhook, one error shape.

v1 is not deprecated. It keeps working unchanged, and v1 and v2 calls can be mixed in one codebase with the same API key and the same IDs. Use the other skills for v1.

**From TypeScript, use `client.v2` of `@zavudev/sdk` 0.59.0 or later.** Every other SDK (`zavudev` on PyPI and the rest) calls v1 only: from those languages, call v2 with `fetch`, `curl` or any HTTP client. The HTTP examples below are what `client.v2` sends.

```typescript
import Zavudev from "@zavudev/sdk";
import type { MessageSendParams } from "@zavudev/sdk/v2"; // v2 types; the root exports v1's

const zavu = new Zavudev(); // reads ZAVUDEV_API_KEY

const params: MessageSendParams = { from: "+13125551212", to: "+56912345678", text: "Hi" };
const { message, routing } = await zavu.v2.messages.send(params);

// Every path id is positional, in path order. Pass the address raw: the SDK encodes it.
await zavu.v2.channels.activate("+13125551212", "whatsapp");
await zavu.v2.agents.channels.disconnect("agent_abc123", "+13125551212", "sms");

// Lists page for you.
for await (const entry of zavu.v2.channels.list()) console.log(entry.address, entry.type);

// Errors are v1's classes; the v2 body `{code, message, details}` is on `err.error`.
try {
  await zavu.v2.messages.send({ to: "+56912345678", text: "Hi" });
} catch (err) {
  if (!(err instanceof Zavudev.ConflictError)) throw err;
  const body = err.error as { code?: string; details?: { connect?: unknown[] } } | undefined;
  if (body?.code === "no_channel") console.log(body.details?.connect);
}
```

Methods: `messages` (`send`, `list`, `retrieve`), `conversations` (`list`, `retrieve`, `listMessages`, `markAsRead`), `contacts` (`list`, `create`, `retrieve`, `update`, `delete`), `channels` (`list`, `connect`, `retrieve`, `activate`, `deactivate`, `disconnect`), `templates` (`list`, `create`, `retrieve`, `delete`, `submit`), `campaigns` (`list`, `create`, `retrieve`, `delete`, `send`, `progress`, `cancel`, `recipients.add`), `webhook` (`retrieve`, `set`, `delete`, `rotateSecret`), `agents` (`list`, `create`, `retrieve`, `update`, `delete`, `test`, `channels.connect`, `channels.disconnect`), `numbers` (`searchAvailable`, `purchase`), `emailDomains` (`list`, `create`, `retrieve`, `delete`, `verify`), `functions` (`list`, `retrieve`, `update`).

The rest of this skill shows the HTTP calls, through the `zavu()` helper under Basics.

## Basics

- Base URL: `https://api.zavu.dev/v2`
- Auth: `Authorization: Bearer $ZAVUDEV_API_KEY` (the same key as v1)
- Rate limit: per project, shared with v1
- Bodies: JSON. A body that is not JSON is `400 invalid_json`.
- Lists: `{ items, nextCursor }`. Pass `nextCursor` back as `cursor`; stop when it is `null`.
- Money: USD, rounded to 6 decimals.
- A test-mode key always sends from the WhatsApp sandbox; `from` and `conversationId` are ignored.

```typescript
const ZAVU = "https://api.zavu.dev/v2";

async function zavu(method: string, path: string, body?: unknown) {
  const res = await fetch(`${ZAVU}${path}`, {
    method,
    headers: {
      Authorization: `Bearer ${process.env.ZAVUDEV_API_KEY}`,
      ...(body !== undefined ? { "Content-Type": "application/json" } : {}),
    },
    body: body !== undefined ? JSON.stringify(body) : undefined,
  });
  if (res.status === 204) return null;
  const json = await res.json();
  if (!res.ok) throw Object.assign(new Error(json.message), { status: res.status, code: json.code, details: json.details });
  return json;
}
```

## Addresses

v2 names where a message leaves from the way the recipient sees it.

| Type | Address |
|------|---------|
| `sms`, `voice`, `whatsapp` | E.164 number |
| `email` | The email address, lowercase |
| `telegram` | `@botname` |
| `instagram` | `@username` |
| `messenger` | Facebook Page id |
| `sms_oneway` | The literal `sms_oneway` (shared numbers, never your own) |

`GET /v2/channels` lists one entry per (address, type): `{address, type, active, v1SenderId}`. A number with WhatsApp and SMS is two entries. `active: false` means connected but switched off. `v1SenderId` is the sender to pass as `Zavu-Sender` on v1 calls.

In URL paths over HTTP, encode the address: `+` is `%2B`. With `client.v2`, pass it raw.

## Connect a channel

`POST /v2/channels` connects what needs no purchase and no Meta signup:

```bash
# Email: the domain must be a verified email domain of the project
curl -X POST https://api.zavu.dev/v2/channels \
  -H "Authorization: Bearer $ZAVUDEV_API_KEY" -H "Content-Type: application/json" \
  -d '{"type": "email", "address": "noreply@yourdomain.com", "fromName": "Acme"}'

# Telegram: bot token from @BotFather
curl -X POST https://api.zavu.dev/v2/channels \
  -H "Authorization: Bearer $ZAVUDEV_API_KEY" -H "Content-Type: application/json" \
  -d '{"type": "telegram", "botToken": "123456789:AA..."}'

# One-way SMS: no number or credential; recipients cannot reply. +1 (US/Canada) numbers need an
# approved 10DLC campaign first (403 ten_dlc_required): use email or Telegram for them meanwhile
curl -X POST https://api.zavu.dev/v2/channels \
  -H "Authorization: Bearer $ZAVUDEV_API_KEY" -H "Content-Type: application/json" \
  -d '{"type": "sms_oneway"}'
```

- **Email domain first:** `POST /v2/email-domains {"domain": "yourdomain.com"}` → publish the returned `dnsRecords` exactly → `POST /v2/email-domains/{emailDomainId}/verify`. Connecting at a missing or unverified domain is `409 email_domain_missing` / `409 email_domain_not_verified`. A domain returned with `verifiable: false` can never verify: remove it and add it again.
- **Two-way SMS and voice** come with a number: `GET /v2/numbers/available?countryCode=US`, then `POST /v2/numbers {"phoneNumber": "+1..."}` buys it and turns SMS on in one call (paid plan). If turning SMS on fails, the number is still bought and the 201 carries `setupError`.
- **WhatsApp, Instagram, Messenger** connect through Meta from Accounts in the dashboard, never through the API. Once connected they appear in `GET /v2/channels`.
- Switch a connection on or off: `POST /v2/channels/{address}/{type}/activate` or `/deactivate`. Activating can charge a month (`chargedCents`, `monthlyCents`). `sms`, `sms_oneway`, `voice` have no switch.

## Send

```bash
curl -X POST https://api.zavu.dev/v2/messages \
  -H "Authorization: Bearer $ZAVUDEV_API_KEY" -H "Content-Type: application/json" \
  -d '{"from": "+13125551212", "to": "+56912345678", "text": "Your code is 123456."}'
```

The body is the v1 message body (`to`, `channel`, `messageType`, `text`, `content`, `subject`, `htmlBody`, `attachments`, `idempotencyKey`, `metadata`, `fallbackEnabled`, ...) plus `from` and `conversationId`. There is no `Zavu-Sender` header.

**Where it leaves from:** `conversationId` if given, else `from`, else the project default.

The `202` is `{ message, routing }`:

```json
{
  "message": { "id": "...", "to": "+56912345678", "from": "+13125551212", "channel": "whatsapp", "status": "queued", "direction": "outbound" },
  "routing": { "channel": "whatsapp", "from": "+13125551212", "reason": "auto_order", "estimatedCost": null }
}
```

`routing.reason` is one of: `sandbox` (test key), `recipient_type` (email `to`), `message_type` (WhatsApp-only type), `explicit` (`channel` given), `conversation` (sender from `conversationId`), `auto_order` (first **active** transport: WhatsApp, then SMS, then one-way SMS). The order ignores price and the WhatsApp 24-hour window. A WhatsApp-to-SMS fallback happens after the 202; read `attemptedChannels` on the message.

Rules that differ from v1:

- Every non-text, non-template `messageType` goes on WhatsApp. An explicit other `channel` is `400`.
- An email `to` with a non-email `channel`, or `channel: "email"` with a phone `to`, is `400`.
- `to` of 8–15 bare digits is a chat id. If nothing can reach a chat id, it is refused with `details.suggestion` (`+<digits>`). Always send phone numbers in E.164 with `+`.
- `ambiguous_from` (409): two channel groups share the `from` address. Add `channel`, or pass one of `details.candidates[].v1SenderId` as `from`.

## Reply in a conversation

```bash
curl -X POST https://api.zavu.dev/v2/messages \
  -H "Authorization: Bearer $ZAVUDEV_API_KEY" -H "Content-Type: application/json" \
  -d '{"conversationId": "js723987cyghwqxxaxcf590qd18axd95", "text": "Thanks, we are on it."}'
```

- `to` may be omitted: it is the thread's contact. A `to` that is not the thread's contact is `400`; nothing is sent.
- The reply leaves from the thread's `from`, the address it was last handled from.
- The WhatsApp 24-hour window still applies: outside it, send a template.

Conversations: `GET /v2/conversations` (`?from=`, `?channel=`, `?search=`), `GET /v2/conversations/{id}`, `GET /v2/conversations/{id}/messages`, `POST /v2/conversations/{id}/read`. Each conversation carries `from` instead of `senderId`. Use `direction` on messages to tell inbound from outbound.

## Webhook

One webhook per project, for every event:

```bash
curl -X PUT https://api.zavu.dev/v2/webhook \
  -H "Authorization: Bearer $ZAVUDEV_API_KEY" -H "Content-Type: application/json" \
  -d '{"url": "https://api.example.com/webhooks/zavu", "events": ["message.inbound", "message.failed", "conversation.new"]}'
```

- The `201` that creates it returns `webhook.secret` once. Store it. Rotate with `POST /v2/webhook/secret`. Remove with `DELETE /v2/webhook`.
- Payload: `{ id, event, timestamp, projectId, from, to, type, data }`. `data` is the v1 event data without `senderId`. On inbound events `from` is the contact and `to` your address.
- `message.inbound` carries `data.conversationId` (always set: the webhook is sent once the message is in its thread), `data.contactId`, and on WhatsApp `data.whatsapp.bsuid`. Key people on `contactId`, not `from`: for WhatsApp username contacts `from` is the phone only when WhatsApp includes it. `contact.identity_linked` fires when a BSUID is tied to a phone number; when that merges two contacts, its `mergedContactId` is the id that went away.
- Verify `X-Zavu-Signature: t=<seconds>,v2=<hex>`: `v2 = HMAC_SHA256(secret, "{t}.{rawBody}")`, hex. New v2 webhooks sign with `v2`. Use the raw body, not re-serialized JSON. See the `webhook-setup` skill for verification code.
- v1 sender webhooks keep receiving events too, as separate deliveries. Once the v2 receiver works, `PUT /v2/webhook` again with `"disableSenderWebhooks": true`. `GET /v2/webhook` lists the ones still on in `senderWebhooks`.

## Errors

Every error is flat:

```json
{ "code": "invalid_request", "message": "Invalid request body", "details": { "errors": [{ "path": ["to"], "message": "Required" }] } }
```

Branch on `code`, never on `message`.

| Code | Do this |
|------|---------|
| `invalid_request` (400) | Fix the fields in `details.errors[].path`. |
| `no_channel` (409) | Nothing can reach `to`. Offer `details.connect`: each item has `type`, `request` (the exact call, or `null` for a dashboard-only channel), `note`, `available`. `details.inactive` lists a connection that only needs activating. |
| `ambiguous_from` (409) | Disambiguate with `channel` or a `details.candidates[].v1SenderId`. |
| `conflict` (409) | Repeated `idempotencyKey`. Nothing was re-sent; the original is in `details.message`. |
| `kyb_required` (403) | One-way SMS is closed to this account until business verification is approved (only when the account is required to verify first; not the default). Use another channel or verify at `/kyb`. |
| `ten_dlc_required` (403) | SMS to +1 needs an approved 10DLC campaign. `details.connect` lists the v1 registration calls; `details.alternatives` lists WhatsApp, Telegram and email. |
| `channel_inactive` (403) | The named `channel` is switched off. Activate it or omit `channel`. |
| `not_found` (404) | Resource or address not in this project. |
| `method_not_allowed` (405) | Use a method from `details.allowed`. |
| `insufficient_balance` (402) | Add funds. |
| `rate_limited`, `daily_limit_exceeded` (429) | Back off; the daily ceiling resets at 00:00 UTC. |

```typescript
try {
  await zavu("POST", "/messages", { to: "+56912345678", text: "Hi" });
} catch (e: any) {
  if (e.code === "no_channel") {
    for (const opt of e.details.connect) {
      console.log(opt.type, opt.available, opt.request ?? "connect in the dashboard", opt.note);
    }
  } else throw e;
}
```

## Other v2 resources

| Path | Purpose |
|------|---------|
| `/v2/contacts` | CRUD. A duplicate identifier is `400 duplicate_identifier`; phone identifiers must be E.164. |
| `/v2/campaigns` | One message to many. Create, `POST /{id}/recipients`, `POST /{id}/send`, `/cancel`, `/progress`. A campaign id is a v1 `broadcastId`. Check `invalid` on the recipients 201: unreachable recipients are skipped, not refused. An email campaign needs a paid plan: on Free, send is `402 paid_plan_required`. |
| `/v2/templates` | WhatsApp templates. Submit with `POST /v2/templates/{id}/submit {"from": "<your WhatsApp number>"}`. |
| `/v2/agents` | AI agents. `provider: "zavu"` needs no key; `openai`, `anthropic`, `google` and `mistral` need `apiKey` on create (never returned). Connect one with `POST /v2/agents/{id}/channels {"address", "type"}`. `sms_oneway` is refused: it receives nothing. `POST /v2/agents/{id}/test` is billed like a live reply on the `zavu` provider. |
| `/v2/numbers` | Search and buy numbers. |
| `/v2/email-domains` | Add, verify, remove email domains. |
| `/v2/functions` | List, get and configure functions. Code and deploys stay on v1 (`npx zavudev deploy`, see the `functions` skill). |

## Still v1 only

Call these on v1 with the same key: voice calls (`/v1/calls`), 10DLC (`/v1/10dlc/*`), regulatory requirements, addresses and documents, owned number list/release, agent tools/flows/knowledge bases, memory, contact channel edits and merge, message reactions/typing/attachments, template sync, sub-accounts, balance, invitations.

## Gotchas

| Trap | Reality |
|------|---------|
| Calling v2 from the SDK's root (`zavu.messages`) | That is v1. Use `zavu.v2.messages`, on `@zavudev/sdk` 0.59.0+. Other SDKs: use HTTP. |
| Importing v2 types from `@zavudev/sdk` | The root exports v1's. Import from `@zavudev/sdk/v2`. |
| Sending a `Zavu-Sender` header | v2 ignores it. Use `from` or `conversationId`. |
| Treating `active: false` as disconnected | It is connected and switched off. `activate` turns it on. |
| Parsing the address `sms_oneway` as a number | It is a literal. One-way SMS leaves from shared numbers. |
| Expecting one delivery per event after adding the v2 webhook | v1 sender webhooks still fire until `disableSenderWebhooks: true`. |
| Reading `costTotal` as what an SMS cost you | Read `charged`. |
