---
name: ai-agent
description: Configure AI agents for automated conversational responses with flows, webhook tools, and knowledge bases.
---

# AI Agent

## When to Use

Use this skill when building AI-powered conversational agents that automatically respond to inbound messages. Covers agent setup, provider selection, flows, tools, and knowledge bases (RAG).

## Architecture

```
Inbound message -> Flow check (keyword/intent match?)
                     -> YES: Execute flow steps
                     -> NO: LLM call with system prompt + context + KB
                -> Agent generates response -> Send reply
```

## Create Agent

Each sender can have one agent:

```typescript
const result = await zavu.senders.agent.create({
  senderId: "snd_abc123",
  name: "Customer Support",
  provider: "openai",
  model: "gpt-4o-mini",
  systemPrompt: "You are a helpful customer support agent for Acme Corp. Be friendly, concise, and helpful. If you don't know the answer, say so.",
  apiKey: process.env.PROVIDER_API_KEY,
  contextWindowMessages: 10,
  includeContactMetadata: true,
  triggerOnChannels: ["sms", "whatsapp"],
  triggerOnMessageTypes: ["text"],
});
console.log(result.agent.id); // agent_xxx
```

**Python:**
```python
result = zavu.senders.agent.create(
    sender_id="snd_abc123",
    name="Customer Support",
    provider="openai",
    model="gpt-4o-mini",
    system_prompt="You are a helpful customer support agent...",
    api_key=os.environ["PROVIDER_API_KEY"],
)
```

**Go:**
```go
result, err := client.Senders.Agent.Create(context.TODO(), zavudev.AgentCreateParams{
    SenderID:     zavudev.String("snd_abc123"),
    Name:         zavudev.String("Customer Support"),
    Provider:     zavudev.String("openai"),
    Model:        zavudev.String("gpt-4o-mini"),
    SystemPrompt: zavudev.String("You are a helpful customer support agent..."),
    APIKey:       zavudev.String(os.Getenv("PROVIDER_API_KEY")),
})
```

**Ruby:**
```ruby
result = client.senders.agent.create(
    sender_id: "snd_abc123",
    name: "Customer Support",
    provider: "openai",
    model: "gpt-4o-mini",
    system_prompt: "You are a helpful customer support agent...",
    api_key: ENV["PROVIDER_API_KEY"],
)
```

**PHP:**
```php
$result = $client->senders->agent->create([
    'senderId' => 'snd_abc123',
    'name' => 'Customer Support',
    'provider' => 'openai',
    'model' => 'gpt-4o-mini',
    'systemPrompt' => 'You are a helpful customer support agent...',
    'apiKey' => getenv('PROVIDER_API_KEY'),
]);
```

## Provider & Model Selection

| Provider | Models | API Key Required |
|----------|--------|-----------------|
| `openai` | `gpt-4o`, `gpt-4o-mini`, `gpt-4-turbo` | Yes |
| `anthropic` | `claude-3-5-sonnet`, `claude-3-haiku` | Yes |
| `google` | `gemini-1.5-pro`, `gemini-1.5-flash` | Yes |
| `mistral` | `mistral-large`, `mistral-small` | Yes |
| `zavu` | Zavu-hosted models | No (included) |

## Update & Toggle Agent

```typescript
// Update configuration
await zavu.senders.agent.update({
  senderId: "snd_abc123",
  systemPrompt: "Updated prompt...",
  temperature: 0.7,
  maxTokens: 500,
});

// Enable/disable
await zavu.senders.agent.update({
  senderId: "snd_abc123",
  enabled: false,
});
```

## Conversational Flows

Flows handle structured conversations (keyword triggers, data collection):

```typescript
const result = await zavu.senders.agent.flows.create({
  senderId: "snd_abc123",
  name: "Lead Capture",
  description: "Capture lead information from interested prospects",
  trigger: {
    type: "keyword",
    keywords: ["info", "pricing", "demo"],
  },
  steps: [
    {
      id: "welcome",
      type: "message",
      config: { text: "Thanks for your interest! Let me get some info." },
      nextStepId: "ask_name",
    },
    {
      id: "ask_name",
      type: "collect",
      config: { variable: "name", prompt: "What's your name?" },
      nextStepId: "ask_email",
    },
    {
      id: "ask_email",
      type: "collect",
      config: { variable: "email", prompt: "What's your email?" },
      nextStepId: "confirm",
    },
    {
      id: "confirm",
      type: "message",
      config: { text: "Thanks {{name}}! We'll reach out at {{email}}." },
    },
  ],
  enabled: true,
  priority: 10,
});
```

