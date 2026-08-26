# 请求示例 / Request examples

## WorkBuddy（Responses API，必须结构化）
```bash
curl -sS http://<NAS_LAN_IP>:8096/workbuddy/default/v1/responses \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_TDAI_API_KEY>" \
  -H "x-team-id: <TEAM_ID>" \
  -H "x-agent-id: <WORKBUDDY_AGENT_ID>" \
  -H "x-task-id: <TASK_ID>" \
  -d '{
    "model": "<YOUR_MODEL>",
    "input": [
      { "type": "message", "role": "user",
        "content": [ { "type": "input_text", "text": "请记住：我喜欢用中文回复" } ] }
    ],
    "stream": true
  }'
```

## OpenClaw（chat completions，仅 header 鉴权）
```bash
curl -sS http://<NAS_LAN_IP>:8096/openclaw/default \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_TDAI_API_KEY>" \
  -H "x-team-id: <TEAM_ID>" \
  -H "x-agent-id: <NAS_OC_AGENT_ID>" \
  -H "x-task-id: <TASK_ID>" \
  -d '{
    "model": "<YOUR_MODEL>",
    "messages": [ { "role": "user", "content": "请记住：我喜欢用中文回复" } ]
  }'
```

## 验证检索（按 agent 分区）
用 Hub 面板 `http://<NAS_LAN_IP>:8125` 或 core 检索接口，按 `x-tdai-agent-id: <AGENT_ID>` 查询探针词是否落在对应分区。
