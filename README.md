# Claude Code Skills

本仓库保存自定义的 Claude Code Skills，用于自动化常见任务。

## Skills 列表

| Skill 名称 | 功能 | 触发词 |
|-----------|------|--------|
| mihomo-update-dat | 更新 mihomo 地址库 | "更新地址库"、"更新 geoip" |
| mihomo-update-sub | 更新 mihomo 机场订阅（订阅+主配置同步），仅日本线路，含备份/验证/回滚 | "更新订阅"、"更新机场"、"更新代理配置" |
| codex-troubleshoot | 修复 Codex CLI 错误 | "codex 错误"、"修复 codex" |

## 安装方法

```bash
# 安装单个 skill
npx skills add https://github.com/wenqiaowuli-coder/claude-skills/tree/main/skills/<skill名> --skill -g -y

# 或克隆整个仓库后手动复制
git clone https://github.com/wenqiaowuli-coder/claude-skills.git
cp -r skills/* ~/.claude/skills/
```

## 使用方法

安装后，在 Claude Code 对话中直接使用触发词即可自动调用对应的 Skill。

## 环境信息

- 创建日期：2026-07-12
- Claude Code 版本：最新
- 系统环境：Ubuntu 24.04
