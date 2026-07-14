# agent-easy-http

> Expose your OpenClaw agent as an HTTP(S) REST API over your network IP — thin proxy over OpenClaw's native `/hooks/agent` with millisecond dispatch and per-request session isolation.

Let other systems on your network call your OpenClaw agent to run tasks, e.g.
`POST http://<your-server-ip>:7720/agent/run`. HTTP by default (zero-friction);
optional HTTPS with self-signed SAN certificates for cross-host / production.

## Features

- **Thin proxy over `/hooks/agent`** — no embedded agent cold-start; millisecond trigger
- **Per-request isolation** — every call runs in its own `hook:<uuid>` session
- **Two entry points** — `POST /agent/run` (open-ended prompt) + `POST /skills/{name}/run` (targeted skill)
- **Result polling** — `GET /result/{run_id}` returns `pending` / `done` / `not_found` + full transcript
- **Mandatory API Key auth** (`X-API-Key`) + deny/expose skill lists (outer firewall)
- **Prompt-injection hardening** — untrusted input wrapped with boundary + anti-injection instructions
- **Multi-agent routing** — `default_agent_id` / `allowed_agent_ids` whitelist
- **HTTP by default, optional HTTPS** with self-signed SAN certs
- **Self-healing watchdog** — PID + `/health` + hook-endpoint checks, auto-recovers within 30s
- **PVC-safe persistence** in workspace + env-var overrides (container friendly)

## Quick Start

```bash
cd <your-workspace>/skills/agent-easy-http

# Install deps
pip install fastapi uvicorn pydantic httpx

# Interactive init (enables OpenClaw hooks + generates API Key; HTTP mode by default)
python3 scripts/server.py init

# Start (defaults to http://0.0.0.0:7720)
python3 scripts/server.py start
```

Call it:

```bash
API_KEY=$(cat <workspace>/.http/secrets/api-keys/agent-easy-http.key)
BASE=http://<your-server-ip>:7720

RESP=$(curl -s -H "X-API-Key: $API_KEY" -H "Content-Type: application/json" \
       -d '{"message":"summarize my week"}' $BASE/agent/run)
RUN_ID=$(echo "$RESP" | jq -r .run_id)

sleep 4
curl -s -H "X-API-Key: $API_KEY" "$BASE/result/$RUN_ID" | jq -r .output
```

## Usage

Full docs in [SKILL.md](./SKILL.md); deployment guide (HTTPS, systemd, Nginx, Docker/K8s,
FAQ) in [references/deployment.md](./references/deployment.md).

## Install in your AI agent

| Agent | Install |
|---|---|
| OpenClaw | `clawhub install agent-easy-http` |
| Claude Code | Manual: copy to `~/.claude/skills/` |
| Cursor | Manual: copy to `.cursor/skills/` |

## Requirements

- `python3` ≥ 3.8 with `fastapi`, `uvicorn`, `pydantic`, `httpx`
- `openssl` (only for HTTPS / cert generation)
- A running OpenClaw gateway with native `/hooks/agent` (the init wizard enables it for you)

## Part of

This skill ships in the [`html-tool-suite`](https://github.com/Songhonglei/html-tool-suite)
repo under `skills/agent-easy-http/`.

## License

MIT (see [LICENSE](./LICENSE))

## Author

Evan Song · [github.com/Songhonglei](https://github.com/Songhonglei)
