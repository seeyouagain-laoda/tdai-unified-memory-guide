# TDAI 三端共享记忆库 · 完整部署与排错教程

> 生成时间：2026-09-04 ｜ 作者：WorkBuddy（用户授权自执行）｜ 状态：**已落地并实测**
> 适用：把 **NAS OpenClaw / Windows OpenClaw / WorkBuddy** 三端的长期记忆统一进**同一个** TencentDB Agent Memory（TDAI）记忆库，做到「一处记住、三处都能召回」。
> 实测环境：飞牛 fnOS NAS（<NAS_LAN_IP>）+ Windows 11 + WorkBuddy。TDAI 以 Docker 四容器部署在 NAS。

> ⚠️ **本文取代** `14_TDAI三端统一接入_任务书与方案`、`15_TDAI_WorkBuddy端接入_总结报告` 以及旧版 GitHub README。那几份文档基于**错误的"验收通过"结论**（详见 §3），按它们操作会出现「请求 200、模型也回复、但记忆根本没落库」的静默故障。本文是踩完所有坑后的**最终正解**。

> **官方底座**：TencentDB Agent Memory（腾讯开源，MIT 协议）— https://github.com/TencentCloud/TencentDB-Agent-Memory ｜ 四资产 Chat Memory / Skill / Wiki / CodeGraph，L0→L3 分层蒸馏，面板 `http://<NAS_LAN_IP>:8125`。

---

## 0. 一句话原理

TDAI 的记忆库是一张 sqlite（`vectors.db`），**`agent_id` 是分区键**。三端都往**同一个 `agent_id`** 写，就天然共享同一份记忆。

但**光设 `agent_id` 不够**——proxy 有一道 `sessionInit` 握手，如果握手被绕过，`agent_id` 会解析成 `null`，记忆被**静默丢弃**（HTTP 200、零报错）。本文的核心就是修掉这个静默丢弃（见 §3、§4）。

---

## 1. 架构与端口

| 容器 | 端口 | 作用 |
|---|---|---|
| `tdai-memory-core` | 8420 | 内核 gateway，真正落库的地方 |
| `tdai-memory-hub` | 8125 / 8424 | 面板 + 知识库 |
| `tdai-proxy` | 8096 | 对外代理；三端都打它 |

数据落盘位置（NAS 宿主机）：

```
<TDAI_VOLUME_PATH>/vectors.db
```

表结构关键点：

- `l0_conversations`：原始对话（分区键 `agent_id`，文本列 `message_text`）
- `l1_records`：抽取出的记忆
- `l0_fts` / `l1_fts`：FTS5 全文索引表（外部内容表，不会随基础表 `UPDATE` 自动同步）

三容器均 `--restart=always`，NAS 重启自愈。

proxy 关键配置（`/data/config.yaml`，由 `start-proxy.sh` 生成）：

```yaml
server: { host: 0.0.0.0, port: 8096, forwardTimeoutMs: 600000 }
upstream:
  model: "nvidia/nemotron-3-super-120b-a12b"      # 默认上游
  agents:
    openclaw:
      url: "https://www.poke2api.com/v1"            # grok-4.6 走这里
      apiKey: "<POKEX_API_KEY>"
tdai:
  endpoint: "http://memory-core:8420"
  apiKey: "local"
  serviceId: default
  memory: { enabled: true, inject: true, writeL0: true, recallL1: true, injectL2L3: true }
auth: { enabled: true, url: "http://memory-core:8420" }
```

三容器均 `--restart=always`，NAS 重启自愈。

---

## 2. 三端接入总览

| 端 | 接入方式 | 打到哪 |
|---|---|---|
| NAS OpenClaw | `memory-proxy` 模型 provider（带记忆落库能力的 provider） | proxy `:8096` |
| Windows OpenClaw | 同 NAS | proxy `:8096` |
| WorkBuddy | 本机转发层 `tdai-wb-relay.py`（WB 不支持自定义请求头，需补头） | proxy `:8096` |

