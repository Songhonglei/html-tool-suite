# Changelog

## v1.0.0 (open-source first release)

- Persistent FastAPI server exposing installed agent Skills as REST API endpoints (one endpoint per Skill)
- Sync / async execution with webhook callbacks (HMAC-signed, SSRF-guarded)
- Multi-engine executor with graceful fallback: OpenClaw → Claude Code SDK → Claude CLI → Codex CLI → LLM API
- Bilingual (EN/ZH) web management console: skill on/off, test runs, job history, logs, metrics, doctor
- HTTP by default, optional HTTPS with self-signed SAN certificates (`upgrade-to-https` one-liner)
- API Key auth (constant-time compare), expose whitelist + deny blacklist, `--no-docs` hardening
- Init wizard (8 steps) + non-interactive mode for agent environments
- Doctor self-check with `--fix` (TLS expiry, SAN mismatch, config drift)
