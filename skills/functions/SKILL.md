---
name: functions
description: Build serverless TypeScript functions on Zavu Cloud — declare agents + tools in code with defineAgent / defineTool, deploy with `npx zavudev deploy`, debug with `npx zavudev agents executions`. Use this skill whenever the user wants code-driven AI agents, custom tool handlers, or event-driven business logic.
---

# Zavu Functions

Zavu Functions = serverless TypeScript on Zavu Cloud + a declarative framework for AI agents.

```ts
import { defineAgent, defineTool } from "@zavudev/functions"

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

That's a full agent + tool. `npx zavudev deploy` reconciles the live state.

## When to use Functions vs the imperative AI Agent API

| Use case | Use |
|---|---|
| Customer wants a code-first agent with custom tool handlers in their own language | **Functions** |
| Tools need to query the user's database, call internal APIs, or transform data before returning | **Functions** |
| User wants reproducible config from a git repo (one source of truth) | **Functions** |
| User wants no-code config via the dashboard | imperative `senders.agent.create` API (see `ai-agent` skill) |
| User needs event-driven handlers (`message.inbound`, `broadcast.status_changed`) without dashboard wiring | **Functions** |

If the user mentions writing code, `defineAgent`, `defineTool`, `npx zavudev deploy`, or "serverless" — use this skill. Otherwise route to `ai-agent`.

## CLI as primary interface

Functions are managed entirely via the `zavu` CLI, not API calls. Install once:

```sh
brew install zavudev/tools/zavu
# or grab a standalone binary from https://github.com/zavudev/zavu-cli/releases

npx zavudev login
```

`npx zavudev login` opens the browser and stores credentials in `~/.zavu/credentials.json`.

## Factory agents

The fastest way to a working voice or text agent: pull a ready-made one into your
codebase, then deploy. Each factory agent is a Zavu Function whose `index.ts`
declares the agent with `defineAgent` and its skills with `defineTool` — so what
you pull is real, editable code you own.

```sh
npx zavudev agents catalog                            # list the factory agents
npx zavudev agents pull fermi --sender <senderId>     # scaffold ./fermi, register it, set SENDER_ID
cd fermi
npx zavudev deploy
```

`npx zavudev agents catalog` lists each agent's id, name, whether it's `voice`, tool
count, and category. `npx zavudev agents pull <id>` scaffolds `./<id>` (override with
`--dir <path>`); `--sender <senderId>` sets the `SENDER_ID` secret so `npx zavudev deploy`
works immediately — omit it and run `npx zavudev fn secrets set SENDER_ID <senderId>`
first. Voice agents come with the `voice` block pre-filled; edit `index.ts` and
redeploy to iterate.

`npx zavudev agents init` runs the whole setup as one guided command: it creates the
sender (buying a phone number if you want), pulls a factory agent, and sets
`SENDER_ID`.

The sender's channels are what the agent answers on, and each connects from the
CLI. Voice and SMS work as soon as the sender has a number; the rich channels
connect with one command each:

```sh
npx zavudev telegram connect --sender <senderId> --token <botFatherToken>
npx zavudev email-domains add example.com                       # then publish DNS + `verify`
```

## Full lifecycle

### 1. Scaffold

```sh
npx zavudev fn init --name order-bot --template blank
cd order-bot
```

Templates available: `blank`, `restaurant-booking`. Run `zavu fn init --help` for the
current list — it is the authority, not this page.

**Prefer a factory agent when one fits.** `zavu agents catalog` lists production
agents (support, lead capture, booking) and `zavu agents pull <id>` scaffolds one
complete with its prompt, its skills and, for voice agents, its voice config.

The init writes `index.ts`, `package.json`, and a `.zavu/config.json` that links this directory to a Function record in the user's project. Once linked, every subsequent command auto-resolves the function.

### 2. Set secrets

Secrets are encrypted env vars injected into the function at deploy time.

```sh
npx zavudev fn secrets set SENDER_ID jx7abc123def456
npx zavudev fn secrets set DATABASE_URL "postgres://..."
npx zavudev fn secrets list
npx zavudev fn secrets unset OLD_KEY
```

Get the sender ID from `npx zavudev senders list`.

### 3. Author the agent + tools

Edit `index.ts`:

```ts
import { defineAgent, defineTool, defineFunction } from "@zavudev/functions"

