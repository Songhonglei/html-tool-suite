# skill-to-http-flash 设计方案

> 把单个 Skill 编译成一个**独立、稳定、毫秒冷启**的 HTTP REST API 微服务。
> 核心理念：**subprocess 直执行，运行期零 LLM / 零 Agent / 零 Gateway 依赖。**

---

## 1. 一句话定位

skill-to-http-flash（下称 flash）解决的是这样一个问题：

> 我有一个带命令行参数的 Python 脚本（一个 skill），想把它变成一个能被浏览器、
> 业务后端、CI 流水线直接 `curl` 的 REST 接口，而且最好能脱离任何 Agent 框架，
> 在一台普通的 Python 机器上独立跑。

flash 的答案是：**一个 skill 编译成一个独立的 FastAPI 微服务**，请求进来后
直接 `subprocess.run([python3, entry.py, --flag value, ...])` 调脚本，把
stdout/stderr/exit_code 包装成统一 envelope 返回。整条链路不经过任何大模型或
Agent 会话。

---

## 2. 为什么是"直执行"

flash 的核心决策是：**不让大模型参与执行**。一个"输入结构化参数、输出结构化
结果"的 skill，执行阶段根本不需要 LLM。对比两种可能的执行方式：

| 维度 | 经 Agent/LLM 执行 | subprocess 直执行（flash 的选择）|
|---|---|---|
| 冷启动 | 秒级~几十秒（模型加载 / 会话初始化）| **毫秒级**（就是起个子进程）|
| 确定性 | LLM 可能改写参数、产生幻觉 | **完全确定**，参数 1:1 映射到 CLI |
| 依赖 | 必须有 LLM key + Agent runtime | **零运行期依赖**，任何 Python 机器可跑 |
| 可调试 | 黑盒 | 直接看等价 CLI 命令，本地复现 |
| 成本 | 每次调用烧 token | **零 token** |

结论：flash 在运行期完全不碰 LLM，请求参数确定性地翻译成命令行直接执行。
LLM 只在**生成期**（`create` 时提取参数 schema）可选参与，运行期完全不碰。

---

## 3. 整体架构

```
                       ┌─────────────── 生成期（create，一次性）───────────────┐
                       │                                                        │
   flash.py create ────┼──▶ ① 解析 SKILL.md frontmatter（flash.entry 等）        │
     --skill <name>    │    ② 提取入参 JSON Schema                              │
                       │        - 有 LLM 配置 → LLM 读 SKILL.md 提参（可选）     │
                       │        - 无配置     → 启发式 fallback（宽松 schema）    │
                       │    ③ 用 server_template.py 渲染出独立 server.py         │
                       │    ④ 生成 API Key + TLS 证书（按需）                    │
                       │    ⑤ 拉起 uvicorn 进程，写 pid/port                     │
                       └────────────────────────────────────────────────────────┘

                       ┌─────────────── 运行期（每次请求，零 LLM）──────────────┐
   外部请求            │                                                        │
   POST /run ──────────┼──▶ ① API Key 校验（hmac.compare_digest 常时比较）       │
   {"date":"...",      │    ② JSON Schema 校验入参                              │
    "chart_id":"x"}    │    ③ JSON → 长 flag CLI 映射                           │
                       │        {"chart_id":"x"} → --chart-id x                 │
                       │    ④ subprocess.run([py, entry.py, --date ..., ...])   │
                       │    ⑤ 收集 stdout/stderr/exit_code                      │
                       │    ⑥ 自适应封装 envelope                               │
   ◀───────────────────┤        stdout 是 JSON → data 字段                     │
   统一 envelope        │        否则          → output 字段                    │
                       └────────────────────────────────────────────────────────┘
```

**一个 skill = 一个独立进程 = 一个端口 = 一份 API Key**。服务本身就是端点，
没有多 skill 路由、没有控制台——极简是设计目标。

---

## 4. 参数映射规则（JSON → CLI）

请求体是 JSON，flash 把它确定性地翻译成命令行参数：

| JSON | 映射到 CLI | 说明 |
|---|---|---|
| `{"date": "2026-05-30"}` | `--date 2026-05-30` | 基础键值 |
| `{"chart_id": "abc"}` | `--chart-id abc` | 下划线 → 横杠 |
| `{"verbose": true}` | `--verbose` | 布尔 true → flag |
| `{"verbose": false}` | （不传） | false / null 不传 |
| `{"tags": ["a", "b"]}` | `--tags a --tags b` | 列表 → 重复 flag |
| `{"opts": {"k": 1}}` | `--opts '{"k": 1}'` | dict → JSON 字符串 |

**安全**：字段名走 `[a-z][a-z0-9_]*` 白名单正则校验，argv 以 list 形式传给
subprocess（`shell=False`），从根上杜绝 shell 注入。

---

## 5. 统一响应 envelope

无论 skill 返回什么，flash 都封装成同一个信封：

