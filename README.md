# Zavu Skills for Claude Code

Claude Code skills for the [Zavu](https://zavu.dev) unified messaging API. Send SMS, WhatsApp, Email, Telegram, and Voice messages with intelligent routing.

## Installation

```bash
npx skills add zavudev/zavu-skills
```

### Optional: MCP Server

For direct API execution from Claude Code:

```bash
claude mcp add --transport stdio zavudev_sdk_api \
  --env ZAVUDEV_API_KEY="your_key" -- npx -y @zavudev/sdk-mcp
```

## Skills

| Skill | Type | Description |
|-------|------|-------------|
| `zavu-rules` | Always loaded | SDK ecosystem, auth, conventions, business rules |
| `send-message` | Workflow | Channel selection decision tree + all message types |
| `broadcast-campaign` | Workflow | Full broadcast lifecycle: create, review, send, monitor |
| `webhook-setup` | Reference + Code | Webhook config, signature verification, event handling |
| `whatsapp-templates` | Reference | Template creation, Meta approval, OTP authentication |
| `ai-agent` | Reference + Workflow | Agent config, flows, tools, knowledge bases |
| `contacts-management` | Reference | Multi-channel contacts, merge, introspection |
| `phone-numbers` | Reference | Search, purchase, regulatory requirements |

## SDK Ecosystem

| Language | Package | Install |
|----------|---------|---------|
| TypeScript | `@zavudev/sdk` | `npm add @zavudev/sdk` |
| Python | `zavudev` | `pip install zavudev` |
| Go | `github.com/zavudev/sdk-go` | `go get github.com/zavudev/sdk-go` |
| PHP | `zavudev/sdk-php` | Composer |
| Ruby | `zavudev/sdk-ruby` | RubyGems |

## Links

- [Documentation](https://docs.zavu.dev)
- [API Reference](https://docs.zavu.dev/api-reference)
- [Dashboard](https://dashboard.zavu.dev)
- [MCP Server](https://www.npmjs.com/package/@zavudev/sdk-mcp)

## License

Apache-2.0
