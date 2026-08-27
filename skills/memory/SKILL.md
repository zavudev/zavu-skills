---
name: memory
description: Give agents memory that survives the conversation — facts recalled by meaning (ctx.memory.add / search) and JSON documents recalled by key (collections), scoped to the project, a contact, or a thread.
---

# Memory

## When to Use

Use this skill when an agent needs to remember something **after the current
invocation ends**:

- "Remember that this customer prefers WhatsApp."
- "What do we already know about this contact?"
- "Store the order I just looked up so the next message doesn't refetch it."
- "Give the bot a long-term memory / knowledge it accumulates."

Two shapes, one API:

| You want to recall by… | Use | Because |
|---|---|---|
| **meaning** — "how do they like to be contacted?" | `memory.add` / `memory.search` | Text is embedded on write; retrieval is semantic |
| **key** — "order ORD-12345" | `memory.collection(name)` | Exact lookup, JSON in and JSON out |

**This is not the knowledge base.** A knowledge base holds documents you author
up front and the agent reads (`ai-agent` skill). Memory holds what the agent
*learns while running*. Use both: the KB for your policies, memory for what this
customer said last Tuesday.

**This is not conversation history.** The last N messages are already in the
agent's context automatically. Memory is for what should outlive them.

## The one thing to get right: scope

Every read and every write happens inside exactly one scope, and scopes are
isolated — a search in one **never** returns a fact from another.

| Scope | Holds | Cap |
|---|---|---|
| `project` | What the agent knows in general. Shared by every conversation. | 10,000 facts |
| `contact` | What it knows about one person, across every thread with them. | 2,000 facts |
| `conversation` | What it knows about one inbox thread. | 1,000 facts |

Default is `project` everywhere — including `memory.add(...)` with no options.
**That is the mistake to avoid**: a fact about one customer written at project
scope is visible to every other customer's conversation.

```ts
// WRONG — every customer's agent can now recall this person's preference
await ctx.memory.add("Prefers WhatsApp over email.");

// RIGHT — filed against the person it is about
await ctx.memory.contact!.add("Prefers WhatsApp over email.");
```

## In a function or tool: `ctx.memory`

Requires `@zavudev/functions` **0.3.0 or later**.

```ts
import { defineTool } from "@zavudev/functions";

export const rememberPreference = defineTool({
  name: "remember_preference",
  description: "Save something the customer told us about how they want to be served.",
  parameters: {
    type: "object",
    properties: { fact: { type: "string", description: "One fact, in plain language." } },
    required: ["fact"],
  },
  handler: async ({ fact }, ctx) => {
    // ctx.memory.contact is already bound to whoever the agent is talking to.
    if (!ctx.memory.contact) return { ok: false, reason: "no contact on this call" };
    await ctx.memory.contact.add(fact, { metadata: { source: "customer" } });
    return { ok: true };
  },
});
```

Recall before answering:

```ts
const hits = await ctx.memory.contact!.search("how they prefer to be contacted", {
  limit: 3,
  minScore: 0.3,
});
// hits: [{ id, text, score, metadata?, createdAt }]
```

### The scope handles

| Handle | Scope | Present when |
|---|---|---|
| `ctx.memory` | project | always |
| `ctx.memory.contact` | the contact being talked to | **optional** — `undefined` in an event handler, or a tool called outside a conversation |
| `ctx.memory.conversation` | the current thread | **optional** — same |
| `ctx.memory.forContact(id)` | a contact you name | always |
| `ctx.memory.forConversation(id)` | a thread you name | always |

`contact` and `conversation` are optional properties, so TypeScript makes you
handle their absence. Do handle it — a tool that assumes a contact throws
`MemoryScopeError` the first time it runs from a cron trigger.

```ts
import { MemoryScopeError } from "@zavudev/functions";

try {
  await ctx.memory.add(fact, { scope: "contact" }); // throws if there is no contact
} catch (e) {
  if (e instanceof MemoryScopeError) { /* fall back to project scope, or skip */ }
}
```

`MemoryScopeError` means *the scope could not be resolved* — never that the
scope is empty. An empty scope is an empty array.

### Collections: JSON by key

