# 请求示例 / Request examples（统一 `agt-shared` 版）

> 三端全部用**同一个** `agent_id = agt-shared`、`team_id = team-nql9lamkba`，且**省略 `x-task-id`**（退化为 agent 级全量召回，三端共享）。
> WorkBuddy 统一走 proxy 的 `/openclaw/default/v1/chat/completions`（经本地 relay 补头），**不要**走 `/workbuddy/default/v1/responses`（那条不落库）。

## 1. OpenClaw（NAS / Windows 相同）— chat completions

OpenClaw 的 `memory-proxy` provider 已替你补好头，只需保证 `x-agent-id: agt-shared`。等价 curl：

```bash
curl -sS http://192.168.31.123:8096/openclaw/default/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_TDAI_ADMIN_KEY>" \
  -H "x-team-id: team-nql9lamkba" \
  -H "x-agent-id: agt-shared" \
  -d '{
    "model": "grok-4.6",
    "messages": [ { "role": "user", "content": "请记住：机房门禁密码是 8823" } ]
  }'
```

## 2. WorkBuddy — 经本地 relay（推荐）

WorkBuddy 模型配置不支持自定义请求头，用转发层 `tdai-wb-relay.py` 补头：

- 监听 `127.0.0.1:8911` → 转发到 `http://192.168.31.123:8096/openclaw/default/v1/chat/completions`
- relay 自动加 `x-agent-id: agt-shared` / `x-team-id: team-nql9lamkba`，每天换新 session `wb-shared-<日期>`
- WorkBuddy 里把记忆模型 API base 指向 `http://127.0.0.1:8911/v1/chat/completions`

等价直连 proxy（绕过 relay，验证用）：

```bash
curl -sS http://192.168.31.123:8096/openclaw/default/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_TDAI_ADMIN_KEY>" \
  -H "x-team-id: team-nql9lamkba" \
  -H "x-agent-id: agt-shared" \
  -H "x-session-id: wb-shared-$(date +%Y%m%d)" \
  -d '{
    "model": "grok-4.6",
    "messages": [ { "role": "user", "content": "请记住探针词 TDAI_SHARED_TEST_XXXX" } ]
  }'
```

> 首次会话 proxy 会返回 `ask_followup_question` 工具调用（sessionInit 表单），回答「是，关联团队资产」即可完成握手；之后记忆落 `agt-shared`。

## 3. 合并历史记忆（停栈后，NAS 上 python3）

```python
import sqlite3
db = "/vol4/docker/volumes/tdai-memory-core-data/_data/vectors.db"
c = sqlite3.connect(db); cur = c.cursor()
for t in ("l0_conversations", "l1_records"):
    cur.execute(f"UPDATE {t} SET agent_id='agt-shared' WHERE agent_id<>'agt-shared'")
c.commit(); c.close()
# 重建 FTS（外部内容表不随 UPDATE 同步）
c2 = sqlite3.connect(db); cur2 = c2.cursor()
cur2.execute("INSERT INTO l0_fts(l0_fts) VALUES('rebuild')")
cur2.execute("INSERT INTO l1_fts(l1_fts) VALUES('rebuild')")
c2.commit(); c2.close()
```

## 4. 验证

```python
import sqlite3
c = sqlite3.connect("/vol4/docker/volumes/tdai-memory-core-data/_data/vectors.db"); cur = c.cursor()
print("agt-shared L0:", cur.execute("SELECT count(*) FROM l0_conversations WHERE agent_id='agt-shared'").fetchone()[0])
print("其它分区残留:", cur.execute("SELECT count(*) FROM l0_conversations WHERE agent_id<>'agt-shared'").fetchone()[0])
```

期望：`agt-shared` 有数据、其它分区 = 0 → 三端已共享同一库。