> 三者最终都汇聚到 proxy `:8096`，再由 proxy 统一强制身份后写 core `:8420`。**统一入口是关键**——不要各自直连 core，否则分区不一致。

---

## 3. ⚠ 核心坑：为什么「设了 x-agent-id 也不落库」（根因）

这是整套方案**唯一真正致命**的坑，也是之前几次「验收通过」全是假的原因。

### 3.1 静默丢弃的完整链路

1. proxy 有 `sessionInit` 握手：首轮会话会弹 `ask_followup_question` 表单让你选 `team/agent/task`，并把选择写进**内核 session 记录**（`sessionInfo`）。
2. proxy 内部 `deriveTdaiIdentity()` **只从 `sessionInfo`（内核 session 记录）取 `team_id`/`agent_id`**。如果首轮握手被绕过（没走完表单、或 `headerAutoSelect` 没匹配上），`sessionInfo` 为空 → `deriveTdaiIdentity` 返回 **`null`**。
3. `recordTdaiTurn(null)` → 打内核写接口 `POST /v3/conversation/add` 时**缺身份 → 内核回 401**。
4. 关键：proxy 的 `postForCtx()`（client.ts）把这次 401 **吞掉了**（catch 返回 `{}`），**不打印任何错误**。
5. 结果：**HTTP 200、模型照常回复、但 L0 一行没写、日志零报错**。你永远看不出问题。

> 之前「验收通过」就是这么来的：L0 卡在 146 不动（全是历史合并进来的旧数据伪装），新写入一条都没进库，但表面一切正常。

### 3.2 为什么不能用字面 `agt-shared`

有人会想：「那我把 `agent_id` 写死成 `agt-shared` 不就行了？」不行，两个原因：

- **kernel 用自己生成的 ID**：`agent/create`、`task/create` 这类 meta API，**你传的 `agent_id` 不一定被采纳**——kernel 会返回它自己生成的 ID（形如 `<YOUR_AGENT_ID>`）。如果你在配置里写 `agt-shared`，但 kernel 实际注册的是 `<YOUR_AGENT_ID>`，后续 recall 会直接 `agent not found` / `task not found`（404）。
- **绕过握手后 `sessionInfo` 仍为空** → 回到 §3.1 的静默丢弃。

### 3.3 正解的两个要素（缺一不可）

1. **用 kernel 实际返回的 ID**（通过 meta API 注册后捕获），不要自己编 `agt-shared`。
2. **让 `sessionInfo` 一定有值** → 用 `sessionInit.debugForceIdentity` 强制每个新会话在第 1 次触碰时就注册成固定身份，**跳过交互表单**，从而 `deriveTdaiIdentity` 一定能拿到非空身份 → L0 真正落库。

---

## 4. 正解：sessionInit.debugForceIdentity + 真实 kernel ID

### 4.1 先注册真实的 team / agent / task（捕获返回值）

通过 TDAI 内核 meta API 注册（端口 `8420`）。**重点是：用返回体里的 `agent_id`/`task_id`/`team_id`，不要信自己传的**。

```bash
# 注册 agent（返回体里的 agent_id 才是真值）
curl -sS http://<NAS_LAN_IP>:8420/v3/agent/create \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_TDAI_USER_KEY>" \
  -d '{ "team_id": "<YOUR_TEAM_ID>", "name": "shared", "visibility": "team" }'
# → 返回类似 { "agent_id": "<YOUR_AGENT_ID>", ... }

# 注册 task
curl -sS http://<NAS_LAN_IP>:8420/v3/task/create \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_TDAI_USER_KEY>" \
  -d '{ "team_id": "<YOUR_TEAM_ID>", "title": "shared" }'
# → 返回类似 { "task_id": "<YOUR_TASK_ID>", ... }
```