```ts
const orders = ctx.memory.contact!.collection("orders");

await orders.set("ORD-12345", { status: "shipped", total: 42.5 });
const order = await orders.get<{ status: string }>("ORD-12345"); // null if absent
await orders.delete("ORD-12345");                                 // true even if absent

const page = await orders.list({ prefix: "ORD-2026", limit: 50 });
// { items: [{ key, value, rev, createdAt, updatedAt }], nextCursor }
```

**Keys are `A-Z a-z 0-9 . _ : @ -`, 1–255 characters — no `+`.** A phone number
in E.164 is not a valid key: `set("+56940560201", …)` is refused with `400`.
Key it by its digits (`56940560201`) or with a prefix (`phone:56940560201`), and
keep the E.164 inside the value.

**`query` needs an index, and indexes are declared at creation.** Writing to an
unknown collection creates it implicitly with **no indexes**, and indexes can
never be added afterwards. If you will ever look items up by a field, create the
collection first:

```bash
npx zavudev memory collections create orders --index status
```

```ts
const shipped = await orders.query({ field: "status", op: "eq", value: "shipped" });
```

Querying an unindexed field throws `field_not_indexed`. The local stub has no
indexes and answers any field, so it warns on the console — confirm a query
against the real API before shipping it.

## From the CLI

Everything takes `--scope project` (default) / `--scope contact:<id>` /
`--scope conversation:<id>`.

```bash
npx zavudev memory add "Orders before 2pm ship same day."
npx zavudev memory search "shipping cutoff" --limit 5 --min-score 0.3
npx zavudev memory list --scope contact:jd7x2k3m4n5p6q7r8s9t0abc
npx zavudev memory delete <memoryId>
npx zavudev memory wipe --scope conversation:<id>     # asks for confirmation

npx zavudev memory collections create orders --index status --default-ttl 2592000
npx zavudev memory collections list
npx zavudev memory collection set orders ORD-1 '{"status":"shipped"}'
npx zavudev memory collection get orders ORD-1
npx zavudev memory collection query orders --field status --eq shipped
npx zavudev memory collection export orders > backup.ndjson
npx zavudev memory collection import orders --file backup.ndjson

npx zavudev memory usage
```

Note the shape of the collection verbs: the **verb comes before the name**
(`memory collection set orders ORD-1`), and collection *management* is a
separate group (`memory collections create`, plural).

## From the REST API

`@zavudev/sdk` does **not** have a memory resource yet — use `curl` or `fetch`.

Scope is passed **in the body** on the two POSTs that write or search facts,
and as a **`?scope=` query parameter** on everything else. That asymmetry is
the most common source of a confusing 404.

```bash
# Remember (scope in the BODY)
curl -X POST https://api.zavu.dev/v1/memory \
  -H "Authorization: Bearer $ZAVU_API_KEY" -H "Content-Type: application/json" \
  -d '{"text":"Prefers WhatsApp.","scope":"contact","contactId":"jd7x..."}'

# Recall (scope in the BODY)
curl -X POST https://api.zavu.dev/v1/memory/search \
  -H "Authorization: Bearer $ZAVU_API_KEY" -H "Content-Type: application/json" \
  -d '{"query":"contact preference","scope":"contact","contactId":"jd7x...","minScore":0.3}'

# List / read / delete (scope in the QUERY STRING)
curl "https://api.zavu.dev/v1/memory?scope=contact:jd7x..." -H "Authorization: Bearer $ZAVU_API_KEY"
curl -X DELETE "https://api.zavu.dev/v1/memory/$MEMORY_ID?scope=contact:jd7x..." \
  -H "Authorization: Bearer $ZAVU_API_KEY"

# Collections
curl -X POST https://api.zavu.dev/v1/memory/collections \
  -H "Authorization: Bearer $ZAVU_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"orders","indexes":[{"field":"status"}]}'

curl -X PUT "https://api.zavu.dev/v1/memory/collections/orders/items/ORD-1?scope=contact:jd7x..." \
  -H "Authorization: Bearer $ZAVU_API_KEY" -H "Content-Type: application/json" \
  -d '{"value":{"status":"shipped"}}'
```

