# FreeLLMAPI 使用说明书

> 文档编号：2HELP.md  
> 适用版本：当前仓库主线（Node 20.18–24、npm 10+）  
> 读者：要安装、配置、把编码智能体接到网关、以及排障的操作员  

本手册按「先能跑起来，再按功能用，再按配置调，最后用 FAQ 自助」组织。所有命令与路径以本仓库为准。若与英文 README 冲突，以源码与 `.env.example` 为准。

---

## 1. 整体功能概述

FreeLLMAPI 把几十家 LLM 提供方的免费额度，以及你自己的 OpenAI 兼容端点，收成**一个**本地 API。你在仪表盘里粘贴各家密钥，系统给你一把统一密钥 `freellmapi-…`。之后：

- 任何 OpenAI SDK 把 `base_url` 指到 `http://localhost:3001/v1`；
- Claude Code 走 `/v1/messages` 或 CLI 生成的配置；
- Codex CLI 走 `/v1/responses`；
- Gemini CLI 走 `/v1beta`；
- Zed / JetBrains 可走开箱即用的 Ollama 模拟；
- 智能体还可以通过 `/mcp` 询问「现在哪些模型活着」。

路由器每次请求都会：挑一个有健康密钥、没触顶的模型 → 内存解密密钥 → 调用上游 → 429/5xx 则冷却并换下一家 → 按密钥记账以免打爆免费层。

你得到的不是「无限 GPT」，而是**叠加后的容量**：目录宣传约 34 家提供方、600+ 免费端点、每月数十亿 token 量级。实际能用多少取决于你添加了哪些密钥、当天各家是否限流、以及模型是否被目录退役。

### 1.1 你能用它做什么

1. 给 Cursor、Cline、Roo、Aider、Continue、OpenCode、Goose 等编码智能体当后端。
2. 在试验台里对比模型、看 Fusion 合稿、试图像/音频。
3. 把本地 llama.cpp / LM Studio / vLLM / Ollama 注册成自定义提供方，与云端免费层一起故障转移。
4. 看 24 小时到 90 天的用量、延迟分位、错误分类。
5. 用声明式 JSON 在 CI 或树莓派上无人值守启动。

### 1.2 你不能指望它做什么

- 不提供 SLA。免费层会不打招呼改额度或下线。
- 不是多用户网关。不要把端口暴露到公网再给同事共用一把密钥。
- 一天之内有效智能会下降：最强的模型日额度最小，耗尽后链路会滑到更小的模型，UTC 午夜附近才回升。
- 不审核内容，不提供 `/v1/moderations`。
- 一次请求不会返回 n>1 个补全。

### 1.3 五分钟心智模型

把网关想成一条**有评分的候车队列**：

- 队列顺序默认不是你手排的死顺序，而是 balanced 老虎机：更稳、更快、更聪明且还有余量的排前面。
- 你可以改成手动 priority，或 smartest / fastest / reliable，或自己拉三个滑条。
- 某把密钥 429 了，它会坐下休息（冷却），队列里的下一位顶上。
- `model: "auto"` 表示「按队列来」；`model: "gemini-2.5-flash"` 表示「只准这家的这个模型，但密钥仍可轮换」。

---

## 2. 安装与第一次运行

### 2.1 一行 Docker（推荐大多数人）

需要本机有 Docker。脚本创建 `~/freellmapi`、生成加密密钥、拉镜像、起容器：

```bash
curl -fsSL https://freellmapi.co/install.sh | bash
```

重复执行安全：`.env` 与加密密钥保留，容器更新到 `:latest`。可用 `FREELLMAPI_DIR`、`PORT`、`HOST_BIND` 覆盖。

打开 http://localhost:3001 。若从另一台设备访问，必须 `HOST_BIND=0.0.0.0`，且只在可信局域网这么做。

### 2.2 Docker Compose（从源码）

```bash
git clone https://github.com/tashfeenahmed/freellmapi.git
cd freellmapi
ENCRYPTION_KEY="$(openssl rand -hex 32)"
printf "ENCRYPTION_KEY=%s\nPORT=3001\n" "$ENCRYPTION_KEY" > .env
docker compose up -d
```

Windows PowerShell 用 RNG 生成 32 字节 hex 写入 `.env`。丢失 `ENCRYPTION_KEY` 等于丢失所有已存的提供方密钥。

### 2.3 本地开发

```bash
npm install
cp .env.example .env   # 填 ENCRYPTION_KEY
npm run dev            # server + Vite
```

仪表盘开发服务器在 5173，API 在 3001。Node 必须 ≥20.18 且 <25。

### 2.4 桌面应用

从 GitHub Releases 下载 macOS `.dmg`（arm64 或 x64）或 Windows `.exe`。macOS 12+。不强制账号：托盘里的统一密钥就是凭据。从源码：`npm run desktop:dev`。日志在托盘「打开日志文件夹」的 `freeapi.log`。

### 2.5 Android Termux（实验）

见 `docs/zh-cn/install/02-android-termux.md`。使用 Node 内置 SQLite，不需要 NDK。

### 2.6 第一件事清单

1. 用浏览器打开仪表盘。远程访问时输入启动日志里的一次性 setup code。
2. 设置邮箱与密码（浏览器安装需要；桌面可跳过）。
3. 打开 **密钥** 页，添加至少一把提供方密钥（Groq / Google / OpenRouter 最容易起步）。
4. 复制页顶统一密钥。
5. 打开 **模型 / 回退链**，确认有启用的模型。空链会 400，不会偷偷用全目录。
6. 试验台发一句「hi」，看是否返回，以及 `X-Routed-Via`。
7. 用 CLI 接上你的智能体（下一章）。

---

## 3. 每一个功能的详细使用说明

### 3.1 统一 API 密钥

**在哪：** 密钥页顶部。  
**干什么：** 所有 `/v1` 调用的 Bearer。  
**注意：** 等同根密码。泄漏后应在仪表盘轮换，并检查请求日志里的陌生 IP。不要把它提交到 git。CLI 用 `FREELLMAPI_API_KEY` 或 `--api-key`。

### 3.2 添加提供方密钥

点「添加密钥」，选平台，粘贴密钥，可选标签。保存时服务器会探测一次。状态：

- healthy：可用于路由
- rate_limited：暂时跳过
- invalid：401/403，不会再打，除非你改密钥或强制重探
- error / unknown：看 last health error

**模型范围：** 一把密钥可以限制只服务某些 model id，避免「备用号」被不相关模型烧掉。

**每密钥代理：** 公司网络下某家必须走不同出口时用。密码在库中加密。

### 3.3 自定义端点

平台选 custom，填 OpenAI 兼容 base URL，例如 `http://127.0.0.1:11434/v1`。然后「发现模型」或手工登记 chat/embedding/image/audio。Docker 里不要写 127.0.0.1 指向宿主机上的 Ollama，应写 `http://host.docker.internal:11434/v1`。云元数据 IP 永远被拒绝。

### 3.4 批量导入导出

把 `.env` 里多行 `GROQ_API_KEY=...` 粘贴到导入框，先预览再勾选。导出去掉密码的代理 URL，格式 JSON / .env / CSV。导出接口有严格 RPM，脚本不要狂刷。

### 3.5 回退链与配置档

**模型页** 看到的是目录。**回退链** 决定 auto 的候选集合与手动模式下的顺序。

- 拖拽调整 priority（仅手动策略时顺序就是命运）。
- 开关 enabled。全部关掉 = 空链 = 400。
- 新建配置档：「编程」「视觉」「便宜」。智能体请求 `model: "auto:编程"` 即可换链而不换密钥。
- 可打开「新模型自动加入配置档」，否则目录同步来的新行不会进你的链。

### 3.6 路由策略

在设置或回退页选择：

| 策略 | 何时用 |
|---|---|
| balanced（默认） | 日常编码，稳优先 |
| smartest | 难重构、需要更强推理 |
| fastest | 补全、短问答 |
| reliable | 演示、CI，只要成功 |
| priority | 你完全不信自动排序 |
| custom | 自己拉可靠性/速度/智能三个滑条 |

单次覆盖：

```json
{ "model": "auto:fast", "messages": [{"role":"user","content":"hi"}] }
```

同义：`auto:fastest`、`auto:smart`、`auto:cheap`（免费池上 cheap≈balanced）、`auto:reliable`。

高峰时段：若你晚间免费中继拥堵，打开 peak hours，选 IANA 时区，系统把部分速度权重改到可靠性。fastest/reliable 不会被改，以免预设身份消失。

任务类型：编码智能体可声明 code，把一点速度权重让给智能。不要和 custom 同时指望它生效——custom 是你手写的向量，请求级不会偷偷改。

### 3.7 钉死模型

`"model": "llama-3.3-70b-versatile"` 只在该逻辑模型的提供方之间转。统一分组打开时，Groq 与别家的同名模型算一组。分组错误用 merges/splits 纠正。

### 3.8 试验台（Playground）

左侧会话列表，中间 composer，右侧可选产物面板。支持：

- 流式输出与 Markdown
- 附件图像（会按设置缩小）
- 语音输入（若有 transcription 模型）
- 采样参数滑条
- Fusion 与普通模型切换
- 会话持久化在 SQLite `playground_conversations`

试验台走同一代理，因此这里能复现的问题，CLI 里一般也能复现。

### 3.9 Fusion 多模型合成

把模型设为 `fusion`。网关并行问多个风格不同的免费模型，再找一个评审模型合成。适合「一份设计方案、一份文案」这种允许花多倍 token 换质量的任务。不适合：

- 工具调用密集的 agent 循环（系统会改走短超时串行，且可能直接不走面板）
- 要省额度的日常补全

在 Fusion 页调整默认 K、最大 K、超时。K 越大越慢越贵（对免费层是「越容易触顶」）。

### 3.10 工具调用

请求里带 OpenAI 风格 `tools`。路由器只考虑标明 supports_tools 的模型；把会拒绝 tools 的模型放到最后。若模型把调用写成文本，网关会尝试救援成 `tool_calls`。若你看到 Anthropic 客户端收到空 `input: {}`，打开 `VALIDATE_TOOL_ARGUMENTS=1` 让坏参数换模型重试。

### 3.11 视觉

消息 content 用 image_url / base64。没有视觉能力的模型会被跳过。超大截图默认会被压到长边 2048。关掉：`IMAGE_NORMALIZE=off`。

### 3.12 嵌入

`POST /v1/embeddings`。密钥页登记 embedding 模型。分析页不把 embedding 与 chat 混在同一条链。

### 3.13 图像生成与 TTS

`/v1/images/generations`、`/v1/audio/speech`。仪表盘有 Image / Audio 页可试。视频页对应实验性 video generations。

### 3.14 压缩

设置里选 off/lossless/standard/aggressive。Agent 长会话工具输出爆炸时，standard 通常是合理起点。请求头 `X-FreeLLM-Compress: lossless` 只能往更保守调。全局 off 时请求头不能擅自打开，防止客户端改写操作员策略。

### 3.15 响应缓存

默认关。打开后**完全相同**的请求（含温度等）在 TTL 内直接内存返回。高温度默认也可缓存，可把 `RESPONSE_CACHE_MAX_TEMPERATURE` 降到 0.2 只缓存近确定性调用。注意持久化缓存会把模型回答明文写入 SQLite。

### 3.16 粘性会话与交接

同一对话尽量留在同一模型，减少「上一句是强模型、下一句是小模型却不知道上下文」的断裂。若仍换模，可开 context handoff，让网关插入一句简短交接。

### 3.17 分析

时间窗 24h 到 90 天。看成功率、token、p50/p95、TTFB、按平台拆分、错误类别。最近调用表可关 IP/UA 记录：`REQUEST_ANALYTICS_LOG_CLIENT=false`。保留天数与最大行数可配，防树莓派磁盘被请求日志写满。

### 3.18 日志

分析菜单下的日志页。实时环缓冲 + 库内 warn/error。不要把 server stdout 原样贴到 GitHub——红线会尽量遮密钥，但仍可能有 URL 查询串。

### 3.19 MCP

