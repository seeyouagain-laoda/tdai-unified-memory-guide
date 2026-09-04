# 请求示例 / Request examples（debugForceIdentity 正解版）

> ⚠️ 本文取代旧版 `agt-shared` 示例。旧版按字面 `agt-shared` 配置会 **recall 404**（kernel 用自己生成的 ID，不是你传的）。
> 正确做法：先用 meta API 注册 team/agent/task，取**返回值**填入 `sessionInit.debugForceIdentity`（见 README §4），三端全部经 proxy `:8096` 落库。
> 所有 `<...>` 均为占位符，按你的环境替换。

## 1. 注册真实 agent / task（取返回值）

```bash
# agent（返回体里的 agent_id 才是真值）
curl -sS http://<NAS_LAN_IP>:8420/v3/agent/create \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_TDAI_USER_KEY>" \
  -d '{ "team_id": "<YOUR_TEAM_ID>", "name": "shared", "visibility": "team" }'
# → { "agent_id": "<YOUR_AGENT_ID>", ... }

# task
curl -sS http://<NAS_LAN_IP>:8420/v3/task/create \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_TDAI_USER_KEY>" \
  -d '{ "team_id": "<YOUR_TEAM_ID>", "title": "shared" }'
# → { "task_id": "<YOUR_TASK_ID>", ... }
```

## 2. OpenClaw（NAS / Windows 相同）— chat completions

OpenClaw 的 `memory-proxy` provider 已替你补好头，只需保证 `x-agent-id` / `x-team-id` 对齐（proxy 的 `debugForceIdentity` 会强制成 `<YOUR_AGENT_ID>` / `<YOUR_TASK_ID>`）。等价 curl：

```bash
curl -sS http://<NAS_LAN_IP>:8096/openclaw/default/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_TDAI_USER_KEY>" \
  -H "x-team-id: <YOUR_TEAM_ID>" \
  -H "x-agent-id: <YOUR_AGENT_ID>" \
  -H "x-session-id: nas-shared-$(date +%Y%m%d)" \
  -d '{
    "model": "grok-4.6",
    "messages": [ { "role": "user", "content": "请记住：机房门禁密码是 8823" } ]
  }'
```

> `x-session-id` 必须**每天换新**（如 `nas-shared-<日期>`），避免撞旧 session 的缓存身份导致写进旧分区。

## 3. WorkBuddy — 经本地 relay（推荐）

WorkBuddy 模型配置不支持自定义请求头，用转发层 `tdai-wb-relay.py` 补头：

- 监听 `127.0.0.1:8911` → 转发到 `http://<NAS_LAN_IP>:8096/openclaw/default/v1/chat/completions`
- relay 自动加 `x-agent-id: <YOUR_AGENT_ID>` / `x-team-id: <YOUR_TEAM_ID>`，每天换新 session `wb-shared-<日期>`
- WorkBuddy 里把记忆模型 API base 指向 `http://127.0.0.1:8911/v1/chat/completions`
- **不要**走 `/workbuddy/default/v1/responses`（那条返回 200 但不落库）

## 4. 合并历史记忆（停栈后，NAS 上 python3）

```python
import sqlite3
db = "<TDAI_VOLUME_PATH>/vectors.db"
c = sqlite3.connect(db); cur = c.cursor()
for t in ("l0_conversations", "l1_records"):
    cur.execute(f"UPDATE {t} SET agent_id='<YOUR_AGENT_ID>' WHERE agent_id<>'<YOUR_AGENT_ID>'")
c.commit(); c.close()
# 重建 FTS（外部内容表不随 UPDATE 同步）
c2 = sqlite3.connect(db); cur2 = c2.cursor()
cur2.execute("INSERT INTO l0_fts(l0_fts) VALUES('rebuild')")
cur2.execute("INSERT INTO l1_fts(l1_fts) VALUES('rebuild')")
c2.commit(); c2.close()
```

## 5. 验证

```python
import sqlite3
c = sqlite3.connect("<TDAI_VOLUME_PATH>/vectors.db"); cur = c.cursor()
print("共享分区 L0:", cur.execute(
    "SELECT count(*) FROM l0_conversations WHERE agent_id='<YOUR_AGENT_ID>'").fetchone()[0])
print("其它分区残留:", cur.execute(
    "SELECT count(*) FROM l0_conversations WHERE agent_id<>'<YOUR_AGENT_ID>'").fetchone()[0])
# 跨端召回：用任一端问「还记得之前的探针词吗？」应召回其它端写入的内容
```

期望：共享分区持续增长、合并后其它分区 = 0、跨端召回成功。
