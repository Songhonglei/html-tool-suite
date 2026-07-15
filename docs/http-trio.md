# HTTP 三剑客 — agent-easy-http / skill-to-http-flash / skill-to-http

> 把 Agent / Skill 的执行能力暴露为 HTTP API 的三个互补 skill。

核心目标——**把 OpenClaw 的 Agent / Skill 执行能力暴露为 HTTP API**，三剑客分别适用不同场景。

# 一句话定位

- [**agent-easy-http**](../skills/agent-easy-http)**  **   —  **极简 Agent 网关**：HTTP 给 Agent 发一句话，原生 hook 毫秒执行 （仅OpenClaw)

- [**skill-to-http-flash**](../skills/skill-to-http-flash)  — **Skill 微服务工厂**：把单个 Skill直执行变稳定的 REST API，毫秒冷启 + standalone（支持非OpenClaw)

- [**skill-to-http **](../skills/skill-to-http)**         **   —  **全能 Skill 网关**：多 Skill 统一服务 + 控制台 + 多执行引擎降级 （支持非OpenClaw)

# 功能说明

## 三剑客关系

- **skill-to-http** 是旗舰，覆盖最全（多 skill / 多引擎 / 控制台 / 历史 / Docker），适合"对内部署一套统一 HTTP 网关"

- **skill-to-http-flash** 为基于"Python 脚本 + JSON Schema" 的REST API直执行工厂，**完全 standalone**（不依赖 OpenClaw / Gateway / LLM 任何链路），适合"我想把这一个 skill 挂出去 + 在任何 Python 机器都能跑"

- **agent-easy-http**  「最薄代理层」，把执行隔离完全交给 OpenClaw 的 /hooks/agent，自身只做 HTTP 转发 + 鉴权 + deny 注入，适合"我只要 HTTP 入口能调 Agent 就行"

> 三者**互补**，覆盖从「一行命令把 Agent 挂上 HTTP」到「企业级多 Skill 全功能网关」的完整频谱。

## 选型建议

> **skill-to-http（简称 s2h）**
> **skill-to-http-flash（简称 flash）**
> **agent-easy-http（简称 aeh / easy）**

> **新手快速选**：只想 Agent 发消息任何 → **agent-easy-http**；
>
> 只想把一个 skill转成HTTP、或者在非openclaw使用 → **skill-to-http-flash**；
>
> 要管一整套 skill 到HTTP的管控 → **skill-to-http**。

| **场景** | **推荐** | **原因** |
| --- | --- | --- |
| 只想**给 Agent 远程喂任务**，不限特定 Skill | **agent-easy-http** | 原生 hook 毫秒派发，最轻量 |
| 每次**只暴露 1 个 Skill**，追求极简启动 | **skill-to-http-flash** | subprocess 直执行，毫秒冷启 |
| 需要**脱离 OpenClaw 独立部署** Skill 服务 | **skill-to-http-flash** | standalone 一等公民，任何 Python 机器可跑 |
| 需要暴露**多个 Skill 统一管理** + 控制台 | **skill-to-http** | 多 Skill 统一网关 + Web UI 控制台 + 反向黑名单 |
| 需要 Skill 走 **CC / Codex / LLM 执行引擎** | **skill-to-http** | 5 引擎自动检测降级 |
| CI/CD 流水线 / Docker 部署 | **skill-to-http** | 正式 Dockerfile + HEALTHCHECK |
| **浏览器前端 HTTPS 调用** | **all** | 都自带 CORS + HTTPS 自签证书 |
| 强 API 鉴权 + 反向黑名单 | **agent-easy-http** 或 **skill-to-http** | 强制 API Key + deny 拦截危险 skill |
| 异步任务需 **webhook 回调** | **agent-easy-http** 或 **skill-to-http** | flash 仅轮询 |
| 异步任务历史可查（重启不丢） | **flash**（JSONL）或 **skill-to-http**（SQLite） | aeh 无历史 |
| **宿主机自启 / 崩溃自重启** | **skill-to-http-flash** | flash.py systemd --user 自动生成 unit |
| **客户端要拿固定 schema 入参 / envelope 出参** | **skill-to-http-flash** | 统一响应 envelope，错误码标准化 |