把 MCP 客户端指到网关 `/mcp`，智能体可列出模型与健康状况。设置里可关默认启用（迁移曾改过默认值）。

### 3.20 交互式 API 文档

浏览器打开 `/v1/docs`，规范在 `/v1/openapi.json`。适合没有 SDK 时手工试。

### 3.21 Premium 目录

免费安装跟随约 30 天延迟的月度快照；付费 live feed 当天拿到新模型与额度修复。这只影响**目录新鲜度**，不影响你是否自托管。在仪表盘 Premium 页查看状态。

### 3.22 备份

设置备份路径或 HTTPS 目标与独立备份密钥。启动时若主库缺失会尝试恢复。不要把备份密钥和 `ENCRYPTION_KEY` 混成同一个还到处复制——备份介质泄漏面不同。

### 3.23 更新检查

设置里可关。桌面自动更新需要签名构建；自己 `npm run dist` 的包检查器可能显示有新版本但 Squirrel 会拒。

### 3.24 出站代理

公司网或 GFW 场景：在密钥页设 SOCKS5h/HTTP 代理，或 env `PROXY_URL`。`h`/`a` 变体在代理侧解析 DNS。Docker 里代理若跑在宿主机，用 `host.docker.internal`。Fetch Relay 模式把出站改成应用层中继，见 `docs/zh-cn/proxy/01-fetch-relay.md`。

### 3.25 声明式启动配置

```json
{
  "keys": [{"platform":"groq","key":"gsk_...","label":"main"}],
  "routing": {"strategy":"balanced"}
}
```

`FREEAPI_CONFIG_PATH` 或 `FREEAPI_CONFIG_JSON`。每次启动幂等应用，适合不可变基础设施。不要把这份 JSON 提交到公开仓库。

---

## 4. 把编码智能体接进来

统一模式：安装 CLI → 生成配置（先 `--dry-run`）→ 备份已存在 → 写入。

```bash
npx freellmapi setup-claude --url http://localhost:3001 --api-key "$KEY"
npx freellmapi setup-codex --url http://localhost:3001 --api-key "$KEY"
npx freellmapi setup-aider --url http://localhost:3001 --api-key "$KEY"
```

还有 cline、continue、opencode、goose、qwen、roo、kilo、crush、dsh、mimo、openclaw、hermes、cursor、atomcode、generic。

原则：

- 默认模型是 `auto`，不要默默写成 `fusion`。
- `--model` 钉死某个 id，即使它此刻额度用尽也会写入，以免生成器擅自换模型。
- `--profile` 给同一工具多套配置目录。
- `freellmapi launch` / `launch-codex` 不把密钥写盘，只注入子进程环境，适合共享机器。
- `freellmapi doctor` 探 `/livez` 并检查各工具配置。

