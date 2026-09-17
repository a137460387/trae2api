# trae2api

把 **Trae**（国内版 / 国际版）包装成本地 **OpenAI / Anthropic 兼容 API**，供 Claude Code、Cursor、Cline、OpenCode 等任意兼容客户端调用。

单进程、零外部服务，直接读取本机已登录 Trae 客户端的凭证并转发请求。

```
┌──────────────┐   OpenAI / Anthropic    ┌───────────┐   Trae 私有协议    ┌──────────────┐
│ Claude Code  │ ──────────────────────► │  trae2api │ ────────────────► │  Trae (CN/SG) │
│ Cursor/Cline │ ◄────────────────────── │   :19950  │ ◄──────────────── │   upstream    │
└──────────────┘        SSE / JSON       └───────────┘      SSE          └──────────────┘
```

## 特性

- **双区域**：国内版（`cn`）与国际版（`sg`）同池共存，默认**按请求自动选区**——用 `gpt-5.4` 就交给国际版账号，用 `glm-5.3` 就交给国内版账号，无需手动切换。
- **多账号池**：热导入本机所有已登录实例；401/配额/风控等故障自动轮换，无需重启。
- **两套协议**：OpenAI Chat Completions + Anthropic Messages，含流式 SSE、tool calls、thinking。
- **推理深度控制**：按模型族注入 reasoning 前缀（`think_effort`），含实测调优矩阵。
- **自动续写**：模型只输出思考、或输出被截断时自动续请求，客户端看到完整结果。
- **零配置启动**：只要本机有一个已登录的 Trae 客户端就能跑。

## 快速开始

```bash
npm install
npm start
# → http://127.0.0.1:19950/v1
```

Windows 可直接双击 `start-gateway.bat`。

验证：

```bash
curl http://127.0.0.1:19950/v1/status -H "Authorization: Bearer sk-trae2api-local"
```

客户端配置（以 Claude Code 为例）：

```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:19950
export ANTHROPIC_AUTH_TOKEN=sk-trae2api-local
```

## 配置

复制 `.env.example` 为 `.env` 后按需修改，常用的几个：

| 变量 | 默认 | 说明 |
|------|------|------|
| `PORT` | `19950` | 监听端口（避开 19900，见 `.env.example` 注释） |
| `API_KEY` | `sk-trae2api-local` | 客户端需携带的 Bearer key |
| `TRAE_EDITION` | 自动 | 强制指定上游区域：`cn` / `sg`。不设则按请求自动选区 |
| `TRAE_PRODUCT` | `solo` | `solo`=TRAE SOLO（排队轻）；`trae`=经典 Trae（排队重） |
| `TRAE_POOL_DIR` | `./accounts` | 账号池目录（含明文 token，勿提交） |
| `TRAE_POOL_SELF_RENEW` | `off` | 是否允许网关自行刷新令牌，见下文 |

## 账号池

网关启动后会自动扫描本机的 Trae 客户端数据目录并收池，**无需手工搬运凭证**。

- 加账号：在某个 Trae 客户端登录该账号，网关下次取凭证时自动收录。
- 多开实例：把 `storage.json` 放到 `%APPDATA%\TraeWork-CN-<N>\User\globalStorage\` 即可（只需目录结构，不必装客户端）。
- 查看：`GET /v1/pool`
- 下架：把成员文件的 `"enabled"` 改为 `false`

细节见 [`accounts/README.md`](accounts/README.md)。

> ⚠️ **池内 token 是明文，等同账号本体。** 该目录已被 `.gitignore` 排除，请勿提交、同步或备份。

### `TRAE_POOL_SELF_RENEW` 为什么默认关闭

网关刷新令牌会走 `ExchangeToken`，而该接口会在服务端**轮换整个令牌族**——这会让同一账号在其它客户端里的登录静默失效。默认 `off` 表示网关只使用已导入的令牌，令牌更新依赖客户端重新登录（热导入会自动接上）。若你不需要保留客户端登录，可设为 `on`。

## API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/v1/chat/completions` | OpenAI 兼容对话（流式/非流式、tools、`think_effort`） |
| POST | `/v1/messages` | Anthropic 兼容对话 |
| GET | `/v1/models` | 可用模型列表 |
| GET | `/v1/status` | 当前账号与上游状态 |
| GET | `/v1/pool` | 账号池状态（含每个账号的区域与可用性） |
| GET/POST/DELETE | `/v1/upstream` | 查看/切换/清除上游区域（`cn` \| `sg`） |
| GET | `/v1` | 完整路由目录与 OpenAPI 文档 |

## 双区域路由

国内版与国际版的**模型目录几乎不重叠**（实测 47 个模型名只有 3 个交集），且同名模型可能分属不同区域。因此：

- 池里同时有 `cn` 与 `sg` 账号时，网关按**请求的模型名**推断该用哪个区域，同一进程可同时服务两侧。
- 模型名无法判断区域、或你只想用一个区域时，用 `TRAE_EDITION` 固定，或调用 `POST /v1/upstream`。

**注意：模型名不跨区通用。** 用错区域的模型名会得到 `4001 param is invalid`。各区域的模型清单见 [`docs/PROTOCOL.md`](docs/PROTOCOL.md)。

## 推理深度（think_effort）

请求体加 `think_effort`（别名 `reasoning_effort`）：`off` / `low` / `high` / `max` / `auto`。

不同模型族对前缀的响应差异很大，矩阵与实测数据见 [`doc/think-effort.md`](doc/think-effort.md)。

## 项目结构

```
src/
  server.js            HTTP 入口、路由、OpenAI/Anthropic 主链路
  realms.js            区域配置表（host/版本/模型归属）——双区域知识的唯一出处
  auth.js              凭证解密、账号池、区域判定、请求头
  trae-decrypt.js      客户端 storage.json 的解密实现
  trae-client.js       上游请求体构造、模型解析、排队降级
  openai-format.js     OpenAI 协议编解码与流解析
  anthropic-format.js  Anthropic 协议编解码与流解析
  think-effort.js      推理深度前缀注入
  auto-continue.js     自动续写判定
  tools.js             本地工具执行（文件/命令/搜索）
  sessions.js          会话持久化
  traffic-logger.js    请求日志与看板数据源
docs/PROTOCOL.md       协议实测记录（最有价值的参考文档）
```

## 文档

| 文档 | 内容 |
|------|------|
| [`docs/PROTOCOL.md`](docs/PROTOCOL.md) | Trae 协议实测大全：认证、请求头、错误码语义、两套模型目录、区域差异 |
| [`doc/think-effort.md`](doc/think-effort.md) | 推理深度控制的前缀方案与 A/B 实验数据 |
| [`accounts/README.md`](accounts/README.md) | 账号池文件格式与安全须知 |

## 免责声明

本项目仅供**个人学习与研究**使用，用于在本地环境中访问你**自己已登录账号**的模型能力。使用者应自行遵守 Trae 及各模型服务方的服务条款。请勿用于商业转售、批量注册、绕过配额或任何违反服务条款的用途。因使用本项目产生的一切后果由使用者自行承担。
