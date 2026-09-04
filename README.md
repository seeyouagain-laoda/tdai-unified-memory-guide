# TDAI 三端共享记忆库 · 完整部署教程

> 适用场景：把 **NAS OpenClaw / Windows OpenClaw / WorkBuddy** 三个端的长期记忆，统一进**同一个** TencentDB Agent Memory（TDAI）记忆库，做到「一处记住、三处都能召回」。
> 实测环境：飞牛 fnOS NAS（192.168.31.123）+ Windows 11 + WorkBuddy。TDAI 以 Docker 四容器部署在 NAS。
> 作者踩过的所有坑都写进来了，照做即可。

---

## 0. 一句话原理

TDAI 的记忆库是一张 sqlite（`vectors.db`），**`agent_id` 是分区键**。只要三个端都往**同一个 `agent_id`** 写，它们就天然共享同一份记忆。官方作用域规则：

> 相同 `team_id` + 相同 `agent_id`（且省略 `x-task-id`） → L0–L3 跨端全量共享。

所以我们只做一件事：**把三端的 `agent_id` 统一成 `agt-shared`**，再把历史记忆也并进去。

---

## 1. 架构与端口

| 容器 | 端口 | 作用 |
|---|---|---|
| `tdai-memory-core` | 8420 | 内核 gateway，真正落库的地方 |
| `tdai-memory-hub` | 8125 / 8424 | 面板 + 知识库 |
| `tdai-proxy` | 8096 | 对外代理；三端都打它 |

数据落盘位置（NAS 宿主机）：

```
/vol4/docker/volumes/tdai-memory-core-data/_data/vectors.db
```

表结构关键点：

- `l0_conversations`：原始对话（分区键 `agent_id`，文本列 `message_text`）
- `l1_records`：抽取出的记忆
- `l0_fts` / `l1_fts`：FTS5 全文索引表（带 `agent_id` 等过滤列，**外部内容表**，不会随基础表 UPDATE 自动同步）

三容器均 `--restart=always`，NAS 重启自愈。

---

## 2. 三端配置对齐（核心步骤）

### 2.1 NAS OpenClaw — `/home/半夏/.openclaw/openclaw.json`

两处都要改成 `agt-shared`：

```jsonc
{
  "plugins": {
    "entries": {
      "memory-tencentdb": {
        "config": {
          "mode": "gateway",
          "server": { "url": "http://127.0.0.1:8420", "apiKey": "local", "instanceId": "agt-shared" }
        }
      }
    }
  },
  "providers": {
    "memory-proxy": {
      "type": "openai",
      "base_url": "http://192.168.31.123:8096/openclaw/default/v1",
      "headers": { "x-agent-id": "agt-shared", "x-team-id": "team-nql9lamkba" }
    }
  }
}
```

改完：`openclaw config validate` → `sudo systemctl restart openclaw-gateway`（NAS systemd 服务 restart 必须 sudo，且要 `MainPID` diff 验证生效）。

### 2.2 Windows OpenClaw — `C:\Users\user\.openclaw\openclaw.json`

与 NAS 完全相同：插件 `instanceId: agt-shared`、provider `memory-proxy` 头 `x-agent-id: agt-shared` / `x-team-id: team-nql9lamkba`。

### 2.3 WorkBuddy — 本地转发层 `tdai-wb-relay.py`

WorkBuddy 的模型配置不支持自定义请求头，所以用一层极薄转发层补头后打到 proxy。关键常量：

```python
TEAM_ID = "team-nql9lamkba"
AGENT_ID = "agt-shared"          # 三端共享分区
# 不发送 x-task-id：让 recall 退化为 agent 级全量，三端共享
API_KEY  = "sk-mem-..."          # 取 NAS 上 /opt/.../global-images/.admin-key 里的 admin key
LISTEN_PORT = 8911
UPSTREAM = "http://192.168.31.123:8096/openclaw/default/v1/chat/completions"

def pick_session(body, headers):
    # ⚠ 必须用「每天全新」的 session id，不能与旧的 wb-desktop-<日期> 撞车
    return "wb-shared-" + datetime.now().strftime("%Y%m%d")
```

WorkBuddy 里把记忆类模型的 API base 指向 `http://127.0.0.1:8911/v1/chat/completions`，模型名经 `MODEL_MAP` 映射成上游真名（如 `tdai-nemotron-120b → grok-4.6`）。

> ⚠ **最容易翻车的一点（session→agent 缓存陷阱）**：proxy 会按 `session id` 缓存首次 `sessionInit` 见到的 `agent_id`。如果复用了一个**旧 session**（它第一次绑定的是别的 agent），哪怕你这次发了 `x-agent-id: agt-shared`，proxy 仍会把记忆写进**旧分区**。所以 session 必须每天换新 id（`wb-shared-<日期>`），绝不复用 `wb-desktop-<日期>` 这类历史 id。

