# ARC Bridge MCP

A self-hosted **MCP server** (spec **2025-11-25**, Streamable HTTP) that
exposes David's **ARC free-tier agent mesh** — the voice-assistant bridge he
runs on Alexa/Echo hardware — as MCP tools. Built for the Amazon
**Build, Ship, Shape** Developer Hackathon, **Alexa+ track**
(self-hosted MCP server).

**Zero dependencies.** Pure Python 3.9+ standard library. No `pip install`.

## What it does

Five tools plus one resource, over `POST /mcp`:

| Tool | What it really does |
|---|---|
| `arc_ask` | Ask the mesh a question. Local mode answers by retrieval over the memory corpus; live mode queues it in `arc_alexa_queue` for the worker (answers async, announces via Echo). |
| `arc_queue_status` | Read back a queued question's record (status, answer/digest). |
| `arc_memory_search` | Search the mesh's shared long-term memory (`arc_memory`). |
| `arc_health` | Worker liveness: heartbeat age + question-queue depth. |
| `arc_notify` | Fan out a notification: `log` (always), `echo` (Voice Monkey), `telegram` (Bot API), `ntfy` (ntfy.sh), `all`. Unconfigured channels fail closed with a reason. |
| `arc://mesh/health` | The health snapshot as a read-only resource. |

## Run it (60 seconds)

```bash
python3 server.py
# backend: LocalBackend
# auth: none (local mode -- bind to 127.0.0.1 ...)
# ARC Bridge MCP on http://127.0.0.1:8765/mcp
```

Local mode needs **no configuration**: it seeds clearly-labeled demo memory
and runs the full MCP lifecycle (initialize → tools → SSE → session close).

Try it with curl (replace `SID` with the `Mcp-Session-Id` from step 1):

```bash
# 1. initialize
curl -s -D - -X POST http://127.0.0.1:8765/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize",
       "params":{"protocolVersion":"2025-11-25","capabilities":{},
                 "clientInfo":{"name":"curl","version":"0"}}}'

# 2. list tools
curl -s -X POST http://127.0.0.1:8765/mcp -H 'Content-Type: application/json' \
  -H "Mcp-Session-Id: SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'

# 3. ask the mesh
curl -s -X POST http://127.0.0.1:8765/mcp -H 'Content-Type: application/json' \
  -H "Mcp-Session-Id: SID" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call",
       "params":{"name":"arc_ask","arguments":{"question":"How do voice replies work?"}}}'
```

## Live mode (the real mesh)

```bash
cp .env.example .env   # fill in values, never commit .env
set -a && source .env && set +a
ARC_BRIDGE_BACKEND=live python3 server.py
```

Live mode wires the same contracts the production mesh uses:

- `arc_ask` → `POST /rest/v1/arc_alexa_queue` (the worker's real queue)
- `arc_memory_search` → `GET /rest/v1/arc_memory`
- `arc_health` → `GET /rest/v1/arc_heartbeats` + queue depth
- `arc_notify` → Voice Monkey announce / Telegram Bot API / ntfy.sh

All credentials come from the environment (see `.env.example`). Nothing is
hardcoded; unconfigured channels fail closed.

## OAuth 2.1 account linking (for Alexa+)

```bash
ARC_BRIDGE_AUTH=oauth ARC_BRIDGE_PUBLIC_URL=https://mcp.example.com \
  python3 server.py
```

This fronts `/mcp` with a local OAuth 2.1 authorization server:

- `GET /.well-known/oauth-protected-resource` — RFC 9728 metadata
- `POST /oauth/register` — RFC 7591 dynamic client registration
- `GET /oauth/authorize` + `POST /oauth/approve` — consent flow (PKCE S256)
- `POST /oauth/token` — code exchange + refresh-token rotation

Unauthenticated `/mcp` requests get `401` with a `WWW-Authenticate`
challenge pointing at the metadata URL — the flow Alexa+ account linking
expects. Token stores are in-memory (single-instance self-hosting).

## Tests

```bash
python3 -m unittest discover -s arc_bridge/tests -v   # 15 tests
```

Covers: backend units (seed, ask, search, health, notify, validation),
full-stack HTTP (initialize → 5 tools → resource → session lifecycle),
and OAuth mode (401 challenge shape, metadata document). The transport and
OAuth internals have 14 more tests in the vendored harness's upstream
package (`../opensource-mini/mini-mcp-harness`).

## Layout

```
server.py                 entry point (env config -> build_app -> serve)
arc_bridge/
  harness.py              vendored mini-mcp-harness (Streamable HTTP + OAuth)
  backends.py             LocalBackend (file harness) / LiveBackend (mesh)
  tools.py                5 tools + 1 resource registration
  tests/                  unit + full-stack HTTP tests
data/                     local-mode JSONL files (gitignored, auto-created)
.env.example              every supported variable, documented
```

## Design notes

- **Stdlib only** is a deliberate choice: the entry runs anywhere Python
  exists, costs $0 forever, and can't break on an SDK major-version bump.
- **Fail-closed everywhere**: unknown sessions → 404, bad auth → 401 with
  the RFC 9728 challenge, unconfigured notify channels → explicit
  `{"ok": false, "detail": ...}`, never a faked success.
- **Origin allowlist** (`ARC_BRIDGE_ALLOWED_ORIGINS`) blocks DNS-rebinding
  from browsers; loopback is always allowed for local dev.
- **Every line of code carries a plain-English `#` comment** — the author
  can explain each line because each line says what it does.

## License

MIT — see `LICENSE`. The vendored `arc_bridge/harness.py` is
mini-mcp-harness v0.1.0 (same license, same author).
