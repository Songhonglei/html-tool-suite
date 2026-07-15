# agent-easy-http 设计方案

> 把 OpenClaw 的 **Agent 能力**通过 HTTP(S) + 内网 IP 暴露给其它系统调用。
> 核心理念：**最薄代理层** —— 自己只做 HTTP 转发 + 鉴权 + 安全加固，
> 把真正的执行与隔离完全交给 OpenClaw 原生 `/hooks/agent`。

> 📐 说明："v3.0" 是本 skill 的**内部架构代号**（基于 OpenClaw 原生 hooks 的
> 轻代理实现），与发布版本号是两个维度。本文以 v3.0 架构为准。

---

## 1. 一句话定位

agent-easy-http（下称 aeh）解决的是这样一个问题：

> 我在一台 OpenClaw 容器/Pod 上跑着一个 Agent，想让团队里其它服务
>（业务后端、脚本、浏览器）能通过一个内网 HTTP 地址给这个 Agent 发任务，
> 让 Agent 自己决定用什么 skill / 工具去完成。

跟 flash 不同，aeh 的调用方传的是**一句自然语言 `message`**，而不是结构化参数。
"让 Agent 决定怎么干"是它的核心场景。

---

## 2. v3.0 的核心决策：从"CLI 冷启动"到"原生 hooks"

aeh v2.0 曾用 `openclaw agent` CLI 方式执行——每个请求 embedded 冷启动约 90s，
而且多请求容易串进同一个 session。v3.0 彻底重构，改用 OpenClaw 原生
`/hooks/agent` 端点：

| 维度 | v2.0（CLI 冷启动）| **v3.0（原生 hooks）**|
|---|---|---|
| 执行引擎 | `openclaw agent` CLI（~90s 冷启动）| **`/hooks/agent`（毫秒触发）**|
| Session 隔离 | 🐛 多请求可能串入同一 session | ✅ **每请求自动 `hook:<uuid>`**|
| Job 管理 | 本地 JSON 持久化 + asyncio.Event | ❌ 删除（OpenClaw 自己管）|
| Callback HMAC | 自定义回调链路 | ❌ 删除（hooks fire-and-forget）|
| sub-agent 开销 | 每次冷启动 | ✅ 零开销 |

一句话：**aeh 不再自己启动 Agent，而是把请求转成 OpenClaw hook 事件，
由 Gateway 毫秒派发到一个独立隔离的 sub-agent session。**

---

## 3. 整体架构

```
   外部系统
      │  POST /agent/run   {"message": "帮我查下今天的天气"}
      │  （或 POST /skills/{name}/run，会自动注入 SKILL.md）
      ▼
 ┌──────────────── agent-easy-http（FastAPI 薄代理层）──────────────┐
 │  ① API Key 校验（X-API-Key）                                     │
 │  ② deny_skills 黑名单过滤（外层防火墙）                          │
 │  ③ Prompt 注入加固（分隔符 + 反注入指令包裹 message）            │
 │  ④ 生成 run_id（= hook session uuid 后缀）                       │
 │  ⑤ POST 到 OpenClaw /hooks/agent                                │
 │       - 自定义 sessionKey = hook:<uuid>                          │
 │       - 指定 default_agent_id / 调用方指定的 agent              │
 └───────────────────────────┬────────────────────────────────────┘
                             │ 毫秒返回 run_id（fire-and-forget）
      ◀──────────────────────┘
   {"success": true, "run_id": "6457...", "_openclaw": {...}}

 ┌──────────────── OpenClaw Gateway（真正执行方）──────────────────┐
 │  在 agent:main:hook:<uuid> 独立 session 里跑 Agent              │
 │  自动加 EXTERNAL_UNTRUSTED_CONTENT 包装（内层防护）              │
 │  Agent 自行决定调什么 skill / 工具                              │
 └─────────────────────────────────────────────────────────────────┘

   外部系统随后轮询：
      GET /result/{run_id}
      ▼
   aeh 从 session JSONL 提取 Agent 完整输出 → {"status":"done","output":...}
```

**aeh 自己不执行任何 Agent 逻辑**，它是一层"翻译 + 加固 + 追踪"的代理。

---

## 4. 两个入口

| 入口 | 调用方传入 | 用途 |
|---|---|---|
| `POST /agent/run` | 只有 `message` | 通用入口，Agent 自己决定干啥（"查 X 数据"）|
| `POST /skills/{name}/run` | `message` + 自动注入该 skill 的 SKILL.md | 引导 Agent 优先用指定 skill |

两者都返回毫秒级的 `run_id`，Agent 在后台跑，用 `run_id` 去 `/result/{run_id}` 拿结果。

---

## 5. run_id 追踪机制

因为 hooks 是 fire-and-forget（发了就返回，不等结果），aeh 需要一套自己的追踪：

1. 请求进来时 aeh **自生成 run_id**（= 它给这次 hook 指定的 session uuid 后缀）。
2. hook 派发到 `agent:main:hook:<uuid>` session 执行。
3. 客户端拿 run_id 轮询 `GET /result/{run_id}`。
4. aeh 去读对应 session 的 JSONL 文件，提取 Agent 的完整回复。

实测时序：触发 → 约 4~6s session 文件可读 → `/result` 返回 `done`。
建议客户端 4s 后首次查询，之后每 1.5s 轮询一次。

