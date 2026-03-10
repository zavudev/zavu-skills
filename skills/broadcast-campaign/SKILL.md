---
name: broadcast-campaign
description: Create and manage broadcast campaigns for bulk messaging across SMS, WhatsApp, Email, and Telegram.
---

# Broadcast Campaign

## When to Use

Use this skill when building code to send messages to multiple recipients in a campaign. Covers the full broadcast lifecycle from creation to monitoring.

## Broadcast Lifecycle

```
draft -> pending_review -> approved -> sending -> completed
                        -> rejected -> (edit) -> pending_review (retry, max 3)
                        -> rejected -> escalated (manual review by Zavu team)
                        -> rejected_final
         approved -> scheduled -> sending -> completed
         sending -> paused
         (any active) -> cancelled
```

## Step-by-Step Workflow

### 1. Create Broadcast

```typescript
const result = await zavu.broadcasts.create({
  name: "Black Friday Sale",
  channel: "sms", // sms | whatsapp | email | telegram | smart
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

### Channel Options

| Channel | Description |
|---------|-------------|
| `sms` | SMS to all contacts |
| `whatsapp` | WhatsApp (requires template for non-window contacts) |
| `email` | Email (requires KYC, needs `emailSubject`) |
| `telegram` | Telegram |
| `smart` | Per-contact intelligent routing |

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

### 3. Send (triggers AI content review)

```typescript
// Send immediately
await zavu.broadcasts.send({ broadcastId });

// Or schedule
await zavu.broadcasts.send({
  broadcastId,
  scheduledAt: "2024-01-15T10:00:00Z",
});
```

### 4. Monitor Progress

```typescript
const progress = await zavu.broadcasts.progress({ broadcastId });
console.log(`${progress.percentComplete}% complete`);
console.log(`Delivered: ${progress.delivered}, Failed: ${progress.failed}`);
console.log(`Estimated completion: ${progress.estimatedCompletionAt}`);
```

### 5. Handle Rejection (if content review fails)

```typescript
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
- Content goes through AI review before sending
- Balance is reserved (estimated cost) when sending
- Max 3 review retry attempts, then escalate
- Can only update/delete broadcasts in `draft` status
- Cancelling doesn't stop already-queued messages
