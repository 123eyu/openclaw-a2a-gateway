# OpenClaw A2A Gateway Plugin

An [OpenClaw](https://github.com/openclaw/openclaw) plugin that implements the [A2A (Agent-to-Agent) protocol v0.3.0](https://google.github.io/A2A/) — enabling your OpenClaw agents to communicate with any A2A-compatible agent platform.

## Features

- **A2A v0.3.0 Compliant** — Full protocol surface: Agent Card discovery, JSON-RPC 2.0, and REST endpoints
- **Bidirectional Communication** — Receive inbound A2A requests *and* send outbound messages to peer agents
- **Gateway RPC Dispatch** — Routes A2A messages to OpenClaw agents via the gateway's internal WebSocket API with automatic session resolution
- **Legacy Fallback** — Falls back to `dispatchToAgent` bridge or `/hooks/wake` when gateway RPC is unavailable
- **Bearer Token Auth** — Optional inbound authentication with configurable bearer tokens
- **Peer Management** — Configure multiple remote A2A peers with independent auth (Bearer or API Key)
- **In-Memory Task Store** — Tracks task lifecycle (working → completed/canceled) per the A2A spec
- **Internal Gateway Extensions** — Custom reliability layer with HMAC-SHA256 signing, outbox pattern, idempotency, and message routing for gateway-to-gateway mesh

## Architecture

```
                          ┌─────────────────────────────────┐
   External A2A Agent ──▶ │  /.well-known/agent.json        │
                          │  /a2a/jsonrpc  (JSON-RPC 2.0)   │
                          │  /a2a/rest     (REST)            │
                          │                                  │
                          │  ┌───────────────────────────┐   │
                          │  │  OpenClawAgentExecutor     │   │
                          │  │  ┌─────────────────────┐   │   │
                          │  │  │ Gateway RPC (WS)    │◀──┼───┼── OpenClaw Gateway
                          │  │  │ Legacy Bridge       │   │   │
                          │  │  │ /hooks/wake fallback│   │   │
                          │  │  └─────────────────────┘   │   │
                          │  └───────────────────────────┘   │
                          │                                  │
   OpenClaw Agent ───────▶│  a2a.send (Gateway Method)      │──▶ Remote A2A Peer
                          └─────────────────────────────────┘
```

## Installation

### 1. Copy the plugin into your OpenClaw workspace

```bash
# From your OpenClaw workspace root
cp -r /path/to/a2a-gateway plugins/a2a-gateway
cd plugins/a2a-gateway
npm install
```

### 2. Register in OpenClaw config

Add the plugin to your `gateway.yaml` or use the CLI:

```bash
openclaw gateway config.patch '{
  "plugins": {
    "a2a-gateway": {
      "enabled": true,
      "config": {
        "agentCard": {
          "name": "My OpenClaw Agent",
          "description": "An AI assistant powered by OpenClaw",
          "url": "https://your-domain.com/.well-known/agent.json",
          "skills": [
            {
              "id": "chat",
              "name": "General Chat",
              "description": "Natural language conversation"
            }
          ]
        },
        "server": {
          "host": "0.0.0.0",
          "port": 18800
        },
        "security": {
          "inboundAuth": "bearer",
          "token": "your-secret-token"
        },
        "routing": {
          "defaultAgentId": "main"
        }
      }
    }
  }
}'
```

### 3. Restart the gateway

```bash
openclaw gateway restart
```

## Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `agentCard.name` | `string` | *required* | Display name in the A2A Agent Card |
| `agentCard.description` | `string` | `"A2A bridge for OpenClaw agents"` | Agent description |
| `agentCard.url` | `string` | auto-generated | Public URL for the Agent Card endpoint |
| `agentCard.skills` | `array` | *required* | List of skills (string or `{id, name, description}`) |
| `server.host` | `string` | `"0.0.0.0"` | Bind address for the HTTP server |
| `server.port` | `number` | `18800` | Port for the A2A HTTP server |
| `peers` | `array` | `[]` | Remote A2A agent peers (see below) |
| `security.inboundAuth` | `"none" \| "bearer"` | `"none"` | Inbound authentication method |
| `security.token` | `string` | `""` | Bearer token for inbound requests |
| `routing.defaultAgentId` | `string` | `"default"` | OpenClaw agent to route inbound messages to |

### Peer Configuration

Each peer in the `peers` array:

| Field | Type | Description |
|-------|------|-------------|
| `name` | `string` | Unique peer identifier (used in `a2a.send`) |
| `agentCardUrl` | `string` | URL to the peer's Agent Card |
| `auth.type` | `"bearer" \| "apiKey"` | Authentication method for outbound requests |
| `auth.token` | `string` | Auth token/key |

**Example with peers:**

```json
{
  "peers": [
    {
      "name": "research-agent",
      "agentCardUrl": "https://research.example.com/.well-known/agent.json",
      "auth": {
        "type": "bearer",
        "token": "peer-secret-token"
      }
    }
  ]
}
```

## Endpoints

Once running, the plugin exposes:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/.well-known/agent.json` | GET | A2A Agent Card (auto-discovery) |
| `/a2a/jsonrpc` | POST | JSON-RPC 2.0 endpoint (`message/send`, etc.) |
| `/a2a/rest` | POST | REST-style endpoint |

## Sending Messages to Peers

From any OpenClaw agent session, use the gateway method:

```
a2a.send({ peer: "research-agent", message: { text: "Summarize recent papers on LLMs" } })
```

The client will:
1. Fetch the peer's Agent Card to discover its service URL
2. Send a JSON-RPC `message/send` request with proper auth headers
3. Return the response

## Agent Dispatch Flow

When an inbound A2A message arrives:

1. **Gateway RPC (preferred)** — Connects to OpenClaw gateway via WebSocket, resolves the target agent session, and dispatches the message. Waits for the agent's response and returns it as a completed A2A task.

2. **Legacy Bridge (fallback)** — If gateway RPC fails, falls back to `dispatchToAgent` API.

3. **Hooks Wake (last resort)** — If all else fails, sends a wake event to `/hooks/wake` so the agent can process the message asynchronously.

## Internal Modules

The `src/internal/` directory contains gateway-to-gateway extensions that go beyond the A2A spec:

| Module | Purpose |
|--------|---------|
| `envelope.ts` | Custom `a2a/v1` envelope format with TTL, hop-count, and loop detection |
| `transport.ts` | HTTP transport layer with `/a2a/v1/inbox` endpoint |
| `security.ts` | HMAC-SHA256 signing, nonce cache, and peer registry |
| `routing.ts` | Route-key and agent-id based message routing |
| `outbox.ts` | Reliable at-least-once delivery with exponential backoff |
| `idempotency.ts` | SHA-256 payload fingerprinting and deduplication |
| `metrics.ts` | Protocol metrics and structured logging |

## Testing

```bash
npm test
```

Runs the test suite with Node.js built-in test runner (`node:test`). Tests cover:
- Agent Card generation with A2A v0.3.0 fields
- Gateway RPC dispatch (preferred path)
- Legacy bridge fallback
- Task cancellation lifecycle
- Outbound peer messaging via `a2a.send`

## Tech Stack

- **Runtime:** Node.js (ESM)
- **A2A SDK:** `@a2a-js/sdk` v0.3.0
- **HTTP:** Express 4
- **Language:** TypeScript 5
- **Testing:** `node:test` + `supertest`

## License

MIT

## Links

- [A2A Protocol Specification](https://google.github.io/A2A/)
- [OpenClaw Documentation](https://docs.openclaw.ai)
- [@a2a-js/sdk](https://www.npmjs.com/package/@a2a-js/sdk)