```json
// GET /result/{run_id}
{
  "status": "done",        // pending | done | not_found
  "run_id": "6457f48c-...",
  "output": "Agent 的完整回复文本",
  "messages": [{"role":"user","text":"..."}, {"role":"assistant","text":"..."}]
}
```

---

## 6. 安全模型（七层）

aeh 是"把 Agent 暴露到网络"，安全是重中之重，分七层防御：

| 层 | 防御 |
|---|---|
| **传输** | HTTP 默认 / HTTPS 可选（TLS + 自签 SAN 证书；`0.0.0.0` 跨主机场景强烈建议启用）|
| **接入** | API Key（`X-API-Key`，32 字符随机，强制）|
| **暴露** | `deny_skills` 黑名单（推荐拒 `finclaw-ai`/`hi-send` 等有副作用的 skill）|
| **Agent 路由** | `allowed_agent_ids` 白名单控制调用方可指定的 agent 范围 |
| **Prompt 注入** | 分隔符 + 反注入指令包裹用户 message |
| **Session 隔离** | OpenClaw `/hooks/agent` 自动 `hook:<uuid>` 每请求独立 |
| **OpenClaw 内层** | Gateway 自动给 hook 消息加 `EXTERNAL_UNTRUSTED_CONTENT` 包装 |

设计取舍：aeh **默认全开放**（所有 skill 都能被调），靠"API Key + 黑名单 +
注入加固 + session 隔离"兜底。强烈建议至少把发外部消息、改系统部署这类
有副作用的 skill 加进 `deny_skills`。

> ⚠️ `GET /agents` 会返回内部所有 agent 的 ID/名称（需鉴权）。API Key 泄露时
> 外部可枚举 agent。不需要动态查询时，建议把 `allowed_agent_ids` 设为固定
> 白名单而非 `["*"]`，缩小攻击面。

---

## 7. OpenClaw hooks 自动启用

aeh v3.0 依赖 OpenClaw 的 `/hooks/agent`，需要 4 项 Gateway 配置：

| 配置项 | 值 | 用途 |
|---|---|---|
| `hooks.enabled` | `true` | 启用 hooks 端点 |
| `hooks.token` | `<32 字符随机>` | 鉴权 token |
| `hooks.allowRequestSessionKey` | `true` | 允许调用方自定义 sessionKey |
| `hooks.allowedSessionKeyPrefixes` | `["hook:"]` | 限制前缀，安全兜底 |

init wizard 自动完成全部配置，并**同步写多个 config 路径**（防配置中心覆盖），
最后热加载 + ping 验证。用户通常不用手动配 hook_url/token——wizard 会从
`openclaw.json` 自动推导。

hook 端点解析优先级：`env` > `config.json` > 自动从 `openclaw.json` 推导。

---

## 8. 自愈守护（watchdog）

aeh 自带 watchdog 脚本，每 30s 检查一次：

- server 进程挂了 → 自动重启。
- 检测到任一 hooks 配置缺失（如被配置中心覆盖）→ 自动调 init_wizard 修复。

容器场景推荐前台跑 `watchdog.sh run`，也支持后台 start/stop/status。
`start` 时默认会自动拉起 watchdog（可用 `AGENT_EASY_HTTP_NO_WATCHDOG` 跳过）。

---

## 9. 数据路径（workspace 内，PVC 安全）

```
<workspace>/.agent-easy-http/     # 本 skill 私有数据
  config.json
  server.pid / server.port
  logs/server.log / watchdog.log

<workspace>/.http/                # 三个 HTTP skill 共享
  certs/server.{crt,key}          # TLS（SAN 支持多 IP）
  secrets/api-keys/agent-easy-http.key
```

所有持久化都放在 workspace 内（容器/K8s 环境下走 PVC，重启不丢），可用
`OPENCLAW_HTTP_ROOT` / `AGENT_EASY_HTTP_DATA_ROOT` 覆盖。跑
`python3 scripts/server.py paths` 一键查所有路径。

---

## 10. 关键工程决策回顾

| 决策 | 选择 | 理由 |
|---|---|---|
| 执行方式 | 转发到原生 `/hooks/agent` | 毫秒派发 + 零冷启动 + 自动 session 隔离 |
| 自己不管 Job | 是 | 交给 OpenClaw，代理层保持极薄 |
| 回调方式 | fire-and-forget + run_id 轮询 | 不阻塞 HTTP 连接，超时语义清晰 |
| 默认全开放 + 黑名单 | 是 | 易用优先，靠多层安全兜底 |
| 默认 HTTP | 是 | 零门槛先跑通，按需升 HTTPS |
| hooks 配置多路径同步 | 是 | 防配置中心（如 Apollo）覆盖 |
| watchdog 自愈 | 是 | 容器环境下配置易被覆盖，需要自动修复 |

---

## 11. 与另外两剑客的关系

- **agent-easy-http（本篇）**：给 Agent 发一句自然语言任务，让 Agent 自己决定
  怎么干。最薄代理层，仅 OpenClaw。
- **skill-to-http-flash**：单 skill、结构化入参、subprocess 直执行、可 standalone。
- **skill-to-http**：多 skill 统一网关 + 控制台 + 多执行引擎降级，企业级一整套。

三者互补，详见 [HTTP 三剑客对比](./http-trio.md)。