---

## 3. 合并历史记忆（多分区 → 统一 `agt-shared`）

如果三个端之前各自用了不同 `agent_id`，历史记忆散在多个分区，需要合并。

### 3.1 先停栈（重要，避免写冲突）

```bash
# NAS 上，root 执行
cd /opt/TencentDB-Agent-Memory/deploy/global-images
docker rm -f tdai-memory-core tdai-memory-hub tdai-proxy
```

### 3.2 备份（必做）

```bash
cp -a /vol4/docker/volumes/tdai-memory-core-data/_data \
      /vol4/docker/volumes/tdai-memory-core-data/_data.bak_merge_$(date +%Y%m%d_%H%M%S)
```

### 3.3 合并 agent_id（NAS 上跑 python3）

```python
import sqlite3
db = "/vol4/docker/volumes/tdai-memory-core-data/_data/vectors.db"
c = sqlite3.connect(db); cur = c.cursor()
old = [r[0] for r in cur.execute(
    "SELECT DISTINCT agent_id FROM l0_conversations WHERE agent_id<>'agt-shared'")]
print("待合并分区:", old)
for t in ("l0_conversations", "l1_records"):
    cur.execute(f"UPDATE {t} SET agent_id='agt-shared' WHERE agent_id<>'agt-shared'")
c.commit(); c.close()
print("合并完成")
```

### 3.4 重建 FTS 索引（必做，否则新分区查不到）

FTS 是外部内容表，`UPDATE` 基础表不会自动同步它。最干净的办法：

```sql
INSERT INTO l0_fts(l0_fts) VALUES('rebuild');
INSERT INTO l1_fts(l1_fts) VALUES('rebuild');
```

> 如果报列不兼容（不同版本 schema 可能多/少 `message_text_original` 之类的 UNINDEXED 列），就 DROP 后按 `PRAGMA table_info(l0_fts)` 的实际列重建，并 `INSERT ... SELECT rowid, <列> FROM l0_conversations` 保留 `rowid`。**先 `PRAGMA table_info` 看真实列，再决定用 rebuild 还是重建。**

### 3.5 合并 profile 目录（可选）

`/data/tdai-memory` 下每个 agent 有独立 profile 子目录，把旧 agent 目录合并进 `agt-shared` 目录即可（同名文件以 `agt-shared` 为准）。

### 3.6 重启栈（见第 4 节）

---

## 4. 启动服务栈（⚠ 别用 start-all.sh）

`start-all.sh` 是**交互式**脚本，还会把配置写回 `.env`——而 `.env` 所在目录是 **root 属主**，用普通用户（`半夏`）跑会在写回处 `Permission denied` 直接 abort，导致栈根本起不来。

**正确做法：root 直跑三个非交互子脚本，并完全脱离 SSH 会话启动。**

一键脚本 `tdai_start_stack.sh`（放 `/opt/TencentDB-Agent-Memory/deploy/global-images/`）：

```bash
#!/usr/bin/env bash
set -e
cd /opt/TencentDB-Agent-Memory/deploy/global-images
export PROXY_FULL_STACK=1
./start-memory-core.sh
./start-memory-hub.sh
PROXY_FULL_STACK=1 ./start-proxy.sh
```

启动（普通用户下用 sudo 提权 + setsid 彻底脱离会话）：

```bash
echo "你的sudo密码" | sudo -S bash -c \
  'setsid bash -c "cd /opt/TencentDB-Agent-Memory/deploy/global-images && bash tdai_start_stack.sh > /tmp/tdai_stack.log 2>&1 < /dev/null"'
```

> 传文件注意：deploy 目录是 root 属主，普通用户 `sftp.put` 会 Permission denied。先 `put` 到 `/tmp`，再 `sudo mv` 进去。
> `PROXY_FULL_STACK=1` 必须带——否则 proxy 只起最小能力，记忆落库失效。

等 ~2 分钟，确认三容器 healthy：

```bash
docker ps --format '{{.Names}} | {{.Status}}' | grep tdai
```

---

## 5. WorkBuddy 转发层启动 + 开机自起

```bash
# 后台启动（pythonw 无控制台、可脱离会话）
"C:\Users\user\.workbuddy\binaries\python\versions\3.13.12\pythonw.exe" \
  "C:\Users\user\.workbuddy\tdai-wb-relay.py"
```

