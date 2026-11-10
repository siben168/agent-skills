# Agent Skills 🚀

通用个人 AI Agent 技能仓库（遵循 [skills.sh](https://skills.sh) 规范）。  
专为 **Claude Code**, **Codex**, **Cursor**, **Gemini CLI**, **Windsurf** 等全生态 AI Coding Agent 设计，支持一次安装、全平台复用。

---

## 📦 技能目录

| 技能名称 | 目录链接 | 简介 | 核心适用场景 |
|---|---|---|---|
| **cloudflare-pages-deploy** | [查看技能文档](skills/cloudflare-pages-deploy/SKILL.md) | Cloudflare Pages 通用自动化无头部署、环境变量/Secret 注入与自定义域名/DNS CNAME 自动热切换指南 | 任何 Web 前端、全栈应用快速上线 Cloudflare 并绑定自定义域名 |
| **agile-docs**（初稿） | [查看技能文档](skills/agile-docs/SKILL.md) | 轻量敏捷文档协作：需求与设计分工、任务台账、事实单一来源与持续演进 | 项目文档建设、需求推进、跨会话交接与文档治理 |

`agile-docs` 的基础规则与模板讨论已完成，PRD、ADR、GD、任务台账及研究归档规则已纳入技能包；维护进度见 [设计共识与模板讨论计划](docs/agile-docs-design.md)。尚未完成场景试用及跨项目、跨工具行为验证。

---

## ⚡ 快速安装与使用

### 方式 1：使用 `npx skills` 全局安装（推荐，所有 AI 工具同时生效）
在终端中执行以下命令，工具会自动为本机已安装的 Claude Code、Codex、Cursor、Gemini CLI 配置软链接：

```bash
# 全局安装本仓库的所有技能
npx skills add siben168/agent-skills -g

# 或者只安装指定的部署技能
npx skills add siben168/agent-skills --skill cloudflare-pages-deploy -g
```

### 方式 2：为特定单项目安装
进入目标项目目录后执行：
```bash
npx skills add siben168/agent-skills
```

### 方式 3：更新技能至最新版本
后续仓库更新后，只需在终端运行：
```bash
npx skills update -g
```

---

## 🔑 凭据准备（直接放项目根目录的 `.env`）

在待部署的 Web 项目根目录下创建 `.env` 文件（请确保加入 `.gitignore`），AI 会**自动扫描并静默读取**，无需在对话中透露敏感信息：

```env
# 平台连接凭据 (必填)
CLOUDFLARE_ACCOUNT_ID=0123456789abcdef0123456789abcdef
CLOUDFLARE_API_TOKEN=cf_pat_xxxxxxxxxxxxxxxxxxxxxxxxx

# 自定义域名与 DNS (使用自定义域名时必填)
CLOUDFLARE_ZONE_ID=0123456789abcdef0123456789abcdef
```

> 💡 **进阶技巧**：如果多个项目使用同一个账号，也可以直接将上述变量写进系统 `~/.bashrc`，在任何新项目中 AI 都能直接免配置读取。

---

## 💡 如何在对话中调用？（直接对 AI 说）

安装后，你在任何 Web 项目中唤起 AI（无论是 Claude Code 终端、Codex 会话、Cursor 聊天框还是 Antigravity/Gemini），直接发送如下自然语言指令即可触发：

> **“使用 `cloudflare-pages-deploy` 技能，帮我把当前前端项目自动构建并部署到 Cloudflare Pages，并绑定域名 `my-app.example.com`。”**

AI 将自动执行：
1. 探测项目构建产物（`npm run build` 生成 `dist/` 或 `out/`）；
2. **自动扫描项目根目录的 `.env` 或系统环境变量**，提取 `CLOUDFLARE_API_TOKEN` 和 `CLOUDFLARE_ACCOUNT_ID` 进行无头部署；
3. 调用 Cloudflare REST API 挂载自定义域名；
4. 自动在 Cloudflare DNS 中增量更新 CNAME 解析并开启 CDN 代理加速。

---

## 📋 免安装直接使用（发给任意 Web 版 AI 的提示词模版）

如果你使用的是网页版 ChatGPT、Claude.ai、DeepSeek 等未接入 CLI 的工具，直接复制以下提示词发给它：

```markdown
你现在是精通 Cloudflare 生态的 DevOps 部署专家。请严格遵循以下自动化规范指导我部署当前 Web 项目：
1. 认证方式：严禁要求我运行需要网页交互的 `wrangler login`，必须统一使用环境变量 `CLOUDFLARE_ACCOUNT_ID` 和 `CLOUDFLARE_API_TOKEN`（需具备 Pages:Edit 和 DNS:Edit 权限）。
2. 构建与部署：先执行 `npm run build`，再通过静默参数 `npx wrangler pages deploy <构建产物目录> --project-name <项目名> --branch main --commit-dirty true --force` 上传。
3. 域名与DNS切换：调用 Cloudflare REST API v4：
   - POST /accounts/{accountId}/pages/projects/{projectName}/domains 绑定自定义域名；
   - PATCH/POST /zones/{zoneId}/dns_records 自动写入或更新对应的 CNAME 解析，指向 `<projectName>.pages.dev`，并设置 proxied: true。
请基于以上规则，检查我的项目结构并提供可执行的完整部署脚本或步骤。
```
