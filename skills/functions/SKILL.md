---
name: functions
description: Build serverless TypeScript functions on Zavu Cloud — declare agents + tools in code with defineAgent / defineTool, deploy with `zavu deploy`, debug with `zavu agents executions`. Use this skill whenever the user wants code-driven AI agents, custom tool handlers, or event-driven business logic.
---

# Zavu Functions

Zavu Functions = serverless TypeScript on Zavu Cloud + a declarative framework for AI agents.

```ts
import { defineAgent, defineTool } from "@zavu/functions"

defineAgent({
  senderId: process.env.SENDER_ID!,
  name: "Bella",
  provider: "zavu",
  model: "openai/gpt-4o-mini",
  prompt: "You are Bella, host at the restaurant. Be brief.",
})

defineTool({
  name: "check_availability",
  description: "Get free reservation slots for a date.",
  parameters: {
    type: "object",
    properties: { date: { type: "string" }, partySize: { type: "number" } },
    required: ["date", "partySize"],
  },
  handler: async ({ date, partySize }) => {
    return { available: true, slots: ["19:00", "21:00"] }
  },
})
```

That's a full agent + tool. `zavu deploy` reconciles the live state.

## When to use Functions vs the imperative AI Agent API

| Use case | Use |
|---|---|
| Customer wants a code-first agent with custom tool handlers in their own language | **Functions** |
| Tools need to query the user's database, call internal APIs, or transform data before returning | **Functions** |
| User wants reproducible config from a git repo (one source of truth) | **Functions** |
| User wants no-code config via the dashboard | imperative `senders.agent.create` API (see `ai-agent` skill) |
| User needs event-driven handlers (`message.inbound`, `broadcast.completed`) without dashboard wiring | **Functions** |

If the user mentions writing code, `defineAgent`, `defineTool`, `zavu deploy`, or "serverless" — use this skill. Otherwise route to `ai-agent`.

## CLI as primary interface

Functions are managed entirely via the `zavu` CLI, not API calls. Install once:

```sh
brew install zavudev/tools/zavu
# or grab a standalone binary from https://github.com/zavudev/zavu-cli/releases

zavu login
```

`zavu login` opens the browser and stores credentials in `~/.zavu/credentials.json`.

## Full lifecycle

### 1. Scaffold

```sh
zavu fn init order-bot --template blank
cd order-bot
```

Templates available: `blank`, `restaurant-booking`, `school-parent-notify`, `ecommerce-order-bot`.

The init writes `index.ts`, `package.json`, and a `.zavu/config.json` that links this directory to a Function record in the user's project. Once linked, every subsequent command auto-resolves the function.

### 2. Set secrets

Secrets are encrypted env vars injected into the function at deploy time.

```sh
zavu fn secrets set SENDER_ID jx7abc123def456
zavu fn secrets set DATABASE_URL "postgres://..."
zavu fn secrets list
zavu fn secrets unset OLD_KEY
```

Get the sender ID from `zavu senders list`.

### 3. Author the agent + tools

Edit `index.ts`:

```ts
import { defineAgent, defineTool, defineFunction } from "@zavu/functions"

defineAgent({
  senderId: process.env.SENDER_ID!,
  name: "Bella",
  provider: "zavu",              // Zavu's AI gateway (charged from project balance)
                                  // Or "openai" / "anthropic" / "google" / "mistral" with BYOK + apiKey
  model: "openai/gpt-4o-mini",   // For "zavu" provider, prefix with the underlying provider
  prompt: "You are Bella…",       // System prompt
  channels: ["whatsapp"],         // Optional: default ["*"] = all channels the sender supports
  // apiKey: process.env.OPENAI_API_KEY  // only for non-zavu providers
})

defineTool({
  name: "lookup_order",
  description: "Get current status of an order. Use when the customer asks about an order they placed.",
  parameters: {
    type: "object",
    properties: { orderId: { type: "string" } },
    required: ["orderId"],
  },
  handler: async (args, ctx) => {
    // ctx: { projectId, functionId, slug, awsRequestId, messageId?, contactPhone?, sessionId?, log }
    const res = await fetch(`https://pos.example.com/orders/${args.orderId}`, {
      headers: { Authorization: `Bearer ${process.env.POS_API_KEY}` },
    })
    return await res.json()
  },
})

// Optional: handle raw events (message.inbound from triggers, HTTP calls).
// NOT needed if you only declare agent + tools.
export default defineFunction(async (event, ctx) => {
  ctx.log("got event", event.type)
})
```

### 4. Deploy

```sh
zavu deploy
```

Output:

```
✓ Deployed in 6.4s
  Agents:
    + Bella           (sender_abc, whatsapp)
  Tools:
    + lookup_order    → Bella
