# skill-to-http

> Expose all your installed agent Skills as HTTP(S) REST API services — one persistent FastAPI server, one endpoint per Skill, with a bilingual web management console.

`skill-to-http` runs a long-lived FastAPI server that scans your skill directories and auto-generates a REST endpoint for every Skill:

```
POST /skills/{name}/run        →  executes the Skill via a sub-agent engine
POST /skills/{name}/run/async  →  job id + webhook callback (HMAC-signed)
GET  /skills                   →  list exposed skills
```

Execution is delegated to a **multi-engine executor with graceful fallback**: OpenClaw → Claude Code SDK → Claude CLI → Codex CLI → plain LLM API. Whatever is available in your environment gets used automatically.

**Part of the HTTP serving trio** (all in [html-tool-suite](https://github.com/Songhonglei/html-tool-suite)):

| Tool | Granularity | Runtime execution |
|---|---|---|
| [agent-easy-http](https://github.com/Songhonglei/html-tool-suite/tree/main/skills/agent-easy-http) | Whole agent as one API | agent hook dispatch |
| **skill-to-http** (this) | All skills, one endpoint each | sub-agent engines w/ fallback |
| [skill-to-http-flash](https://github.com/Songhonglei/html-tool-suite/tree/main/skills/skill-to-http-flash) | One skill, one microservice | subprocess, zero LLM |

## Features

- **One endpoint per Skill** — auto-scans multiple skill dirs (`~/.openclaw/workspace/skills`, `~/.claude/skills`, `~/.cursor/skills`, custom via `SKILL_HTTP_SKILL_DIRS`), nested layouts supported
- **Sync + async execution** — async jobs with TTL cleanup, webhook callbacks (HMAC-signed, SSRF-guarded, retry with backoff)
- **Multi-engine executor** — OpenClaw / Claude Code SDK / Claude CLI / Codex CLI / LLM fallback, auto-detected, per-request override
- **Bilingual web console (EN/ZH)** — skill on/off toggles, test runs, job history, logs, metrics, TLS cert management, doctor self-check; one-click language switch
- **HTTP by default, HTTPS on demand** — self-signed SAN certificates, `upgrade-to-https` one-liner, cert renew/import
- **Security hardening** — API Key auth (constant-time compare), expose whitelist + deny blacklist, `--no-docs` mode, API key via env (not argv)
- **Init wizard** — 8-step interactive setup, or `--non-interactive` one-liner for agent environments
- **Doctor** — self-check with `--fix` (TLS expiry, SAN mismatch, config drift, executor availability)

## Quick Start

```bash
# Install dependencies
cd /path/to/skill-to-http
pip install -r requirements.txt

# Start (must run from scripts/ dir)
cd scripts
python3 server.py
```

First launch runs an 8-step init wizard (TLS / API Key / expose scope / deny list). Config lands in `<workspace>/.skill-to-http/config.json` (or `~/.skill-to-http/config.json` fallback).

One-liner for agent environments (skip wizard, defaults):

```bash
python3 server.py --non-interactive --expose-skill "*"
```

Then open the API docs at `http://localhost:8080/docs`, and the management console:

```bash
bash start-console.sh        # console at http://0.0.0.0:9000
```

## Usage

Full documentation (executors, config reference, env vars, HTTPS, security notes) in [SKILL.md](./SKILL.md).

## Install in your AI agent

| Agent | Install |
|---|---|
| OpenClaw | `clawhub install skill-to-http` |
| Claude Code | Manual: copy to `~/.claude/skills/` |
| Cursor | Manual: copy to `.cursor/skills/` |
| Others / standalone | Clone this repo and copy `skills/skill-to-http/`; set `SKILL_HTTP_SKILL_DIRS` |

## License

MIT (see [LICENSE](./LICENSE))

## Author

Evan Song · [github.com/Songhonglei](https://github.com/Songhonglei)

## Changelog

See [CHANGELOG.md](./CHANGELOG.md).