defineAgent({
  senderId: process.env.SENDER_ID!,
  name: "Bella",
  provider: "zavu",              // Zavu's AI gateway (charged from project balance)
                                  // Or "openai" / "anthropic" / "google" / "mistral" with BYOK + apiKey
  model: "openai/gpt-4o-mini",   // For "zavu" provider, prefix with the underlying provider
  prompt: "You are Bella…",       // System prompt
  channels: ["whatsapp"],         // Optional: default ["*"] = all channels the sender supports
  // apiKey: process.env.OPENAI_API_KEY  // only for BYOK providers (openai / anthropic / google / mistral)
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
// The object form is what binds events — `on` is the subscription.
export default defineFunction({
  on: ["message.inbound"],
  handler: async (event, ctx) => {
    ctx.log("got event", event.type)
  },
})
```

### 4. Deploy

```sh
npx zavudev deploy
```

Output:

```
✓ Deployed in 6.4s
  Agents synced:
    + Bella
  Tools synced:
    + lookup_order
```

| Marker | Meaning |
|---|---|
| `+ name` | Created |
| `~ name` | Existed and was written |
| `= name (unchanged)` | Existed and nothing differed — **only on newer backends; you may never see this** |

**Do not use this output to confirm an edit landed.** `~` is printed for every
agent and tool that already existed, whether or not anything about it actually
changed, so a redeploy with no edit at all is byte-identical to one that rewrote
the prompt. `=` is emitted only by newer backends; if you never see it, yours
does not distinguish the two cases.

To actually verify a change reached production, ask the agent:

```sh
npx zavudev agents test --agent <agentId> --message "<something your edit changes>" --json
```

Put a distinctive word in the prompt you edited and check it comes back. That is
one command, it charges nothing, and unlike the deploy summary it answers the
question you actually have.

**Read the lines above the ✓.** Warnings print before the success line and cover
the cases where a green deploy did not do what it looks like: a manifest probe
that threw (nothing was synced, and the command exits non-zero), tools attached
to an agent whose channels will never call them, or a second agent landing on a
sender that already has one — where only the first will ever answer.

### Local development

The scaffold is a real TypeScript project: `zavu init` and `zavu agents pull`
write a `tsconfig.json` and a `package.json` that declares `@zavudev/functions`,
`typescript` and `@types/node` as devDependencies.

### Exposing a function over HTTP

A function that needs its own endpoint — a webhook you host, a tool URL you
control — is created with `--http`:

```sh
npx zavudev fn init --template blank --http
npx zavudev agents pull kepler --sender "$SENDER_ID" --http
```

The URL exists once something is deployed behind it. `deploy` prints it, and you
can ask for it any time:

```sh
npx zavudev fn info                 # status, and the public URL when HTTP is on
npx zavudev fn info --json          # same, machine-readable
```

Toggling it on an existing function is a CLI action too — it applies to the
already-deployed function, no redeploy:

```sh
npx zavudev fn http enable
npx zavudev fn http disable
```

**Run `npm install` before anything local.** Nothing resolves until you do:

```sh
cd <your-function-dir>
npm install
```

Without it, `zavu fn invoke` fails with `Cannot find module '@zavudev/functions'`
and `npx tsc` silently fetches an unrelated deprecated package instead of the
compiler. Deploy works either way — the runtime layer supplies the module in
production and it is never shipped from your machine — so this only bites the
local loop, which is exactly the loop you use to verify your work.

Commit `.zavu/` — it holds the `functionId`, and there is no command to look one
up, so a teammate who clones without it cannot deploy.

`zavu fn invoke` runs your code with **Bun**. Install it from https://bun.sh.
Every other command runs under plain Node.

### 5. Test

**Local invocation (skip cloud round-trip):**

```sh
# Call a tool handler with synthetic args
npx zavudev fn invoke --tool lookup_order --args '{"orderId":"ORD-001"}'

# Simulate an inbound event for defineFunction
npx zavudev fn invoke --event message.inbound --data '{"from":"+14155551234","text":"hi"}'
```

`--args` is checked against the tool's own `parameters` schema before the handler
runs — required fields, types, and closed `enum` values. A mismatch fails with
the exact field and the expected shape, and never calls the handler:

```
✗ arguments do not match the tool's schema:
   • missing required "orderId"
   • "score" must be one of "hot" | "warm" | "cold", got "banana"
  expected: { orderId: string, score: hot|warm|cold, notes: string? }
```

If the project's dependencies are not installed, this stops with a note telling
you to run `npm install` rather than surfacing Bun's raw module-resolution error.

Exit codes are worth branching on: **2** means you called it wrong (unknown
tool, arguments that fail the schema), **1** means the handler itself threw, **0**
means it ran. A green run therefore means the handler actually executed with
valid input, not merely that nothing crashed.

**Test the agent's brain without sending anything:**

```sh
npx zavudev agents list                # your agents, with their ids
npx zavudev agents test --agent <agentId> --message "where is order ORD-001?"
```

Prints what the agent would reply, plus tokens, latency and how many knowledge
chunks it used. Nothing is delivered and nothing is charged, so it is safe to run
on every prompt edit.

> **Tools do NOT run on the text channel.** The plain text path does not offer
> tools to the model — it will answer "let me check that, one moment" and then do
> nothing. Tools run on **voice**, and inside a flow's `tool` step. `agents test`
> warns you when an agent has tools that its channels will never call.
> Test the handlers themselves with `fn invoke --tool`.

**End-to-end:** send a real message to the sender's WhatsApp/SMS/Telegram number,
or place a call for a voice agent. The agent runs the LLM and replies on the same
channel.

### 6. Debug

When something fails, walk the chain top-down:

```sh
# 1. Did the inbound reach the agent?
npx zavudev agents executions list --sender <senderId>

# 2. Detail of any failed run
npx zavudev agents executions get <executionId> --sender <senderId>

# 3. Live tool handler logs (your console.log calls)
npx zavudev fn logs --tail
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
  voice?: VoiceConfig,           // Add to make the agent answer phone calls. See "Voice agents".
})
```

## Voice agents

Add a `voice` block to a `defineAgent` to make it answer phone calls. When
present with `enabled: true`, the sender's number is answered by the agent. The
LLM runs **co-located in the voice network** for the lowest latency, independent
of the text `model` — set `voice.model` to pick the call model, or omit it to
derive one from the text `model`. Removing the block reverts the agent to
text-only on the next deploy.

```ts
defineAgent({
  senderId: process.env.SENDER_ID!,
  name: "Fermi",
  provider: "zavu",
  model: "openai/gpt-4o-mini",      // text model
  channels: ["voice", "whatsapp"],
  voice: {
    enabled: true,
    model: "openai/gpt-4o",         // co-located voice model (optional; derived from `model` if omitted)
    greeting: "Hi, I'm Fermi. How can I help?",
    language: "en",                 // BCP-47; auto-detected if omitted
    interruptible: true,            // caller can barge in
    maxCallDurationMinutes: 10,
  },
  prompt: "You are Fermi…",
})
```

### voice reference

```ts
voice: {
  enabled: boolean,              // Required. true = answer/place calls; removing the block reverts to text-only.
  model?: string,                // Co-located call model, e.g. "openai/gpt-4o". Derived from the text model if omitted.
  greeting?: string,             // Opening line. Max 1000 chars. If omitted, the agent waits for the caller.
  greetings?: Record<string, string>, // Per-language greeting keyed by language tag: { es: "Hola…" }. Used when the caller's language differs from the one `greeting` is written in. Factory agents ship with this set.
  language?: string,             // BCP-47 (e.g. "en", "es", "pt-BR"). Auto-detected if omitted.
  ttsVoiceId?: string,           // Voice used for speech synthesis.
  voiceSpeed?: number,           // Speech rate 0.5–1.5. Default 1.0.
  interruptible?: boolean,       // Caller can barge in while the agent speaks. Default true.
  maxCallDurationMinutes?: number, // Hard cap on call length. Default 15.
  maxIdleSeconds?: number,       // End the call after this much silence (5–300). Default 30.
  voicemailAction?: "hangup" | "leave_message", // On answering-machine detection (outbound). Default "hangup".
  voicemailMessage?: string,     // Spoken when voicemailAction is "leave_message". Falls back to greeting.
  transferPhoneNumber?: string,  // E.164. Gives the agent a tool to transfer the call to a human.
}
```

Voice requires the **Voice Agents** feature enabled for your team and a phone
number assigned to the sender. An out-of-range or misspelled field makes
`npx zavudev deploy` warn and ship the agent as text-only rather than fail — fix the
value and redeploy.

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
  log: (...args) => void,        // console.log proxy that appears in `npx zavudev fn logs --tail`
}
```