OpenAI 兼容客户端最小 Python 例子：

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:3001/v1", api_key="freellmapi-...")
print(client.chat.completions.create(
    model="auto",
    messages=[{"role":"user","content":"ping"}],
).choices[0].message.content)
```

---

## 5. 配置说明

配置有三层，优先级从高到低因项而异，但记忆口诀是：

1. **单请求头**（只影响这一次）：`X-FreeLLM-Cache`、`X-FreeLLM-Compress`、`model: auto:fast`、Idempotency-Key。
2. **仪表盘 settings 表**（热更新）：策略、权重、Fusion、压缩、缓存开关、护栏。
3. **环境变量 / `.env`**（启动时）：密钥、端口、超时、代理、路径。

生产必须设 `ENCRYPTION_KEY`（64 hex）。开发未设会写 `.encryption-key` 文件。

### 5.1 端口与监听

- `PORT` 默认 3001。
- `HOST` 默认 `::` 双栈。
- Docker `HOST_BIND` 默认 127.0.0.1。

### 5.2 限流与超时

- 代理入口默认 120 RPM/IP，仪表盘 600，导出 10。
- `FALLBACK_TIME_BUDGET_MS` 默认 45s，到点不再开新尝试。
- 提供方超时 `PROVIDER_TIMEOUT_<PLATFORM>`，NVIDIA 默认更长。
- 流中停滞默认 90s，可按平台覆盖。

### 5.3 安全相关

- `FREEAPI_BLOCK_PRIVATE_PROVIDER_URLS=true` 在仪表盘暴露给他人时必须开。
- `TRUST_PROXY` 默认 false；前面有 Caddy 时设 1，否则分析 IP 全是 127.0.0.1。
- `CSP_UPGRADE_INSECURE_REQUESTS` 在纯 HTTP 局域网保持自动/关，否则静态资源会被浏览器改写成 https 然后加载失败。

### 5.4 数据位置

- SQLite：`server/data/freeapi.db` 或 `FREEAPI_DB_PATH`
- 加密密钥文件：数据库旁 `.encryption-key`
- 桌面：OS 用户数据目录（见安装文档）
- Docker：具名卷

备份先停写入或使用内置备份，不要拷正在写的 WAL 当完整备份。

### 5.5 路由覆盖

`MODEL_ROUTING_OVERRIDES={"gpt-4o":0.2}` 乘数 0–2。0 表示自动路由永不选它，手动 priority 仍可选。id 精确、区分大小写、跨平台按 model id 匹配。

---

## 6. 日常运维建议

- 至少两家提供方密钥，一家挂了还能工作。
- 每周看分析页：若某平台成功率暴跌，先看健康而不是先改策略。
- 目录同步失败时路由器继续用旧目录，不要为此重启循环。
- 笔记本休眠后若首请求失败，唤醒检测应重探；若没有，手动在密钥页「全部检查」。
- 树莓派把 `REQUEST_ANALYTICS_MAX_ROWS` 调低。
- 升级 Docker 镜像前备份 `.env` 与数据卷。

---

## 7. 常见问题 100 问

### FAQ 1. 安装后浏览器打不开 3001

确认容器 `docker ps` 在跑，端口映射是 3001:3001，且你访问的是运行 Docker 的那台机器。远程设备需要 HOST_BIND=0.0.0.0。

### FAQ 2. 页面一直转圈

常见于从局域网 IP 访问但绑定仍是 127.0.0.1。看 docker-compose 端口发布地址。

### FAQ 3. 仪表盘白屏且控制台 ERR_SSL

纯 HTTP 环境被 CSP upgrade-insecure-requests 误伤。升级到已修复版本或设 CSP_UPGRADE_INSECURE_REQUESTS=false。

### FAQ 4. 第一次远程打开要求 setup code

这是防抢注。看 server 日志里打印的一次性码。本机回环浏览器不需要。

### FAQ 5. 忘记仪表盘密码

桌面版看日志文件里的重置码；Docker 用 docker logs。按忘记密码流程走。

### FAQ 6. 提示 Encryption key missing

生产必须提供 ENCRYPTION_KEY。生成：node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"。

### FAQ 7. 换机器后所有密钥无效

你没有带走 ENCRYPTION_KEY / .encryption-key。库文件在但解不开。只能重新添加密钥。

### FAQ 8. 添加 Groq 密钥显示 invalid

检查是否复制了空格、密钥是否被吊销、出站代理是否拦了 api.groq.com。

### FAQ 9. Google 密钥健康检查失败

确认是 AI Studio / Gemini API key，不是 OAuth 客户端密钥。Gemma 冷启动慢不算失败，超时过短才会误判。

### FAQ 10. ModelScope 一探测就扣魔粒

验证走付费 1 token 补全。成功结果默认缓存 24h。MODELSCOPE_VALIDATE_CACHE_MS=0 会把额度探穿。

### FAQ 11. 自定义 Ollama 连不上

Docker 内 127.0.0.1 是容器自己。用 host.docker.internal，并让 Ollama 监听 0.0.0.0。

### FAQ 12. 报 SSRF / blocked URL

云元数据与 link-local 永远禁止。VPS 上应再开 FREEAPI_BLOCK_PRIVATE_PROVIDER_URLS。

### FAQ 13. 导入 .env 全部 skipped

平台检测失败或重复。看预览里 detectedPlatform 与 duplicates。

### FAQ 14. 导出只有 10 次就 429

导出限流就是 10 RPM，防止脚本爆破。等一分钟。

### FAQ 15. 统一密钥 401

Bearer 前缀、复制缺段、或用了提供方原始密钥而不是 freellmapi- 开头的统一密钥。

### FAQ 16. 模型列表是空的

没有启用密钥，或目录尚未同步，或过滤了 available=true 而所有模型都不可用。

### FAQ 17. 请求 400 active chain is empty

回退链没有启用的模型。去 Fallback 页打开几个，或换 profile。

### FAQ 18. 一直落到很小的模型

强模型日额度用尽是预期行为。看限流条和下次 UTC 午夜，或加更多提供方。

### FAQ 19. latency 忽快忽慢

Cerebras/Groq 很快，社区中继和 Horde 很慢。fastest 策略可偏向速度，但不能消灭方差。

### FAQ 20. stream 到一半断了

提高 PROVIDER_STREAM_STALL_TIMEOUT_MS，或按平台覆盖。NVIDIA 预填长提示可能数分钟无首字节，应提高 PROVIDER_TIMEOUT_NVIDIA。

### FAQ 21. 首 token 极慢但不算超时

首字节预算取 chat timeout 与 stall 的较大者。把对应 PROVIDER_TIMEOUT 加大。

### FAQ 22. 工具调用没有发生

模型不支持 tools 会被跳过；若链上没有任何 tools 模型，会耗尽。看诊断里 lacks tool-calling。

### FAQ 23. tool 参数是空对象

打开 VALIDATE_TOOL_ARGUMENTS，让网关换模型。或换更强的 tools 模型。

### FAQ 24. Claude Code 连不上

用 setup-claude 生成，确认 ANTHROPIC_BASE_URL 指向网关根而不是 /v1。统一密钥作 ANTHROPIC_AUTH_TOKEN。

### FAQ 25. Codex 报 responses 404

网关已实现 /v1/responses。检查 base URL 是否多写或少写 /v1，以及版本是否过旧。

### FAQ 26. Cursor 里要用哪套

OpenAI 兼容：Override OpenAI Base URL 为 http://localhost:3001/v1。或运行 setup-cursor。

### FAQ 27. Zed 看不到模型

打开 Ollama 模拟，把 Zed 指到网关的 Ollama 端口/路径，见客户端文档。

### FAQ 28. Gemini CLI 鉴权失败

它走 /v1beta 而非 /v1/chat/completions。用文档中的 Gemini 配方，不要当 OpenAI 客户端配。

### FAQ 29. Fusion 特别慢

默认 4 路并行再评审。降 fusion_default_k，或不要对短问题用 fusion。

### FAQ 30. Fusion 只返回一个模型的话

成功草稿不足法定人数 2，系统会直接返回唯一幸存者，这是特性。

### FAQ 31. 缓存返回了过期答案

精确匹配且在 TTL 内。改 prompt 或关缓存，或等 TTL。不要指望语义缓存。

### FAQ 32. 打开缓存后磁盘变大

RESPONSE_CACHE_PERSIST 把明文答案写入 SQLite。可关持久化只留内存。

### FAQ 33. 压缩后模型胡言乱语

改回 standard 或 lossless。保真门会放行失败压缩，但 aggressive 仍可能切掉对你重要的旧上下文。

### FAQ 34. 图像发上去上游 400

开 IMAGE_NORMALIZE，把 webp/gif 转成 jpeg/png。或确认模型 supports_vision。

### FAQ 35. 分析页没有数据

还没有成功/失败请求，或保留策略把行剪光了，或看错时间窗。

### FAQ 36. 分析 IP 都是 127.0.0.1

前面有反代时设 TRUST_PROXY=1。不要在公网无脑 true。

### FAQ 37. 日志里没有密钥但有提示词

请求日志默认记 token 数不是全文。仍注意 playground 会话表存了对话。

### FAQ 38. 健康检查把额度用光

减少密钥数、拉长周期，或依赖 ModelScope 类缓存。健康检查设计上已交错限速。

### FAQ 39. 休眠后第一次请求失败

唤醒检测会丢连接池并重探。若仍失败，手动 Check all keys。

### FAQ 40. SQLite busy

busy_timeout 已是 5s。不要用第三个进程直接写同一 db。备份用内置接口。

### FAQ 41. 如何迁移到新服务器

拷贝 db、.encryption-key、.env，权限 0600/0700，同一 Node 大版本。

### FAQ 42. WAL 文件能不能删

正常关闭会自己处理。强删可能丢最近写入。先停进程。

### FAQ 43. 如何完全重置

停服务，删 db 与密钥文件，保留 .env 中的 ENCRYPTION_KEY 或一起删后重新添加密钥。

### FAQ 44. pm2 内存涨

空闲约 40MB。缓存条目上限 RESPONSE_CACHE_MAX_ENTRIES。泄漏先看是否有未释放的流。

### FAQ 45. IPv6 only 主机容器无网

Docker daemon 开 ipv6 与 ip6tables。见安装文档。

### FAQ 46. 公司 SSL 检查中间人

自定义 CA 需进入 Node 信任库；或对提供方走已信任的正向代理。

### FAQ 47. SOCKS 报只给了 IP

对 Tor 用域名且 socks5h。本地字面量 IP 默认绕过代理。

### FAQ 48. 想让代理也走本地 Ollama

FREEAPI_PROXY_LOCAL_DESTINATIONS=true，仅当 ssh -D 这种场景。

### FAQ 49. Fetch Relay 400 loopback

中继 URL 不能是 127.0.0.1，除非策略放行。用 https 中继。

### FAQ 50. 幂等 409

同一 Idempotency-Key 配了不同正文。换 UUID 或完全相同重放。

### FAQ 51. 并发两个相同幂等键

进行中窗口不去重，仍可能双花。客户端应串行重试。

### FAQ 52. 空 chain 但目录很多模型

目录 ≠ 链。必须把模型放进当前 profile 并启用。

### FAQ 53. auto:某中文名不生效

profile 名要与创建时一致。CLI --profile 只允许字母数字点下划线连字符。

### FAQ 54. 统一分组把两个无关模型并一起

用 Unify splits 拆开，或 merges 指到正确 into。

### FAQ 55. 改 intelligence rank 好像没变化

层（Frontier/Large/…）仍然严格主导。只在同一档内 rank 才显著。看评分面板。

### FAQ 56. priority 策略下评分无用

是的。priority 完全按你的顺序，只跳过不可用项。

### FAQ 57. custom 权重被高峰改掉

不会。高峰豁免 fastest/reliable；任务偏向再豁免 custom。

### FAQ 58. 社区先验是什么

可选把匿名聚合的成败计数当 Beta 起点。本地数据多了会稀释。可关。

### FAQ 59. 惩罚检查器里全是红的

短期 429 累积。等冷却，或分散到更多密钥。clear penalties 仅调试用。

### FAQ 60. 如何让某模型永不被 auto 选

MODEL_ROUTING_OVERRIDES 设 0，或在链里 disable。

### FAQ 61. 如何强制只用本地模型

只启用 custom 密钥，关掉云密钥，或单独 profile。

### FAQ 62. OpenRouter 日限额

PROVIDER_DAILY_REQUEST_CAP_OPENROUTER=50 之类可再加一道本地闸。

### FAQ 63. 多把 Groq 密钥如何轮转

默认按密钥分数；可改 least-remaining。并发帽 MAX_CONCURRENT_REQUESTS_PER_KEY_GROQ。

### FAQ 64. 429 之后要等多久

看 Retry-After 或仪表盘冷却剩余。耗尽正文也会给 Soonest reset。

### FAQ 65. 支付 402

该免费层要绑卡或试用金花完。会进入长冷却，换别家。

### FAQ 66. model_not_found 一串

平台短期下线一批模型。网关会在多次未找到后跳过该平台剩余项。等目录同步。

### FAQ 67. Reka / OpenCode 免费没了

已从名册退役或停叫。删旧密钥，看当前目录。

### FAQ 68. Cohere 能用吗

技术上有适配器，但条款不适合个人家用。自行判断。

### FAQ 69. NVIDIA 说仅评估

条款限制生产。超时也更长。不要当 SLA 后端。

### FAQ 70. GitHub Models 限实验

同样仅原型。密钥权限按 GitHub 文档。

### FAQ 71. 国内千帆/方舟/LongCat/星火

多数要实名。LongCat 可用邮箱注册。绑定失败会 401。

### FAQ 72. AI Horde 特别慢

志愿算力排队。匿名密钥 0000000000 优先级最低。无工具。

### FAQ 73. Sail 要轮询

专用适配器已处理后台 job。超出月赠金会开始计费，注意绑卡。

### FAQ 74. embedding 维度不对

各家维度不同，不要把 Groq 向量和本地模型向量混进同一索引。

### FAQ 75. TTS 没声音

确认 audio 模型已登记且密钥有该模态。浏览器要允许自动播放。

### FAQ 76. 视频生成 501

该构建可能未接此提供方或模型未启用。看 video 路由测试与媒体表。

### FAQ 77. MCP 工具列表空

设置里关闭了 MCP，或鉴权失败。

### FAQ 78. 命令面板快捷键

Mac 用 Cmd，其它用 Ctrl。见导航栏搜索图标。

### FAQ 79. 语言错了

设置或浏览器语言；60 语 locale 在 client/src/i18n/locales。

### FAQ 80. 暗色闪白

内联引导脚本被拦。不要改 index.html 引导除非同时更新 CSP 哈希。

### FAQ 81. 桌面托盘图标消失

macOS 托盘可见性有专门逻辑；看 tray-visibility 测试所述平台差异。

### FAQ 82. Linux 桌面打不开

chrome-sandbox 打包问题。用官方包或按 after-pack 脚本。

### FAQ 83. Windows 杀毒误报

Electron 常见。用官方签名 Release。

### FAQ 84. CI 里 npm test 失败 SQLITE

server 测试关闭 fileParallelism。不要自行改成高度并行写同一路径。

### FAQ 85. 贡献时文档语言

改英文 docs/en 再同步 zh-cn，见 TRANSLATION.md。

### FAQ 86. 如何加新提供方

docs 里 adding-a-new-provider：类型、register、测试、目录，不要写死进 legacy 迁移。

### FAQ 87. 密钥健康 unknown 很久

调度还没轮到，或出站全失败。点单钥匙重试。

### FAQ 88. Playground 会话丢了

换了 db 路径或桌面与 Docker 不是同一份库。

### FAQ 89. max_tokens 被截断

请求级 REQUEST_MAX_TOKENS_BUDGET 或模型上下文估算。看 413 正文。

### FAQ 90. 连续失败 503

MAX_CONSECUTIVE_UPSTREAM_FAILS 熔断。池子真的全挂时宁可快失败。

### FAQ 91. 想看每次尝试耗时

FALLBACK_DETAIL_HEADER=1，注意头里会有提供方错误文本。

### FAQ 92. HSTS 为什么不开

本地 HTTP 单用户代理，开 HSTS 会把局域网安装打进 https 黑洞。

### FAQ 93. CORS 拦了自己的前端

DASHBOARD_ORIGINS 加上你的来源。默认只放行 Vite localhost。

### FAQ 94. body 413

REQUEST_BODY_LIMIT_MB 默认 25。超长视觉会话再调，同时依赖图像规范化。

### FAQ 95. seed 不生效

透传给支持的提供方；很多免费端点忽略 seed。不是网关丢了。

### FAQ 96. logprobs 没有

同样取决于上游。网关不伪造。

### FAQ 97. n=3 被拒

明确不支持。自己循环三次请求。

### FAQ 98. moderations 404

未实现。

### FAQ 99. 多租户怎么做

不要改几行就上公网。需要全新鉴权、配额隔离、法律条款。本项目设计反对这一点。

### FAQ 100. 树莓派 1GB 内存行不行

可以跑，关掉持久化缓存、降低分析保留、不要 Fusion K=8。


## 8. 功能对照速查

| 你想做的事 | 去哪里 |
|---|---|
| 加密钥 | 仪表盘 → 密钥 |
| 拿统一密钥 | 密钥页顶部 |
| 改 auto 顺序 | 回退链 / 策略 |
| 接 Claude Code | `npx freellmapi setup-claude` |
| 看为什么选了这个模型 | 响应头 + 惩罚检查器 + 分析 |
| 省额度 | 关 Fusion、开压缩、降 K、加缓存（仅重复请求） |
| 提高成功率 | reliable 策略、更多密钥、避开高峰或打开 peak hours |
| 本地模型优先 | custom 端点 + 单独 profile |
| 备份 | 设置备份 / 拷贝加密后的 db 与密钥文件 |
| 升级 | install.sh 再跑或拉新镜像，保留 .env |

## 9. 请求头与模型字符串速查

- `Authorization: Bearer freellmapi-…`
- `model`: `auto` / `auto:fast` / `auto:<profile>` / 具体 id / `fusion`
- `X-FreeLLM-Cache: on|off`
- `X-FreeLLM-Compress: lossless|standard|aggressive`
- `Idempotency-Key: <uuid>`
- 响应：`X-Routed-Via`、`X-Fallback-Trail`

## 10. 手册维护说明

本使用说明书与源码中的路由、设置键、CLI 子命令对齐。新增提供方或新 setup-* 生成器时，应在第 3 章与第 4 章各加一小节，并在 FAQ 增补至少一条失败模式。


---

## 11. 仪表盘逐页操作手册（详细）

本章按你打开 http://localhost:3001 之后实际看到的导航顺序书写。每一页都说明「进来做什么、按钮含义、失败时看哪里」。所有文案以界面 i18n 键的中文理解为准；若你把界面换成其它语言，控件位置不变。

### 11.1 登录与首次设置

浏览器从本机打开时，若库里还没有用户，你会看到创建账户表单：邮箱与密码。密码哈希存在 `users` 表，明文不会进 SQLite。从另一台机器打开同一端口时，还必须填写服务器日志里打印的一次性 setup code——这是为了防止你刚用 `HOST_BIND=0.0.0.0` 把端口暴露到局域网时，被别人抢先注册成管理员。桌面应用走另一条路径：托盘里已经有统一密钥，不必先登录；但若你在桌面里仍然打开浏览器版仪表盘，登录门会按「是否在 Electron 壳内」决定。改邮箱或改密码在头像菜单里。忘记密码时，服务器打印一次性重置码；桌面用户请打开日志文件夹里的 `freeapi.log`，因为从访达启动的应用没有终端。

登录成功后会发 session cookie。不要用把 cookie 拷到脚本里当 API 密钥——数据面必须用 `freellmapi-` 开头的统一密钥。Session 过期后仪表盘会跳回登录，正在进行的试验台流式输出会中断，需要重发。

### 11.2 模型页（聊天表）

这是默认落地页。表格列出目录里的聊天模型：提供方、显示名、上下文窗口、能力徽章（工具/视觉）、限额、启用开关。搜索框过滤 id 与名称。排序可按智能、速度、名称。统一分组打开时，同一逻辑模型会折成一行，展开后看到各提供方端点。点行进入模型详情：限额、怪癖、最近延迟、是否在当前链上。

「启用」关掉的模型不会进入 `auto`。若你关掉了当前链上全部模型，下一请求会 400 空链，而不是偷偷用全目录——这是有意的，以免你以为自己在用精选链，实际打到了实验模型。新模型经目录同步出现后，若未打开自动加入配置档，它们只出现在目录表，不会出现在链上，需要你手工拖进去。

月度额度条来自目录的 `monthly_token_budget` 与本地记账，是估算不是账单。没有申报月度预算的模型这条是空的，不代表无限。

### 11.3 嵌入 / 图像 / 视频 / 音频页

这些页与聊天表分开，因为路由入口不同。嵌入页登记 embedding 模型并显示维度提示（各家维度不同，不能混进同一个向量库而不做迁移）。图像页可试 `/v1/images/generations` 的提示词与尺寸。音频页试 TTS，可下载。视频页对应实验能力，若提供方或目录未启用，生成会失败，请看错误而不是反复点。详情页与聊天模型详情类似，含自定义媒体端点编辑。

### 11.4 Fusion 页

用来调面板大小、评审超时、看最近 Fusion 花费的 token 倍数。请记住 Fusion 默认不是给 Agent 工具循环用的。你在这里调高 K，试验台里一次「写首诗」会打出 K 路草稿加评审，免费额度会成倍消失。建议日常 K=3 或 4，只有要质量对比时再升高，且硬顶 8 不能突破。

### 11.5 试验台

左侧是会话列表，数据在 `playground_conversations`。新建会话、重命名、删除都只影响本机库。中间是 composer：支持 Markdown 发送、换行与快捷键发送、附件、语音（若有转写模型）。右侧设置轨选模型（含 auto 与 fusion）、温度、max_tokens。流式输出用 SSE，中途点停止会发 abort，路由器把其中止与故障转移区分开，不会把你的取消当成上游 500 去冷却密钥。

把试验台当成「生产问题复现器」：CLI 里奇怪的工具调用，先在这里用同一模型字符串重放。图像附件会走规范化，所以试验台发得过的大截图，Agent 侧通常也能过，除非 Agent 自己把图缩得不认识。

### 11.6 密钥页

这是整个系统的控制中枢。顶部统一密钥只显示一次完整值，之后掩码，丢失就轮换。下面按提供方分组：每组有密钥列表、健康点、模型清单、配额信号与展望。添加密钥对话框会按前缀猜测平台，也可强制指定。自定义提供方要填 base URL 与至少一个模型。发现模型会打上游 `/v1/models`（若对方需要鉴权）。测试模型对话框对勾选的 id 发最小 chat，用来确认不是「列得出来但一调就 404」。

导入区接受 .env 风格文本，先 preview 再 import selected，避免把注释行当密钥。导出要明白：导出文件含可调用的上游密钥，等同备份，应与 ENCRYPTION_KEY 一样进密码管理器，不要进 git。

出站代理、Anthropic 兼容说明、客户端配置档、备份、配额条都在本页后部。若页面很长，用命令面板（Ctrl/Cmd+K）搜「proxy」或「backup」。

### 11.7 智能体页

把 CLI 配方、兼容矩阵、URL 令牌入口集中展示，避免只在 README 里。按文档生成配置；生成前在本页确认网关 URL 与统一密钥。URL 令牌给不能设头的客户端：签发、复制带令牌的 URL、随时撤销。撤销立即生效，适合误贴到聊天软件之后。

### 11.8 分析页与日志页

时间窗选择影响所有图。成功率低时先看错误分类是 429 还是 401，再决定是加密钥还是修密钥。p95 飙高常常是某一家排队，换 fastest 或避开该平台。日志页级别过滤、提供方过滤、时间格式。只存 warn/error 到库，info 级别在内存环，重启会丢——这是设计，免得树莓派被日志写死。

### 11.9 Premium 页

显示目录源是 live 还是月度快照、上次同步时间、签名是否通过。免费安装延迟是产品行为不是 bug。不要为「邻居已经有某个新模型而我没有」反复重启容器，等同步窗口或订阅 live。

### 11.10 设置对话框

从顶栏齿轮进入。分组大致包括：通用（语言、主题、更新检查）、路由（策略、自定义权重、高峰、任务偏向、探索开关）、护栏（token 预算、连续失败熔断、工具参数校验）、压缩、缓存、MCP、分析保留的只读提示（真正的天数在 env）。改完即写入 settings 表，下一请求生效，不必重启。启动项（ENCRYPTION_KEY、PORT）不会出现在可编辑列表里，只能改 .env。

### 11.11 命令面板与快捷键

Ctrl/Cmd+K 打开。可跳转页面、聚焦搜索、打开设置。Mac 与其它平台修饰键不同，导航栏有提示。这在密钥页很长时比滚动可靠。

### 11.12 空状态与新手引导

没有密钥时会看到 getting-started 引导：加密钥、复制统一密钥、试一次 chat。不要跳过「至少一家健康密钥」这一步去先调 Fusion，否则只会得到耗尽错误。

---

## 12. 协议与客户端详细用法

### 12.1 OpenAI Chat Completions 逐项

把官方 SDK 的 `base_url` 设为 `http://localhost:3001/v1`（注意包含 /v1，与 Anthropic 根 URL 习惯不同）。`api_key` 用统一密钥。`model` 推荐先 `auto`。`stream=True` 时按 SSE 读，最后一帧可能带 usage（若请求了 `stream_options.include_usage`）。`tools` 数组与 OpenAI 相同。`tool_choice` 支持 none/auto/required/具名函数。`response_format` 在不支持的平台会被路由器直接跳过该平台，避免 400 循环。`temperature`、`top_p`、`stop`、`seed`、`logprobs` 能透传就透传，上游忽略也不报错。不要传 `n`>1。视觉用 content 数组，`image_url` 可 http 或 data URL。

