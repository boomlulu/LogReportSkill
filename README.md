# LogReportSkill

Claude Code skills that wrap [LogReportSvr](https://github.com/boomlulu/LogReportSvr) — query Unity game session logs as native skill actions.

## 收录的 skills

| skill | 用途 |
|---|---|
| [log-svr](log-svr/) | 查询 Unity 游戏日志的 8 个端点 + 3 条常用排查链路（无需 MCP，直接 curl 后端） |

## 安装

一次性 clone + 软链：

```bash
# 1) clone 本仓库到本地稳定路径
git clone git@github.com-boomlulu:boomlulu/LogReportSkill.git ~/.claude/.skill-sources/LogReportSkill

# 2) 每个 skill 用软链到 ~/.claude/skills/
mkdir -p ~/.claude/skills
ln -sfn ~/.claude/.skill-sources/LogReportSkill/log-svr ~/.claude/skills/log-svr
```

> 没配 `github.com-boomlulu` SSH 别名的同事用 HTTPS：
> `git clone https://github.com/boomlulu/LogReportSkill.git ~/.claude/.skill-sources/LogReportSkill`

## 更新

```bash
cd ~/.claude/.skill-sources/LogReportSkill && git pull
```

软链自动指到最新版本，无需重做。

## 配置 API Key（log-svr 必需）

`log-svr` 的 5/8 个端点需要 `X-API-Key`。在 LogReportSvr dashboard `http://124.220.6.174:8080/dashboard/` 注册一个属于自己的 AppKey，保存到本地：

```bash
mkdir -p ~/.config/logreport && chmod 700 ~/.config/logreport
printf '%s' 'YOUR_KEY_HERE' > ~/.config/logreport/key
chmod 600 ~/.config/logreport/key
```

## 触发

在 Claude Code / Claude App 里说："看日志"、"查 error"、"session xxx 出啥问题"、"最近什么在炸"、"fingerprint xxx 是啥"，对应 skill 自动加载。
