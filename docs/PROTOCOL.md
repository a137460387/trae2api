# Trae 协议实测记录

本文记录通过实测与抓包得到的 **Trae 上游协议细节**——认证、请求头、错误码语义、模型目录与区域差异。这些结论来自对官方客户端的观测，而非官方文档；每一项都注明了观测日期，便于判断时效性。

> 这些知识是本项目最难重建的部分。若你发现某项已过期，欢迎提 issue 或 PR 更正。

## 请求池：CN vs SOLO

| 产品 | 常见 function | 热门模型排队 |
|---|---|---|
| Trae CN | `chat_v3` 通用池 | **很重**（常 1000+） |
| TRAE SOLO | `solo_work_lite` / `create_agent_task` | **较轻** |

本项目默认 `TRAE_PRODUCT=solo`，指定模型走 `solo_work_lite`，不要退回 CN 的 `chat_v3` 主路径。

## 已对齐（可用）

### 认证
- 目录：`%APPDATA%\TRAE SOLO CN\User\globalStorage\storage.json`
- 解密 `iCubeAuthInfo://icube.cloudide`（tc / AES-128-CBC）

### Headers（与 SOLO 日志一致）
| Header | 来源 |
|---|---|
| `x-app-id` | 固定 app id |
| `x-device-id` | `iCubeAuthInfo://icube-dc:<id>` |
| `x-machine-id` | `telemetry.machineId` |
| `x-ide-version` | manifest `appVersion` |
| `x-ide-version-code` | 日期型 |
| `x-device-brand` / `x-os-version` | SOLO 默认，env 可覆盖 |
| `x-flow-traceparent` | 本地生成 |
| `Authorization` | `Cloud-IDE-JWT <token>` |

### 对话通道
| 模式 | 本地 | 说明 |
|---|---|---|
| `auto` | `llm_utils_chat` + `inline_chat` | 最轻 |
| 指定模型 | `llm_utils_chat` + **`solo_work_lite`** + `config_name` | 与 SOLO 同池 |
| 完整 Agent | `create_agent_task` 骨架 | 未 100% 复刻 |

实测：`solo_work_lite + glm-5.2` 正常；`chat_v3 + glm-5.2` 易大排队（SOLO CN 默认实例场景）。  
注意：`TRAE_DATA_DIR` 固定到 `TraeWork-CN-N` 这类目录时，目录名不含 "solo" → `isSoloProduct()` 判 false → 默认走 `chat_v3`；要切回 SOLO 池需显式 `TRAE_PRODUCT=solo`。  
**2026-09-17 事故复现**：用暂存目录（名不含 solo）起临时网关测新导入账号，全部请求默认 `chat_v3`，新模型齐齐 4001，症状极像"账号无此模型权限"；`TRAE_PRODUCT=solo` 后同批模型全部可用。测账号/开账号池时务必显式 solo，勿凭 4001 下账号能力结论。

### 模型目录（重要）

`llm_utils_chat` 的 `config_name` 必须在 SOLO `get_detail_param(function=solo_work_lite)` 返回列表中。  
**不在列表**（如历史 `glm-5.1`）会立刻 `4001 param is invalid`——以 HTTP 200 + SSE error 事件返回，不触发换模/换号。

CN SOLO 模型名有两套格式（2026-09-12 实测）：

1. **第一方模型**：裸 `config_name`。`solo_work_lite` 目录共 38 个：`glm-5.2`、`glm-5-turbo`、`glm-5`、`glm-5.3`、`DeepSeek-V4-Pro/-Flash`（含 `-Official`）、`qwen3.8-max`、`qwen-3.7-plus`、`kimi-k3`、`kimi-k2.7-code`、`kimi-k2.6`、`Doubao-Seed-2.*`、内部代号 `sagitta`/`aquila`/`seed-code-pro-0430` 等。快照：`scripts/_solo-models-clean.json`。  
   **2026-09-17 复测（双账号、最小请求逐一验证）**：新代第一方裸名 `glm-5.3`/`kimi-k3`/`kimi-k2.7-code`/`qwen3.8-max`/`glm-5-turbo`/`Doubao-Seed-2.1-Pro` 经网关直调 `solo_work_lite` **全部可用**（此前"新模型 4023/需 custom_models 渠道"的记载不适用于第一方裸名），已收录 `model-config.json`：`glm-5.3`→tier1、`qwen3.8-max`→tier2。