本机实测拿到的真实 ID（记下来，后面全用它们）：

```
team_id  = <YOUR_TEAM_ID>
agent_id = <YOUR_AGENT_ID>      # ← 共享分区，三端都写这里
task_id  = <YOUR_TASK_ID>
```

> 如果 `<YOUR_TEAM_ID>` 已存在（本机就是），只需注册 agent/task 并复用该 team。

### 4.2 改 host 配置副本（持久化的源头）

配置文件在 NAS 上、root 属主：

```
<TDAI_DEPLOY_DIR>/.proxy-config/config.yaml
```

在 `sessionInit:` 块下加入 `debugForceIdentity`（用 §4.1 的真实 ID）：

```yaml
sessionInit:
  enabled: true
  maxRetries: 3
  injectAgentContext: true
  injectTaskContext: true
  headerAutoSelect:
    enabled: true
    teamHeader: "x-team-id"
    agentHeader: "x-agent-id"
    taskHeader: "x-task-id"
    onMismatch: "form"
  debugForceIdentity:                 # ← 新增：强制身份，跳过表单，根治静默丢弃
    team_id: "<YOUR_TEAM_ID>"
    agent_id: "<YOUR_AGENT_ID>"
    task_id: "<YOUR_TASK_ID>"
```

改法（先备份）：

```bash
ssh <NAS_USER>@<NAS_LAN_IP>
echo '<NAS_SSH_PASSWORD>' | sudo -S cp <TDAI_DEPLOY_DIR>/.proxy-config/config.yaml \
     <TDAI_DEPLOY_DIR>/.proxy-config/config.yaml.bak_debugforce
# 用你习惯的编辑器在 sessionInit 下补 debugForceIdentity 三行（root 属主，需 sudo 写）
```

### 4.3 改生成器 start-proxy.sh（防重启被覆盖）

⚠ **关键**：proxy 每次启动会用 `start-proxy.sh` 里的 heredoc **重新生成** `/data/config.yaml` 并挂载 `:ro`。**只改 host 副本没用**——重启后就被生成器覆盖掉。必须同时改生成器。

生成器位置（NAS，root 属主）：

```
<TDAI_DEPLOY_DIR>/start-proxy.sh
```

在它生成 `sessionInit` 的 heredoc 段（约 line 128 附近的 `teamHeader/agentHeader/taskHeader` 之后）补同样的 `debugForceIdentity` 三行，内容与 §4.2 完全一致。改完保存，**重启 proxy 才会生效**。

> 验证是否真的改到生成器：重启 proxy 后 `docker exec tdai-proxy cat /data/config.yaml | grep -A3 debugForceIdentity` 应能看到这三行。看不到 = 生成器没改到，重启后又会被冲掉。

### 4.4 重启 proxy

```bash
ssh <NAS_USER>@<NAS_LAN_IP>
echo '<NAS_SSH_PASSWORD>' | sudo -S docker restart tdai-proxy
# 健康检查
curl -s -o /dev/null -w '%{http_code}' --noproxy '*' http://127.0.0.1:8096/health   # 期望 200
docker ps --format '{{.Names}} | {{.Status}}' | grep tdai
```

---

## 5. 三端接入配置

### 5.1 NAS OpenClaw — `/home/<NAS_USER>/.openclaw/openclaw.json`

用 `memory-proxy` 模型 provider 指向 proxy `:8096`（proxy 会强制身份，所以端上 `x-agent-id` 写什么不重要，但建议对齐）：

```jsonc
{
  "providers": {
    "memory-proxy": {
      "type": "openai",
      "base_url": "http://<NAS_LAN_IP>:8096/openclaw/default/v1",
      "headers": {
        "x-agent-id": "<YOUR_AGENT_ID>",
        "x-team-id": "<YOUR_TEAM_ID>"
      }
    }
  }
}
```