```

The reconcile is idempotent — re-running with no changes shows `0 created, 0 updated, 0 deleted`.

### 5. Test

**Local invocation (skip cloud round-trip):**

```sh
# Call a tool handler with synthetic args
zavu fn invoke --tool lookup_order --args '{"orderId":"ORD-001"}'

# Simulate an inbound event for defineFunction
zavu fn invoke --event message.inbound --data '{"from":"+14155551234","text":"hi"}'
```

**End-to-end:** send a real message to the sender's WhatsApp/SMS/Telegram number. The agent runs the LLM, calls tools, and replies via the same channel.

### 6. Debug

When something fails, walk the chain top-down:

```sh
# 1. Did the inbound reach the agent?
zavu agents executions list --sender <senderId>

# 2. Detail of any failed run
zavu agents executions get <executionId> --sender <senderId>

# 3. Live tool handler logs (your console.log calls)
zavu fn logs --tail
```

The `--json` flag on `executions list` returns the full payload including `errorMessage` for parseable diagnostics.

## defineAgent reference

```ts
defineAgent({
  senderId: string,              // Required. The sender that receives inbound + dispatches the agent.
  name: string,                  // Required. Displayed in dashboard.
  provider: "zavu" | "openai" | "anthropic" | "google" | "mistral",
  model: string,                 // For "zavu": prefix with underlying provider e.g. "openai/gpt-4o-mini"
  prompt: string,                // System prompt.
  apiKey?: string,               // Required for non-"zavu" providers.
  channels?: string[],           // Default ["*"]. Subset of: sms, whatsapp, telegram, email, instagram, voice
  messageTypes?: string[],       // Default ["text"]. Filter by message type.
  temperature?: number,          // 0-2.
  maxTokens?: number,            // Cap on output tokens.
  contextWindowMessages?: number,// Past N messages included as context. Default 10.
  sessionTimeoutMinutes?: number,// Reset conversation context after N minutes. Default 60.
  includeContactMetadata?: boolean, // Inject contact's metadata into the system prompt. Default true.
  enabled?: boolean,             // Default true.
})
```

## defineTool reference

```ts
defineTool({
  name: string,                  // Required. snake_case, max 64 chars.
  description: string,           // Required. The LLM reads this to decide WHEN to call the tool.
  parameters: {
    type: "object",
    properties: { /* JSON Schema */ },
    required?: string[],
  },
  handler: async (args, ctx) => any,  // Required. Return any JSON-serializable value.
  agent?: string,                // Optional: which agent owns this tool. Defaults to the only agent in the file.
  enabled?: boolean,             // Default true.
})
```

### Handler `ctx` shape

```ts
{
  projectId: string,
  functionId: string,
  slug: string,
  awsRequestId: string,
  messageId?: string,            // ID of the triggering inbound (when called by agent)
  contactPhone?: string,
  sessionId?: string,            // Active flow session if any
  log: (...args) => void,        // console.log proxy that appears in `zavu fn logs --tail`
}
```

## defineFunction reference (optional)

Use only if you want to handle:
- **Raw HTTP requests** (function exposed at a public URL — set `httpEnabled: true` via dashboard)
- **Native event triggers** (`message.inbound`, `broadcast.completed`, etc — configured via `zavu fn triggers add`)

```ts
export default defineFunction(async (event, ctx) => {
  if (event.type === "message.inbound") {
    // event.data: { from, text, channel, messageId, ... }
  }
  return { ok: true }
})
```

## Triggers (event subscriptions)

To make `defineFunction` react to Zavu events:

```sh
zavu fn triggers list
zavu fn triggers add --events message.inbound --senders <senderId>
zavu fn triggers add --events broadcast.completed --senders any
zavu fn triggers toggle <triggerId>
zavu fn triggers rm <triggerId>
zavu fn triggers events       # list available event types
```

Triggers use signed internal invocations (no HMAC verification needed inside the handler).

## Versions + rollback

Every `zavu deploy` creates an immutable version.

```sh
zavu fn versions list           # alias: zavu fn history
zavu fn rollback 4              # go back to version 4
```

The function metadata in the dashboard tracks the active version + lets you rollback from the UI too.

## Runtime versions

Each function pins to a specific runtime layer at first deploy. Subsequent deploys keep the same pin (immutable for stability).

```sh
zavu deploy --update-runtime   # opt-in upgrade to latest runtime
```

Only opt in when there's a security advisory or feature you want — `zavu deploy` without the flag is safe forever.

## Pricing model

Functions are billed by **invocation units**, memory-weighted:

| Memory | Units per call |
|---|---|
| 128 MB | 1 |
| 256 MB | 2 |
| 512 MB | 4 |
| 1024 MB | 8 |

Each plan includes a monthly quota; overage rolls into the next Stripe invoice via metered billing.

| Plan | Included units | Overage rate |
|---|---|---|
| Free | 100k | Hard cap (invocations blocked) |
| Hobby | 1M | $5 / 1M |
| Standard | 5M | $4 / 1M |
| Growth | 10M | $3 / 1M |

Set memory at function creation or via dashboard. Lower memory = cheaper. Most tool handlers fit in 128 MB.

## Common patterns

### Take over a manual agent

If the user already created an agent via the dashboard or `zavu agents create`, declaring it in code with the same `senderId + name` will TAKE OVER that agent — Zavu marks it `managedByFunctionId` and the dashboard locks manual edits. The function source becomes source-of-truth.

To go back to manual control: delete the function (`zavu fn delete`) and the agent is freed.

### Per-environment senders

```ts
defineAgent({
  senderId: process.env.NODE_ENV === "production"
    ? process.env.PROD_SENDER_ID!
    : process.env.DEV_SENDER_ID!,
  // ...
})
```

Then `zavu fn secrets set NODE_ENV production` on prod, `... development` on dev. Same code, different agents.

### BYOK (Bring Your Own Key)

For OpenAI / Anthropic / Google / Mistral, pass `apiKey` directly:

```ts
defineAgent({
  senderId: process.env.SENDER_ID!,
  provider: "openai",
  model: "gpt-4o-mini",
  apiKey: process.env.OPENAI_API_KEY,
  prompt: "...",
})
```

`zavu fn secrets set OPENAI_API_KEY sk-...`. The agent uses the key directly — no Zavu balance consumed for LLM calls.

### Pinning a tool to a specific agent (multi-agent functions)

When a function declares more than one `defineAgent`, tools default-attach to the first one. To pick explicitly:

```ts
defineTool({
  name: "lookup_order",
  agent: "Bella",   // Match by agent's name field
  // ...
})
```

### Calling Zavu APIs from inside a handler

Each function gets a scoped `ZAVU_API_KEY` injected automatically — use the SDK to call back:

```ts
import { Zavudev } from "@zavudev/sdk"

