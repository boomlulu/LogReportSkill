# log-svr 安装与使用

LogReportSvr Unity 日志查询 skill — 让 Claude Code 直接 `curl` 后端拉日志，无需 MCP。本文给人看，配套的 [SKILL.md](SKILL.md) 是给 agent 看的工具手册。

## 前置

- Claude Code 已装（CLI / VS Code 扩展 / Claude App 任选）
- 能拿到目标设备的 `device_id`
- 能访问 `http://124.220.6.174:8080`（公司内网 / VPN 通即可）

## 安装

```bash
# 1) clone 本仓库到本地稳定路径
git clone https://github.com/boomlulu/LogReportSkill.git ~/.claude/.skill-sources/LogReportSkill

# 2) 软链 log-svr 到 ~/.claude/skills/
mkdir -p ~/.claude/skills
ln -sfn ~/.claude/.skill-sources/LogReportSkill/log-svr ~/.claude/skills/log-svr
```

### 验证

1. 重启 Claude Code 会话
2. 看 system reminder 的 available-skills 列表里有没有 `log-svr` — 出现就 OK

## 触发口径

skill 只在 **同时满足** 这两条时才激活：

1. 用户说出完整短语 **"去 LogReport 查日志"**
2. 提供 `device_id`

否则不触发。device_id 缺失时 skill 会反问，不会擅自全表扫。

### 触发 / 不触发对照

| 输入 | 是否触发 |
|---|---|
| 去 LogReport 查日志，device_id=abc123 看看最近一次 session | ✅ |
| 去 LogReport 查日志，device_id 是 xyz789，session id 是 5e8a... | ✅ |
| 看下日志有没有 error | ❌（缺触发短语） |
| 查 error / 崩溃了 / 出问题了 | ❌（缺触发短语） |
| 去 LogReport 查日志，最近什么在炸 | ❌（缺 device_id，会被反问） |

## 能做什么

8 个工具 + 3 条常用排查链路，覆盖：

- 看某设备最近运行情况
- 单 session 分诊 → 拉错误日志 → 看上下文
- 全局 top error fingerprint 趋势
- 按 tag / fingerprint / keyword 跨 session 搜

详细工具列表 + curl 样例见 [SKILL.md](SKILL.md)。

## 三条典型对话示例

### 1. 看这台机最近的崩溃

> 去 LogReport 查日志，device_id=abc123 看看最近 3 次运行有没有崩溃

agent 会：
1. `list_sessions` 拉最近 3 条
2. 对每条调 `get_session_summary`
3. 报告 error / warning 数量 + 是否疑似启动崩溃
4. 等你说"钻进最新那次" 再拉具体 error 日志 + 上下文

### 2. 全局 fingerprint 排查

> 去 LogReport 查日志，device_id=abc123 看下最近什么 fingerprint 在炸

agent 会：
1. `list_recent_errors` 拿 top fingerprint + 趋势（is_new / count_prev_window）
2. 你挑一个，让 agent 调 `get_issue` 拿 stack + recent_sessions
3. 从 recent_sessions 挑一条钻 `get_session_summary` + `get_log_context`

### 3. 按 tag 查问题

> 去 LogReport 查日志，device_id=abc123 看 tag=Network 最近有没有错

agent 会：
1. `search_logs(device_id, tag:Network, level:error, group_by:session)`
2. 挑命中最多的 session 钻 summary + 该 tag 全量日志

## 更新

```bash
cd ~/.claude/.skill-sources/LogReportSkill && git pull
```

软链自动跟最新版本。

## 卸载

只卸 log-svr：

```bash
rm ~/.claude/skills/log-svr
```

完全卸掉本仓库的所有 skill + 删本地源：

```bash
find ~/.claude/skills -maxdepth 1 -type l -lname '*/.skill-sources/LogReportSkill/*' -delete
rm -rf ~/.claude/.skill-sources/LogReportSkill
```

## 故障排除

| 现象 | 原因 / 修复 |
|---|---|
| available-skills 列表里没有 log-svr | 软链没建对，检查 `readlink ~/.claude/skills/log-svr`；或没重启 Claude Code |
| 说了短语 skill 没触发 | 触发口径只认完整短语，简写 / 改字都不行；device_id 也得给出来 |
| skill 触发但 agent 一直反问 device_id | 你确实没在请求里写 device_id；补上即可 |
| curl 报 HTTP 401 | URL 漏了 `?device_id=$DEVICE`，所有端点都要带，包括 `/errors/recent` / `/issues/:fp` |
| curl 报 HTTP 404 | session_id / fingerprint 拼错或不存在；先用 `list_sessions` 或 `list_recent_errors` 找正确的 id |
| 连不上 124.220.6.174:8080 | 公司 VPN / 防火墙；或确认机器对外访问能力 |

## 参考

- 工具手册（agent 用）：[SKILL.md](SKILL.md)
- 后端 base URL：`http://124.220.6.174:8080`
- 仓库：https://github.com/boomlulu/LogReportSkill
- 发布新 skill：见仓库根目录 [PUBLISHING.md](../PUBLISHING.md)