### Trigger Types

| Type | Description |
|------|-------------|
| `keyword` | Matches specific keywords in message |
| `intent` | Matches detected intent |
| `always` | Runs on every message |
| `manual` | Only triggered via API |

### Step Types

| Type | Description |
|------|-------------|
| `message` | Send a message |
| `collect` | Collect user input into a variable |
| `condition` | Branch based on conditions |
| `tool` | Call a webhook tool |
| `llm` | Make an LLM call |
| `transfer` | Transfer to human agent |

### Flow Operations

```typescript
// List flows
const flows = await zavu.senders.agent.flows.list({ senderId: "snd_abc123" });

// Update flow
await zavu.senders.agent.flows.update({
  senderId: "snd_abc123",
  flowId: "flow_abc123",
  enabled: false,
});

// Duplicate flow
await zavu.senders.agent.flows.duplicate({
  senderId: "snd_abc123",
  flowId: "flow_abc123",
  newName: "Lead Capture (Copy)",
});

// Delete flow
await zavu.senders.agent.flows.delete({
  senderId: "snd_abc123",
  flowId: "flow_abc123",
});
```

## Webhook Tools

Tools let the agent call your backend during conversations:

```typescript
const result = await zavu.senders.agent.tools.create({
  senderId: "snd_abc123",
  name: "get_order_status",
  description: "Get the current status of a customer order",
  webhookUrl: "https://api.example.com/webhooks/order-status",
  webhookSecret: process.env.WEBHOOK_SECRET,
  parameters: {
    type: "object",
    properties: {
      order_id: { type: "string", description: "The order ID to look up" },
    },
    required: ["order_id"],
  },
});

// Test tool
await zavu.senders.agent.tools.test({
  senderId: "snd_abc123",
  toolId: "tool_abc123",
  testParams: { order_id: "ORD-12345" },
});
```

## Knowledge Bases (RAG)

Add documents for the agent to reference via retrieval-augmented generation:

```typescript
// Create knowledge base
const kb = await zavu.senders.agent.knowledgeBases.create({
  senderId: "snd_abc123",
  name: "Product FAQ",
  description: "Frequently asked questions about our products",
});

// Add document
await zavu.senders.agent.knowledgeBases.documents.create({
  senderId: "snd_abc123",
  kbId: kb.knowledgeBase.id,
  title: "Return Policy",
  content: "Our return policy allows returns within 30 days of purchase...",
});

// List documents
const docs = await zavu.senders.agent.knowledgeBases.documents.list({
  senderId: "snd_abc123",
  kbId: kb.knowledgeBase.id,
});
```

## Monitoring

```typescript
// Get agent stats
const stats = await zavu.senders.agent.stats({ senderId: "snd_abc123" });
console.log(`Invocations: ${stats.totalInvocations}`);
console.log(`Tokens: ${stats.totalTokensUsed}`);
console.log(`Cost: $${stats.totalCost}`);

// List executions
const executions = await zavu.senders.agent.executions.list({
  senderId: "snd_abc123",
  status: "error",
  limit: 20,
});
for (const exec of executions.items) {
  console.log(exec.id, exec.status, exec.errorMessage);
}
```

### Execution Statuses

| Status | Description |
|--------|-------------|
| `success` | Agent generated response successfully |
| `error` | Execution failed (LLM error, tool error, etc.) |
| `filtered` | Response blocked by safety filters |
| `rate_limited` | Provider rate limit exceeded |
| `balance_insufficient` | Account balance too low to process |

## Delete Agent

```typescript
await zavu.senders.agent.delete({ senderId: "snd_abc123" });
```

## Constraints

- One agent per sender
- System prompt: max 10,000 characters
- Context window: 1-50 messages
- Temperature: 0-2
- Max tokens: 1-4,096
- Tool name: max 100 characters
- Tool description: max 500 characters
- Document content: max 100,000 characters
- Knowledge base name: max 100 characters
- Provider `zavu` doesn't require an API key (uses Zavu-hosted models)
- All other providers require your own API key