Python 与 Node 示例见第 4 章。其它语言凡能改 base URL 的官方库都可以。若库写死了 api.openai.com 且不能改，用反向代理或换库。

### 12.2 Completions 与编辑器幽灵文本

Continue 等会打 `/v1/completions`。网关把它转成 chat 再路由，因此限额与聊天共用。prompt 字段不要再包一层 chat 模板，除非你的编辑器已经包了。max_tokens 对补全要小，否则浪费额度且延迟差。

### 12.3 Responses API（Codex）

Codex CLI 的主路径。setup-codex 会写对。手工对接时看 OpenAPI。流式工具调用一旦发出 delta 就不能在中途因参数校验切模型，这是协议限制，不是漏测。非流式仍可校验工具参数并换模型。

### 12.4 Anthropic Messages 与 Claude Code

`ANTHROPIC_BASE_URL` 指向网关**根**（无 /v1），`ANTHROPIC_AUTH_TOKEN` 为统一密钥。模型 id 用网关的 auto 或具体 id。system、thinking、工具结果块由 anthropic-map 翻译。若 Claude Code 报模型找不到，先 `curl /v1/models` 看 id，再 `--model` 钉死。零留存用 `freellmapi launch`，密钥不写进 `~/.claude`。

### 12.5 Gemini CLI

走 `/v1beta` generateContent。不要把 Gemini CLI 配成 OpenAI 兼容模式还指望 grounding 原样工作。搜索接地是 Gemini 特有请求字段，其它提供方没有等价物，路由器不会假装有。

### 12.6 Ollama 模拟

在设置中打开后，Zed、JetBrains 把 Ollama 地址指到本网关。tags 列表来自当前可用模型。这是模拟不是真 Ollama：没有 `ollama pull`，模型来自你配置的密钥。关模拟则这些客户端连不上，但不影响 /v1。

### 12.7 嵌入、图像、音频

embeddings 输入文本数组，输出向量，模型必须是 embedding 表里的 id，不能用 chat 模型名碰运气。图像生成 body 与 OpenAI 类似，`prompt` 必填。TTS 输入文本，返回音频字节。这些模态的故障转移链与聊天分开，缺模型时不会降级去用聊天模型画画。

---

## 13. 环境变量逐项说明（操作员版）

下列每一项都来自 `.env.example` 与 `lib/config.ts`。改 env 通常要重启进程；能热更新的会注明「也可在仪表盘改」。

ENCRYPTION_KEY：64 位 hex，生产必填。用来加密库里的上游密钥。泄漏等于泄漏全部提供方密钥。轮换用 server 脚本 `rotate-encryption-key`，不要只改 env 而不重加密行。

PORT 与 HOST：监听。Docker 场景 HOST_BIND 才影响「宿主机哪张网卡被发布」。很多人把二者搞混：容器内可以听 ::，但 compose 只发布 127.0.0.1:3001。

PROXY_RATE_LIMIT_RPM 与 ADMIN_RATE_LIMIT_RPM：入口防刷，不是上游免费层。设 0 关闭。局域网多人误用同一实例时，应保持默认而不是关闭。

REQUEST_BODY_LIMIT_MB：视觉会话变长时先开图像规范化，再考虑加这个值。盲目加到 200 只会让内存被 base64 撑爆。

IMAGE_NORMALIZE 一组：总开关、长边、阈值 KB、JPEG 质量。关掉后，webp 上游 400 只能自己解决。

PROXY_URL / PROXY_MODE / FETCH_RELAY_TOKEN / NO_PROXY：出站。Docker 访问宿主机代理必须 host.docker.internal。Fetch Relay 只接受 http/https 中继，不是 SOCKS。

PROVIDER_DAILY_REQUEST_CAP_*：在提供方自己的限额之外再加本地闸，适合 OpenRouter 这种共享日限额。

PROVIDER_TIMEOUT_* 与 STREAM_STALL：前者管「等到首字节」，后者管「已经出字但卡住」。NVIDIA 与 Horde 往往要加大前者。

FALLBACK_TIME_BUDGET_MS：单次 HTTP 内不再开新尝试的墙钟。Agent 一轮工具若在 45 秒内跨了很多家仍失败，会耗尽。调大能多试，但客户端也容易先超时。

RESPONSE_CACHE 一组：精确缓存。Agent 循环几乎不命中。TTL、最大条数、最大温度、是否落盘。落盘是明文回答。

FREELLMAPI_COMPRESSION：off/lossless/standard/aggressive。请求头只能降级。

FREEAPI_BLOCK_PRIVATE_PROVIDER_URLS：仪表盘能被别人打开时必须 true。

REQUEST_ANALYTICS_*：保留天数、最大行、是否记 IP。树莓派把行数调小。

FREEAPI_DB_PATH 与 DIR_HARDENING：自定义路径时注意权限。不要指向 /tmp 还指望自动 chmod 0700。

FREEAPI_DB_BACKUP_*：路径或 URL、token、独立密钥、间隔。启动缺库会恢复，所以备份密钥要和主密钥一起保管。

FREEAPI_CONFIG_PATH/JSON：声明式密钥与策略。CI 注入，不要打进公开镜像。

FREELLMAPI_CONTEXT_HANDOFF：换模插入交接。默认关，避免污染不需要的客户端。

FREELLMAPI_UPDATE_CHECK：可关，完全离线时建议关。

DASHBOARD_ORIGINS：仪表盘与 API 分域时加。

CSP_UPGRADE_INSECURE_REQUESTS：HTTP 局域网保持自动。

TRUST_PROXY：反代后看真 IP。公网不要 true 除非你信任整条链。

MODEL_ROUTING_OVERRIDES：JSON 乘数。0 让 auto 永不选该 id。

CLIENT_DIST 与 FREEAPI_ENV_PATH：嵌入桌面或定制包时用。

MODELSCOPE_VALIDATE_CACHE_MS：防健康检查烧魔粒。

VALIDATE_TOOL_ARGUMENTS、REQUEST_MAX_TOKENS_BUDGET、MAX_CONSECUTIVE_UPSTREAM_FAILS：护栏，默认偏保守（校验关、预算关、熔断关），按需打开。

MAX_CONCURRENT_REQUESTS_PER_KEY：默认不限制。某家并发 429 时再设。

---

## 14. CLI 子命令与生成器详解

`npx freellmapi --help` 列出命令。全局选项：`--url` 默认环境变量 FREELLMAPI_URL 或 localhost:3000（注意与 server 默认 3001 可能不同，要以你实际端口为准）、`--api-key`、`--profile`、`--model`、`--dry-run`。

setup-* 命令会拉 `/v1/models?available=true`，所以网关必须在跑且密钥有效。dry-run 打印 diff 不写盘。写入前备份已存在文件，且不覆盖用户已手写的未知键——这是生成器的核心承诺，升级 CLI 不应毁掉你的自定义 env。

launch / launch-codex：spawn 子进程，把密钥放进环境变量，进程退出即无盘留存。适合演示电脑。doctor：探 livez，检查已知工具配置路径，退出码区分「网关死了」和「配置漂移」。`--timeout` 必须是毫秒数字，写 `5s` 会被拒绝而不是默默当默认值。

profile 名只允许字母数字点下划线连字符，不能是 `.` 或 `..`，防止写出家目录以外。

各生成器目标路径因工具而异：Claude 在 `~/.claude/settings.json` 或 profiles 子目录；其它见 tools.ts 与客户端文档。生成后用对应工具自己的命令验证（`claude`、`codex`、`aider`）。

---