A collection is defined once per project; its **items are per-scope**, so the
same collection holds a separate set of documents for the project, for each
contact, and for each conversation. `DELETE /v1/memory/collections/{name}`
removes it and its items in **every** scope.

## Writing facts an agent can actually find

- **One fact per `add`.** A paragraph with five unrelated facts embeds as an
  average of all five and is retrieved well by none of them.
- **Write it the way it will be asked for.** Retrieval is semantic, so
  "Prefers to be contacted in the morning" is found by "when should I call
  them"; "pref: AM" is found by nothing.
- **Set `minScore` whenever your code branches on the result.** Without it a
  scope always returns its `limit` closest facts however distant — so "found
  nothing relevant" is indistinguishable from "found the right thing". Start at
  `0.3`.
- **`metadata` is not searchable.** Search matches `text` only; metadata comes
  back with the hit for you to filter on afterwards.
- **Use `ttlSeconds` for anything temporary** (60s to 2 years). A fact with no
  TTL is kept until deleted, and counts against the scope cap forever.
- **Don't re-remember what you can look up.** An order status belongs in a
  collection, not as a fact — facts are for what you would otherwise forget.

## Limits and errors

| Limit | Value |
|---|---|
| Fact text | 8 KB |
| Fact metadata | 2 KB (keys 64 chars, values 512) |
| Search results | 20 max, 5 by default |
| Collection document | 200 KB serialized, 32 levels deep |
| Indexes per collection | 2, fixed at creation |
| Batch operations | 25 per call |
| TTL | 60 seconds to 2 years |

| Error | Means |
|---|---|
| `400 field_not_indexed` | The field carries no index, or the collection was auto-created without any |
| `404 memory_not_found` / `item_not_found` | Not in **that scope** — check you passed the scope it was written with |
| `409 revision_conflict` | An `If-Match` write lost a race. Re-read and retry |
| `413 document_too_large` | Over 200 KB. Memory is not a blob store — put media in file storage and keep the URL |
| `429 memory_scope_full` | The scope hit its cap. Delete, or use a narrower scope |
| `429 memory_limit_exceeded` | The plan's monthly unit quota is spent |
| `503 memory_unavailable` | Memory is not enabled on this deployment |

**Reads and deletes are never refused for quota reasons** — only writes are. Your
data stays readable and removable whatever the billing state.

## Test mode and local development

- A **test-mode API key** writes to a separate partition that self-destructs
  after 30 days and is never billed. Test data is invisible to live keys.
- **Locally**, `createLocalContext()` gives you a `ctx.memory` backed by an
  in-process stub, so tools are testable without network.

```ts
import { createLocalContext } from "@zavudev/functions";
const { memory } = createLocalContext({ contactId: "test-contact" });
```

**The stub is not a simulator, and two behaviours differ enough to mislead you:**

- **`search` matches substrings, not meaning.** The stub has no embedding
  model, so it falls back to keyword matching and scores every hit `1`.
  `search("WhatsApp")` finds "Prefers WhatsApp over email"; the semantically
  correct `search("how they prefer to be contacted")` finds **nothing**. Live,
  it is the other way round. So a local `0` is not evidence your code is
  broken, a local hit is not evidence your query is good, and `minScore` does
  nothing locally.
- **`query` answers any field**, because the stub has no indexes. Live it
  throws `field_not_indexed`. The stub warns on the console when it serves one.

Use the stub to check that your tool *wires up* — that it calls the right
scope, handles a missing contact, and shapes its return value. Confirm anything
that depends on retrieval quality against the real API, with a test-mode key.

## Billing

Metered in **units** from real capacity, where a write costs 5x a read, plus
storage. `npx zavudev memory usage` (or `GET /v1/memory/usage`) reports the
month's units, storage bytes, the plan quota, and `capReached`. Free includes
250,000 units, 0.5 GB, 10 collections and 5,000 project facts; paid tiers scale
from there.

## Related Skills

- **`functions`** — the `defineTool` / `defineAgent` surface `ctx.memory` lives on
- **`ai-agent`** — knowledge bases (documents you author) versus memory (what the agent learns)
- **`contacts-management`** — the `contactId` that `contact` scope is keyed on
