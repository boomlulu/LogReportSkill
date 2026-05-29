# 发布 skill 到 LogReportSkill

本仓库是 boomlulu 团队的共享 Claude Code skill 分发仓。任何新 skill 走以下流程发到这里，下游同事 `git pull` + 软链就装上了。

## 仓库约定

```
LogReportSkill/
├── README.md             # 终端用户安装入口
├── PUBLISHING.md         # 本文档
└── <skill-name>/         # 每个 skill 一个顶级文件夹
    └── SKILL.md          # 必备；frontmatter + 正文
```

- 文件夹名 = SKILL.md frontmatter 的 `name` 字段，必须一致
- 一个 skill 只做一件事，复杂能力拆多个 skill
- 文件夹里可以放 SKILL.md 之外的辅助文件（templates、evals 等），但 SKILL.md 是入口

## SKILL.md frontmatter

```yaml
---
name: <kebab-case-skill-name>
description: >
  <精准的触发描述。明确写"用户说 X 时触发"，"用户不说 X 时不触发"。
  必须的入参列为硬性入参，没给就先反问，绝不擅自补全。>
allowed-tools: ["Bash", "Read", ...]
---
```

字段说明：

- **name**：kebab-case，全局唯一，作为 `~/.claude/skills/<name>` 路径
- **description**：YAML `>` 块标量，agent 的 system prompt 里就是按这段判断要不要加载 skill。**这是 skill 命中率的核心**
- **allowed-tools**：白名单数组，列出 skill 正文里允许调用的工具名

## 写好 description（最重要）

**反面教材**：列一堆模糊触发词

> 用户说"看日志"、"查 error"、"崩溃了"、"有没有报错"、"出问题了" 时触发

→ 几乎任何对话都会误触发。

**正面写法**：固定短语 + 硬性入参 + 拒绝兜底

> 仅在用户明确说"去 LogReport 查日志"且同时提供 device_id 时才触发。任何其他说法一律不触发。device_id 缺失时必须先反问，绝不擅自全表扫。

可参照 `log-svr/SKILL.md` 的 description 作为模板。

## 发布新 skill 流程

### 1. clone 仓库到稳定路径（首次）

```bash
git clone git@github.com-boomlulu:boomlulu/LogReportSkill.git ~/dev/LogReportSkill
cd ~/dev/LogReportSkill
```

> 已 clone 过就 `cd ~/dev/LogReportSkill && git pull`。

### 2. 建 skill 文件夹 + SKILL.md

```bash
mkdir my-new-skill
$EDITOR my-new-skill/SKILL.md
```

frontmatter 按上面模板填，正文建议包含：

- 能力一句话总结
- 入口判断（什么情况进入 / 什么情况拒绝）
- 工具或命令列表（每个带 curl / Bash / 等可复制范例）
- 常用链路（多步排查的固定剧本）
- 故障排除（已知错误码 → 修复指引）

### 3. 更新顶层 README.md 收录表

在 "## 收录的 skills" 表里加一行：

```markdown
| [my-new-skill](my-new-skill/) | 一句话用途说明 |
```

### 4. 本地装上自测

```bash
ln -sfn ~/dev/LogReportSkill/my-new-skill ~/.claude/skills/my-new-skill
```

打开新 Claude Code 会话，说触发短语：

- skill 出现在 system reminder 的 available-skills 列表 → description 触发口径生效
- 用户说触发短语 → skill 被加载，按 SKILL.md 正文行事

改 SKILL.md 后 symlink 自动跟，无需重装。

### 5. commit + push

```bash
git add my-new-skill/ README.md
git -c user.name='boomlulu' -c user.email='boom.chatgpt.plus.01@gmail.com' commit -m "feat(<skill-name>): <一句话描述>"
git push origin main
```

### 6. 通知同事

同事一次性安装（首次）：

```bash
git clone git@github.com-boomlulu:boomlulu/LogReportSkill.git ~/.claude/.skill-sources/LogReportSkill
ln -sfn ~/.claude/.skill-sources/LogReportSkill/my-new-skill ~/.claude/skills/my-new-skill
```

或已装过本仓库的同事，pull 后只软链新 skill：

```bash
cd ~/.claude/.skill-sources/LogReportSkill && git pull
ln -sfn ~/.claude/.skill-sources/LogReportSkill/my-new-skill ~/.claude/skills/my-new-skill
```

## 修改已有 skill

直接改 `<skill>/SKILL.md`，commit + push。下游 `git pull` 拿到。无需重装软链。

## 删除 skill

1. `git rm -r <skill-name>/`
2. 顶层 README "收录的 skills" 表删对应行
3. commit + push
4. 通知同事卸载：`rm ~/.claude/skills/<skill-name>`

## 调试 description 触发命中率

- description 写完后，开新 Claude Code 会话，看 system-reminder 里 skill 列表是否出现 → 出现说明加载成功
- 在 description 里能触发的短语 + 不能触发的短语都列几个例子，逼 agent 不要乱触发
- 触发命中率低 → 把固定短语写得更突出（用 **加粗** 或引号）
- 误触发率高 → 加"任何其他说法一律不触发"这种硬性兜底句

可选：用 `skill-creator` skill 跑评测，量化命中率。

## 风格约定

- 中文写正文，命令 / 参数 / 字段名用英文
- frontmatter `description` 用 `>` 块标量，第 2 行起前导 2 空格（YAML）
- 代码块标语言（`bash` / `yaml` / `json` 等），方便渲染高亮

## 安全约定

- **不要**把 token / 密钥 / 密码硬编码到 SKILL.md
- 鉴权改成读环境变量 或 本地文件（`~/.config/<svc>/token`）
- 公开仓库 README **不要**贴真实 host / IP / credentials；用 placeholder

## 维护责任

- 每个 skill 的 SKILL.md 顶部留作者署名（comment 形式）
- 大改动（description 收紧、API 变更）写 commit message 解释 why
- 半年没人维护的 skill 在 README 表里标 `[deprecated]`