## 15. 备份、升级与迁移操作步骤

升级 Docker：在同一目录再跑 install.sh 或 compose pull && up -d。先确认数据卷还在。升级前复制 .env。若发行说明提到迁移，看 `db:migration:status`。legacy_baseline 不能 down，回滚只能恢复备份文件。

手工备份：停进程，拷贝 db、-wal、-shm、.encryption-key、.env。或使用内置加密备份，不必停，但要理解备份是周期快照不是同步复制。

迁到新机器：三样缺一不可——库、主密钥、.env 里你改过的项。权限改为仅所有者。先在新机器用同一 Node 大版本起，看日志迁移是否成功，再改 DNS。

轮换 ENCRYPTION_KEY：用官方脚本，它会解密再加密所有行。只改 env 会导致启动后全部密钥解密失败，健康检查全 invalid。

---

## 16. 安全使用清单

只在本机或可信局域网监听。公网暴露必须前面加 TLS 与至少一层你自己的鉴权，并且明白统一密钥仍是全能的。定期轮换统一密钥。导出文件当密码一样管。自定义 URL 防 SSRF。不要把 docker logs 贴到公开 issue。贡献代码用测试密钥。桌面与 Docker 不要混用两个库还以为会话会同步。

条款：叠加免费层用于个人开发通常是项目存在的理由，但转售、给同事共用端点、把免费层当生产 SLA，会同时违反多家 ToS 与本项目的设计范围。

---

## 17. 性能与容量规划（自托管）

空闲内存约 40MB 量级。缓存条目、分析行、并发流是三大变量。树莓派：关持久化缓存、分析行 1 万、不要 Fusion K=8、不要同时 20 路视觉。x86 小 VPS：注意出站带宽，base64 图像会打满廉价 VPS 的流量包。并发 Agent：为敏感提供方设每密钥并发帽，避免自己 429 自己。磁盘：WAL 模式 db 会增长到分析上限，靠保留任务裁剪。

---

## 18. 功能×客户端矩阵（使用层）

Claude Code：Messages + setup-claude + launch。Codex：Responses + setup-codex。Cursor：OpenAI base URL 或 setup-cursor。Cline/Roo/Kilo：各自 setup，模型 auto。Aider：setup-aider。Continue：chat 与 completions。Gemini CLI：v1beta。Zed/JetBrains：Ollama 模拟。Goose/Qwen/Crush/Hermes/OpenClaw/AtomCode/DeepSeek Harness：见 tools.ts 生成物。任意 curl：/v1/chat/completions。

选不到的客户端：能改 OpenAI base URL 就能用；只能走 Azure 专有头的，需要自己垫一层，本项目不模拟 Azure 部署名。

---

## 19. 常见问题 100 问（详解）

下面每一问都给出原因、处理步骤与预防。先用页内搜索关键词。

### FAQ 1　为什么推荐 Docker 而不是直接 npm start？

Docker 把 Node 版本、原生模块 better-sqlite3、客户端静态资源与入口脚本一次打好，避免本机 Node 25、Python 环境或缺编译链导致安装失败。桌面用户若不想碰容器，应下载官方安装包而不是自行 npm。开发者改代码才用 npm run dev。混用时注意两套进程不要抢 3001 和同一 db 文件，否则会出现 SQLITE_BUSY 或密钥看起来「时有时无」。生产用镜像还方便你只备份卷和 .env。

### FAQ 2　install.sh 会不会覆盖我的密钥？

设计上重复执行保留 .env 与加密密钥，只更新容器镜像。尽管如此，第一次运行前仍应自己备份。阅读脚本再 pipe 到 bash 更稳妥：curl 到文件、less 查看、再 bash。自定义 FREELLMAPI_DIR 时确认目录归属是你的用户，以免 root 写出你事后改不了的文件。

### FAQ 3　默认端口 3001 被占用怎么办？

改 PORT 并重启。所有客户端的 base URL、CLI --url、Ollama 模拟地址、桌面配置都要一起改，漏改一个就会出现「仪表盘好用但 Claude 连旧端口」的假活。Docker 还要改映射。改完用 curl /livez 验证。不要让两个 FreeLLMAPI 实例默默绑不同端口却共用一个库。

### FAQ 4　IPv6 与 IPv4 我该听哪个？

默认 HOST=:: 双栈，IPv6 不可用的主机应自动回退。若你发现只能用 [::1]:3001 打不开 127.0.0.1:3001，是操作系统双栈策略问题，把 HOST=0.0.0.0 或 127.0.0.1 即可。纯 IPv6 的 Docker 还要在 daemon.json 开 ipv6，否则容器内连 DNS 都没有。

### FAQ 5　仪表盘语言如何固定成简体中文？

设置里选 zh-CN，或让浏览器 Accept-Language 靠前。locale 文件是 client/src/i18n/locales/zh-CN.json。若某键显示成英文，是翻译缺键，开发模式 check:i18n 会失败；发行版偶发缺键会回退。不要手工改构建后的 dist 文案，升级会被覆盖。

### FAQ 6　暗色主题刷新为什么闪白？

首屏主题必须在 CSS 之前由 index.html 内联脚本写入。该脚本的 SHA 写进 CSP。你若用广告拦截或改了 index.html 却没改哈希，脚本被拦，就会闪白甚至白屏。修复：恢复官方 index.html，或同步更新 INLINE_BOOTSTRAP_SHA 与测试。

### FAQ 7　命令面板搜不到某个设置？

有的项只在 .env，不在设置对话框。命令面板主要跳转页面与打开已实现的设置项。出站代理在密钥页后部不在齿轮里。用文档搜环境变量名比在面板里硬搜更有效。

### FAQ 8　为什么没有「用户管理」？

单用户软件。第二个人应自己部署实例。共享统一密钥等于共享你所有上游账号。若你需要团队，应寻找多租户网关，而不是在本项目开一个 users 表就上线。

### FAQ 9　健康状态 unknown 是否不能用？

unknown 表示还没探成功，不一定坏。路由器在没有明确 invalid 时仍可能尝试，取决于实现细节；为稳妥请点一次检查。长期 unknown 看 last_health_error 与出站代理。不要在 unknown 时删除密钥，先看网络。

### FAQ 10　rate_limited 的密钥还会被选中吗？

冷却期内该模型+密钥会被跳过。冷却结束后自动回来，无需手工。若整家平台日限额触顶，会看到提供方级跳过。惩罚分会让它在冷却外也暂时排后，但不会永久除名，以便恢复。

### FAQ 11　invalid 之后我改了密钥为什么还是 invalid？

保存新密钥应重新探测。若你是直接改数据库密文，状态不会动。用界面编辑。ModelScope 类会缓存验证结果，短时间内仍显示旧状态，等缓存过期或重启。401 被分类为 invalid 后，网关避免用一把死密钥把其它请求打爆。

### FAQ 12　如何确认加密真的生效？

打开 SQLite 看 api_keys.encrypted_key，应是密文不是 sk- 开头。没有 ENCRYPTION_KEY 的开发模式会在库旁写 .encryption-key。不要把该文件和 db 一起发到聊天软件。轮换脚本跑完，指纹变化，旧备份需旧密钥才能解。

### FAQ 13　掩码密钥如何判断是哪一把？

看标签和前缀。导入预览也显示 prefix。不要靠「最后四位」在多把类似 Groq 键之间猜，给每把写 label=备用-2。导出后再导入，重复检测会标 duplicate。

### FAQ 14　模型范围选错导致 auto 变空？

若唯一健康密钥的 scope 不含链上模型，路由会报无可用密钥。把 scope 设回全部，或把链改成范围内的 id。scope 是密钥维度不是全局禁用模型。多把密钥时只有被限制的那把受影响。

### FAQ 15　自定义端点要不要带 /v1？

要与上游约定一致。Ollama OpenAI 兼容接口通常是 http://host:11434/v1。少写 /v1 会 404。Docker 里 host 不能是 127.0.0.1。URL 策略拦截元数据地址，写 169.254.169.254 会被拒，这是保护不是误报。

### FAQ 16　发现模型按钮没反应？

上游 /v1/models 可能不需要密钥就返回公开列表，或相反需要密钥。看网络错误。有的上游列表含付费模型，登记后一调用就 402，需要你用范围或手工删。发现任务尊重墓碑，你删过的自定义 id 不会被悄悄写回。

### FAQ 17　导入时平台识别错了怎么办？

预览里改 platform 再导入。前缀启发式不是合同。错当成 custom 会缺 base URL。导入后看分组是否在预期提供方下，用测试模型打一条。

### FAQ 18　导出 CSV 用 Excel 打开乱码？

UTF-8 无 BOM 时老 Excel 会乱。用 Numbers、LibreOffice 或先转码。CSV 里有密钥，不要发邮件。JSON 更适合再导入。

### FAQ 19　统一密钥能当提供方密钥填回去吗？

不能。统一密钥只通过本网关验证。把它填进 Groq 控制台毫无意义。相反，把 Groq 密钥当成统一密钥用会出现 401。看前缀：freellmapi- 对本网关，gsk_ 对 Groq，AIza 对 Google，以此类推。

### FAQ 20　auto 和具体模型如何选择？

日常 Agent 用 auto，让额度与故障转移工作。演示或对比用具体 id。CI 要确定性时用具体 id 或 reliable 策略加短链。fusion 只用于允许花多倍额度的质量任务。auto:profile 用于同一把统一密钥服务多种工作负载。

### FAQ 21　priority 策略下拖拽没有立刻变化？

热更新应立刻影响下一请求。若你拖的是未启用行或看的是分组折叠行，检查是否改到了当前活动配置档。多配置档时确认顶栏当前档。客户端若钉死了模型，与链顺序无关。

### FAQ 22　balanced 为什么不总选最聪明的？

权重 50% 可靠性。聪明但上午已经失败多次的模型会让位。这是为了让 Agent 循环不要卡在「最强但正在 429」的端点。要最聪明用 smartest，并接受下午可能更快触顶。

### FAQ 23　高峰时段时区填错会怎样？

窗口按 IANA 名解释，填错会回退 UTC。你若在中国晚上想降速权，应填 Asia/Shanghai 而不是靠服务器（容器常是 UTC）的本地钟。start=end 表示空窗口，不会变成全天高峰。

### FAQ 24　探索开关该不该开？

开：Thompson 采样让未知新模型有机会。关：更利用已知分数。长任务建议关探索以免来回换模型。短请求开探索有助于发现新目录条目其实很快。

### FAQ 25　社区先验安全吗？

若启用，用的是聚合成败不是你的提示词。仍建议看清设置再开。本地样本会稀释社区值，老实例几乎等于只用自己的数据。完全离线应关。

### FAQ 26　如何解释 X-Fallback-Trail？

从左到右是尝试过的跳。含平台、模型、失败原因摘要。最后成功的一跳不一定出现在 trail 的失败列表里，同时看 X-Routed-Via。打开 Detail 头会看到耗时，但可能含上游错误文本，注意分享。

### FAQ 27　耗尽信息里的 ETA 不准？

来自最近冷却到期时间，不是提供方官方日历的完美镜像。UTC 日切的日限额可能比 ETA 更关键。把 ETA 当「至少再等这么久别死循环」而不是精确闹钟。

### FAQ 28　为什么有时 429 有时 503？

额度耗尽偏 429，连续上游失败熔断偏 503。客户端应都重试，但 503 更可能是池子坏了要换本地模型，429 更可能是等窗口。不要对 401 重试同一密钥。

### FAQ 29　流式输出重复或乱序？

先排除客户端 SSE 解析 bug。网关会尽量透传。半流式失败后不会在同一响应里换模型续写，以免两家模型各写一半。若看到截断，看 stall 超时与上游断开。重试应是新的 HTTP 请求。

### FAQ 30　工具调用被执行了两次？

那是 Agent 侧重试或并行子代理，不是网关把一次调用拆两次（除非你并发打了两次 HTTP）。幂等键可保护非流式重复提交，但进行中的并发相同键仍可能竞态。工具副作用要在 Agent 层做去重。

### FAQ 31　vision 模型被跳过？

