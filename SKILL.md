---
name: arkcli-dsh-model-onboard
description: 把火山方舟（或其它兼容 OpenAI / Anthropic 协议的模型网关）的模型安全接进 DSH 模型选择器：先用零成本手段判定权威 baseURL 与凭证切面，再改 cordis.patch.yml、做最小真实调用验真、失败整文件回滚。当用户说「把 XX 模型加进 DSH」「模型选择器里看不到方舟模型」「接 Agent Plan / Coding Plan 模型」「配置了 provider 但选不到」「调用报 API 密钥无效 AUTH」时使用。
---

# arkcli-dsh-model-onboard（方舟模型接入 DSH）

## 触发条件

- 需要把**新模型**加进 DSH 模型选择器（本机 profile：`C:\Users\Sxg\.dsh\profiles\desktop`）
- 现象是「模型选择器里看不到某组」「加了 provider 但选不到」
- 调用报 `API 密钥无效 / AUTH / 401`
- 目标不只是「试一下」，而是要在 DSH 里长期可用

## 输入

| 输入 | 示例 | 说明 |
|---|---|---|
| 模型 ID 列表 | `doubao-seed-evolving` / `kimi-k3.1-pro` / `glm-5.3` | 用 **调用 ID**（套餐别名），不要自造版本号 |
| credential ref 名 | `ARKV3_API_KEY` | DSH 侧凭据引用名；由 Models 页写入，**不要**写进 YAML |
| 套餐类型 | Agent Plan / Coding Plan / 平台按量 | 决定 baseURL 与协议，见下表 |

## 第 0 步：开工约束（先取证，再动手）

1. 先读配置与相关源码/文档，给出「现象 → 可能根因」候选，再决定改什么；
2. 在拿到 **0 成本鉴别证据**（dry-run / 假模型名探针 / schema 校验）之前，**不要**要求用户重启应用或更换 Key；
3. 每个结论标注性质：**实测 / 推定 / 未验证**；
4. 改配置前先备份并记 SHA256，改后给 diff，失败整文件回滚。

## 第 1 步：零成本判定端点与凭证切面

```bash
ARKCLI_NO_UPDATE_NOTIFIER=1 ARKCLI_CALLER_TYPE=ai_agent ARKCLI_CALLER_NAME=dsh \
ARKCLI_SKILL_NAME=arkcli-chat \
arkcli +chat --model <model-id> --dry-run --format json "hi"
```

读 `summary.execution_context`：

| 字段 | 含义 |
|---|---|
| `base_url` | **权威端点**，不要用文档默认值猜 |
| `credential_kind` | `agent_plan` / `platform` / … |
| `data_plane` | `agent_plan` 等 |

映射表（2026-11-10 实测）：

| 通道 | baseURL | 协议 |
|---|---|---|
| Agent Plan（套餐） | `https://ark.cn-beijing.volces.com/api/plan/v3` | `openai-responses` |
| Coding Plan（套餐） | `https://ark.cn-beijing.volces.com/api/coding` | `anthropic-messages` |
| 平台按量 | `https://ark.cn-beijing.volces.com/api/v3` | `openai-completions` / `openai-responses` |

## 第 2 步：零成本探针（区分鉴权 vs 模型）

用**不存在的模型名**发请求，只产生错误、不产生 token：

```bash
curl -s -o out.txt -w "HTTP=%{http_code}\n" -X POST "<base>/responses" \
  -H "Authorization: Bearer <key>" -H "Content-Type: application/json" \
  --data-binary '{"model":"definitely-not-a-model-xyz","input":"hi"}'
```

| 状态码 | 结论 | 下一步 |
|---|---|---|
| 401 | 鉴权层（Key 错 / Key 与端点不匹配 / 端点走错数据面） | 先核对 baseURL，不要急着换 Key |
| 403 | 权限层（实名 / 开通 / 白名单） | 去控制台核对开通与实名 |
| 404 / `UnsupportedModel` | 鉴权已通过，只是模型名或套餐不支持 | 换正确调用 ID 或换套餐通道 |
| 429 | 限流 / 额度耗尽 | 查 usage，不要重试轰炸 |

## 第 3 步：改配置

