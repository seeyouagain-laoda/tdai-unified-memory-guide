# 三端统一记忆实战指南 · Unified Agent Memory Across WorkBuddy + OpenClaw (NAS & Host)

> 把 **WorkBuddy**、**主机 OpenClaw**、**NAS OpenClaw** 三个 AI 客户端的对话沉淀进同一个团队记忆库，实现共享记忆。
> Built on Tencent's open-source **TencentDB Agent Memory (Memory Hub)**. 官方仓库：https://github.com/TencentCloud/TencentDB-Agent-Memory

## 这是什么 / What this is
- 同团队共享 L2/L3（场景档案 / 人格画像）
- 按 agent 隔离 L0/L1（原始对话 / 记忆卡片）

## 架构 / Architecture
- 部署在 NAS（Docker）：memory-core(8420) + memory-hub 面板(8125) + memory-knowledge(8424) + **tdai-proxy(8096)**。
- 三端都连 `tdai-proxy`。
- 路由：OpenClaw → `/openclaw/default`（chat completions，仅 header 鉴权）；WorkBuddy → `/workbuddy/default/v1/responses`（OpenAI Responses API，需 session-init 绑定）。

## 关键配置（占位符，换成你部署时生成的值）
| 项 | 占位符 |
|---|---|
| 团队 team | `<TEAM_ID>` |
| NAS OpenClaw agent | `<NAS_OC_AGENT_ID>` |
| 主机 OpenClaw agent | `<HOST_OC_AGENT_ID>` |
| WorkBuddy agent | `<WORKBUDDY_AGENT_ID>` |
| task | `<TASK_ID>` |
| user | `<USER_ID>` |
| API Key | `<YOUR_TDAI_API_KEY>` |
| 模型 | `nvidia/nemotron-3-ultra-550b-a55b`（示例） |
| NAS 内网 IP | `<NAS_LAN_IP>` |
| Proxy 地址 | `http://<NAS_LAN_IP>:8096` |

> team / agent / user / API Key 由 Memory Hub 部署时生成。

## 接入三端 / Connect the three clients

### NAS / 主机 OpenClaw
- base URL：`http://<NAS_LAN_IP>:8096/openclaw/default`
- 请求头：`x-team-id: <TEAM_ID>`、`x-agent-id: <NAS_OC_AGENT_ID>`（主机用 `<HOST_OC_AGENT_ID>`）、`x-task-id: <TASK_ID>`
- 标准 chat completions，无特殊格式要求。

### WorkBuddy ⚠️ 最易踩坑
- 端点：`http://<NAS_LAN_IP>:8096/workbuddy/default/v1/responses`
- 预设头：`x-team-id` / `x-agent-id` / `x-task-id`
- 需 session-init 绑定。

## ⚠️ 头号坑：WorkBuddy 请求体必须是结构化 Responses API 格式
`extractWorkbuddyUserText` 严格要求标准 Responses API 的 `input[]` 结构。错误写法（普通 chat 格式）：
```json
{ "role": "user", "content": "请记住：我喜欢用中文回复" }   // ❌ content 是字符串、缺 type
```
→ 用户消息提取返回 null → L0 静默不落库、无日志。

正确写法：
```json
{
  "model": "<YOUR_MODEL>",
  "input": [
    { "type": "message", "role": "user",
      "content": [ { "type": "input_text", "text": "请记住：我喜欢用中文回复" } ] }
  ],
  "stream": true
}
```

## 验证 / Verify
1. 三端各发唯一探针词。
2. 等后台提炼完成（约 10s 或满 5 轮）。
3. 用 core 检索接口 / Hub 面板按 agent 分区查询：探针词应只落在对应 agent 分区，其他 agent 为空 → 共享 + 隔离正确。

## 常见坑 / Pitfalls
| 级别 | 坑 | 解法 |
|---|---|---|
| P0 | WB 请求体格式错 | 用结构化 Responses API 格式 |
| P1 | header 选错字段 | `sessionInit.headerAutoSelect` 只认 `x-team-id`/`x-agent-id`/`x-task-id`，不认 `x-tdai-*` |
| P2 | hook-cache FK 警告 | 性能级，不影响 L0，可忽略或清理 |

## 引用 / Reference
- **腾讯官方库（本项目底座）**：https://github.com/TencentCloud/TencentDB-Agent-Memory ｜ MIT 协议，团队级记忆中枢，四资产 Chat Memory / Skill / Wiki / CodeGraph，L0→L3 分层蒸馏。
- 官方安装：仓库内 `INSTALL.md` / `INSTALL_CN.md`；面板 `http://localhost:8125`。

## English summary
This guide shows how to give three AI clients — **WorkBuddy**, **OpenClaw on your NAS**, and **OpenClaw on your host PC** — a single shared long-term memory, using Tencent's open-source **TencentDB Agent Memory** (Memory Hub, MIT license).

- Deploy the Hub (core + hub + knowledge + proxy) on your NAS via Docker. All three clients point at the proxy (`http://<nas>:8096`).
- OpenClaw uses `/openclaw/default` (chat completions, header-only auth). WorkBuddy uses `/workbuddy/default/v1/responses` (OpenAI Responses API, requires session-init).
- Shared memory: same `team` ⇒ L2/L3 shared across agents. Isolation: different `agentId` ⇒ L0/L1 isolated.
- **Critical gotcha for WorkBuddy**: the request body MUST be the structured Responses API shape — `input: [{ type:"message", role:"user", content:[{ type:"input_text", text:"..." }] }]`. A plain `{ role, content:"string" }` chat body makes the user message silently dropped (no error, no log), so it looks like "WorkBuddy has no memory".
- Reference: https://github.com/TencentCloud/TencentDB-Agent-Memory
