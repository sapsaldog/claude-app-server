# symphony-claude

A **JSON-RPC 2.0 Claude Code App Server** conforming to the **OpenAI Symphony Codex** protocol spec.

No API key required. Authentication is handled by the `claude` CLI (`claude auth`).

Clients communicate over **stdio** (default) or **WebSocket**, using newline-delimited JSON (NDJSON).

---

## Install

### Homebrew

```bash
brew tap sapsaldog/symphony
brew install symphony-claude
```

### From source

```bash
pnpm install
pnpm run build
```

---

## Requirements

- Node.js >= 18
- [Claude Code CLI](https://claude.ai/code) installed and authenticated (`claude auth`)

---

## Quick start

```bash
# Start with WebSocket + QR code (recommended)
symphony-claude start

# Custom port
symphony-claude start --port 4000

# Plain WebSocket (no TLS)
symphony-claude start --no-tls

# stdio mode (for piped/programmatic use)
symphony-claude
```

On startup you'll see:

```
  symphony-claude  ·  WebSocket (TLS)
  ─────────────────────────────────
  Local:    wss://localhost:3284?key=AbC123
  Network:  wss://192.168.x.x:3284
  Pair Key: AbC123

  ▄▄▄ ... QR code ...
  Scan to connect
```

A random 6-character **pair key** is generated each time the server starts. The key is embedded in the QR code URL. Clients connecting without a valid key are rejected (close code 4401).

Scan the QR code from any device on the same Wi-Fi to connect.

---

## Transports

| Command | Transport | Notes |
|---------|-----------|-------|
| `symphony-claude start` | WebSocket :3284 | Shows QR code, binds to all interfaces |
| `symphony-claude start --port N` | WebSocket :N | Custom port |
| `symphony-claude start --no-tls` | WebSocket :3284 | Plain `ws://` (no TLS) |
| `symphony-claude --transport ws` | WebSocket :3284 | No QR code |
| `symphony-claude` | stdio | For piped/programmatic use |

### `--no-tls`

By default, WebSocket mode uses a self-signed TLS certificate (`wss://`). Pass `--no-tls` to disable TLS and use plain `ws://` instead. This is useful for local development or when TLS is handled by a reverse proxy.

---

## Protocol overview

Messages are newline-delimited JSON, following [JSON-RPC 2.0](https://www.jsonrpc.org/specification). Both `snake_case` and `camelCase` parameter names are accepted for Codex protocol compatibility.

### Handshake

```jsonc
// Client → Server
{ "jsonrpc": "2.0", "method": "initialize", "params": {
    "client": { "name": "my-app", "version": "1.0.0" },
    "cwd": "/path/to/project"
  }, "id": 1 }

// Server → Client (response)
{ "jsonrpc": "2.0", "result": {
    "server": { "name": "symphony-claude", "version": "1.0.0" },
    "capabilities": { ... }
  }, "id": 1 }

// Server → Client (notification, async)
{ "jsonrpc": "2.0", "method": "initialized", "params": { "server": "symphony-claude" } }
```

---

## Methods

### Thread management

| Method | Params | Returns |
|--------|--------|---------|
| `thread/start` | `{ cwd?, permissionMode? }` | `{ thread: { id, created_at } }` |
| `thread/resume` | `{ threadId }` | `{ thread: { id, turns[], cwd, … } }` |
| `thread/fork` | `{ threadId }` | `{ thread: { id, forked_from, created_at } }` |

### Turn management

| Method | Params | Returns |
|--------|--------|---------|
| `turn/start` | `{ threadId, content, model? }` or `{ threadId, input: [{ type, text }] }` | `{ turn: { id } }` |
| `turn/steer` | `{ threadId, content }` | `{ turn_id }` |
| `turn/interrupt` | `{ threadId }` | `{ turn_id, status }` |

`turn/start` returns immediately; the agent streams back **notifications** until `turn/completed`.

### Discovery

| Method | Returns |
|--------|---------|
| `model/list` | List of available Claude models |
| `skills/list` | List of available tools (Read, Bash, …) |
| `app/list` | (stub, always empty) |

### Approval

| Method | Params | Description |
|--------|--------|-------------|
| `approval/respond` | `{ threadId, approved, permissionMode? }` | Respond to a permission prompt |

---

## Server notifications

After `turn/start`, the server streams these notifications:

| Notification | When |
|-------------|------|
| `turn/started` | Turn began |
| `item/progress` | Streaming text delta — `{ turn_id, delta: { type, text } }` |
| `item/created` | Item finalized (text, tool_call, tool_result) |
| `usage/update` | Token usage update — `{ turn_id, usage: { input_tokens, output_tokens, total_tokens } }` |
| `turn/completed` | Turn finished — `{ turn_id, status, items_count, usage?, cost_usd? }` |
| `turn/failed` | Turn failed — `{ turn_id, error }` |
| `turn/permission_denied` | Permission denied — `{ turn_id, denials }` |

---

## Permission modes

| Mode | Behaviour |
|------|-----------|
| `default` | Prompts for bash and file writes |
| `acceptEdits` | Auto-approves file writes; prompts for bash |
| `bypassPermissions` | Approves all tools automatically |

---

## Example session (stdio)

```bash
symphony-claude
```

```jsonc
// Initialize
{"jsonrpc":"2.0","method":"initialize","params":{"client":{"name":"demo","version":"1.0"},"cwd":"/tmp/my-project"},"id":1}

// Start a thread
{"jsonrpc":"2.0","method":"thread/start","params":{"cwd":"/tmp/my-project","permissionMode":"acceptEdits"},"id":2}

// Start a turn
{"jsonrpc":"2.0","method":"turn/start","params":{"threadId":"<id>","content":"List the files in this project."},"id":3}
```

---

## Architecture

```
src/
  index.ts       CLI entry — parses subcommand / flags, shows QR code
  protocol.ts    JSON-RPC 2.0 types and helpers
  types.ts       Domain types: Thread → Turn → Item
  transport.ts   stdio and WebSocket transports
  tools.ts       Built-in skills catalog
  server.ts      ClaudeAppServer — method handlers + claude CLI runner
```

Each turn spawns:
```
claude --print --output-format stream-json --include-partial-messages
       --permission-mode <mode>
       --session-id <id>    # first turn of a thread
       --resume <id>        # subsequent turns
```

---

## Models

Default model can be overridden per turn:

```json
{ "method": "turn/start", "params": { "threadId": "…", "content": "…", "model": "claude-haiku-4-5" } }
```

Available: `claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`

---

## Related

- [Symphony (fork)](https://github.com/sapsaldog/symphony) — Modified Symphony client for Claude Code integration
- [Symphony (original)](https://github.com/openai/symphony) — OpenAI's original Symphony
