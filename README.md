# LogReportSkill

Claude Code skills that wrap [LogReportSvr](https://github.com/boomlulu/LogReportSvr) — query Unity game session logs as native skill actions.

## 收录的 skills

| skill | 用途 |
|---|---|
| [log-svr](log-svr/) | 查询 Unity 游戏日志的 8 个端点 + 3 条常用排查链路。安装与使用见 [USAGE](https://github.com/boomlulu/LogReportSvr/blob/main/USAGE.md) |

## 安装

一次性 clone + 软链：

```bash
# 1) clone 本仓库到本地稳定路径
git clone https://github.com/boomlulu/LogReportSkill.git ~/.claude/.skill-sources/LogReportSkill

# 2) 每个 skill 用软链到 ~/.claude/skills/
mkdir -p ~/.claude/skills
ln -sfn ~/.claude/.skill-sources/LogReportSkill/log-svr ~/.claude/skills/log-svr
```

## 更新

```bash
cd ~/.claude/.skill-sources/LogReportSkill && git pull
```

软链自动指到最新版本，无需重做。

## 卸载

只卸某一个 skill（保留本地源，不影响其他 skill）：

```bash
rm ~/.claude/skills/log-svr
```

完全卸掉本仓库的所有 skill + 删本地源：

```bash
# 1) 找出指向 .skill-sources/LogReportSkill 的所有软链并删除
find ~/.claude/skills -maxdepth 1 -type l -lname '*/.skill-sources/LogReportSkill/*' -delete

# 2) 删本地源
rm -rf ~/.claude/.skill-sources/LogReportSkill
```

> 不会动 Claude Code 本身的任何配置。下次想装回来重新跑"安装"那节即可。

## 触发

`log-svr` skill 仅在用户说 **"去 LogReport 查日志"** 这条固定短语 + 提供 `device_id` 时才会加载。没说短语 / 没给 device_id 的模糊请求（"看日志"、"查 error"、"崩溃了" 等）不会触发，避免误激活。

示例：

> 去 LogReport 查日志，device_id=abc123def4567 看看最近一次 session

device_id 缺失时，skill 会反问而不是擅自全表扫。