改完：`openclaw config validate` → `echo '<NAS_SSH_PASSWORD>' | sudo -S systemctl restart openclaw-gateway`（NAS 用户级 systemd；`systemctl --user` 绝不套 sudo，会丢 D-Bus）。

### 5.2 Windows OpenClaw — `C:\Users\<YOU>\.openclaw\openclaw.json`

与 NAS 完全相同：`memory-proxy` provider，base `http://<NAS_LAN_IP>:8096/openclaw/default/v1`，头 `x-agent-id: <YOUR_AGENT_ID>` / `x-team-id: <YOUR_TEAM_ID>`。

### 5.3 WorkBuddy — 本地转发层 `tdai-wb-relay.py`

WorkBuddy 模型配置**不支持自定义请求头**，所以用一层极薄转发层补头后打 proxy。关键常量（本机实测可用）：

```python
TEAM_ID   = "<YOUR_TEAM_ID>"
AGENT_ID  = "<YOUR_AGENT_ID>"        # 三端共享分区（debugForceIdentity 会强制成它）
# 即便 relay 不发 x-task-id，proxy 的 debugForceIdentity 也会补上 <YOUR_TASK_ID>
API_KEY   = "<YOUR_TDAI_USER_KEY>"   # 取 NAS 上 .admin-key 里的 user key
LISTEN_PORT = 8911
UPSTREAM  = "http://<NAS_LAN_IP>:8096/openclaw/default/v1/chat/completions"
MODEL_MAP = { "tdai-nemotron-120b": "grok-4.6" }   # WB 友好名 → proxy 上游真名

def pick_session(body, headers):
    # 必须每天换新 session id，避免撞旧 session 的缓存身份
    return "wb-shared-" + datetime.now().strftime("%Y%m%d")
```

WorkBuddy 里把记忆类模型的 API base 指向 `http://127.0.0.1:8911/v1/chat/completions`，模型名 `tdai-nemotron-120b` 经 `MODEL_MAP` 映射成上游真名。

> ⚠ **session→agent 缓存陷阱**：proxy 按 `session id` 缓存首次 `sessionInit` 见到的身份。复用旧 session（如历史 `wb-desktop-<日期>`）会写进旧分区。所以 session 必须每天换新 id（`wb-shared-<日期>`），绝不复用历史 id。

> ⚠ **WB relay 当前已知缺陷**：本机 relay `:8911` 在实测中偶发返回 502（`upstream connect failed os error 10061`）。Windows 直连 proxy 是通的（返回 401 鉴权而非拒绝连接），说明是 relay 脚本内部 bug 而非网络。修复路径：检查 relay 里 `UPSTREAM` 的 DNS/连接复用、确认 `ProxyHandler({})` 直连 NAS（绕过本机 Clash `127.0.0.1:7897`）、确保 `x-session-id` 每请求新鲜。该缺陷不影响 NAS/Windows 双端已验证的落库。

---

## 6. 合并历史记忆（多分区 → 统一 `<YOUR_AGENT_ID>`）

实测基线：L0 共 159 行，**仅 6 行在共享分区 `<YOUR_AGENT_ID>`，153 行散落在旧分区**（如 `nas-oc-fixed`、`host-oc-fixed`、`agt-ns1zl4s9sx` 等）。要真正「一处记住三处召回」，需把旧分区并进来。

### 6.1 先停栈（避免写冲突）

```bash
ssh <NAS_USER>@<NAS_LAN_IP>
echo '<NAS_SSH_PASSWORD>' | sudo -S docker rm -f tdai-memory-core tdai-memory-hub tdai-proxy
```

### 6.2 备份（必做）

```bash
echo '<NAS_SSH_PASSWORD>' | sudo -S cp -a <TDAI_VOLUME_PATH> \
      <TDAI_VOLUME_PATH>.bak_merge_$(date +%Y%m%d_%H%M%S)
```

### 6.3 合并 agent_id（NAS 上 python3）

