# OpenCode Claude Bridge

> A drop-in local proxy that lets **Claude Code** run on **OpenCode Zen** — DeepSeek V4
> and other OpenCode models — with full `reasoning_content` replay so thinking-mode tool
> calls keep working.

Claude Code speaks the Anthropic `/v1/messages` API. OpenCode Zen exposes models through
an OpenAI-compatible `/v1/chat/completions` endpoint. This bridge translates between the two
protocols **and** preserves the DeepSeek `reasoning_content` history that thinking-mode and
tool-calling flows require.

---

## Features

- **Protocol translation** — Anthropic `/v1/messages` ↔ OpenAI-compatible
  `/v1/chat/completions`. Claude Code runs unchanged.
- **Thinking-mode support** — DeepSeek `reasoning_content` is surfaced to Claude Code
  as native Anthropic `thinking` content blocks (visible in the UI).
- **Reasoning replay** — DeepSeek requires prior `reasoning_content` to be sent back on
  later tool-call turns. The bridge caches it by **tool-call ID**, **assistant-text hash**,
  and **recent tool context**, and replays it automatically so sessions never fail with
  `reasoning_content must be passed back`.
- **Visible thinking** — streaming `reasoning_content` deltas are emitted as
  `thinking_delta` blocks in real time.
- **Model rotation & fallback** — configurable `models` / `fallbackModels` lists. The
  bridge retries the next model on rate limits (429) and upstream errors
  (500/502/503/504) or `model unavailable` (400); genuine request/config errors are not
  hidden.
- **DeepSeek-aware extensions** — `thinking` and `reasoning_effort` are forwarded to
  DeepSeek model names only, so experimental non-DeepSeek chat models are not polluted
  with DeepSeek-specific fields.
- **Robust tool calling** — coalesces adjacent assistant tool-call turns, repairs orphaned
  tool results, and converts DeepSeek-rejected forced `tool_choice` into system
  instructions.
- **Streaming & non-streaming** — full SSE translation with correct Anthropic
  `message_start` / `content_block_*` / `message_delta` / `message_stop` events and usage
  mapping (including cache-hit / cache-miss tokens).
- **Local-first & zero-dependency** — pure Node.js (`>=18`), no `npm install` needed.
  Listens on loopback by default.

---

## Architecture

```
Claude Code                  Bridge (this server)                OpenCode Zen
   │                              │                                    │
   │  POST /v1/messages           │                                    │
   │  (Anthropic format)          │                                    │
   ├─────────────────────────────►│                                    │
   │                              │  Anthropic → OpenAI payload        │
   │                              │  (+ reasoning_content replay)      │
   │                              ├───────────────────────────────────►│
   │                              │  POST /v1/chat/completions         │
   │                              │  (Bearer = ANTHROPIC_API_KEY)      │
   │                              │◄───────────────────────────────────┤
   │                              │  SSE (content + reasoning_content) │
   │  SSE thinking + text + tool  │                                    │
   │◄─────────────────────────────┤                                    │
   │                              │  cache reasoning for next turns    │
```

1. Claude Code sends an Anthropic-style request to the bridge (e.g. `http://127.0.0.1:8787`).
2. The bridge converts messages, tools, `tool_choice`, `thinking`, and `output_config.effort`
   into an OpenAI-compatible payload and attaches previous `reasoning_content` where DeepSeek
   requires it.
3. The request is forwarded to OpenCode Zen with your OpenCode Go API key as a Bearer token.
4. Streaming responses are translated back into Anthropic `thinking` / `text` / `tool_use`
   content blocks, and the reasoning is cached for later replay.

### Request & fallback flow