## 功能对比总览

| **维度** | **agent-easy-http**  **（全科医生）** | **skill-to-http-flash v2.0**  （专科医院） | **skill-to-http**  （三甲医院） |
| --- | --- | --- | --- |
| 定位 | 极简 Agent 代理 | 单 Skill subprocess 直执行 + standalone | 多 Skill 全能网关 |
| 服务模型 | Agent 的轻量 HTTP 代理层 | 单 Skill 独立微服务（一对一） | 多 Skill 统一服务 + 管理 |
| 执行链路 | /hooks/agent 原生派发 | **subprocess.run 直执行**（不依赖 Gateway/LLM） | 5 引擎降级（OpenClaw/CC/Codex/Claude CLI/LLM） |
| 执行入参 | 自然语言任务描述 | **JSON → 长 flag CLI**（{"foo":"x"} → --foo x） | 运行时动态生成（LLM 解析 SKILL.md） |
| 响应 envelope | 任务结果 + run_id | **统一 envelope**（success/exit_code/elapsed_ms/data|output/stderr/truncated） | 同步：结果；异步：job_id |
| 启动开销 | **毫秒级**（原生 hook 派发） | **毫秒级**（subprocess 直执行） | 0.5s ~ 5s（按引擎） |
| Standalone | OpenClaw 内 | ✅ **一等公民**（--skill-dir/--data-dir flag + FLASH_SKILL_DIR/FLASH_DATA_DIR env） | OpenClaw / CC / Claude CLI / Codex |
| 默认 TLS | **HTTP**（零门槛，按需 HTTPS） | **HTTP**（零门槛，按需 HTTPS） | **HTTPS**（init wizard 推 self-signed；可选 HTTP） |
| 认证 | **强制 API Key**（X-API-Key） | 可选 API Key（X-API-Key） | API Key（X-API-Key）+ 反向黑名单 |
| 反向黑名单 | ✅（deny_skills 注入到 prompt） | ❌（单 Skill 无此需求） | ✅（config.deny_skills） |
| 管理控制台 | 无 | 无——服务即端点 | ✅ 独立 Web UI |
| 同步 timeout | 默认 60s | **默认 60s**，超时自动截断 | 可配 |
| 输出截断 | 无 | **同步 512 KB 截断 / 异步不截断** | 可配 |
| 异步执行 | 支持（webhook 回调） | 支持（轮询，无 webhook）+ JSONL 持久化 | 支持（webhook + 轮询）+ SQLite 持久化 |
| 历史持久化 | 无 | JSONL append-only + logrotate（重启不丢） | SQLite（7 天自动清理） |
| 错误码标准化 | 基础 | **7 类 error_type**（validation_failed/entry_not_found/spawn_failed/timeout/rate_limited/internal_error/exit_code≠0） | 基础 |
| CORS | 默认 * | 默认 *（可 env 收紧） | 默认 *（可 config 收紧） |
| systemd | 无 | ✅（flash.py systemd 子命令） | 无 |
| Docker | 无 | 无 | ✅（Dockerfile + HEALTHCHECK） |
| 持久化目录 | <workspace>/.agent-easy-http/ | <workspace>/.skill-to-http-flash/ | <workspace>/.skill-to-http/ |
| 共用证书目录 | <workspace>/.http/certs/ | 同左（flash-<skill>/ 子目录隔离） | 同左 |
| 核心依赖 | fastapi + uvicorn + pydantic + httpx | fastapi + uvicorn + cryptography + pydantic + jsonschema | fastapi + uvicorn + cryptography + pydantic + starlette + (claude_agent_sdk 可选) |
| 上线门槛 | 低 | **最低**（standalone 任何机器可跑） | 中 |