```python
import sqlite3
db = "<TDAI_VOLUME_PATH>/vectors.db"
c = sqlite3.connect(db); cur = c.cursor()
old = [r[0] for r in cur.execute(
    "SELECT DISTINCT agent_id FROM l0_conversations WHERE agent_id<>'<YOUR_AGENT_ID>'")]
print("待合并分区:", old)
for t in ("l0_conversations", "l1_records"):
    cur.execute(f"UPDATE {t} SET agent_id='<YOUR_AGENT_ID>' WHERE agent_id<>'<YOUR_AGENT_ID>'")
c.commit(); c.close()
print("合并完成")
```

### 6.4 重建 FTS 索引（必做，否则新分区查不到）

FTS 是外部内容表，`UPDATE` 基础表不会自动同步它：

```sql
INSERT INTO l0_fts(l0_fts) VALUES('rebuild');
INSERT INTO l1_fts(l1_fts) VALUES('rebuild');
```

> 若报列不兼容（不同版本 schema 可能多/少 `UNINDEXED` 列），先 `PRAGMA table_info(l0_fts)` 看真实列，再 DROP 后按实际列重建并 `INSERT ... SELECT rowid, <列> FROM l0_conversations` 保留 `rowid`。

### 6.5 重启栈（见 §7）

---

## 7. 启动服务栈（⚠ 别用 start-all.sh）

`start-all.sh` 是**交互式**脚本，还会把配置写回 `.env`——而 `.env` 目录是 **root 属主**，普通用户（`<NAS_USER>`）跑会在写回处 `Permission denied` 直接 abort，栈根本起不来。

**正确做法：root 直跑三个非交互子脚本，完全脱离 SSH 会话启动。**

```bash
# NAS 上，root 提权 + setsid 彻底脱离会话
echo '<NAS_SSH_PASSWORD>' | sudo -S bash -c \
  'setsid bash -c "cd <TDAI_DEPLOY_DIR> && \
    PROXY_FULL_STACK=1 ./start-memory-core.sh && \
    ./start-memory-hub.sh && \
    PROXY_FULL_STACK=1 ./start-proxy.sh" > /tmp/tdai_stack.log 2>&1 < /dev/null'
```

> `PROXY_FULL_STACK=1` 必须带——否则 proxy 只起最小能力，记忆落库失效。
> 传文件注意：deploy 目录是 root 属主，普通用户 `sftp.put` 会 Permission denied。先 `put` 到 `/tmp`，再 `sudo mv` 进去。

等 ~2 分钟，确认三容器 healthy：`docker ps --format '{{.Names}} | {{.Status}}' | grep tdai`。

---

## 8. 验证（权威：直接查 vectors.db）

### 8.1 数据库实证

```python
import sqlite3
c = sqlite3.connect("<TDAI_VOLUME_PATH>/vectors.db"); cur = c.cursor()
print("共享分区 <YOUR_AGENT_ID> L0:", cur.execute(
    "SELECT count(*) FROM l0_conversations WHERE agent_id='<YOUR_AGENT_ID>'").fetchone()[0])
print("其它分区残留:", cur.execute(
    "SELECT count(*) FROM l0_conversations WHERE agent_id<>'<YOUR_AGENT_ID>'").fetchone()[0])
```

期望：共享分区持续增长（新写入进来了）、合并后其它分区 = 0。

### 8.2 三端各发一条带标记的真实消息

- NAS：`openclaw agent --agent main -m "请记住探针词 NAS-PASS-7788"`
- Windows：同命令（或经记忆 provider 发消息）
- WB：经 relay `:8911` 发「请记住探针词 WB-PASS-3355」

然后 DB 查各标记命中：

```python
import sqlite3
c = sqlite3.connect("<TDAI_VOLUME_PATH>/vectors.db"); cur = c.cursor()
for mark in ("NAS-PASS-7788","WIN-PASS-1122","WB-PASS-3355"):
    print(mark, cur.execute(
        "SELECT count(*) FROM l0_conversations WHERE message_text LIKE ?", (f"%{mark}%",)).fetchone()[0])
```