2. **第三方模型（`<渠道>//<模型名>`，暂不可经网关调用）**：`get_detail_param` 目录里各 `custom_model_*` 条目带 `custom_models` 白名单（如 `Kimi-CN//kimi-k3`、`aliyuncs//qwen3.8-max`、`bigmodel-plan//glm-5.3`；约 167 个，渠道 `anthropic`/`openai`/`openrouter`/`vercel`/`deepseek`/`aliyuncs`/`volcengine`/`bigmodel`/`zai`/`siliconflow`/`gitee`/`Kimi-CN/Global`/`MiniMax-CN/Global` 等；`-plan`/`-agent-plan` 后缀疑似订阅计费通道）。这属于官方客户端 UI 选项层，**裸 `llm_utils_chat` 通道不认这些名字**：原样透传（请求体 `config_name`）→ `4001 param is invalid`（2026-09-12 实测 `anthropic//claude-opus-4-6`）；以 `config_name=custom_model_1M_text` + `model=Kimi-CN//kimi-k3` 直调返回 `4023 model is unknown`——客户端还携带 `encrypted_prompt_set` 等网关未复刻的加密参数（2026-09-10 实测）。另注意网关别名解析会子串匹配本地键、把 `//` 名悄悄改写（如 `deepseek//deepseek-v4-pro` 含本地键 `deepseek-v4-pro` → 上游实收第一方 `DeepSeek-V4-Pro`），看似调用成功实则不是第三方路由。真第三方路由需官方客户端的云端 agent 通道（`solo_agent_lite` + 完整 `model_info`：`ak`/`base_url`/`custom_model_type`），见 `TODO.md`。

两套目录各有独占模型、互不通用：`qwen3.8-flash`、`kimi-k2.8-preview`、`minimax-m2*`、GLM 旧系（4.6/4.7/5.1）只在 `chat_v3`（51 个）；`glm-5-turbo`、`kimi-k2.6`、`Doubao-Seed-2.0-Code`、`custom_model_claude`、`sagitta` 等只在 `solo_work_lite`。需要 `chat_v3` 独占模型时请求体显式传 `"function": "chat_v3"`（重排队，勿作主路径）。

**2026-09-10 实测（chat_v3 池 / TraeWork-CN-N 账号，最小请求逐一验证）**：

- 可用：`glm-5.2`、`glm-5`、`kimi-k2.6`、`qwen-3.7-plus`、`DeepSeek-V4-Pro`、`DeepSeek-V4-Flash`、`custom_model_deepseek_reasoner`（deepseek-r1，`toolcall_compatible=false`）
- 4001：`kimi-k3`、`kimi-k2.7-code`、`glm-5-turbo`（连同映射到它的 `glm-5.1`）、doubao 全系、`qwen3.8-max`——即上表"solo_work_lite 独占"名单，chat_v3 池本就没有，属目录差异而非全局下架
- `get_detail_param(chat_v3)` 返回 41 个 config_name，与本地 `model-config.json` 差异：+32（多为 `title_generation`/`fast_apply` 等内部工具模型，勿用 `fetch-models.js --update` 全量收录）/ −22

拉表：`node scripts/dump-model-detail.js`（SOLO 表，function 读 `TRAE_SOLO_FUNCTION`，默认 `solo_work_lite`）；`node scripts/fetch-models.js`（`chat_v3` 表，对比本地 `model-config.json`）。本地 `model-config.json` 别名可映射到上述两种格式；排队/错误降级也会跳过 4001/4023。

### OAuth
SOLO ClientID：`en1oxy7wnw8j9n`（`product.json` → `authConfig.SOLO.stable`）

## 未完成：`create_agent_task`

真机：

```text
POST /api/agent/v3/create_agent_task
body ≈ 180–210KB
agent_type=function=solo_work_lite
```

卡点：summary template / `encrypted_prompt_set` 非 tc，需抓完整明文 body。  
步骤见 `TODO.md`。探测：`node scripts/probe-solo-create-agent.js --model glm-5.2`。

## `.env` 要点

```env
TRAE_PRODUCT=solo
TRAE_EDITION=cn
# TRAE_DATA_DIR=%APPDATA%/TRAE SOLO CN
# TRAE_SOLO_FUNCTION=solo_work_lite
# TRAE_OAUTH_CLIENT_ID=en1oxy7wnw8j9n
```

陷阱：设 `TRAE_DATA_DIR` 后 `detectEdition()` 的 cn/sg 候选解析到**同一文件**（mtime 相等）→ 误判 `sg`。固定实例时必须同时显式 `TRAE_EDITION=cn`。
