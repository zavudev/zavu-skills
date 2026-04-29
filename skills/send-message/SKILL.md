---
name: send-message
description: Send messages via SMS, WhatsApp, Email, Telegram, or Voice with channel selection logic.
---

# Send Message

## When to Use

Use this skill when building code to send messages through the Zavu API. Covers channel selection, message types, and error handling.

## Channel Selection Decision Tree

```
Is recipient an email address?
  -> YES: channel = "email" (requires KYC verification)
Is message type non-text (image, video, buttons, list, template, etc.)?
  -> YES: channel = "whatsapp" (auto-selected)
Need voice call / TTS?
  -> YES: channel = "voice"
Need guaranteed delivery to specific channel?
  -> YES: channel = "sms" | "whatsapp" | "telegram"
Want cost-optimized routing?
  -> YES: channel = "auto" (ML-powered smart routing)
Default?
  -> channel = "sms" (or omit for default)
```

## Basic Messages

### SMS (default)

**TypeScript:**
```typescript
const result = await zavu.messages.send({
  to: "+14155551234",
  text: "Your verification code is 123456",
});
console.log(result.message.id);
```

**Python:**
```python
result = zavu.messages.send(
    to="+14155551234",
    text="Your verification code is 123456",
)
print(result.message.id)
```

**Go:**
```go
result, err := client.Messages.Send(context.TODO(), zavudev.MessageSendParams{
    To:   zavudev.String("+14155551234"),
    Text: zavudev.String("Your verification code is 123456"),
})
fmt.Println(result.Message.ID)
```

**Ruby:**
```ruby
result = client.messages.send(to: "+14155551234", text: "Your verification code is 123456")
puts result.message.id
```

**PHP:**
```php
$result = $client->messages->send([
    'to' => '+14155551234',
    'text' => 'Your verification code is 123456',
]);
echo $result->message->id;
```

### WhatsApp Text

```typescript
const result = await zavu.messages.send({
  to: "+14155551234",
  channel: "whatsapp",
  text: "Hello from Zavu!",
});
```

### Email

```typescript
const result = await zavu.messages.send({
  to: "user@example.com",
  channel: "email",
  subject: "Your order has shipped",
  text: "Hi John, your order #12345 has shipped.",
  htmlBody: "<h1>Order Shipped</h1><p>Your order #12345 has shipped.</p>",
  replyTo: "support@example.com",
});
```

### Voice (Text-to-Speech)

```typescript
const result = await zavu.messages.send({
  to: "+14155551234",
  channel: "voice",
  text: "Your verification code is 1 2 3 4 5 6",
  voiceLanguage: "en-US", // optional, auto-detected from country code
});
```

## WhatsApp Rich Messages

### Image

```typescript
await zavu.messages.send({
  to: "+14155551234",
  messageType: "image",
  text: "Check out this product!", // caption
  content: { mediaUrl: "https://example.com/image.jpg" },
});
```

### Document

```typescript
await zavu.messages.send({
  to: "+14155551234",
  messageType: "document",
  content: {
    mediaUrl: "https://example.com/invoice.pdf",
    filename: "invoice.pdf",
  },
});
```

### Video / Audio

```typescript
// Video
await zavu.messages.send({
  to: "+14155551234",
  messageType: "video",
  text: "Watch this!",
  content: { mediaUrl: "https://example.com/video.mp4" },
});

// Audio
await zavu.messages.send({
  to: "+14155551234",
  messageType: "audio",
  content: { mediaUrl: "https://example.com/audio.mp3" },
});
```

### Sticker

```typescript
await zavu.messages.send({
  to: "+14155551234",
  messageType: "sticker",
  content: { mediaUrl: "https://example.com/sticker.webp" },
});
```

### Location

```typescript
await zavu.messages.send({
  to: "+14155551234",
  messageType: "location",
  content: {
    latitude: 37.7749,
    longitude: -122.4194,
    locationName: "San Francisco",
    locationAddress: "123 Main St, San Francisco, CA",
  },
});
```