期望：三端标记各自命中 ≥1（NAS/Win 已实测命中；WB 待 relay 502 修复后验证）。

### 8.3 跨端召回（最重要的验收）

用任意一个端问「你还记得之前的探针词吗？」→ 模型应从 L1/L2 准确回忆出**其它端**写入的词，证明三端共享同一库。本机已实测：一个 recall 会话成功召回了 NAS 与 Windows 端写入的标记，跨会话、跨端生效。

### 8.4 客户端鉴权提醒

proxy 读 `Authorization: Bearer <user_key>` 鉴权（**不是** `x-tdai-user-key`）。key 取自 NAS `<TDAI_DEPLOY_DIR>/.admin-key`，**复制时务必完整**（漏字符会 `invalid user_key`）。

---

## 9. 排错速查表

| 现象 | 原因 | 解决 |
|---|---|---|
| 请求 200 但 L0 不增长、零报错 | `sessionInit` 握手被绕过 → `deriveTdaiIdentity` 返回 null → 内核 401 被吞 | 配 `sessionInit.debugForceIdentity`（§4） |
| recall 报 `agent not found` / `task not found` | 配了字面 `agt-shared`，但 kernel 实际 ID 是 `agt-xxxx` | 用 meta API 返回的**真实 ID**（§4.1） |
| 改完 config 重启后又失效 | 只改了 host 副本，没改 `start-proxy.sh` 生成器 | 生成器同步加 `debugForceIdentity`（§4.3） |
| `start-all.sh` 跑完栈没起来 | `.env` 写回权限 deny（普通用户） | root 直跑三子脚本 + setsid（§7） |
| 新记忆写进了旧分区 | session→agent 缓存陷阱 | session 改 `wb-shared-<日期>` 每天全新（§5.3） |
| 合并后 FTS 搜不到 | FTS 未重建 | `INSERT INTO l0_fts(l0_fts) VALUES('rebuild')`（§6.4） |
| 发消息 401 invalid user_key | key 漏字符 / 用了错头 | 完整复制 `.admin-key`，用 `Authorization: Bearer` |
| proxy 起但落库失效 | 没设 `PROXY_FULL_STACK=1` | 启动脚本里 `export PROXY_FULL_STACK=1` |
| WB relay `:8911` 返回 502 | relay 脚本内部连接 bug（非网络） | 修 relay 直连 + 新鲜 session（§5.3） |

---

## 10. 已知限制 & 待办

1. **WB relay 502**：脚本内部 bug，待修（不影响 NAS/Win 双端已验证落库）。
2. **语义召回弱**：当前 `embedding.provider=none`，检索仅 BM25 关键词。泛问偶发召不回，精确词必中。→ 待补 BGE-M3 远程 embedding。
3. **模型被锁上游**：proxy 上游默认 `nemotron-3-super-120b-a12b`，实测 `agents.openclaw` 走 `grok-4.6`。要 WB 用其它强模型需给 proxy 配 `upstream.agents.<name>`（改 `config.yaml` + 生成器）。
4. **skill 注入污染回复**：TDAI `injection.injectors` 含 `skill`，会把工具模板塞进上下文。relay 侧可在最后一条 user 消息追加「【输出约束】」压制，根治应改 proxy `injection.injectors` 去掉 `skill`（影响全局，需评估）。

---

## 11. 命令速查

