---
name: sandboxes
description: Run code, commands and services in isolated Linux machines over /v1/sandboxes — create, run commands sync or in the background, read and write files, expose a port, stop and resume. Paid plans only, billed per second from the balance.
---

# Sandboxes

## When to Use

Use this skill when code needs to **run somewhere**, isolated from the caller:

- "Run the Python the agent just wrote and tell me the output."
- "Build this repo and run its tests."
- "Start a dev server and give me a URL to it."
- "Keep a workspace between sessions and pick it up later."

A sandbox is an Ubuntu machine with no access to the Zavu account, to other
sandboxes, or to any secret not passed in `env`.

**Not available on Free.** Creating or resuming answers
`402 paid_plan_required` (`details.dashboardUrl: "/upgrade"`). **Live keys
only** to start one or run commands; a test-mode key can only read, stop and
delete.

## Billing (tell the user before creating one)

| Size | Machine | Per hour |
|---|---|---|
| `small` | 2 vCPU, 4 GB | $0.0225 |
| `default` | 4 vCPU, 8 GB | $0.045 |
| `large` | 8 vCPU, 16 GB | $0.09 |

- $0.01 per start (create or resume), returned if it fails before running.
- Machine time per second while `starting` or `running`, one-minute minimum per run.
- Stopped costs nothing; the disk is kept 14 days (`expiresAt`).
- A start needs the balance (and the monthly spend cap) to cover the $0.01 plus
  15 minutes at that size, or it answers `402 insufficient_balance` /
  `402 spend_cap_reached`.
- A running sandbox is stopped automatically at its `timeoutSeconds` (default
  3600, max 86400), when the balance or cap no longer covers 15 more minutes,
  or when the team leaves its paid plan.

**Always stop or delete what you created.** A forgotten sandbox bills until its timeout.

## Lifecycle

`starting` → `running` → `stopping` → `stopped` → (resume) → `starting`.
`failed` carries `failureReason`. Commands and file calls made while
`starting` wait for it; on `stopped`/`failed` they answer
`409 sandbox_not_running`.

## Workflow (REST)

All requests: `Authorization: Bearer $ZAVU_API_KEY`, base `https://api.zavu.dev`.

```bash
# 1. Create. Pass idempotencyKey so a retry does not create (and charge) twice:
#    a replay answers 409 with the sandbox the first call created.
curl -X POST https://api.zavu.dev/v1/sandboxes \
  -H "Authorization: Bearer $ZAVU_API_KEY" -H "Content-Type: application/json" \
  -d '{"size":"default","timeoutSeconds":1800,"idempotencyKey":"job-42","env":{"API_URL":"https://api.example.com"},"metadata":{"job":"42"}}'
# -> 202 {"sandbox":{"id":"...","status":"starting",...}}

# 2. Write a file (≤5 MiB, under /home/user or /tmp; relative paths are under /home/user)
curl -X PUT https://api.zavu.dev/v1/sandboxes/$ID/files \
  -H "Authorization: Bearer $ZAVU_API_KEY" -H "Content-Type: application/json" \
  -d '{"path":"main.py","content":"print(2 + 2)\n"}'

# 3. Run it synchronously (timeoutSeconds 1-120, default 30)
curl -X POST https://api.zavu.dev/v1/sandboxes/$ID/commands \
  -H "Authorization: Bearer $ZAVU_API_KEY" -H "Content-Type: application/json" \
  -d '{"command":"python3 main.py"}'
# -> 200 {"result":{"exitCode":0,"stdout":"4\n","stderr":"","timedOut":false,...}}

# 4. Stop (billing ends now). Resume later, or delete.
curl -X POST https://api.zavu.dev/v1/sandboxes/$ID/stop -H "Authorization: Bearer $ZAVU_API_KEY"
```

### Long-running commands

Anything over 120 s goes in the background:

```bash
curl -X POST https://api.zavu.dev/v1/sandboxes/$ID/commands \
  -H "Authorization: Bearer $ZAVU_API_KEY" -H "Content-Type: application/json" \
  -d '{"command":"npm ci && npm test","cwd":"app","background":true}'
# -> 202 {"command":{"id":"CMD_ID","status":"running"}}

curl https://api.zavu.dev/v1/sandboxes/$ID/commands/CMD_ID -H "Authorization: Bearer $ZAVU_API_KEY"
# -> {"command":{"status":"running"|"finished"|"unknown","exitCode":...,"stdout":"(last 1 MiB)",...}}
```

A command is never retried. If the call fails mid-way the command may still be running — check before re-sending a non-idempotent one.

### Ports

```bash
curl -X POST https://api.zavu.dev/v1/sandboxes/$ID/ports \
  -H "Authorization: Bearer $ZAVU_API_KEY" -H "Content-Type: application/json" \
  -d '{"port":3000}'
# -> {"port":{"port":3000,"url":"https://<random>.zavu.run/?_token=...","public":false}}
```

The service must listen on `0.0.0.0`. HTTP and WebSockets both work. The
default URL carries an access token in its query string — anyone with the full
URL can open it, so do not paste it in public. A browser keeps the token in a
cookie after the first visit; other clients send `?_token=` on every request.
`"public": true` gives an open URL. Same port, same URL, across stop and
resume (`503` while stopped, `404` once deleted). At most 20 ports per sandbox
(`409 port_limit_reached`).

### Files

| Call | Endpoint |
|---|---|
| Read | `GET /v1/sandboxes/{id}/files?path=...&encoding=utf8\|base64` |
| Write | `PUT /v1/sandboxes/{id}/files` `{path, content, encoding?}` |
| Delete | `DELETE /v1/sandboxes/{id}/files?path=...&recursive=true` |
| List | `GET /v1/sandboxes/{id}/files/list?path=/home/user` |

### Other calls

| Call | Endpoint |
|---|---|
| List | `GET /v1/sandboxes?status=running&limit=50&cursor=...` |
| Get | `GET /v1/sandboxes/{id}` |
| Resume | `POST /v1/sandboxes/{id}/resume` `{timeoutSeconds?}` — billed like a create |
| Delete | `DELETE /v1/sandboxes/{id}` — stops, settles, deletes the disk; irreversible |

## SDK

`@zavudev/sdk` 0.63.0 and later has `client.sandboxes` (`create`, `list`,
`retrieve`, `stop`, `resume`, `delete`) with `client.sandboxes.commands`
(`run`, `retrieve`), `client.sandboxes.files` (`read`, `write`, `delete`,
`list`) and `client.sandboxes.ports` (`expose`). On an older SDK, use the REST
calls above.

## Limits and errors

| | |
|---|---|
| Starting or running at once, per team | 10 → `403 plan_limit_reached` |
| Starts per day (creates + resumes) | 200 → `429 daily_limit_exceeded`, resets 00:00 UTC |
| File through the API | 5 MiB → `413 file_too_large` |

| Code | Do this |
|---|---|
| `402 paid_plan_required` | The team is on Free; it must upgrade. |
| `402 insufficient_balance` / `spend_cap_reached` | Add funds / raise the cap in Billing. |
| `409 sandbox_not_running` | Resume it first. |
| `409 sandbox_not_stopped` | Only a stopped sandbox can be resumed. |
| `409 sandbox_expired` | Disk deleted 14 days after stopping; create a new one. |
| `503 sandbox_unavailable` | No capacity; retry shortly. Nothing was charged for machine time. |