内容里没有被识别为图像、或模型 supports_vision=0、或图像规范化失败。看诊断 no vision support。纯文本请求不会只为了「它能看图」去选视觉模型。

### FAQ 32　压缩导致漏掉系统提示里的规则？

lossless 不应丢掉规则。standard 可能滤工具输出和过期文件读。aggressive 可能按相关性砍旧轮。把关键规则放在稳定 system 前缀；重复出现的 system 会被冻结防压坏。出问题改回 lossless 并开保真统计对照。

### FAQ 33　缓存命中但答案不对场景？

精确哈希意味着提示完全相同。若你改了不可见空白或工具 schema 顺序，会 miss。命中却「不对」通常是你以为提示不同其实相同，或 TTL 内上游模型已更新但你还想要新口吻——这正是缓存的代价，关之。

### FAQ 34　如何给 CI 单独限额？

目前主要靠独立部署或接受统一密钥全能。派生令牌是后续方案。眼下可用反代按路径限流，或 PROXY_RATE_LIMIT 配合独立端口。不要把根密钥打进 fork 的公开仓库。

### FAQ 35　Playground 和 API 用量是否分开？

不分。试验台走同一代理同一账本。你在界面里 Fusion 一首诗，一样消耗 Groq 日限额。演示前先看限额条。

### FAQ 36　分析里的费用节省是什么？

估算值，把 token 乘上某种对照单价，不是你的信用卡账单。免费层真实成本是时间与额度。不要用这个数做财务审计。

### FAQ 37　日志会记录完整 prompt 吗？

请求分析默认记 token 与错误摘要。试验台会话表有对话内容。server_logs 是 warn/error。仍避免在 prompt 里贴生产密钥。关 REQUEST_ANALYTICS_LOG_CLIENT 可少记 IP。

### FAQ 38　MCP 连上后智能体会改我的链吗？

当前 MCP 侧重查询。不要授予不必要的写权限。未来若加 set_profile，应只在本机使用。把 MCP 暴露到公网等于把控制面暴露出去。

### FAQ 39　桌面和 Docker 同时开会怎样？

抢端口、抢库或各写各的。选一种当权威。桌面适合笔记本托盘；服务器用 Docker。数据不自动同步。

### FAQ 40　macOS 提示未签名？

用官方 Release 的已公证构建。自己 dist 的包用于开发。自动更新依赖签名，未签名检查器可能显示有新版本却升不上去，属预期。

### FAQ 41　Windows 防火墙弹窗？

允许本地回环即可。若你 HOST_BIND 到 0.0.0.0，防火墙询问的是局域网入站，按你的信任决定。不要在咖啡厅 Wi-Fi 放开。

### FAQ 42　Linux systemd 如何写？

WorkingDirectory 指向安装目录，EnvironmentFile=.env，ExecStart=node server dist，User 非 root，Restart=on-failure。保护 .encryption-key 权限。日志进 journald，注意红线仍要避免把 env 打进 ExecStart 命令行。

### FAQ 43　PM2 集群模式可以吗？

多进程写同一 SQLite 不是本项目设计点。用 fork 单实例。要水平扩展应拆实例拆库，而不是集群打同一文件。空闲内存很小，单进程通常够个人 Agent 用。

### FAQ 44　如何验证目录签名失败时的行为？

断开外网，路由器应用上一份目录继续服务。仪表盘 Premium 页显示同步错误。这比「清空模型表」安全。修好网络后等调度或手动触发同步。

### FAQ 45　新模型三天了还没有？

免费安装可延迟约 30 天。Premium live 当天。再检查 tombstone 与 overrides 是否把模型按住了。社区文档里的模型名未必等于本目录 id。

### FAQ 46　如何永久去掉某个模型？

链上 disable、overrides、tombstone、或乘数 0。仅在表里 enabled=0 可能被同步策略影响，以文档中 catalog 同步语义为准。自定义模型删除走墓碑防发现任务复活。

### FAQ 47　OpenRouter 与直连 Groq 同时存在会双花吗？

auto 可能在两家之间选，不是一次请求打两家（Fusion 除外）。同一提示不会同时消耗两家，除非故障转移或 Fusion 面板。账单式双花不存在于单次非 Fusion 请求。

### FAQ 48　本地 70B 和云端 Flash 如何本地优先？

建 profile：最前 custom 本地模型，后面才是云。或乘数压低云模型。没有自动「难才上云」除非你用后续策略引擎。现在靠链顺序+priority 策略最直观。

### FAQ 49　Agent 把 fusion 设成默认模型怎么办？

改生成器默认是 auto，已尽量避免。若用户配置文件被写成 fusion，改回 auto。doctor 未来应警告。Fusion 对工具循环又慢又危险。

### FAQ 50　如何测试故障转移是否工作？

临时禁用第一模型或用错误密钥标 invalid，发请求看 trail。不要在生产密钥上故意打爆限额作测试，用自定义桩或测试库。单元测试是更好的位置。

### FAQ 51　429 学习到的限额会不会错？

learnLimitFromError 从响应体/头解析，解析错了可能把限额写得过小，模型会过早跳过。可在模型覆盖里改回。看到离谱 RPM=1 时检查上游错误原文（脱敏后）。

### FAQ 52　为什么凌晨又突然变聪明？

日限额 UTC 午夜重置，强模型回到链首。这是 README 写过的预期。按本地时区理解「凌晨」。高峰设置与此独立。

### FAQ 53　粘性会话导致一直用差模型？

粘性避免来回换。若第一次落到差模型，30 分钟可能钉住。换会话、等窗口、或钉死更好的 id。交接开启时换模会插入说明，但粘性本身会减少换模。

### FAQ 54　context handoff 会泄露系统提示吗？

注入的是简短交接，不是把上一模型的完整思维链给人看。仍可能包含对话主题。敏感项目保持关闭。

### FAQ 55　图像规范化会不会裁掉关键 UI？

按长边缩放，不裁切内容（GIF 只首帧）。文字小的截图缩放后 OCR/模型更难看清，必要时关规范化或提高长边 cap。透明 PNG 会尽量保留。

### FAQ 56　音频输入语言不对？

转写模型取决于你登记的 transcription 模型，网关不自动检语言。选对模型。浏览器麦克风权限拒绝时按钮无波形。

### FAQ 57　嵌入向量写入后模型换了怎么办？

旧向量与新模型不在同一空间。要换嵌入模型必须重建索引。网关不负责向量库。分析页不会警告这一点，需要你自己的数据管线纪律。

### FAQ 58　如何只允许 embeddings 不让聊天？

目前没有完善的派生令牌。可用防火墙只放行 /v1/embeddings，或单独实例只配嵌入密钥。把聊天模型全从链上移除。

### FAQ 59　docs 页 OpenAPI 与真实行为不一致？

以测试与源码为准，并提 issue。/v1/docs 方便探索不是规范的唯一真相。重大行为应有 vitest。

### FAQ 60　贡献一个提供方最少改哪些文件？

shared/types.ts 的 Platform、providers/index.ts 注册、可能专用适配器、keys 白名单、测试、文档。模型行进目录源不是进 baseline 迁移。阅读 adding-a-new-provider 中文文档。

### FAQ 61　测试失败显示 SQLITE_BUSY？

不要并行跑 server 测试文件打同一路径。官方 npm test -w server 已关 fileParallelism。自己写脚本时也要串行。内存库用例应完全隔离。

### FAQ 62　钩子 contributing-check 不让提交？

按 CONTRIBUTING 写说明。不要 --no-verify 绕过除非你知道在做什么。项目用钩子保证贡献说明与测试习惯。

### FAQ 63　如何阅读 debug 路由语义测试？

routing-semantics 与 proxy-* 场景测试是行为规范。改路由器先跑这些。测试名往往含 issue 号，是历史回归。

### FAQ 64　能否关闭所有出站以免泄漏？

自定义仅本地端点、关目录同步与更新检查、关健康检查对外探测（可能影响状态）。完全离线可用，但目录会过时，新模型不会出现。

### FAQ 65　树莓派 SD 卡损坏风险？

SQLite WAL 对突然掉电比想象中敏感。用质量好的卡，备份到网络。减少分析写入。考虑把 db 放 USB 盘。busy_timeout 不能防止卡满。

### FAQ 66　容器时区与高峰？

高峰用 IANA 字段，不依赖容器 TZ。分析时间线按存储的 UTC。看图时注意浏览器本地化。

### FAQ 67　多个 profile 如何给不同 Agent？

请求 model=auto:名字。生成器 --profile 改的是工具自己的配置目录，不是网关 profile。两个概念都叫 profile，容易混：网关 profile 是链；CLI profile 是 ~/.claude/profiles/foo。

### FAQ 68　客户端配置档（client_profiles）是什么？

按 UA 分类给默认参数，与回退链配置档不同。分析里能看到 caller。不要指望它能隔离额度。

### FAQ 69　如何解读配额展望 exhausted？

观察样本不足会是 insufficient_data，过期是 stale。exhausted 表示按最近速率会先用完再等到重置。加密钥或降速。低置信度不要当监控绝对阈值。

### FAQ 70　neurons / kudos 单位是什么？

提供方自己的计量，不是 token。账本用 metric 枚举区分。对比图时不要把 kudos 当 token 加总。Horde 尤其不同。

### FAQ 71　Sail 会不会花到付费？

月赠金用完且绑了支付方式会付费。健康检查与真实流量都算。不希望付费就不要绑卡或用完即停该密钥。适配器轮询后台 job，超时设置要理解 flex 模型更慢。

### FAQ 72　GitHub Models 配额在哪里看？

GitHub 文档与本目录限额。条款限实验。网关不会比官方显示更准。402/403 按冷却处理。

### FAQ 73　国内提供方 401 一直出现？

实名与账号绑定未完成（魔搭尤其）。token 能创建不代表能推理。先在官方 playground 成功再贴进本网关。不要用国际卡期望绕过实名。

### FAQ 74　为什么删除了 Moonshot 直连？

历史迁移：直连免费层变化，改由其它路由或目录表达。旧密钥行可能还在，应清理。看 changelog 与退役逻辑。

### FAQ 75　Fusion 评审模型会不会选面板里同一家？

评审走 auto 链，可能重叠。多样化主要在面板。若你只要风格差异，用 K 小一点并依赖 familyKey 打散。评审失败会尝试其它模型直到上限。

### FAQ 76　法定人数 2 是什么意思？

少于此数量的成功草稿不值得评审，直接返回幸存者。全失败才走普通错误。因此 K=4 但三家 429 时你得到的是单模型答案，不是 Fusion 合成，这是特性。

### FAQ 77　如何限制 Fusion 花费？

降 K、降超时、不要对长上下文用、单独 profile 不含贵模型。没有「Fusion 月预算」独立开关时，靠总限额与自觉。

### FAQ 78　试验台 Markdown 代码高亮错语言？

highlight 库自动检测，可指定围栏语言。与网关无关。大段日志可能卡 UI，属前端。

### FAQ 79　附件大小上限？

受 JSON 体限制与规范化阈值约束。超大文件应先在本地缩小。不要用试验台传视频当视觉帧，视频页是另一条生成 API。

### FAQ 80　停止生成后密钥还在冷却？

你的 abort 不应当 429。若实现误分类，看 isClientAbortError 测试。若冷却确实增加了，提 bug。正常停止只释放租约。

### FAQ 81　多标签页仪表盘会不会冲突？

settings 写同一库，后写覆盖。不要两个浏览器同时拖同一条链。React Query staleTime 30s，一个标签页改密钥，另一个最多几十秒内旧数据。

### FAQ 82　CORS 错误但 curl 正常？

浏览器才管 CORS。把仪表盘源加入 DASHBOARD_ORIGINS。用 file:// 打开前端必失败。开发用 Vite 5173 已默认放行。

### FAQ 83　反向代理后 WebSocket 失败？

当前主路径是 HTTP SSE 不是必须 WS。若你的反代缓冲了 SSE，会出现「一下全出来」。关缓冲、设合适 timeout，Caddy/nginx 对 SSE 有常见配方。keepAlive 75s 就是为反代准备的。

### FAQ 84　为什么不用 HSTS？

