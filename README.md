# Zavu Skills for Claude Code

Claude Code skills for the [Zavu](https://zavu.dev) unified messaging API. Send SMS, WhatsApp, Email, Telegram, and Voice messages with intelligent routing.

## Installation

```bash
npx skills add zavudev/zavu-skills
```

### Optional: MCP Server

For direct API execution from Claude Code:

```bash
claude mcp add zavudev --scope user \
  -e ZAVUDEV_API_KEY="your_key" -- npx -y @zavudev/sdk-mcp
```

## Skills

| Skill | Type | Description |
|-------|------|-------------|
| `zavu-rules` | Always loaded | SDK ecosystem, CLI tooling, auth, conventions, business rules |
| `functions` | Workflow + Reference | Zavu Functions: serverless TypeScript with `defineAgent` / `defineTool`, `npx zavudev deploy`, debugging |
| `send-message` | Workflow | Channel selection decision tree + all message types |
| `api-v2` | Reference + Workflow | API v2 over HTTP: addresses instead of senders, connect channels, reply by conversation, one project webhook, flat errors |
| `broadcast-campaign` | Workflow | Full broadcast lifecycle: create, review, send, monitor |
| `conversations` | Reference | Read the inbox: list threads, fetch messages, mark read, reply from the right sender |
| `webhook-setup` | Reference + Code | Webhook config, signature verification, event handling |
| `whatsapp-templates` | Reference | Template creation, Meta approval, OTP authentication |
| `ai-agent` | Reference + Workflow | Agent config via the imperative API (dashboard / REST) — for no-code agent setups |
| `memory` | Reference + Workflow | Agent memory: facts recalled by meaning (`ctx.memory`), JSON collections recalled by key, scoped per project/contact/thread |
| `sandboxes` | Reference + Workflow | Isolated Linux machines over `/v1/sandboxes`: create, run commands, files, ports, stop/resume, billing and limits |
| `voice-agent` | Workflow + Reference | AI voice agents: `voice` block on `defineAgent`, placing/inspecting calls, transcripts, human handoff |
| `contacts-management` | Reference | Multi-channel contacts, merge, introspection |
| `phone-numbers` | Reference | Search, purchase, regulatory requirements |

## SDK Ecosystem

| Language | Package | Install |
|----------|---------|---------|
| TypeScript | `@zavudev/sdk` | `npm add @zavudev/sdk` |

`@zavudev/sdk` is the only maintained SDK. It covers API v1 and API v2 (`client.v2`). The Python, Go, PHP, Ruby, Java, Kotlin and C# packages are no longer updated: they have no API v2 and miss v1 fields added since September 2026. Do not use them in new code. From any language other than TypeScript, call the REST API directly (`https://api.zavu.dev`, `Authorization: Bearer $ZAVUDEV_API_KEY`) or generate a client from the OpenAPI specs at `https://docs.zavu.dev/openapi.json` (v1) and `https://docs.zavu.dev/api-reference/v2/openapi.json` (v2).

## Links

- [Documentation](https://www.zavu.dev/docs)
- [API Reference](https://www.zavu.dev/docs/api-reference)
- [Dashboard](https://dashboard.zavu.dev)
- [MCP Server](https://www.npmjs.com/package/@zavudev/sdk-mcp)

## License

Apache-2.0
