---
name: log-svr
description: >
  查询 LogReportSvr Unity 游戏日志。仅在用户明确说"去 LogReport 查日志"这条短语且同时提供了 device_id 时才触发。任何其他说法（"看日志"/"查 error"/"崩溃了"/"有没有报错"等）一律不触发，由其他 skill 或主对话处理。device_id 是硬性入参，若用户只说了短语没给 device_id，必须先反问要 device_id，绝不擅自全表扫或换其他口径开干。
allowed-tools: ["Bash"]
---

# log-svr

LogReportSvr Unity 日志查询。本 skill 不依赖 MCP，直接 `curl` 打后端 HTTP API。

## 接入点

- **Base URL**：`http://124.220.6.174:8080`
- **Auth**：`X-API-Key` 头（带 `device_id` 的 3 个端点可免）
- **取 Key**：dashboard `http://124.220.6.174:8080/dashboard/` 注册一个 AppKey，或找管理员要
- **存放（首次配置）**：

  ```bash
  mkdir -p ~/.config/logreport && chmod 700 ~/.config/logreport
  printf '%s' 'YOUR_KEY_HERE' > ~/.config/logreport/key && chmod 600 ~/.config/logreport/key
  ```

后续脚本里：`KEY=$(cat ~/.config/logreport/key)`，请求加 `-H "X-API-Key: $KEY"`。

## 入口判断（device_id 是硬要求）

skill 已被触发，意味着用户说了"去 LogReport 查日志"。**必须先确认 device_id 已给**：

- ✅ 用户给了 device_id → 从工具 1 (`get_session`) 或工具 2 (`list_sessions`) 起步
- ❌ 用户没给 device_id → 立刻反问"请给我一个 device_id"，**不要**回退到 list_recent_errors / search_logs 模糊找补。
- ❌ 用户给的是 session_id / fingerprint / tag 但没 device_id → 也要先反问 device_id，可以同时让用户确认"是不是同一台设备的 <session_id/fp/tag>"。

device_id 确认后，按下面 8 个工具 + 3 条链路操作。

## 8 个工具

> 所有响应都带 `_tip` 字段，**优先读它**作为下一步指引。

### 1. `get_session` — 该设备最新一条 session（按 started_at DESC 取第 1 条）

```bash
DEVICE=xxx
curl -sS "http://124.220.6.174:8080/api/v1/agent/sessions?device_id=$DEVICE&limit=1" | jq '.sessions[0]'
```

### 2. `list_sessions` — 该设备最近 N 条

```bash
DEVICE=xxx
curl -sS "http://124.220.6.174:8080/api/v1/agent/sessions?device_id=$DEVICE&limit=10&since=7d" | jq .
```

默认窗口 30d，按 `started_at` DESC 排。可叠 `&platform=iOS` / `&since=24h` / `&until=...`。

### 3. `get_session_summary` — 单 session 分诊（按 level 计数 + 样本）

```bash
KEY=$(cat ~/.config/logreport/key)
SID=xxx
curl -sS -H "X-API-Key: $KEY" \
  "http://124.220.6.174:8080/api/v1/agent/sessions/$SID/summary?samples_per_level=all" | jq .
```

返回 `total_by_level` 和每级 `{ samples, total, has_more }`。响应 200KB 预算，超了按 debug→info→warning→error 顺序裁剪。

### 4. `get_session_logs` — 单 session 分页日志

```bash
KEY=$(cat ~/.config/logreport/key)
SID=xxx
curl -sS -H "X-API-Key: $KEY" \
  "http://124.220.6.174:8080/api/v1/agent/sessions/$SID/logs?level=error&limit=50" | jq .
```

参数：`level`（error/warning/info/debug）、`tag`（模糊）、`keyword`（模糊）、`limit`（≤200）、`offset`。

### 5. `get_log_context` — 围绕某条日志的上下文窗口