# 用户使用说明

> 适合「拿到任务想立刻动手」的同学。三个 skill 的安装入口都一样：对当前 agent 说「装一下 xxx」 或者手工在Skill Hub搜索安装。

## agent-easy-http（给 Agent 发任务的 HTTP 入口）

**适合场景**：你想用任何能发 HTTP 请求的工具（curl / 浏览器 / 业务后端）远程喂一句话给 Agent 跑。

**3 步用起来**：

1. **安装并启动**

  - 跟当前 agent 说「 agent-easy-http 启动服务」

- 启动成功后会返回服务地址（形如 `http://10.x.x.x:7720）和` API Key

  - 自带 watchdog 自愈，挂了会自己拉起来，平时不用管

2. **调用**

```bash
curl -X POST http://10.x.x.x:7720/agent/run \
  -H "X-API-Key: 你的 key" \
  -H "Content-Type: application/json" \
  -d '{"message": "帮我跑下 hello-env 看看环境信息"}'
# 立即返回 {"run_id": "xxx-xxx", ...}
```

3. **查结果**

```bash
curl http://10.x.x.x:7720/result/你的run_id \
  -H "X-API-Key: 你的 key"
# 返回 pending / done + agent 完整输出
```

**装好后日常用法**——只要跟当前 agent 说人话即可，不用记命令：

| 你要做的事 | 跟 agent 这么说 |
| --- | --- |
| 查服务状态 | "agent-easy-http 在跑吗 / 服务还活着吗" |
| 停止服务 | "把 agent-easy-http 停了" |
| 重启服务 | "重启一下 agent-easy-http" |
| 看证书、API Key、config 在哪 | "agent-easy-http 的文件都放哪了" |
| 看现在能调到哪些 skill | "agent-easy-http 现在能调哪些 skill" |
| 看默认 agent 路由 | "agent-easy-http 默认路由到哪个 agent" |
| **白名单**（只放出指定 skill） | "agent-easy-http 只暴露 hello-env 和 hub-skill-query" |
| **黑名单**（拉黑危险 skill） | "agent-easy-http 把 html-go-live 和 skill-release-plus 拉黑" |
| 清空白名单恢复全暴露 | "agent-easy-http 把 skill 暴露范围放开" |
| 热加载新配置不重启 | "agent-easy-http reload 一下配置" |

> **设计哲学**：agent-easy-http 默认全开放（所有 skill 都能被调），靠 **HTTPS + API Key + 反向黑名单** 兜底。建议至少把发外部消息（finclaw-ai / hi-send）、改系统部署（html-go-live / skill-release-plus）这类有副作用的 skill 拉黑。

---

## skill-to-http-flash（把一个 Skill 挂成独立 HTTPS 微服务）

**适合场景**：你有一个 Python skill（带 JSON Schema 入参），想把它变成一个稳定的 REST 接口给浏览器 / 业务后端调用，最好还能脱离 OpenClaw 在普通服务器上跑。

**3 步用起来**：

1. **创建服务**

  - 跟当前 agent 说「用skill-to-http-flash，给 xxx-skill 创建一个 HTTP 服务」

  - Skill会拉一个独立微服务进程，给你返回服务地址 + API Key + 入参 schema

1. **看你的接口长什么样**

```bash
curl http://localhost:7780/schema
# 返回这个 skill 的入参 JSON Schema，告诉你哪些字段必填、什么类型
```

2. **调用**

```bash
curl -X POST http://localhost:7780/run \
  -H "X-API-Key: 你的 key" \
  -H "Content-Type: application/json" \
  -d '{"date": "2026-05-30", "chart_id": "abc"}'
# 返回统一 envelope: success / exit_code / elapsed_ms / data 或 output
```

**装好后日常用法**——同样跟当前 agent 说人话即可：