## defineFunction reference (optional)

Use only if you want to handle:
- **Raw HTTP requests** (function exposed at a public URL — `zavu fn init --http` at creation, or `zavu fn http enable` on an existing one)
- **Native event triggers** (`message.inbound`, `broadcast.status_changed`, etc — configured via `npx zavudev fn triggers add`)

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
npx zavudev fn triggers list
npx zavudev fn triggers add --events message.inbound --senders <senderId>
npx zavudev fn triggers add --events broadcast.status_changed --senders any
npx zavudev fn triggers toggle <triggerId>
npx zavudev fn triggers rm <triggerId>
npx zavudev fn triggers events       # list available event types
```

Triggers use signed internal invocations (no HMAC verification needed inside the handler).

## Versions + rollback

Every `npx zavudev deploy` creates an immutable version.

```sh
npx zavudev fn versions list           # alias: npx zavudev fn history
npx zavudev fn rollback 4              # go back to version 4
```

The function metadata in the dashboard tracks the active version + lets you rollback from the UI too.

## Runtime versions

Each function pins to a specific runtime layer at first deploy. Subsequent deploys keep the same pin (immutable for stability).

```sh
npx zavudev deploy --update-runtime   # opt-in upgrade to latest runtime
```

Only opt in when there's a security advisory or feature you want — `npx zavudev deploy` without the flag is safe forever.

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

If the user already created an agent via the dashboard or `npx zavudev agents create`, declaring it in code with the same `senderId + name` will TAKE OVER that agent — Zavu marks it `managedByFunctionId` and the dashboard locks manual edits. The function source becomes source-of-truth.

To go back to manual control: delete the function (`npx zavudev fn delete`) and the agent is freed.

### Cleaning up a function you created

`fn delete` cascades: the Lambda, triggers, secrets, deployment history, and
every agent and tool the function owns. It asks you to type the slug back.

Non-interactively, assert the slug up front:

```sh
npx zavudev fn delete --confirm <slug>
```

This is deliberately not a blind `-y`. You still have to name the thing, so a
wrong directory or a stale id fails instead of deleting something else — which
is what makes it safe to hand to a script or an agent cleaning up after itself.

### Per-environment senders

```ts
defineAgent({
  senderId: process.env.NODE_ENV === "production"
    ? process.env.PROD_SENDER_ID!
    : process.env.DEV_SENDER_ID!,
  // ...
})
```

Then `npx zavudev fn secrets set NODE_ENV production` on prod, `... development` on dev. Same code, different agents.

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

`npx zavudev fn secrets set OPENAI_API_KEY sk-...`. The agent uses the key directly — no Zavu balance consumed for LLM calls.

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

- Slug: lowercase alphanumeric + hyphens, max 50 chars, unique per project.
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