### Contact Card

```typescript
await zavu.messages.send({
  to: "+14155551234",
  messageType: "contact",
  content: {
    contacts: [
      { name: "John Doe", phones: ["+14155551234", "+14155555678"] },
    ],
  },
});
```

### Interactive Buttons (max 3)

```typescript
await zavu.messages.send({
  to: "+14155551234",
  messageType: "buttons",
  text: "How would you rate your experience?",
  content: {
    buttons: [
      { id: "great", title: "Great!" },
      { id: "okay", title: "It was okay" },
      { id: "poor", title: "Not good" },
    ],
  },
});
```

### List Message

```typescript
await zavu.messages.send({
  to: "+14155551234",
  messageType: "list",
  text: "Select an option:",
  content: {
    listButton: "View Options",
    sections: [{
      title: "Products",
      rows: [
        { id: "prod_1", title: "Product A", description: "$10.00" },
        { id: "prod_2", title: "Product B", description: "$20.00" },
      ],
    }],
  },
});
```

### Template Message

```typescript
await zavu.messages.send({
  to: "+14155551234",
  messageType: "template",
  content: {
    templateId: "tpl_abc123",
    templateVariables: { "1": "John", "2": "ORD-12345" },
  },
});
```

### CTA URL (Call-to-Action button)

```typescript
await zavu.messages.send({
  to: "+14155551234",
  messageType: "cta_url",
  text: "Check out our latest collection",
  content: {
    ctaDisplayText: "View Products",          // max 20 chars
    ctaUrl: "https://example.com/products",
    ctaHeaderType: "image",                    // optional: text | image | video | document
    ctaHeaderMediaUrl: "https://example.com/header.jpg",
    footerText: "Limited time offer",          // optional, max 60 chars
  },
});
```

### Reaction

```typescript
await zavu.messages.react({
  messageId: "msg_abc123",
  emoji: "\ud83d\udc4d",
});
```

## Sender Override

```typescript
await zavu.messages.send({
  to: "+14155551234",
  text: "Hello!",
  'Zavu-Sender': "snd_abc123",
});
```

## Idempotency

```typescript
await zavu.messages.send({
  to: "+14155551234",
  text: "Payment confirmed",
  idempotencyKey: "payment_confirm_order_123",
});
```

## Disable Automatic Fallback

By default, WhatsApp messages auto-fallback to SMS on failure. To disable:

```typescript
await zavu.messages.send({
  to: "+14155551234",
  channel: "whatsapp",
  text: "WhatsApp only — no SMS fallback",
  fallbackEnabled: false,
});
```

## Get Status & List Messages

```typescript
// Get single message
const msg = await zavu.messages.get({ messageId: "msg_abc123" });
console.log(msg.message.status); // queued | sending | sent | delivered | read | failed

// List with filters + pagination
let cursor: string | undefined;
do {
  const result = await zavu.messages.list({ status: "delivered", limit: 50, cursor });
  for (const message of result.items) {
    console.log(message.id, message.status);
  }
  cursor = result.nextCursor ?? undefined;
} while (cursor);
```

## Common Errors

| Error Code | Meaning | Fix |
|------------|---------|-----|
| `whatsapp_window_closed` | 24h window not open | Use template message instead |
| `url_not_verified` | Message has unverified URLs | Submit URLs via `/v1/urls` first |
| `url_shortener_blocked` | URL shortener detected | Use full destination URL |
| `email_kyc_required` | Email needs KYC | Complete verification in dashboard |
| `urls_blocked_unverified` | Unverified account + URLs | Complete KYC verification |

## Constraints

- Button titles: max 20 chars, max 3 buttons
- List row titles: max 24 chars, descriptions: max 72 chars, max 10 rows per section
- Email subject: max 998 chars
- Voice language codes: `en-US`, `es-ES`, `pt-BR`, etc. (auto-detected if omitted)
- Media messages auto-select WhatsApp channel
