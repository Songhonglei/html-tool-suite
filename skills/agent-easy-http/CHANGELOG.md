# CHANGELOG

## v1.0.1 (open-source, suite integration)

- 📦 并入 [`html-tool-suite`](https://github.com/Songhonglei/html-tool-suite) 套件（`skills/agent-easy-http/`）
- 📝 Repository URL 指向 suite repo；README 增加 "Part of" 说明
- （代码无变化，仅文档/归属对齐）

## v1.0.0 (open-source first release)

首个开源发布版本。基于 OpenClaw 原生 `/hooks/agent` 的轻代理架构（内部架构代号 v3.0）。

**核心能力**
- 把 OpenClaw agent 能力通过 HTTP(S) + 网络 IP 暴露为 REST API
- 默认 HTTP 零门槛模式；可选 HTTPS + 自签 SAN 证书
- API Key 强制鉴权（`X-API-Key`）
- deny_skills 黑名单（外层防火墙）+ expose_skills 白名单
- Prompt 注入加固（分隔符 + 反注入指令包裹外部输入）
- 每请求自动 `hook:<uuid>` session 隔离
- `POST /agent/run`（通用入口）+ `POST /skills/{name}/run`（指定 skill）
- `GET /result/{run_id}` 查询 agent 执行结果（pending / done / not_found + 完整对话）
- 多 agent 路由（`default_agent_id` / `allowed_agent_ids` 白名单）
- watchdog 自愈守护：PID 存活 + `/health` 探活 + hook 端点配置双重检查，30s 内自愈
- workspace 内持久化（PVC 安全）+ 环境变量覆盖（容器友好）

> 本版本源自内部实践迭代，开源时已剥离内部专属路径 / 配置中心引用 / 内部 skill 示例，
> 并将配置同步路径通用化为 `OPENCLAW_CONFIG_SYNC_PATHS` 环境变量（默认空）。