| 你要做的事 | 跟 agent 这么说 |
| --- | --- |
| 看我装了哪些 flash 服务 | "我现在有哪些 flash 服务在跑" |
| 给某个 skill 起服务 | "把 hello-env 用 flash 起一下" |
| 看某个服务状态 | "hello-env 的 flash 服务还在吗" |
| 停止某个服务 | "把 hello-env 的 flash 服务停了" |
| 重启某个服务 | "重启 hello-env 的 flash 服务" |
| skill 升级了重新生成 | "hello-env 升级了，flash 重新生成一下" |
| 移除某个服务 | "把 hello-env 的 flash 服务删了" |
| 开机自启 | "让 hello-env 的 flash 服务开机自启" |
| 看接口长什么样 | "hello-env 的 flash 接口入参是什么" |
| 看历史调用记录 | "hello-env 的 flash 服务最近调了哪些" |
| 重生证书 | "hello-env 的 flash 证书重新生成一下" |

> **设计哲学**：flash 一个服务只挂一个 skill，没有白/黑名单概念——要管控就控制谁能拿到 API Key 即可。

**Standalone 跑法**（任何能跑 Python 的机器都行，不用装 OpenClaw）：

```bash
export FLASH_SKILL_DIR=/你的/skills/目录
export FLASH_DATA_DIR=/你的/数据/目录
python3 flash.py create --skill 你的skill名
```

---

## skill-to-http（多 Skill 统一网关 + Web 控制台）

**适合场景**：你要对外提供一整套 skill 服务（不止一个），希望有统一的管理后台、历史记录、多种执行引擎降级（OpenClaw / CC / Codex / Claude CLI / LLM 任选）。

**3 步用起来**：

1. **安装 + 启动**

  - 跟当前 agent 说「帮我装 skill-to-http 并启动」或在SkillHub直接安装

  - Skill会启动一个 HTTPS 网关 + Web 控制台，返回访问地址和 API Key

  - 默认走 HTTPS（自签证书，首次访问浏览器会提示信任一下）

2. **打开控制台**：浏览器访问启动时返回的地址，可视化看所有 skill、最近执行历史、运行日志，也能直接点按钮调用。

3. **API 调用**

```bash
# 列出当前服务暴露的所有 skill
curl https://localhost:8080/skills -H "X-API-Key: 你的 key"

# 同步执行某个 skill
curl -X POST https://localhost:8080/skills/你的skill名/run \
  -H "X-API-Key: 你的 key" \
  -H "Content-Type: application/json" \
  -d '{"参数": "值"}'

# 异步执行 + webhook 回调
curl -X POST https://localhost:8080/skills/你的skill名/run/async \
  -H "X-API-Key: 你的 key" \
  -d '{"参数": "值", "webhook_url": "https://你的回调地址"}'
```

**调试**：skill-to-http 自带控制台，直接在控制台里点按钮跑。

**Docker 部署**：skill 内自带 Dockerfile + HEALTHCHECK，CI/CD 流水线可直接拿来用。

---

# 技术使用说明

## 核心架构差异

三个 Skill 的差异核心在「**谁来执行、怎么执行**」。

### agent-easy-http （薄代理层 + 原生 hook）

```plaintext
外部请求 → FastAPI 代理 → OpenClaw 原生 /hooks/agent → 自动派发 sub-agent（hook:<uuid> 隔离）→ webhook/轮询拿结果
```

- 委托给 OpenClaw 原生 /hooks/agent 端点

- 启动开销**毫秒级**（旧版 embedded CLI 冷启动 ~90s，v3.0 已淘汰）

- 每次请求自动落到独立 hook:<uuid> session，天然隔离

- 适合"远程喂一句任务给 Agent 执行"的极简场景

- v3.0 起代码量约 1100 → 1200 行（server.py + 3 辅助脚本约 2600 行）

### skill-to-http-flash v2.0 （subprocess 直执行 + standalone）

