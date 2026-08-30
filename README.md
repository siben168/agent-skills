# Agent Skills

个人通用 AI Agent 技能仓库（支持 Claude Code, Codex, Cursor, Gemini CLI 等）。

基于 [skills.sh](https://skills.sh) 规范构建。

## 包含技能清单

| 技能名称 | 目录 | 描述 |
|---|---|---|
| **cloudflare-pages-deploy** | [`skills/cloudflare-pages-deploy`](skills/cloudflare-pages-deploy/SKILL.md) | 通用 Cloudflare Pages 自动化无头部署、环境变量/Secret 注入与自定义域名/DNS CNAME 自动热切换指南 |

---

## 安装使用

### 1. 全局安装（推荐，所有 AI 工具通用）
```bash
npx skills add siben168/agent-skills -g
```

### 2. 为当前特定项目安装
```bash
npx skills add siben168/agent-skills
```

### 3. 更新所有技能至最新版本
```bash
npx skills update -g
```