```bash
KEY=$(cat ~/.config/logreport/key)
SID=xxx ; SEQ=287
curl -sS -H "X-API-Key: $KEY" \
  "http://124.220.6.174:8080/api/v1/agent/sessions/$SID/logs?around_sequence=$SEQ&before=20&after=10" | jq .
```

`before` / `after` 默认 20 / 10，最大 200。

### 6. `search_logs` — 设备内跨 session 全文搜

```bash
DEVICE=xxx
curl -sS "http://124.220.6.174:8080/api/v1/agent/logs/search?device_id=$DEVICE&keyword=NullReferenceException&level=error&since=24h&limit=50" | jq .
```

参数：`tag` / `keyword` / `fingerprint` / `exception_type` / `platform` / `app_version` / `is_fatal` / `group_by`（=`session` / `platform` / `app_version` / `device_model`）。默认窗口 24h，仅 `tag` 时扩到 7d。

### 7. `list_recent_errors` — 全局滚动窗口 top-N fingerprint + 趋势

```bash
KEY=$(cat ~/.config/logreport/key)
curl -sS -H "X-API-Key: $KEY" \
  "http://124.220.6.174:8080/api/v1/agent/errors/recent?since=24h&limit=20&include_trend=true" | jq .
```

每项含 `count`、`affected_sessions`、`platforms`、`count_prev_window`、`is_new`。关注 `is_new:true` 或 `count` 暴涨的。

### 8. `get_issue` — 单 fingerprint 详情

```bash
KEY=$(cat ~/.config/logreport/key)
FP=a1b2c3d4e5f60718
curl -sS -H "X-API-Key: $KEY" \
  "http://124.220.6.174:8080/api/v1/agent/issues/$FP?since=7d" | jq .
```

返回 `sample_message` / `sample_stack` / `exception_type` / `first_seen_at` / `last_seen_at` / `affected_sessions` / `affected_devices` / `breakdown`（by_version/by_platform/by_device_model）/ `recent_sessions[10]`。

## 3 条常用链路

### 链路 A — 已知 device_id，看这台机最近的崩溃

1. 工具 2 拉最近 3 条 session（`limit=3`）
2. 对每条调工具 3 看 error/warning 数量
3. 错误最多 / 最新那条调工具 4（`level=error`）拉全 error
4. 可疑行（每条日志的 `sequence` 字段）调工具 5 看上下文
5. 输出：是否规律性、是否启动崩溃（duration 异常短）、典型错误

### 链路 B — 全局排查（不知道设备）

1. 工具 7 看 top fingerprint 和趋势
2. 选一个值得查的 fingerprint，调工具 8 拿 `sample_stack` + `recent_sessions[]`
3. 从 recent_sessions 挑 `last_seen_at` 最新的 session_id，调工具 3
4. 工具 5 看现场，输出根因猜测

### 链路 C — 按 tag 查问题（device_id 已知）

1. 工具 6（`device_id` + `tag` + `level=error` + `group_by=session`）拿聚合
2. 挑命中最多 / 最近的 session
3. 工具 3 看整体，工具 4（`tag=...&limit=100`）拉该 tag 的所有日志
4. 必要时工具 5 看现场

## 诊断节奏

1. 先看 summary 别一上来拉日志
2. error 看完看 warning 找前因
3. 用 `tag` / `keyword` / `level` 收敛，避免无脑翻页
4. 跨 session 找通病用工具 6（设备内）或工具 7（全局）

## 故障排除

- **HTTP 401**：没设 `X-API-Key`，且当前端点 URL 里没有 `device_id` query → 加 `-H "X-API-Key: $KEY"`
- **HTTP 404 on `/sessions/foo/summary`**：session_id 不存在，先用工具 2 / 工具 6 找正确的 id
- **响应 truncated**：`get_session_summary` 超 200KB 自动裁，看 `truncated_reason`；用工具 4 配 `level` 单独拉
- **连不上 124.220.6.174:8080**：检查腾讯云安全组入站 8080；或走 MCP（如能用）`https://mcp.bombgit.top:8443/mcp`