本地 HTTP 安装会被 HSTS 记住 https 然后再也打不开。这是给单用户本机代理的正确默认。公网反代上的 TLS 由你的 Caddy 开 HSTS，不由本进程开。

### FAQ 85　CSP 过严导致图表不显示？

默认 img-src self data，connect-src self。外链字体或分析脚本会被拦。不要随意 unsafe-inline script。改 CSP 必须改测试。

### FAQ 86　sharp 安装失败影响什么？

图像规范化不可用，大图与 webp 更容易上游 400。其它路由仍工作。可选依赖安装问题常见于无 libc 的发行版，用官方 Docker 可避开。

### FAQ 87　better-sqlite3 编译失败？

缺 build tools。Android/Termux 用 Node 内置 sqlite 工厂。官方 Docker 已编好。Electron 要 rebuild 原生模块，desktop 脚本已包。

### FAQ 88　如何确认正在用的是哪份源码？

看 FREELLMAPI_COMMIT_SHA（官方镜像注入）、仪表盘版本、或 docker inspect。自己绑卷覆盖 dist 时容易出现「改了 TS 但容器还在跑旧 dist」。

### FAQ 89　声明式配置和界面谁赢？

每次启动幂等应用 JSON，可能把你在界面里改的策略改回去。GitOps 就只改 JSON；手玩实例就不要设 FREEAPI_CONFIG_JSON。密钥重复导入应按幂等语义去重。

### FAQ 90　备份恢复后会话还在吗？

若备份含 playground 表则在。URL 令牌、用户哈希、用量都在库里。恢复等于时间倒流，冷却与限额窗口也会倒流，可能短暂允许额外请求，这是快照恢复的本质。

### FAQ 91　如何安全地分享「我的配置」？

分享策略名、链上模型 id、env 中非密钥项。不要分享 .env、导出文件、setup code、session。截图注意掩码。

### FAQ 92　Agent 报模型 context 不够？

路由器会跳过上下文窗口小于估算 prompt 的模型。估算是 chars/4 一类启发式，可能偏乐观或悲观。压缩、缩小图、换大窗口模型。413 是护栏预算不是窗口。

### FAQ 93　max_tokens 被悄悄改小？

请求级预算把未指定的输出封顶到剩余额度。或提供方自己 cap。看返回 usage 与截断 finish_reason。

### FAQ 94　seed 不同结果仍不同？

免费端点常忽略 seed。网关透传但不保证。要确定性只能选声称支持且实测支持的模型，并关探索。

### FAQ 95　logprobs 空？

上游没给。网关不编造。不要用空 logprobs 当错误。

### FAQ 96　能否做 A/B 两个模型？

客户端发两次或用 Fusion。没有内置分流百分比路由。可用两个 profile 手工交替。

### FAQ 97　如何暂停所有云端？

禁用云密钥或关掉链上云模型，只留 custom。桌面后续方案有一键，现在用密钥开关最快。健康检查仍可能偶尔探测，ModelScope 缓存可减少。

### FAQ 98　更新检查打 GitHub 限流？

匿名检查有限额。可关检查或设专用 token 环境变量（不要用随意 GITHUB_TOKEN）。企业代理要放行 api.github.com。

### FAQ 99　文档中英不一致听谁？

源码与测试、然后英文 docs、然后中文翻译。本手册按源码写。发现过时请对照 .env.example。

### FAQ 100　我只想当「本地 OpenAI 代理」要不要学路由？

至少加一把密钥、复制统一密钥、model=auto。其余用默认 balanced。等出现下午变笨或 429 再读策略章。不学也能工作，学了才能在免费层活得久。


## 20. 使用说明书收尾

若本文与界面不一致，以你运行的版本源码为准。建议把本手册与 `1DESIGN.md`（为什么这样设计）、`3SOLUTION.md`（下一步演进）、`SOURCE.md`（文件地图）一起放在仓库根目录，供操作员与二次开发者分工阅读：操作员读本手册第 1–6、11–19 章即可完成日常使用。

### 21.密钥轮换与多把备用

当你拥有同一提供方的多把密钥时，网关可以在一把进入冷却后改走另一把，而不必换模型。实践里给每把写清楚标签，例如「主号」「备用」「公司代理出口」。健康检查会按提供方交错，避免同一分钟把所有号都探一遍。若你发现备用号从未被选中，打开密钥选择策略 least-remaining 或检查 model_scope 是否把备用号限制死了。轮换统一密钥等于作废所有客户端里保存的旧 Bearer，需要同步改 CLI 配置与 CI 密钥仓库。上游密钥轮换则在提供方控制台吊销旧码后，在仪表盘编辑同一行，不要新增一行导致限额计数器分裂。导出旧备份在轮换后即过时，应销毁。桌面托盘展示的是统一密钥，不是上游密钥，避免两者混淆。长期来看，派生令牌会让「给 CI 的那一把」与「给人的那一把」分开，但在该功能落地前，用独立部署或严格的网络隔离来降低泄漏面。记住：多把密钥的价值是额度与故障隔离，不是把同一请求打到所有号上加倍速度，除非你在客户端自己做并行，那会加倍消耗免费层，通常得不偿失。

### 21.回退链的日常保养

目录每周都可能上下线模型。养成每周打开回退链看一眼的习惯：有没有 retired 还亮着、有没有新的工具模型没被自动加入、有没有实验性小模型挤在 smartest 策略下抢流量。配置档建议至少两份：coding 与 cheap。coding 放支持工具且上下文较大的模型；cheap 放速度快的补全模型。Agent 用 auto:coding，编辑器幽灵文本用 cheap 或 fastest。空链是 400，保养时不要一次全关再慢慢开，会让正在跑的 Agent 集体失败。拖拽顺序在 priority 策略下就是命运，在 balanced 下只影响并列时的稳定排序，不要以为拖到最上面就一定被选。启用开关才是硬排除。新模型自动加入很方便，但也把未经验证的端点放进生产链，对稳定性敏感的人应关掉并手工审核。把链当作「生产白名单」，把目录表当作「库存」。

### 21.分析页怎么读才有用

先看时间窗是 24 小时还是 7 天。成功率低于 90% 先看错误分类：429 是额度问题，401 是密钥问题，超时是网络或模型排队，4xx 其它可能是上下文或工具。平台拆分能告诉你是一家坏了还是大家都坏。大家都坏作出站网络或代理先查。p95 高而 p50 正常说明长尾，往往是某家 Horde 或冷启动。TTFB 高说明排队或思考模型。token 图上午陡下午平，符合强模型触顶。不要用分析页当模型质量评测，它不看答案对不对。最近调用表用来抓「是哪个客户端在打」，异常 UA 或 IP 意味着密钥泄漏。把保留天数设到你磁盘能承受的范围，树莓派不要 90 天全开。百分位需要足够样本，几个请求的 p95 没有意义。对比改策略前后，用同一时间窗同一工作日，否则周末与工作日的 Agent 负载不同。

### 21.日志页与 stdout 的分工

仪表盘日志页给人类看 warn/error。stdout 给 systemd/docker logs。贴 issue 前确认红线已遮密钥，仍要人工扫一遍 URL 查询串和自定义头。调试路由时看 X-Fallback-Trail 比看日志更快。把日志级别长期开到 debug 会拖慢并可能记下更多敏感字段，默认不要这么做。桌面用户没有终端，必须会打开日志文件夹。轮转策略在桌面文档里，不要手工删正在写入的文件。分析行与 server_logs 是两张表，清分析不会清错误日志。磁盘报警时先降保留再删文件。

### 21.压缩档位选择实验

用同一段长 Agent 轨迹在试验台分别 off、lossless、standard、aggressive 跑一遍，看答案是否仍执行工具。lossless 适合你只想省重复 JSON。standard 适合工具输出爆炸。aggressive 只在上下文真正放不下且你能接受旧轮被砍时用。请求头只能往下降，所以全局 aggressive 时客户端无法要求 off，这是操作员锁。保真门失败会放行原文，因此你看到「开了压缩但 token 没变」不一定是 bug。统计项在压缩 API，可看平均压缩比。不要和响应缓存同时误解：压缩改了提示，缓存键也会变。

### 21.缓存该不该开

精确缓存对「同一句翻译来回点」有用，对 Agent 几乎无用。打开前问自己：会不会有温度大于 0 的请求被冻住口吻？会不会把错误答案记住一小时？持久化会把回答写入 SQLite 明文，共享机器上这是隐私问题。TTL 一小时对新闻类问答过长，对稳定系统提示的分类过短。最大条数防止内存涨。按温度阈值只缓存近确定性调用是较稳的折中。用头 X-FreeLLM-Cache 在评测脚本里临时开，不必改全局。

### 21.视觉工作流

截图前先自己缩到可读，再交给网关规范化。长边 2048 对代码字体可能偏小，可提高 cap。GIF 只首帧，不要用 GIF 教模型看动画。多图请求确认模型 supports_vision。没有视觉模型时诊断会写 lacks vision。把视觉模型放进单独 profile，避免 auto 为了一张图把整条文本链打乱。自定义视觉端点同样要登记能力位，否则路由器当作纯文本模型跳过图。

### 21.嵌入工作流

选定一家嵌入模型后不要轻易换。把模型 id 写进你的向量库元数据。网关不保证同一 display name 跨提供方向量空间相同。限额与聊天分开，但仍消耗该密钥的提供方日限额如果它们共享账户级配额。批量嵌入注意 RPM，可在客户端排队。不要用 chat 模型假装 embedding。

### 21.媒体生成工作流

图像与 TTS 容易被 Agent 误调用烧光。能关则在链上不放媒体模型，需要时再开。SiliconFlow 等免费媒体有自己的限额。试验台先试再给 Agent 工具。视频实验性，失败看路由测试与媒体表是否有行。自定义媒体端点登记走另一套 register，不要填进 chat 模型表。

### 21.代理与 Fetch Relay

公司网用 HTTP/SOCKS 正向代理。GFW 场景 DNS 也污染时用 socks5h。Docker 访问宿主机代理用 host.docker.internal 且代理放行非 loopback。Fetch Relay 把出站改成你控制的 HTTPS 中继，适合不能任意 TCP 的环境。Relay URL 不能是环回除非策略放行。不要把 FETCH_RELAY_TOKEN 与统一密钥设成同一个字符串。NO_PROXY 填内部域名。本地 Ollama 默认绕过代理，除非你刻意走 ssh -D。

### 21.声明式配置的 GitOps

把不含密钥的策略 JSON 提交仓库，密钥用 CI 注入环境变量再渲染进 FREEAPI_CONFIG_JSON。每次部署幂等。不要让人在仪表盘点着玩同时又 GitOps，会互相覆盖。变更走 PR 审查乘数与链。密钥轮换改密钥仓库不是改 git。

### 21.桌面应用日常

托盘看统一密钥与是否在跑。悬浮窗看实时 QPS。日志文件夹解决一切「没有终端」问题。更新用官方签名包。Linux sandbox 问题用官方构建。不要用桌面去当 7×24 服务器，那是 Docker/systemd 的事。笔记本休眠有唤醒重探，仍建议合盖前停长任务。

### 22.1 场景：早晨开工先看限额条再开 Claude Code

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.2 场景：下午模型变笨时切换 reliable 并加本地模型

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.3 场景：演示前把策略改 fastest 并钉死一个健康模型

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.4 场景：CI 失败时用 doctor 看网关是否活着

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.5 场景：出差只带笔记本用桌面版与本地 Ollama

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.6 场景：公司代理更换端口后只改密钥页出站代理

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.7 场景：新同事要试用时让他自建实例而不是给统一密钥

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.8 场景：发现 GitHub 上误贴导出文件立即轮换全部上游密钥

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.9 场景：目录同步失败时不重启死循环而检查时钟与签名

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.10 场景：树莓派磁盘 90% 时降分析保留并关缓存持久化

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.11 场景：IPv6 宿舍网络改 HOST 与 Docker daemon

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.12 场景：Windows 开发用桌面版避免 WSL 端口混乱

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.13 场景：macOS 升级后重新放行防火墙与托盘权限

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.14 场景：给 Aider 与 Claude 同时用不同网关 profile

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.15 场景：评测脚本开缓存与温度 0

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.16 场景：视觉 Agent 单独 profile 与更高图像长边

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.17 场景：Horde 仅作为链尾避免拖死 p95

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.18 场景：ModelScope 关掉频繁健康检查缓存保持 24h

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.19 场景：NVIDIA 仅评估条款下只在试验台偶尔调用

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