```json
{
  "success": true,
  "exit_code": 0,
  "elapsed_ms": 240,
  "data": {"rows": [...]},   // stdout 是合法 JSON 时（模式 A）
  "output": null,            // stdout 不是 JSON 时（模式 B，纯文本）
  "stderr": null,            // 仅 success=false 时透出
  "truncated": false         // 仅同步 + stdout > 512KB 时
}
```

- **`data` 与 `output` 互斥**：flash 自适应判断 stdout 是不是合法 JSON，
  是就放 `data`（结构化），不是就放 `output`（文本）。客户端拿到啥一目了然。
- **业务失败也是 HTTP 200**：skill 退出码非 0 属于"业务失败"，用
  `success:false + stderr` 表达，**HTTP 状态码只留给框架级错误**。这样客户端
  不用靠 HTTP code 猜业务结果。

---

## 6. 错误码设计

7 类标准 `error_type`，区分"框架问题"和"业务问题"：

| error_type | HTTP | 触发 | 归类 |
|---|---|---|---|
| `validation_failed` | 400 | JSON Schema 校验失败 | 框架 |
| `entry_not_found` | 500 | entry 文件不存在或非 .py | 框架 |
| `spawn_failed` | 500 | subprocess 启动失败 | 框架 |
| `timeout` | 408 | 超过 timeout_seconds | 框架 |
| `rate_limited` | 429 | 异步并发超过上限（默认 20，可用 `FLASH_MAX_ASYNC` 配置）| 框架 |
| `internal_error` | 500 | 未预期异常（兜底）| 框架 |
| `exit_code != 0` | **200** | 业务执行失败 | **业务** |

设计原则一句话：**业务失败用 envelope 表达，HTTP 状态码只表达框架问题。**

---

## 7. 同步 vs 异步

| | 同步 `POST /run` | 异步 `POST /run/async` |
|---|---|---|
| 返回 | 直接等结果 | 立即返回 `job_id` |
| 超时 | 默认 60s | 无（后台跑）|
| 输出 | > 512KB 截断（`truncated:true`）| 不截断 |
| 拿结果 | 响应体 | 轮询 `GET /jobs/{id}` |
| 持久化 | — | JSONL append-only（重启不丢）|
| 并发控制 | — | Semaphore（默认 20，`FLASH_MAX_ASYNC` 可配）|

异步任务写入 JSONL 落盘，单文件 > 10MB 自动 logrotate，保留 7 份。内存 job
的 TTL 是 1 小时，过期后仍可从 JSONL 查历史。

---

## 8. Standalone —— 一等公民

flash 最重要的特性：**不依赖任何 Agent 框架**。任何能跑 Python 的机器都能用。

```bash
# 方式 A：环境变量
export FLASH_SKILL_DIR=/your/skills
export FLASH_DATA_DIR=/your/flash-data
python3 flash.py create --skill my-skill

# 方式 B：CLI flag
python3 flash.py --skill-dir /your/skills --data-dir /your/flash-data \
  create --skill my-skill
```

skill 目录探测优先级：`--skill-dir` flag > `FLASH_SKILL_DIR` env > 一组多 Agent
默认目录（如 `~/.claude/skills`、`~/.cursor/skills`、`./skills`、`/app/skills`
等）> 兜底。**CI runner / 公网 VPS / 同事笔记本 / 私有云都能直接跑。**

---

## 9. TLS 与鉴权

- **默认 HTTP**（零门槛，先跑通），按需一键切 HTTPS。
- 自带精简版 `_cert.py`（优先 openssl CLI，fallback 到 cryptography 库），
  首次启用自动生成含 SAN 的自签证书，**不依赖 skill-to-http**。
- API Key 走 `X-API-Key` header，比较用 `hmac.compare_digest`（常时比较，
  防 timing 攻击），401 不回显 header 名（减少信息泄漏）。

---

## 10. 关键工程决策回顾

| 决策 | 选择 | 理由 |
|---|---|---|
| 执行方式 | subprocess 直执行 | 毫秒冷启 + 完全确定 + 零 token |
| 一个 skill 一个服务 | 是 | 极简、隔离、独立扩缩容 |
| 业务失败用 HTTP 200 | 是 | HTTP code 只表达框架错误，语义清晰 |
| data/output 自适应 | 是 | 结构化/文本都优雅承载 |
| 优雅退出不用 `os._exit` | 是 | 避免绕过 atexit/finally，防 JSONL flush 被截断 |
| 并发上限可配 | `FLASH_MAX_ASYNC` | 默认 20，按机器能力调整 |
| 生成期可选 LLM | 是 | 提参更准，但无配置也能靠 fallback 跑通 |

---

## 11. 与另外两剑客的关系

- **agent-easy-http**：给 Agent 发一句自然语言任务，适合"让 Agent 自己决定怎么干"。
- **skill-to-http**：多 skill 统一网关 + 控制台 + 多执行引擎降级，适合企业级一整套。
- **skill-to-http-flash（本篇）**：单 skill、结构化入参、直执行、可 standalone，
  适合"把这一个 skill 稳定挂出去，任何机器都能跑"。

三者互补，详见 [HTTP 三剑客对比](./http-trio.md)。