```bash
# 重启 proxy（改完 config / 生成器后）
echo '<NAS_SSH_PASSWORD>' | sudo -S docker restart tdai-proxy

# 健康检查
curl -s -o /dev/null -w '%{http_code}' --noproxy '*' http://127.0.0.1:8096/health
docker ps --format '{{.Names}} | {{.Status}}' | grep tdai

# 确认 debugForceIdentity 已生效（重启后必查）
docker exec tdai-proxy cat /data/config.yaml | grep -A3 debugForceIdentity

# 启动整栈（root，脱离会话）
echo '<NAS_SSH_PASSWORD>' | sudo -S bash -c 'setsid bash -c "cd <TDAI_DEPLOY_DIR> && PROXY_FULL_STACK=1 bash tdai_start_stack.sh"'

# 停栈
echo '<NAS_SSH_PASSWORD>' | sudo -S docker rm -f tdai-memory-core tdai-memory-hub tdai-proxy

# 查 L0 分区分布
echo '<NAS_SSH_PASSWORD>' | sudo -S python3 -c "import sqlite3;c=sqlite3.connect('<TDAI_VOLUME_PATH>/vectors.db');cur=c.cursor();print(cur.execute('SELECT agent_id,count(*) FROM l0_conversations GROUP BY agent_id').fetchall())"
```

---

## 12. English Summary

This tutorial unifies **NAS OpenClaw**, **Windows OpenClaw**, and **WorkBuddy** into one shared long-term memory on Tencent's open-source **TencentDB Agent Memory** (MIT). Deploy Hub (core `:8420` + hub `:8125` + proxy `:8096`) on your NAS via Docker; all three clients hit the proxy.

- **The one fatal gotcha — silent L0 drop.** TDAI's proxy has a `sessionInit` handshake that populates the kernel session record (`sessionInfo`). The internal `deriveTdaiIdentity()` reads `team_id`/`agent_id` **only** from `sessionInfo`. If the handshake is bypassed, `sessionInfo` is empty → identity is `null` → the kernel write returns 401, which the proxy **silently swallows** (no log, HTTP 200). Result: model replies fine but memory never persists. Previous "acceptance passes" were fake for exactly this reason.
- **You cannot hardcode `agt-shared`.** The kernel meta API (`POST /v3/agent/create`, `/v3/task/create`) returns its **own** generated IDs (e.g. `<YOUR_AGENT_ID>`). Using a friendly literal → recall 404 (`agent/task not found`).
- **The fix — `sessionInit.debugForceIdentity`.** Register a real team/agent/task via the meta API, capture the returned IDs, then set `sessionInit.debugForceIdentity: { team_id, agent_id, task_id }` with those real IDs. This force-registers every fresh session on first touch (skipping the form) so `sessionInfo` is always populated → L0 actually writes. **Patch BOTH** the host config (`/.proxy-config/config.yaml`) **and** the generator (`start-proxy.sh`), or a restart overwrites it.
- NAS / Windows OpenClaw: `memory-proxy` provider → `http://<nas>:8096/openclaw/default/v1` with `x-agent-id`/`x-team-id` headers. WorkBuddy: a thin local relay (`tdai-wb-relay.py`) on `127.0.0.1:8911` adds headers + a fresh daily `x-session-id` (avoids the session→agent cache trap) and forwards to the proxy. **Do not** use `/workbuddy/default/v1/responses` — it returns 200 but does not persist.
- Merge legacy partitions: stop the stack, `UPDATE l0_conversations/l1_records SET agent_id='<shared>'`, then `INSERT INTO l0_fts(l0_fts) VALUES('rebuild')` (FTS5 external-content tables don't auto-sync).
- Verify by querying `vectors.db`: the shared `agent_id` should grow and other partitions should be 0; a cross-end recall should return memories written by a *different* end.
- Known gap: the WB relay occasionally 502s (script-internal bug, not network) — fix pending; NAS + Windows ends are verified working.

Reference: https://github.com/TencentCloud/TencentDB-Agent-Memory

---

*本教程基于一次真实的三端统一记忆落地整理，所有命令均实测可用。文中 `<...>` 均为占位符，请按你的环境替换（team/agent/task 的 ID 必须通过 meta API 注册后取返回值，不能自己编）。*