### 22.20 场景：OpenRouter 加本地日限额闸防止一天用光

在该场景下，先确认网关进程与统一密钥仍有效，再用最少改动达到目的。具体来说，操作员应打开仪表盘对应页核对健康密钥、当前链与策略，避免同时改五个开关导致无法回滚。若请求失败，复制 X-Fallback-Trail 和状态码，对照本手册 FAQ 的 429/401/400 空链/超时条目。不要为了该场景关闭全部安全默认（SSRF 拦截、导出限流、CSP）。完成后把策略改回日常 balanced，以免第二天全部流量走演示配置。涉及密钥的步骤走仪表盘而不是手改 SQLite。涉及客户端的步骤先 dry-run 再写配置文件。涉及 Docker 的步骤记住 127.0.0.1 是容器自己。该场景的成功标准是：一次真实请求返回 200，Routed-Via 指向预期的提供方类别，限额计数有增加。若未达标准，停下来查网络与健康，而不是加大 Fusion K 或关掉限流账本。记录你改过的 env 与设置，便于下一次重复该场景。

## 23. 配置项白话对照

### `ENCRYPTION_KEY`

主密钥，丢了等于丢上游密钥。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `PORT`

监听端口默认 3001。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `HOST`

监听地址默认双栈。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `HOST_BIND`

Docker 发布地址默认本机。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `PROXY_RATE_LIMIT_RPM`

数据面 IP 限流。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `ADMIN_RATE_LIMIT_RPM`

仪表盘 IP 限流。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `REQUEST_BODY_LIMIT_MB`

JSON 体上限。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `IMAGE_NORMALIZE`

入站图像规范化总开关。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `PROXY_URL`

出站正向代理。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `PROXY_MODE`

forward 或 fetch-relay。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `NO_PROXY`

绕过代理的主机。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `PROVIDER_TIMEOUT_NVIDIA`

示例：按平台超时。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `FALLBACK_TIME_BUDGET_MS`

单请求故障转移墙钟。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `RESPONSE_CACHE`

精确缓存开关。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `FREELLMAPI_COMPRESSION`

压缩档位。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `FREEAPI_DB_PATH`

SQLite 路径。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `FREEAPI_CONFIG_JSON`

声明式配置。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `TRUST_PROXY`

反代信任。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `MODEL_ROUTING_OVERRIDES`

模型分数乘数。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `VALIDATE_TOOL_ARGUMENTS`

工具参数校验故障转移。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `REQUEST_ANALYTICS_RETENTION_DAYS`

分析保留天数。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `SERVER_LOGS_RETENTION_DAYS`

错误日志保留。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `FREELLMAPI_UPDATE_CHECK`

更新检查。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `DASHBOARD_ORIGINS`

额外 CORS 源。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `CSP_UPGRADE_INSECURE_REQUESTS`

CSP 升级 https。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `FREEAPI_BLOCK_PRIVATE_PROVIDER_URLS`

拦截私网自定义 URL。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `MODELSCOPE_VALIDATE_CACHE_MS`

魔搭验证缓存。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `MAX_CONSECUTIVE_UPSTREAM_FAILS`

连续失败熔断。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `RESPONSE_CACHE_PERSIST`

缓存落盘明文。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。

### `FREELLMAPI_CONTEXT_HANDOFF`

换模交接说明。修改后若该项属于启动配置则重启进程，若属于 settings 热更新则下一请求生效。不清楚时改完用一次 curl 验证，并看启动日志是否警告未知变量（拼写错误会被 env-drift 指出）。不要把真实密钥提交进 git。Docker 用户改的是 compose 目录的 .env 不是容器内临时文件。桌面用户注意 FREEAPI_ENV_PATH。该项与其它限额、超时、安全开关可能叠加，只改一个观察现象再改下一个。


## 24. 故障排查决策树（中文详述）

### 打不开界面

先 curl 本机端口，再查绑定地址、防火墙、容器是否重启循环、CSP 白屏、setup code。沿着这条树走时每次只改变一个变量，并记下 X-Fallback-Trail。把「我改了很多还是不行」变成「最后一次改的是超时，现象从 504 变成 429」，问题就会收敛。仍不行则用试验台最小 prompt「hi」排除 Agent 侧复杂性，再用 curl 排除浏览器。最小复现成功后再把 tools、图像、长上下文一项项加回。这是所有网关排障的同一方法，免费层只是多了额度这一维。不要在排障时打开 Fusion，它会让失败模式变成多路混合，更难看。不要在排障时关闭限流账本，否则你只能得到「打爆之后更糟」。排障结束恢复日常策略与压缩设置。

### 界面开但 API 401

统一密钥前缀、Bearer 拼写、用了上游密钥、令牌撤销、时钟导致 session 问题只影响仪表盘不影响 API。沿着这条树走时每次只改变一个变量，并记下 X-Fallback-Trail。把「我改了很多还是不行」变成「最后一次改的是超时，现象从 504 变成 429」，问题就会收敛。仍不行则用试验台最小 prompt「hi」排除 Agent 侧复杂性，再用 curl 排除浏览器。最小复现成功后再把 tools、图像、长上下文一项项加回。这是所有网关排障的同一方法，免费层只是多了额度这一维。不要在排障时打开 Fusion，它会让失败模式变成多路混合，更难看。不要在排障时关闭限流账本，否则你只能得到「打爆之后更糟」。排障结束恢复日常策略与压缩设置。

### 401 以外的失败

看状态码：400 空链或请求形状，413 体或预算，429 额度，503 熔断，超时看 stall 与 PROVIDER_TIMEOUT。沿着这条树走时每次只改变一个变量，并记下 X-Fallback-Trail。把「我改了很多还是不行」变成「最后一次改的是超时，现象从 504 变成 429」，问题就会收敛。仍不行则用试验台最小 prompt「hi」排除 Agent 侧复杂性，再用 curl 排除浏览器。最小复现成功后再把 tools、图像、长上下文一项项加回。这是所有网关排障的同一方法，免费层只是多了额度这一维。不要在排障时打开 Fusion，它会让失败模式变成多路混合，更难看。不要在排障时关闭限流账本，否则你只能得到「打爆之后更糟」。排障结束恢复日常策略与压缩设置。

### 成功但很笨

看 Routed-Via 是否 Small，限额是否见顶，压缩档，粘性，策略是否 fastest 误用在难题。沿着这条树走时每次只改变一个变量，并记下 X-Fallback-Trail。把「我改了很多还是不行」变成「最后一次改的是超时，现象从 504 变成 429」，问题就会收敛。仍不行则用试验台最小 prompt「hi」排除 Agent 侧复杂性，再用 curl 排除浏览器。最小复现成功后再把 tools、图像、长上下文一项项加回。这是所有网关排障的同一方法，免费层只是多了额度这一维。不要在排障时打开 Fusion，它会让失败模式变成多路混合，更难看。不要在排障时关闭限流账本，否则你只能得到「打爆之后更糟」。排障结束恢复日常策略与压缩设置。

### 成功但很慢

Horde/冷启动/思考模型/代理。换 fastest 或去掉链尾慢模型。沿着这条树走时每次只改变一个变量，并记下 X-Fallback-Trail。把「我改了很多还是不行」变成「最后一次改的是超时，现象从 504 变成 429」，问题就会收敛。仍不行则用试验台最小 prompt「hi」排除 Agent 侧复杂性，再用 curl 排除浏览器。最小复现成功后再把 tools、图像、长上下文一项项加回。这是所有网关排障的同一方法，免费层只是多了额度这一维。不要在排障时打开 Fusion，它会让失败模式变成多路混合，更难看。不要在排障时关闭限流账本，否则你只能得到「打爆之后更糟」。排障结束恢复日常策略与压缩设置。

### 工具不调

链上有无 tools 模型，拒绝 tools 的排序，校验开关，救援失败看原始文本。沿着这条树走时每次只改变一个变量，并记下 X-Fallback-Trail。把「我改了很多还是不行」变成「最后一次改的是超时，现象从 504 变成 429」，问题就会收敛。仍不行则用试验台最小 prompt「hi」排除 Agent 侧复杂性，再用 curl 排除浏览器。最小复现成功后再把 tools、图像、长上下文一项项加回。这是所有网关排障的同一方法，免费层只是多了额度这一维。不要在排障时打开 Fusion，它会让失败模式变成多路混合，更难看。不要在排障时关闭限流账本，否则你只能得到「打爆之后更糟」。排障结束恢复日常策略与压缩设置。

### 只在 Docker 失败

127.0.0.1、IPv6、代理 allow-lan、host.docker.internal。沿着这条树走时每次只改变一个变量，并记下 X-Fallback-Trail。把「我改了很多还是不行」变成「最后一次改的是超时，现象从 504 变成 429」，问题就会收敛。仍不行则用试验台最小 prompt「hi」排除 Agent 侧复杂性，再用 curl 排除浏览器。最小复现成功后再把 tools、图像、长上下文一项项加回。这是所有网关排障的同一方法，免费层只是多了额度这一维。不要在排障时打开 Fusion，它会让失败模式变成多路混合，更难看。不要在排障时关闭限流账本，否则你只能得到「打爆之后更糟」。排障结束恢复日常策略与压缩设置。

### 只在桌面失败

日志文件、未签名更新、端口被本机其它实例占用。沿着这条树走时每次只改变一个变量，并记下 X-Fallback-Trail。把「我改了很多还是不行」变成「最后一次改的是超时，现象从 504 变成 429」，问题就会收敛。仍不行则用试验台最小 prompt「hi」排除 Agent 侧复杂性，再用 curl 排除浏览器。最小复现成功后再把 tools、图像、长上下文一项项加回。这是所有网关排障的同一方法，免费层只是多了额度这一维。不要在排障时打开 Fusion，它会让失败模式变成多路混合，更难看。不要在排障时关闭限流账本，否则你只能得到「打爆之后更糟」。排障结束恢复日常策略与压缩设置。


## 25. 各智能体接入核对表

### Claude Code

接入Claude Code时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。

### Codex CLI

接入Codex CLI时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。

### Cursor

接入Cursor时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。

### Cline

接入Cline时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。

### Roo Code

接入Roo Code时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。

### Aider

接入Aider时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。

### Continue

接入Continue时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。

### Gemini CLI

接入Gemini CLI时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。

### Zed

接入Zed时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。

### JetBrains AI

接入JetBrains AI时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。

### Goose

接入Goose时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。

### OpenCode

接入OpenCode时：运行对应 setup 或按文档改 base URL；确认网关已有健康密钥与非空链；用 auto 而不是 fusion；先发一句 ping 再开长任务；观察 Routed-Via；长会话再开 standard 压缩；视觉任务换 profile；CI 场景不要用根密钥进公开日志。生成器 dry-run 先看 diff。升级网关后若工具自己改了配置格式，再跑一次 setup，生成器承诺不覆盖你的未知键。若该智能体只支持 Ollama，打开模拟。若该智能体只支持 Anthropic，走 Messages 而不是硬把 OpenAI 字段塞进去。核对表打勾后再把本机端口暴露给局域网。


## 26. 使用说明书版本说明

本手册覆盖安装、逐页操作、协议接入、环境变量、CLI、备份安全、一百条常见问题、场景化操作、配置白话对照与故障决策树。若你只做一件事：加一把健康密钥、复制统一密钥、把 Agent 的 base URL 指过来、模型用 auto。其余章节在失败时再查。字数很长是为了把真实世界的免费层坑写全，不是要求你一次读完。建议收藏 FAQ 搜索，按状态码与客户端名称查找。与设计说明书、优化方案、源码清单分工明确，避免在使用手册里寻找函数级说明。感谢你自托管。请善待各家免费额度，也不要把端口暴露给整个互联网。祝路由顺畅。