const zavu = new Zavudev({ apiKey: process.env.ZAVU_API_KEY })

defineTool({
  name: "send_followup",
  handler: async (args, ctx) => {
    await zavu.messages.send({
      to: ctx.contactPhone!,
      text: "Thanks for your order!",
    })
    return { sent: true }
  },
})
```

The auto-provisioned key has `messages:send`, `messages:read`, `contacts:read` scopes. For broader access create a manual API key and inject as a secret.

## Dashboard

`https://dashboard.zavu.dev/functions/<id>` shows tabs:

- **Code** — current draft source (editable in browser)
- **Triggers** — event subscriptions
- **Agents & Tools** — what this function manages, with deep-links to executions
- **Dependencies** — npm packages used by the bundle
- **Secrets** — encrypted env vars (values are write-only)
- **Versions** — deploy history + rollback
- **Logs** — runtime stdout/stderr of recent invocations
- **Settings** — memory, timeout, httpEnabled, delete

## Reference docs

- Overview: https://docs.zavu.dev/concepts/functions
- Quickstart: https://docs.zavu.dev/guides/functions/quickstart
- CLI reference: https://docs.zavu.dev/guides/functions/cli
- defineAgent: https://docs.zavu.dev/guides/functions/defining-agents
- defineTool: https://docs.zavu.dev/guides/functions/defining-tools
- Debugging guide: https://docs.zavu.dev/guides/functions/debugging
- Examples: https://docs.zavu.dev/guides/functions/examples/restaurant

## Constraints

- Slug: lowercase alphanumeric + hyphens, max 23 chars (Lambda name limit).
- Source bundle: ≤ 900 KB compressed.
- Total env size: 4 KB across all secrets.
- Secret key format: `[A-Z_][A-Z0-9_]*`. Reserved prefixes: `AWS_`, `LAMBDA_`, `_HANDLER`, `_X_AMZN`.
- Timeout: ≤ 30 s (configurable, default 10 s).
- Memory: 128 / 256 / 512 / 1024 MB.
- Tools per agent: 16.
- Agents per function: no hard cap, but typically 1.

## Anti-patterns

- **Don't use the imperative `senders.agent.create` API in parallel with a managed function**. If the function declares an agent, the function owns it — manual edits get blocked.
- **Don't hardcode the `senderId`** in the source. Always read from `process.env.SENDER_ID` so the same code works across envs.
- **Don't import heavyweight deps you only use in one tool**. Each call cold-starts; trim dependencies to keep latency down.
- **Don't `console.log` secrets**. Function logs are visible to anyone with project access.