```plaintext
外部请求 → 独立 FastAPI 微服务 → subprocess.run([python3, scripts/main.py, --foo, value, ...]) → 统一 envelope 返回
```

- **subprocess 直调 skill 入口脚本**，不走 agent / LLM / Gateway / hooks 任何链路

- 一对一编译：每个 Skill 一个独立服务，入口在 frontmatter flash.entry 声明

- **standalone 一等公民**：任何能跑 Python 的机器都能跑，不依赖 ~/.openclaw

- 入参 JSON → 长 flag CLI 映射（{"foo_bar":"x"} → --foo-bar x）

- 出参：自适应 envelope（stdout 是 JSON 走 data 字段 / 否则走 output 字段）

- 7 类标准 error_type，业务失败用 envelope 表达（HTTP 200 + success=false）

- systemd 子命令一键生成 unit；JSONL 持久化 jobs（重启不丢）

- v2.0 **不兼容 v1.x**，老 project 一律 recreate

### skill-to-http  （Agent 全能网关 + 多引擎降级）

```plaintext
外部请求 → FastAPI 网关 → 5 引擎自动检测降级 → 执行 → 返回
```

- 5 引擎降级：OpenClaw（~5s） / CC SDK（~0.5s，最快） / Claude CLI（~2.5s） / Codex CLI（~3s） / 纯 LLM（无工具，仅自包含 skill）

- 唯一支持脱离 OpenClaw 环境独立部署的 skill（cc / codex / llm 模式）

- 多 Skill 统一管理 + Web UI 控制台 + SQLite 历史持久化

- v1.2.0 起加 config.deny_skills 反向黑名单

## 执行引擎对比

| 引擎 | agent-easy-http | skill-to-http | skill-to-http-flash |
| --- | --- | --- | --- |
| **subprocess 直执行**（毫秒级） | ❌ | ❌ | ✅ |
| OpenClaw 原生 /hooks/agent（**毫秒级**） | ✅ | ❌ | ❌ |
| OpenClaw Gateway sessions_spawn（~5s） | ❌ | ✅ | ❌ |
| Claude Code SDK（~0.5s，最快） | ❌ | ✅ | ❌ |
| Claude CLI（~2.5s） | ❌ | ✅ | ❌ |
| Codex CLI（~3s） | ❌ | ✅ | ❌ |
| LLM Fallback（无工具调用） | ❌ | ✅ | ❌ |

## API接口总览

### API 端点速查表

| 端点 | agent-easy-http | skill-to-http-flash | skill-to-http |
| --- | --- | --- | --- |
| GET /health | ✅ | ✅ | ✅ |
| GET /schema | — | ✅（单 skill 入参 schema） | — |
| GET /skills | ✅（列表 + 校验） | — | ✅（多 skill 列表） |
| GET /skills/{name} | — | — | ✅（单 skill 详情） |
| GET /agents | ✅（列 OpenClaw 已配置 agent） | — | — |
| GET /metrics | ✅ | — | — |
| POST /agent/run | ✅（自然语言任务，毫秒返回 run_id） | — | — |
| POST /skills/{name}/run | ✅（按 skill 名调，hook 派发） | — | ✅（同步，按 skill 名） |
| POST /skills/{name}/run/async | — | — | ✅（异步 + webhook 回调） |
| POST /run | — | ✅（同步，本服务唯一 skill） | — |
| POST /run/async | — | ✅（异步 + JSONL 持久化） | — |
| GET /jobs/{id} | — | ✅（轮询异步结果） | ✅（同上） |
| GET /result/{run_id} | ✅（从 sessions JSONL 提 agent 输出） | — | — |
| GET /history / /history/{job_id} / /history/stats | — | — | ✅（SQLite 历史） |
| GET /api/logs | — | — | ✅（运行日志） |
| POST /admin/reload | ✅（重载 config / agent 列表） | — | ✅（重载 config / skills） |

### 共用规范（TLS / 鉴权 / 持久化）