配置文件：`C:\Users\Sxg\.dsh\profiles\desktop\cordis.patch.yml`（**只改这一个文件**）

1. **备份**：复制到 `D:\DS肥鱼专区\_backup\cordis.patch-before-<用途>-<时间戳>.yml`，记录 SHA256；
2. 在 `- id: llm-pi-ai` 的 `config.providers` 下新增（或扩展）provider：

```yaml
      arkv3:
        displayName: 方舟大模型 v3
        apiKeyEnv: ARKV3_API_KEY      # 只写 ref 名，不写密钥值
        api: openai-responses
        baseURL: https://ark.cn-beijing.volces.com/api/plan/v3
        models:
          - id: doubao-seed-evolving
            name: doubao-seed-evolving
            input:
              - text
```

3. **校验 YAML**（用 profile 自带的 js-yaml）：

```bash
node -e "const y=require('C:/Users/Sxg/.dsh/profiles/desktop/node_modules/js-yaml');const fs=require('fs');const d=y.load(fs.readFileSync(process.argv[1],'utf8'));console.log(d.find(x=>x.id==='llm-pi-ai').config.providers)" "<patch.yml>"
```

4. 给出「新增行」diff（禁止改既有行）。

## 第 4 步：确认 credential 已就绪（最容易漏）

- 检查 `~/.dsh/.credentials.yaml` 的 `refs:` 是否有该 `apiKeyEnv`（**只输出长度/前缀/指纹，不打印明文**）
- 未配置时：`listModels()` 返回空数组 → 选择器里**整组静默消失**（不报错，设置页仍有行）
- 补 Key 的唯一正确路径：DSH **设置 → 模型 → 该 provider 行**粘贴（只写存储，不落 YAML）
- 注意：套餐通常只有一把 Key，「新建一把专用 Key」未必可行；先用现有的走通，再谈隔离

## 第 5 步：最小真实调用验真

```bash
curl -s -o out.txt -w "HTTP=%{http_code}\n" -X POST "<base>/responses" \
  -H "Authorization: Bearer <key>" -H "Content-Type: application/json" \
  --data-binary '{"model":"<model-id>","input":"hi","max_output_tokens":16}'
```

- 期望 `HTTP=200` 且返回体里 `"model":"<model-id>"`、有 `id`（`resp_...`）
- 记录 response id 作为证据；不要用「配置写进去了」当验真

## 第 6 步：DSH 侧验收

| 验收项 | 判定 |
|---|---|
| 选择器出现该分组 | 用户 F5 后可见（新分组可能在列表下方，需滚动） |
| 真调用走对路由 | `~/.dsh/.dshw-usage.json` 的 events 里出现该 model；会话事件 provider 正确 |
| 原 provider 未受影响 | 原有模型仍可选、可调 |
| 回滚可用 | 备份文件可整文件覆盖回去 |

## 回滚

```powershell
Copy-Item "D:\DS肥鱼专区\_backup\cordis.patch-before-<用途>-<时间戳>.yml" `
  "$env:USERPROFILE\.dsh\profiles\desktop\cordis.patch.yml" -Force
```

回滚后在设置页删掉对应 provider 行（若有残留），确认选择器恢复原状。

## 反例边界（不许越界）

- **套餐 Key 不能跨数据面用**：Agent Plan Key 打 `/api/v3` 会 401，不要读成「Key 无效」
- **TTS / ASR 不是文本模型**：不要在 `models` 里声明，`input` 只允许 `text` / `image`
- **不要用「平台文档默认端点」猜套餐端点**：一律以 `arkcli --dry-run` 的 `base_url` 为准
- **不要在 YAML 里写密钥明文**：只写 `apiKeyEnv` ref
- **不要一次加 20 个模型**：按批加，每批逐个通一次最小调用再扩
- **401 不必然等于 Key 无效**（本次踩坑），先按第 2 步分层

## 快速参考：DSH 侧机制（排查依据）

- `dsh-llm-pi-ai` 的 `listModels()`：`MISSING_CREDENTIAL` → `return []`
- `buildModelCatalog()`：`filter(group.models.length > 0)` → 空组不渲染
- `cordis.patch.yml` 是 profile 的用户层配置，settings 写入与外部编辑都会收敛；运行中改动以 F5 验证为准，不生效再重启