For each `/v1/messages` request the bridge builds one OpenAI-compatible payload and then runs
a **single-provider rotation loop** over the configured model list (see
[How Model Rotation Works](#how-model-rotation-works)):

```
payload built once
        │
        ▼
for model in [ fallbackModels OR models ]:
        ├─ POST /chat/completions (model)
        ├─ 2xx ...............► return translated stream
        ├─ retryable (429/5xx/400-unavailable)
        │                     └─► log + try NEXT model
        └─ fatal (401/403/other 400)
                              └─► throw (no rotation)
        │
        ▼
all models exhausted ──► 502 "All configured OpenCode models failed"
```

The rotation pool is `fallbackModels` when it is non-empty; otherwise it is `models`. This is
a failover **within OpenCode Zen**, not a switch between unrelated providers.

---

## How Model Rotation Works

OpenCode Claude Bridge rotates **between models on the same OpenCode Zen upstream** when the
current model is temporarily unusable. Rotation is deliberate: it hides transient failures but
never hides a genuine request or authentication error.

A model is skipped and the next one is tried when the upstream returns:

| Status | Body condition | Rotates? | Reason |
|--------|----------------|----------|--------|
| `429` | — | ✅ | Rate limited. |
| `500`, `502`, `503`, `504` | — | ✅ | Temporary upstream failure. |
| `400` | body matches `model is unavailable` / `model not found` / `no allowed providers` / provider-routing errors | ✅ | Model unavailable or not routable right now. |
| `400` | other (bad JSON, invalid `reasoning_effort`, malformed request) | ❌ | Your request is wrong — rotate would just fail again. |
| `401`, `403` | — | ❌ | Auth failure — all models share the same key, so rotating is pointless. |
| other `4xx` | — | ❌ | Client error; do not retry. |

Notes:

- The rotation list is `fallbackModels` when non-empty, otherwise `models`
  (see [Configuration](#configuration)). For typical use, set `models` to your preferred
  order and leave `fallbackModels` empty.
- If every model in the list fails, the bridge returns `502` with
  `All configured OpenCode models failed` and the last error attached.
- Network errors / aborts also advance to the next model (or abort if the client disconnected).

---

## Requirements

- **Node.js >= 18** (uses built-in `fetch` and `AbortController`). No third-party packages.
- An **OpenCode Zen / OpenCode Go API key** (used as the Claude Code `ANTHROPIC_API_KEY`).
- **Claude Code** installed and configured to point at this bridge (see below).

---

## Quick Start

```bash
# 1. Clone or copy this folder
# 2. Copy the example config and adjust if needed
cp config.example.json config.json

# 3. Start the bridge
node server.js --config ./config.json

# 4. Configure Claude Code (see "Claude Code Integration" below)
```

Or use the platform launchers:

```bash
# Linux / macOS
./start.sh

# Windows
start.cmd
```

---

## Configuration

Configuration is read from `config.json` (or `--config <path>` / the
`CLAUDE_OPENCODE_PROXY_CONFIG` env var). Every value can be overridden by an environment
variable. See `config.example.json` for the full annotated schema.

### Essential options

| Key | Env var | Default | Notes |
|-----|---------|---------|-------|
| `listen.host` | `CLAUDE_OPENCODE_PROXY_HOST` | `127.0.0.1` | Bind host. Keep loopback unless you expose intentionally. |
| `listen.port` | `CLAUDE_OPENCODE_PROXY_PORT` | `8787` | 1–65535. |
| `upstream.baseUrl` | `CLAUDE_OPENCODE_PROXY_UPSTREAM_BASE_URL` | `https://opencode.ai/zen/v1` | `/v1` appended if missing. |
| `models` | — | configured list | Preferred rotation order. Use raw Go model IDs (e.g. `deepseek-v4-flash-free`, `kimi-k2.6`), **not** `opencode-go/...`. |
| `fallbackModels` | — | `[]` | If non-empty, **replaces** `models` as the rotation pool. Leave `[]` to rotate over `models`. |
| `reasoningContent` | `CLAUDE_OPENCODE_REASONING_CONTENT` | `auto` | `auto` = send to DeepSeek models only; `always`/`never` force it. |
| `reasoningCachePath` | `CLAUDE_OPENCODE_REASONING_CACHE` | `~/.claude/opencode-claude-bridge-reasoning-cache.json` | Supports `~`. |
| `upstreamTimeoutMs` | `CLAUDE_OPENCODE_UPSTREAM_TIMEOUT_MS` | `600000` | `0` disables the timeout. |

Set `CLAUDE_OPENCODE_LOG_USAGE=1` to print raw → translated token usage in the logs.

---

## Claude Code Integration

Configure Claude Code so its API base URL and key point at the bridge instead of Anthropic.
Add this to your Claude Code `settings.json` (or set the same environment variables):

```jsonc
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:8787",
    "ANTHROPIC_API_KEY": "YOUR_OPENCODE_API_KEY",
    "ANTHROPIC_MODEL": "deepseek-v4-flash-free",
    "ANTHROPIC_SMALL_FAST_MODEL": "deepseek-v4-flash-free",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-flash-free",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-flash-free",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash-free",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4-flash-free",
    "CLAUDE_CODE_EFFORT_LEVEL": "max",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "CLAUDE_CODE_ATTRIBUTION_HEADER": "0"
  }
}
```

Key variables:

- **`ANTHROPIC_BASE_URL`** — must be the bridge address (`http://127.0.0.1:8787`).
- **`ANTHROPIC_API_KEY`** — your **OpenCode Go** key. The bridge forwards it as a Bearer
  token to OpenCode Zen. It is never validated or logged locally.
- **`ANTHROPIC_MODEL` / `ANTHROPIC_*_MODEL`** — set every model alias to a DeepSeek V4
  (or other configured) Go model ID so Claude Code always routes through the bridge.
- **`CLAUDE_CODE_SUBAGENT_MODEL`** — keeps subagents on the same backend.
- **`CLAUDE_CODE_EFFORT_LEVEL`** — requested reasoning effort hint (see below).
- **`API_TIMEOUT_MS`** — raise it for long thinking sessions.

> **Security note:** The bridge binds to loopback (`127.0.0.1`) by default. Only expose
> it beyond that if you fully control the network and trust every client. Your OpenCode
> Go API key is sent upstream on every request; it is never stored or logged by the bridge.

---

## Thinking Level & Effort Mapping

The bridge gives Claude Code a native-looking **thinking** experience on DeepSeek V4 and
exposes the effort control end to end.

### How thinking is shown

When DeepSeek returns `reasoning_content`, the bridge emits Anthropic-compatible `thinking`
content blocks so Claude Code displays the reasoning inline. The same reasoning is cached
for later tool-call history replay. Anthropic `redacted_thinking` blocks are intentionally
not mapped (DeepSeek needs readable `reasoning_content`).

### Effort mapping

Claude Code can send Anthropic-format `thinking` (`enabled`/`disabled`) and
`output_config.effort` fields. The bridge translates them to DeepSeek/OpenAI-compatible
fields **for DeepSeek model names only**:

| Claude Code / Anthropic input | Sent to DeepSeek as `reasoning_effort` |
|-------------------------------|----------------------------------------|
| `none`, `disabled`, `no_think` | `no_think` |
| `low`, `minimal` | `low` |
| `medium`, `high`, `xhigh`, `max`, (anything else) | `high` |

- `thinking` `enabled`/`disabled` is passed through as the DeepSeek `thinking` extension.
- The bridge does **not** force thinking from `config.json` — per-session `/effort` stays
  owned by Claude Code. If Claude Code sends no `thinking` field, DeepSeek uses its own
  default (thinking is enabled by default; complex agent requests are typically treated as
  max-effort).
- `CLAUDE_CODE_EFFORT_LEVEL=max` asks Claude Code to request the highest available effort
  from the backend. It is a *hint*, not a guarantee: session state, `/effort`,
  `effortLevel`, and DeepSeek/OpenCode Zen normalization can all interact. Lower or remove
  it for faster responses.

Simple prompts may produce no visible thinking because the model emits no `reasoning_content`.

---

## Running the Bridge

```bash
# Direct
node server.js --config ./config.json

# Custom port via env
CLAUDE_OPENCODE_PROXY_PORT=9000 node server.js

# Background (Linux/macOS)
nohup node server.js --config ./config.json > bridge.log 2>&1 &

# Windows background
start /b node server.js --config ./config.json
```

---

## Autostart (Optional)

Install a background launcher so the bridge starts with your session:

```bash
# Linux
./scripts/install-autostart-linux.sh
./scripts/uninstall-autostart-linux.sh

# macOS
./scripts/install-autostart-macos.sh
./scripts/uninstall-autostart-macos.sh

# Windows (PowerShell)
./scripts/install-autostart-windows.ps1
./scripts/uninstall-autostart-windows.ps1
```

---

## Health & Endpoints

| Method & Path | Purpose |
|---------------|---------|
| `GET /health` | Bridge status, config path, listen/upstream URLs. |
| `GET /health?probe=upstream` | Also probes OpenCode Zen `/models` (requires a valid key). |
| `POST /shutdown` | Graceful shutdown (loopback clients only; flushes cache). |
| `GET /v1/models` | Lists configured `models`. |
| `POST /v1/messages` | Anthropic Messages API (translated to OpenCode Zen). |
| `POST /v1/chat/completions` | Raw OpenAI-compatible pass-through. |
| `OPTIONS *` | CORS preflight (`*`). |

```bash
curl http://127.0.0.1:8787/health
curl -H "x-api-key: YOUR_OPENCODE_API_KEY" "http://127.0.0.1:8787/health?probe=upstream"
```

---

## Reasoning Cache

The bridge persists `reasoning_content` to a single JSON file so thinking survives restarts
and is replayed on later tool-call turns. It is keyed three ways:

1. **Tool-call ID** — reasoning attached to a specific tool use.
2. **Assistant-text hash** — reasoning for a plain assistant message.
3. **Tool-context hash** — reasoning for the set of tool calls + assistant text in a turn.

Eviction respects `reasoningCacheMaxEntries`, `reasoningCacheMaxAgeMs`, and
`reasoningCacheMaxSizeBytes`. The cache is flushed on shutdown (SIGINT/SIGTERM) and
`/shutdown`.

---

## Project Structure

```
opencode-claude-bridge/
├── server.js                 # Main proxy server (zero dependencies)
├── config.json               # Your configuration (copy from config.example.json)
├── config.example.json       # Annotated example config
├── package.json              # npm metadata + scripts
├── start.sh                  # Linux/macOS launcher
├── start.cmd                 # Windows launcher
├── scripts/
│   ├── install-autostart-*.sh/.ps1   # Autostart installers per OS
│   ├── uninstall-autostart-*.sh/.ps1 # Autostart removers per OS
│   └── trim-reasoning-cache.js       # Manual cache trim utility
├── test/
│   └── server.test.js        # Built-in test suite (run: npm test)
├── LICENSE                   # MIT
└── .gitignore                # Ignores local caches & logs
```

---

## Development

```bash
# Syntax check
npm run check

# Run tests (Node native test runner)
npm test
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `reasoning_content must be passed back` | Ensure the bridge is running and its cache file is intact; don't delete it mid-session. |
| No visible thinking | Use a DeepSeek model; simple prompts may emit no reasoning. Non-DeepSeek models don't get the thinking extensions. |
| `Upstream API key is not set` | Set `ANTHROPIC_API_KEY` in Claude Code to your **OpenCode Go** key. |
| `All configured OpenCode models failed` | Check the model IDs in `config.json` (raw Go IDs, not `opencode-go/...`) and your key/quota. |
| Requests time out | Raise `upstreamTimeoutMs` and Claude Code's `API_TIMEOUT_MS`. |
| `413` on large payloads | Raise `requestBodyLimitBytes`. |

---

## Security Notes

- The bridge binds to **loopback (`127.0.0.1`)** by default. Only expose it beyond that if
  you fully control the network and trust every client — the `/shutdown` endpoint is
  restricted to loopback clients, but the proxy forwards your API key upstream.
- Your OpenCode Go API key is sent to OpenCode Zen on every request. It is never logged or
  stored by the bridge.
- The reasoning cache is written to your home directory (`~/.claude/...`). It contains model
  reasoning text from your sessions.

---

## References

- [DeepSeek thinking-mode guide](https://api-docs.deepseek.com/guides/thinking_mode) —
  `reasoning_content` behavior and thinking-mode tool-call history requirements.
- [OpenCode Zen](https://opencode.ai) — upstream provider.
- [DeepSeek API docs](https://api-docs.deepseek.com) — model and reasoning details.

---

## Limitations

- **Single upstream.** Rotation is failover *within* OpenCode Zen. There is no built-in
  multi-provider switch; pointing `upstream.baseUrl` at a different endpoint changes the
  target but does not combine providers.
- **Reasoning replay is best-effort.** It targets DeepSeek-style `reasoning_content`. Other
  OpenCode models may not emit it, in which case thinking simply won't appear.
- **`thinking` / `reasoning_effort` are DeepSeek-only extensions.** They are forwarded only
  when the model name looks like a DeepSeek model, so non-DeepSeek models keep the standard
  OpenAI shape.
- **Local proxy, not a hosted service.** The bridge runs on your machine and binds to
  `127.0.0.1` by default. Exposing it publicly is out of scope.
- **Upstream shape can change.** Translation depends on OpenCode Zen's OpenAI-compatible
  contract; upstream API changes may require a bridge update.

---

## Roadmap

- Per-model timeouts and backoff between rotation attempts.
- Optional provider plugins behind a stable adapter interface (without changing the core
  translation path).
- Richer health probe output (per-model reachability).
- Configurable rotation policy (strict vs. aggressive) via `config.json`.

These are ideas, not commitments — see [Contributing](#contributing) to shape them.

---

## Contributing

Pull requests and issues are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) first:
run `npm run check` and `npm test`, keep diffs focused, and never commit real API keys
(use the `YOUR_OPENCODE_API_KEY` placeholder). Security issues go through
[SECURITY.md](SECURITY.md), not public issues.

---

## OpenCode Attribution

OpenCode Claude Bridge is an **independent, community-maintained project**. It is **not** an
official OpenCode project and is not affiliated with or endorsed by the OpenCode maintainers.

[OpenCode Zen](https://opencode.ai) is a supported **upstream / platform dependency**: the
bridge translates Claude Code's Anthropic-format requests into OpenCode Zen's
OpenAI-compatible API and forwards them there. We mention OpenCode only where it is
technically relevant (configuration, routing, and upstream errors). All trademarks and
service names belong to their respective owners.

---

## License

MIT — see [LICENSE](LICENSE).