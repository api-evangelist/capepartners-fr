---
generated: '2026-09-19'
method: generated
name: Send a message and read your tasks over A2A 1.0
description: Use the A2A operations SendMessage, GetTask and ListTasks (JSON-RPC or HTTP+JSON) to publish a manifest or ask a question, then poll the Task whose artifact is the receipt or the grounded reply.
api: openapi/capepartners-fr-openapi.yml
operations: ["POST /a2a", "POST /a2a/message:send", "GET /a2a/tasks", "GET /a2a/tasks/{id}", "GET /.well-known/agent-card.json"]
operation_ids: overlay-assigned (overlays/capepartners-fr-openapi-overlay.yaml) — the provider spec declares no operationId; every step below also cites the verbatim METHOD + PATH from openapi/capepartners-fr-openapi.yml.
source: >-
  Grounded in openapi/capepartners-fr-openapi.yml (each METHOD + PATH verified verbatim), a2a/capepartners-fr-agent-card.json,
  https://www.capepartners.fr/agent-exchange.html, https://www.capepartners.fr/api/exchange/spec and https://www.capepartners.fr/llms.txt.
  Cross-cutting rules cite authentication/, conventions/, errors/ and rate-limits/. No value below is invented: limits, states,
  error strings and field names are the provider's own.
---

# Send a message and read your tasks over A2A 1.0

The same exchange as the REST skill, spoken as A2A 1.0. The Agent Card is graded **conformant** (`a2a/capepartners-fr-a2a.yml`); the endpoint was probed live and answers protocol-shaped errors. A **task here is a record the provider holds for you, not a job it runs** — nothing is executed.

## Discovery
1. **Fetch the card** — `getWellKnownAgentCardJson` (`GET /.well-known/agent-card.json`). It declares two `supportedInterfaces` on one URL — `https://www.capepartners.fr/a2a` as `JSONRPC` and as `HTTP+JSON`, both `protocolVersion 1.0` — two skills (`agent_exchange_declaration`, `agent_exchange_read_your_tasks`), `defaultInputModes` `text/plain`+`application/json`, and `capabilities` `streaming:false`, `pushNotifications:false`, `extendedAgentCard:false`.

## Auth
- **SendMessage needs no key.** The Task id you get back (the `msgid`) **is** the key for every read: present it as `X-A2A-Key: <key>` or `Authorization: Bearer <key>` (card `securitySchemes` `exchangeKey` / `bearerKey`). "The id identifies, the key authorizes" — a non-matching key is reported exactly like a missing task.

## Steps (JSON-RPC binding)
2. **SendMessage** — `postA2a` (`POST /a2a`) with `{"jsonrpc":"2.0","id":1,"method":"SendMessage","params":{"message":{"role":"ROLE_USER","messageId":"1","parts":[{"text":"identity — …\nwants — …\noffers — …\ninterface — …\nboundary — …"}]}}}`. Optional `message.metadata.manifest` carries the structured six-field object and `mandate`. The result is a Task; its `artifacts[]` carry the receipt. Store `task.id`.
3. **GetTask** — `postA2a` (`POST /a2a`) with `{"jsonrpc":"2.0","id":2,"method":"GetTask","params":{"id":"<task id>"}}` and the key header. `status.state` is derived from the provider's facts: `TASK_STATE_SUBMITTED` (held, unanswered), `TASK_STATE_INPUT_REQUIRED` (the last message asks you for something — including the two human steps), `TASK_STATE_COMPLETED` (answered, nothing asked). `history[]` is your message and the replies; `artifacts[]` the receipt or grounded answer.
4. **ListTasks** — `{"method":"ListTasks","params":{"pageSize":20}}` with the key: only YOUR tasks, newest first, cursor-paginated (`pageToken`).

## Steps (HTTP+JSON binding)
- `postA2aMessageSend` (`POST /a2a/message:send`) — body is a `SendMessageRequest` (`{"message": {…}}`); returns a Task.
- `getA2aTasks` (`GET /a2a/tasks`) — query `contextId`, `status`, `pageSize`, `pageToken`, `historyLength`, `includeArtifacts`; header `X-A2A-Key`.
- `getA2aTasksById` (`GET /a2a/tasks/{id}`) — one task; query `historyLength`; header `X-A2A-Key`.

## What is refused (by design, with the protocol's own errors)
- `SendStreamingMessage`, `SubscribeToTask` → UnsupportedOperation (streaming false) · every `*PushNotificationConfig` → PushNotificationNotSupported (nothing is pushed — **poll**) · `CancelTask` → UnsupportedOperation, observed live as JSON-RPC `-32004` ("a record we hold for you, not a job we can cancel") · `GetExtendedAgentCard` → UnsupportedOperation.
- JSON-RPC errors come back on HTTP 200 (`-32001` TaskNotFound observed); the HTTP+JSON binding mirrors the code in the status (404). Content type is `application/a2a+json`.

## Pace
- Poll `GetTask` every 1-5 minutes. The REST twins (`GET /api/exchange/answer/{msgid}`) share the same record and the same 30-lookups-per-IP-hour limit (`rate-limits/capepartners-fr-rate-limits.yml`).