开机自起：在 `C:\Users\user\AppData\Roaming\Microsoft\Windows\Start Menu\Startup\` 放 `tdai-wb-relay.bat`：

```bat
@echo off
start "" "C:\Users\user\.workbuddy\binaries\python\versions\3.13.12\pythonw.exe" "C:\Users\user\.workbuddy\tdai-wb-relay.py"
```

验证监听：`(netsh / python socket)` 连 `127.0.0.1:8911` 应可达；relay 日志 `tdai-wb-relay.log` 出现 `START relay`。

---

## 6. 验收

### 6.1 数据库实证（权威）

```python
import sqlite3
c = sqlite3.connect("/vol4/docker/volumes/tdai-memory-core-data/_data/vectors.db")
cur = c.cursor()
print("agt-shared L0:", cur.execute(
    "SELECT count(*) FROM l0_conversations WHERE agent_id='agt-shared'").fetchone()[0])
print("其它分区残留:", cur.execute(
    "SELECT count(*) FROM l0_conversations WHERE agent_id<>'agt-shared'").fetchone()[0])
print("WB-PKT8-UI0J 在 agt-shared:", cur.execute(
    "SELECT count(*) FROM l0_conversations WHERE agent_id='agt-shared' AND message_text LIKE '%WB-PKT8-UI0J%'").fetchone()[0])
```

期望：`agt-shared` 有数据、其它分区 = 0。

### 6.2 实时写测试（验证缓存陷阱已修）

用 WB 路径（真 admin key）向 proxy 发消息，proxy 首次会返回 `ask_followup_question` 工具调用（sessionInit 表单），回答「是，关联团队资产」走完握手，再发一条带标记的话。最终确认：

- HTTP 200（用 `Authorization: Bearer <真key>`，**key 必须一字不差**，漏字符会 `invalid user_key`）
- session 绑定到 `agt-shared`（不再掉回旧分区）

> L0 实时写是**会话级异步落盘**（会话完成才刷新），测试标记不一定立刻进库，属正常；历史合并证据已确证共享成立。

### 6.3 客户端鉴权提醒

proxy 读 `Authorization: Bearer <user_key>` 鉴权（**不是** `x-tdai-user-key`）。key 取自 NAS `/opt/TencentDB-Agent-Memory/deploy/global-images/.admin-key`，**复制时务必完整**（作者曾因漏 `Ay` 两字符误判系统故障）。

---

## 7. 排错速查表

| 现象 | 原因 | 解决 |
|---|---|---|
| `start-all.sh` 跑完栈没起来 | `.env` 写回权限 deny（普通用户） | root 直跑三子脚本 + setsid |
| 新记忆写进了旧分区 | session→agent 缓存陷阱 | session 改 `wb-shared-<日期>` 每天全新 |
| 合并后 FTS 搜不到 | FTS 未重建 | `INSERT INTO l0_fts(l0_fts) VALUES('rebuild')` |
| 发消息 401 invalid user_key | key 漏字符 / 用了错头 | 完整复制 `.admin-key`，用 `Authorization: Bearer` |
| 发消息 401 missing user_key | 没带鉴权头 | 加 `Authorization: Bearer <key>` |
| proxy 起但落库失效 | 没设 `PROXY_FULL_STACK=1` | 启动脚本里 `export PROXY_FULL_STACK=1` |
| WB 不共享记忆 | relay 没起 / 端口 8911 没监听 | 启动 relay，加 Startup 自起 |

---

## 8. 命令速查

```bash
# 启动栈（root，脱离会话）
echo PW|sudo -S bash -c 'setsid bash -c "cd /opt/TencentDB-Agent-Memory/deploy/global-images && PROXY_FULL_STACK=1 bash tdai_start_stack.sh"'

# 停栈
docker rm -f tdai-memory-core tdai-memory-hub tdai-proxy

# 健康检查
docker ps --format '{{.Names}} | {{.Status}}' | grep tdai
curl -s -o /dev/null -w '%{http_code}' --noproxy '*' http://127.0.0.1:8420/health

# 合并 agent_id（停栈后，NAS python3）
# 见 3.3 / 3.4
```

---

## 9. 给后来者的建议

1. **新接入任何一端**，只要把 `x-agent-id` 设 `agt-shared`、`x-team-id` 设 `team-nql9lamkba`、省略 `x-task-id`，就自动进同一库。
2. **session id 永不复用历史值**，每天换新。
3. **改完任何 NAS systemd 服务必须 `MainPID` diff 验证**才算生效。
4. 合并历史记忆前**先备份 volume**，FTS 必须重建。
5. 别用 `start-all.sh` 普通用户直跑，用 root + 三子脚本。

---

*本教程基于一次真实的三端统一记忆落地整理，所有命令均实测可用。涉及敏感信息（IP、密码、key）已脱敏，请按你的环境替换。*
