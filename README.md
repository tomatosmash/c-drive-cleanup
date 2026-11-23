# c-drive-cleanup

一个用于安全清理 Windows C 盘空间的 Claude Code skill。

## 功能

先只读探查定位 C 盘占用大头 → 优先清用户级零风险缓存 → 大目录(WSL / VM 沙箱)迁 D 盘不删 → 系统级清理写成脚本让用户管理员运行 → 每个不可逆操作先确认。

## 安装

把本仓库的 `SKILL.md` 复制到 Claude 的 skills 目录:

- **项目级**(仅当前项目生效):`<项目>/.claude/skills/c-drive-cleanup/SKILL.md`
- **用户级**(所有项目生效):`~/.claude/skills/c-drive-cleanup/SKILL.md`

即:

```bash
mkdir -p ~/.claude/skills/c-drive-cleanup
cp SKILL.md ~/.claude/skills/c-drive-cleanup/
```

## 使用

在 Claude Code 里说「C 盘满了」或「清理 C 盘」即可触发,或显式调用 `/c-drive-cleanup`。

## 注意

- 删除类操作务必先确认,尤其是游戏存档、应用数据、`Downloads` 个人文件。
- auto 权限模式下,应用数据类缓存删除可能被拦,skill 会写成脚本让你手动运行。