三个 skill 共用 TLS / API Key 持久化规范：

| 共用资源 | 路径 |
| --- | --- |
| TLS 证书目录 | <HTTP_ROOT>/certs/（aeh / s2h 共用；flash 用子目录 certs/flash-<skill>/） |
| API Key 持久化 | <HTTP_ROOT>/secrets/api-keys/<skill>.key（0600） |
| 鉴权 header | **X-API-Key: <key>** |
| HTTP_ROOT 默认 | <workspace>/.http/（OpenClaw 环境，PVC 持久化） / ~/.http/（兜底） |
| HTTP_ROOT 覆盖 | OPENCLAW_HTTP_ROOT 环境变量 |

**v2.0 起 flash 不再依赖 skill-to-http**：自带精简版 _cert.py（openssl + cryptography fallback），首次启动自动生成 825 天有效期自签证书（含 SAN）。

### skill-to-http-flash standalone 模式

flash 是**一等公民 standalone**，任何能跑 Python 的机器都能跑：

```bash
# 方式 A：环境变量
export FLASH_SKILL_DIR=/your/skills
export FLASH_DATA_DIR=/your/flash-data
python3 flash.py create --skill my-skill

# 方式 B：CLI flag
python3 flash.py \
  --skill-dir /your/skills \
  --data-dir /your/flash-data \
  create --skill my-skill
```

适合：CI runner / 公网 VPS / 同事笔记本 / 私有云 / 任何不装 OpenClaw 的机器。

### skill-to-http-flash REST API 入参 / 出参格式（新）

#### 请求

```bash
POST /run
Content-Type: application/json

{"date": "2026-05-26", "chart_id": "abc", "fetch_scale": "large"}
```

flash 把 JSON 映射成 long flag CLI：

```bash
python3 scripts/main.py --date 2026-05-26 --chart-id abc --fetch-scale large
```

- 下划线 → 横杠：{"foo_bar": "x"} → --foo-bar x

- 布尔 true → --key（flag style）；false / null → 不传

- 列表 → 重复 flag：{"tags":["a","b"]} → --tags a --tags b

#### 响应（统一 envelope）

```json
{
  "success": true,
  "exit_code": 0,
  "elapsed_ms": 240,
  "data": {"rows": [...]},   // stdout 是合法 JSON 时（模式 A 自适应）
  "output": null,            // stdout 不是 JSON 时（模式 B 文本）
  "stderr": null,            // 仅 success=false 时透出
  "truncated": false         // 仅同步 + stdout >512KB 时
}
```

data 与 output **互斥**，任意时刻只有一个有值。

#### 7 类错误码

| error_type | HTTP | 触发 |
| --- | --- | --- |
| validation_failed | 400 | JSON Schema 校验失败 |
| entry_not_found | 500 | entry 文件不存在或非 .py |
| spawn_failed | 500 | subprocess 启动失败 |
| timeout | 408 | 超过 timeout_seconds |
| rate_limited | 429 | /run/async 并发超过上限（默认 20，可用 FLASH_MAX_ASYNC 配置） |
| internal_error | 500 | 未预期异常（兜底） |
| _exit_code != 0_ | **200** | 业务执行失败（HTTP 仍 200 + envelope.success=false + stderr 透出） |

设计原则：**业务失败用 envelope 表达，HTTP 状态码只表达框架问题**。

# 快速参考

| 资源 | 链接 |
| --- | --- |
| 各 skill 的 SKILL.md | 见 suite repo 各 skill 目录下的 SKILL.md |
| flash v2.0 设计文档 | references/design-v2-direct-execution.md（skill 内） |
| flash v1 → v2 迁移指南 | references/migration-from-v1.md（skill 内） |
| flash standalone 完整指南 | references/standalone-usage.md（skill 内） |
| 三 skill HTTPS 标准化方案 | skill-to-http 内 references/tls-auth-standard.md |

