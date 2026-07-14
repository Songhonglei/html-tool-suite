# html-tool-suite

A suite of skills for **HTML generation, publishing, and sharing** — built for AI agents
(OpenClaw, Claude Code, Cursor, …).

> Sibling suite: [`better-agent-skills`](https://github.com/Songhonglei/better-agent-skills)
> (workspace / memory / cross-agent tooling) ·
> [`build-better-skills`](https://github.com/Songhonglei/build-better-skills)
> (skill engineering lifecycle).

---

## Stages

| Stage | Skill | Description |
|---|---|---|
| Sharing | [`pinme-share`](./skills/pinme-share) | Upload any file/dir to PinMe (public IPFS) → short shareable URL |
| Serving | [`agent-easy-http`](./skills/agent-easy-http) | Expose your OpenClaw agent as an HTTP(S) REST API over your network IP |

*More skills coming as the suite grows.*

---

## Install a skill

| Agent | Install |
|---|---|
| **OpenClaw** | `clawhub install <skill-name>` (e.g. `clawhub install pinme-share`) |
| **Claude Code** | `cp -r skills/<skill-name> ~/.claude/skills/` |
| **Cursor** | `cp -r skills/<skill-name> .cursor/skills/` |
| **Manual** | Copy the entire `skills/<skill-name>/` directory into your agent's skills folder |

Each skill is **self-contained** — no cross-skill imports, no shared runtime.

---

## License

[MIT](./LICENSE) © 2026 Evan Song

## Author

Evan Song · [github.com/Songhonglei](https://github.com/Songhonglei)
