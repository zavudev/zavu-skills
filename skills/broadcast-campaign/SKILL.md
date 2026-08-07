---
name: broadcast-campaign
description: Create and manage broadcast campaigns for bulk messaging across SMS, WhatsApp, Email, and Telegram.
---

# Broadcast Campaign

## When to Use

Use this skill when building code to send messages to multiple recipients in a campaign — including bulk email. Covers the full broadcast lifecycle from creation to monitoring.

## Broadcast Lifecycle

```
draft -> pending_review -> pending_admin_review -> approved -> sending -> completed
                        -> rejected -> (edit) -> pending_review (retry, max 3)
                        -> rejected -> escalated (manual review by Zavu team)
                        -> rejected_final
         approved -> scheduled -> sending -> completed
         sending -> paused (can resume)
         (any active) -> cancelled
         (any) -> failed (permanent failure)
```

## Step-by-Step Workflow

### 1. Create Broadcast

```typescript
const result = await zavu.broadcasts.create({
  name: "Black Friday Sale",
  channel: "sms", // smart | sms | sms_oneway | whatsapp | telegram | email
  text: "Hi {{name}}, check out our Black Friday deals! Code: FRIDAY20",
});
const broadcastId = result.broadcast.id; // brd_xxx
```

**Python:**
```python
result = zavu.broadcasts.create(
    name="Black Friday Sale",
    channel="sms",
    text="Hi {{name}}, check out our Black Friday deals! Code: FRIDAY20",
)
broadcast_id = result.broadcast.id
```

**Go:**
```go
result, err := client.Broadcasts.Create(context.TODO(), zavudev.BroadcastCreateParams{
    Name:    zavudev.String("Black Friday Sale"),
    Channel: zavudev.String("sms"),
    Text:    zavudev.String("Hi {{name}}, check out our Black Friday deals! Code: FRIDAY20"),
})
broadcastID := result.Broadcast.ID
```

**Ruby:**
```ruby
result = client.broadcasts.create(
    name: "Black Friday Sale",
    channel: "sms",
    text: "Hi {{name}}, check out our Black Friday deals! Code: FRIDAY20",
)
broadcast_id = result.broadcast.id
```

**PHP:**
```php
$result = $client->broadcasts->create([
    'name' => 'Black Friday Sale',
    'channel' => 'sms',
    'text' => 'Hi {{name}}, check out our Black Friday deals! Code: FRIDAY20',
]);
$broadcastId = $result->broadcast->id;
```

### Channel Options

| Channel | Description |
|---------|-------------|
| `smart` | Per-contact intelligent routing |
| `sms` | SMS to all contacts |
| `sms_oneway` | One-way SMS (no replies) |
| `whatsapp` | WhatsApp (requires template for non-window contacts) |
| `telegram` | Telegram |
| `email` | Email (needs `emailSubject`) — **recommended path for bulk email** |

### Message Types

| Type | Description |
|------|-------------|
| `text` | Plain text (default) |
| `image` | Image with optional caption |
| `video` | Video message |
| `audio` | Audio message |
| `document` | Document file |
| `template` | WhatsApp pre-approved template |

### Email Broadcast (with HTML body)

For bulk email campaigns, use `channel: "email"` with an HTML body. This is the recommended way to send mass email through Zavu.

```typescript
const result = await zavu.broadcasts.create({
  name: "Newsletter",
  channel: "email",
  emailSubject: "Special offer for {{name}}",
  text: "Hi {{name}}, check out our latest sale!",   // plain text fallback
  emailHtmlBody: "<h1>Hi {{name}}!</h1><p>Check out our latest sale.</p>",
  metadata: { campaign_id: "camp_123", region: "US" },
});
```

### 2. Add Contacts (batch, max 1000/request)

```typescript
const result = await zavu.broadcasts.contacts.add({
  broadcastId: broadcastId,
  contacts: [
    { recipient: "+14155551234", templateVariables: { name: "John" } },
    { recipient: "+14155555678", templateVariables: { name: "Jane" } },
  ],
});
console.log(result.added, result.duplicates, result.invalid);
```

### 3. Send (triggers content review)

Sending requires **two** verifications, and passing one is not enough:

| Check | What it is | If missing |
| --- | --- | --- |
| KYC | Identity of the person behind the account | `403 kyc_required` |
| KYB | The business itself, reviewed by a person | `403 kyb_required` |

Creating and editing drafts needs neither — everything above works unverified,
and only this call is blocked.

```typescript
// Send immediately
await zavu.broadcasts.send({ broadcastId });

// Or schedule
await zavu.broadcasts.send({
  broadcastId,
  scheduledAt: "2024-01-15T10:00:00Z",
});
```

**A draft never goes straight out — what happens next depends on the channel:**

| Broadcast | What this call does |
| --- | --- |
| WhatsApp on a Meta-approved template | Skips review (Meta already vetted it) and starts sending |
| Email | Automated review; sends as soon as it passes |
| SMS, Telegram, smart, everything else | Automated review, then `pending_admin_review` — a person approves before it sends |

So a `202` means *accepted*, not *sending*. Poll the status rather than
assuming; only WhatsApp-template broadcasts move to `sending` immediately.

Calling send on a broadcast that is already `approved` or `scheduled` sends or
reschedules it directly, since it has already been through review.

### 4. Monitor Progress

```typescript
const progress = await zavu.broadcasts.progress({ broadcastId });
console.log(`${progress.percentComplete}% complete`);
console.log(`Delivered: ${progress.delivered}, Failed: ${progress.failed}, Skipped: ${progress.skipped}`);
console.log(`Estimated completion: ${progress.estimatedCompletionAt}`);
```

Per-contact statuses: `pending`, `queued`, `sending`, `delivered`, `failed`, `skipped` (excluded — opted out, duplicate, or invalid).

### 5. Handle Rejection (if content review fails)

```typescript
// Check remaining review attempts
const broadcast = await zavu.broadcasts.get({ broadcastId });
console.log(`Review attempts: ${broadcast.reviewAttempts}/3`);

// Edit content
await zavu.broadcasts.update({
  broadcastId,
  text: "Updated message content with {{name}}",
});

// Retry review (max 3 attempts)
await zavu.broadcasts.retryReview({ broadcastId });

// Or escalate to manual review
await zavu.broadcasts.escalate({ broadcastId });
```

## Other Operations

```typescript
// Reschedule
await zavu.broadcasts.reschedule({
  broadcastId, scheduledAt: "2024-01-16T14:00:00Z",
});

// Cancel (pending contacts skipped, queued may still deliver)
await zavu.broadcasts.cancel({ broadcastId });

// List contacts in broadcast
const contacts = await zavu.broadcasts.contacts.list({
  broadcastId, status: "delivered", limit: 100,
});

// Delete (draft only)
await zavu.broadcasts.delete({ broadcastId });
```

## Template Variables

Use `{{variable_name}}` in broadcast text. Override per-contact via `templateVariables`:

```typescript
await zavu.broadcasts.contacts.add({
  broadcastId,
  contacts: [
    { recipient: "+14155551234", templateVariables: { name: "John", order_id: "ORD-001" } },
  ],
});
```

## Constraints

- Max 1000 contacts per `add` request (batch for larger lists)
- Sending requires BOTH KYC and KYB (`403 kyc_required` / `kyb_required`); drafting requires neither
- Content goes through review before sending, except WhatsApp on a Meta-approved template
- Most channels also wait on a human (`pending_admin_review`) after the automated pass
- Balance is reserved (estimated cost) when sending
- Max 3 review retry attempts, then escalate
- Can only update/delete broadcasts in `draft` status
- Cancelling doesn't stop already-queued messages
