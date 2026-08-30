---
name: cloudflare-pages-deploy
description: 通用 Cloudflare Pages 自动化无头部署与自定义域名绑定技能。适用于任何 Web 项目（静态站点、SPA、SSR/全栈 Pages Functions），提供基于 API Token 的非交互式 Wrangler CLI 与 REST API 自动化部署、环境变量注入、域名自动绑定与 DNS 解析、以及零停机蓝绿换代机制。
---

# Cloudflare Pages 通用自动化部署与自定义域名绑定指南

> **适用 Agent**: Claude Code, Codex, Cursor, Gemini CLI, Windsurf, Cline 等任何支持任务执行的 AI Coding Agent。  
> **核心目标**: 帮助 AI Agent 理解并全自动执行任意 Web 项目向 Cloudflare Pages 的构建、无头上传、环境变量注入及自定义域名 CNAME 热切换。

---

## 🤖 AI Agent 快速启动与执行法则 (AI Agent Guidelines)

当用户提出类似以下需求时，AI 应当**立即激活本技能**：
- *"帮我把这个项目部署到 Cloudflare Pages"*
- *"配置 Cloudflare 自动部署并绑定我的域名"*
- *"写一个脚本把前端发布到 Cloudflare"*

### AI 执行的 5 项核心铁律：
1. **不要请求用户网页登录**：严禁指导用户运行需要浏览器交互的 `wrangler login`，一律使用 `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID` 的无头认证模式。
2. **先探测产物目录再部署**：先阅读 `package.json` 中的 `scripts.build`，执行构建后确认输出目录（如 `dist/`、`out/`、`build/`）存在，切勿盲目上传根目录。
3. **保护敏感信息**：通过 REST API 写入环境变量时，密码、Token、UUID 等必须声明为 `"type": "secret_text"`，绝不能使用明文存储。
4. **单文件 Worker 必须加 `--no-bundle`**：如果上传的产物本身已经包含打包好的 `_worker.js`，调用 Wrangler 时必须带上 `--no-bundle`，防止 Wrangler 二次编译损坏代码。
5. **域名绑定两步走**：将自定义域名绑定到 Pages 后，必须同时检查并更新 Cloudflare DNS 的 CNAME 解析，指向 `<project-name>.pages.dev`，并确保 `proxied: true`。

---

## 1. 适用场景与项目框架对照

AI 在执行部署前，应根据项目类型确定构建命令与上传路径：

| 项目类型 | 典型框架 | 构建命令 | 生产发布目录 (Upload Dir) | 特殊注意事项 |
|---|---|---|---|---|
| **标准 SPA** | Vite (React/Vue/Svelte) | `npm run build` | `dist/` | 需考虑前端路由刷新（配置 `_routes.json`） |
| **纯静态站** | HTML/CSS/JS, Hugo | 无 / `hugo` | `.` 或 `public/` | 检查是否有 `index.html` 入口 |
| **静态导出** | Next.js (`output: 'export'`) | `npx next build` | `out/` | 确保 Next 配置了静态导出模式 |
| **现代 SSG** | Astro, Nuxt | `npm run build` | `dist/` 或 `.output/public/` | 按框架官方规范确认 output 路径 |
| **全栈边缘计算** | Pages Functions / Worker | 自定义打包 | 包含 `_worker.js` 的目录 | 上传命令必须加 `--no-bundle` |

---

## 2. 账号连接与前置凭据 (Keys & Permissions)

AI 执行自动化部署所需的环境变量规范：

### 2.1 引导用户准备 API Token
若用户尚未创建 Token，AI 需指导用户进入 **Cloudflare 控制台 -> My Profile -> API Tokens -> Create Custom Token**，配置以下权限：
- **Account** -> **Cloudflare Pages: Edit**（必选，用于创建/配置项目与绑定域名）
- **Account** -> **Workers KV Storage: Edit**（可选，仅当应用使用 KV 时需要）
- **Zone** -> **DNS: Edit**（自定义域名必选，用于自动配置 CNAME 解析）

### 2.2 部署环境变量清单 (`.env`)
```env
# 平台连接凭据 (必填)
CLOUDFLARE_ACCOUNT_ID="<32位十六进制 Account ID>"
CLOUDFLARE_API_TOKEN="<具备上述权限的 Custom API Token>"

# 自定义域名与 DNS (使用自定义域名时必填)
CLOUDFLARE_ZONE_ID="<32位十六进制 Zone ID，位于域名 Overview 页>"
CUSTOM_DOMAIN="app.yourdomain.com"

# 项目配置 (可选，有默认值)
PAGES_PROJECT_NAME="my-web-app"
PRODUCTION_BRANCH="main"
COMPATIBILITY_DATE="2026-01-20"
```

---

## 3. 标准部署操作流程 (SOP)

AI 请按以下 4 个步骤依次自动化执行：

### 步骤 1：本地构建与产物校验
```bash
# 检查依赖并构建
npm run build

# 验证产物是否存在 (以 dist 为例)
ls -la dist/index.html
```

### 步骤 2：通过 Wrangler CLI 无头静默上传
在当前环境中导出静默变量并执行 CLI：
```bash
export CLOUDFLARE_API_TOKEN="<API_TOKEN>"
export CLOUDFLARE_ACCOUNT_ID="<ACCOUNT_ID>"
export CLOUDFLARE_LOAD_DEV_VARS_FROM_DOT_ENV="false"
export WRANGLER_SEND_METRICS="false"

npx wrangler pages deploy ./dist \
  --project-name "${PAGES_PROJECT_NAME}" \
  --branch "${PRODUCTION_BRANCH:-main}" \
  --commit-dirty true \
  --commit-message "Automated headless deployment" \
  --force
```

### 步骤 3：注入环境变量（仅全栈/带配置应用需要）
通过 Cloudflare REST API v4 为项目注入变量：
```bash
curl -s -X PATCH "https://api.cloudflare.com/client/v4/accounts/${CLOUDFLARE_ACCOUNT_ID}/pages/projects/${PAGES_PROJECT_NAME}" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "deployment_configs": {
      "production": {
        "compatibility_date": "'"${COMPATIBILITY_DATE}"'",
        "env_vars": {
          "ENV_VAR_KEY": { "type": "plain_text", "value": "public_val" },
          "SECRET_KEY": { "type": "secret_text", "value": "private_secret" }
        }
      }
    }
  }'
```

### 步骤 4：自动关联自定义域名并更新 DNS CNAME 解析
使用户通过 `https://<CUSTOM_DOMAIN>` 即可访问：

```bash
# 4.1 关联域名至 Pages 项目
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/${CLOUDFLARE_ACCOUNT_ID}/pages/projects/${PAGES_PROJECT_NAME}/domains" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{ "name": "'"${CUSTOM_DOMAIN}"'" }'

# 4.2 检查现有 DNS 记录并 Upsert CNAME
RECORD_ID=$(curl -s -X GET \
  "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records?type=CNAME&name=${CUSTOM_DOMAIN}" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" | jq -r '.result[0].id // empty')

TARGET="${PAGES_PROJECT_NAME}.pages.dev"

if [ -n "$RECORD_ID" ]; then
  # 更新现有解析
  curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records/${RECORD_ID}" \
    -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{"type":"CNAME","name":"'"${CUSTOM_DOMAIN}"'","content":"'"${TARGET}"'","proxied":true,"ttl":1}'
else
  # 新建解析
  curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records" \
    -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{"type":"CNAME","name":"'"${CUSTOM_DOMAIN}"'","content":"'"${TARGET}"'","proxied":true,"ttl":1}'
fi
```

---

## 4. 开箱即用的自动化部署脚本 (`deploy-cf.mjs`)

为避免在终端分段拼接 cURL 命令，AI 可直接在目标项目根目录下生成并执行如下标准 Node.js 脚本：

```javascript
#!/usr/bin/env node
import { spawn } from "node:child_process";

const {
  CLOUDFLARE_ACCOUNT_ID,
  CLOUDFLARE_API_TOKEN,
  CLOUDFLARE_ZONE_ID,
  CUSTOM_DOMAIN,
  PAGES_PROJECT_NAME = "my-web-app",
  BUILD_DIR = "dist"
} = process.env;

if (!CLOUDFLARE_ACCOUNT_ID || !CLOUDFLARE_API_TOKEN) {
  console.error("❌ 缺少环境变量: 请先配置 CLOUDFLARE_ACCOUNT_ID 与 CLOUDFLARE_API_TOKEN");
  process.exit(1);
}

async function cfApi(path, options = {}) {
  const res = await fetch(`https://api.cloudflare.com/client/v4${path}`, {
    ...options,
    headers: {
      "Authorization": `Bearer ${CLOUDFLARE_API_TOKEN}`,
      "Content-Type": "application/json",
      ...options.headers,
    },
  });
  const data = await res.json();
  if (!res.ok || data.success === false) {
    throw new Error(`Cloudflare API 异常: ${JSON.stringify(data.errors)}`);
  }
  return data.result;
}

async function main() {
  console.log(`🚀 [1/3] 上传产物目录 [${BUILD_DIR}] 到 Pages 项目 [${PAGES_PROJECT_NAME}]...`);
  await new Promise((resolve, reject) => {
    const child = spawn("npx", [
      "wrangler", "pages", "deploy", BUILD_DIR,
      "--project-name", PAGES_PROJECT_NAME,
      "--branch", "main",
      "--commit-dirty", "true",
      "--force"
    ], {
      stdio: "inherit",
      env: {
        ...process.env,
        CLOUDFLARE_API_TOKEN,
        CLOUDFLARE_ACCOUNT_ID,
        WRANGLER_SEND_METRICS: "false"
      }
    });
    child.on("exit", code => code === 0 ? resolve() : reject(new Error(`Wrangler 退出码: ${code}`)));
  });

  if (CUSTOM_DOMAIN && CLOUDFLARE_ZONE_ID) {
    console.log(`🔗 [2/3] 正在关联自定义域名: ${CUSTOM_DOMAIN}...`);
    try {
      await cfApi(`/accounts/${CLOUDFLARE_ACCOUNT_ID}/pages/projects/${PAGES_PROJECT_NAME}/domains`, {
        method: "POST",
        body: JSON.stringify({ name: CUSTOM_DOMAIN })
      });
    } catch (e) {
      console.log(`ℹ️ 域名关联提示 (可能已存在): ${e.message}`);
    }

    console.log(`🌐 [3/3] 正在更新 Cloudflare DNS CNAME 解析...`);
    const records = await cfApi(`/zones/${CLOUDFLARE_ZONE_ID}/dns_records?type=CNAME&name=${encodeURIComponent(CUSTOM_DOMAIN)}`);
    const target = `${PAGES_PROJECT_NAME}.pages.dev`;
    const payload = JSON.stringify({ type: "CNAME", name: CUSTOM_DOMAIN, content: target, proxied: true, ttl: 1 });

    if (records.length > 0) {
      await cfApi(`/zones/${CLOUDFLARE_ZONE_ID}/dns_records/${records[0].id}`, { method: "PATCH", body: payload });
    } else {
      await cfApi(`/zones/${CLOUDFLARE_ZONE_ID}/dns_records`, { method: "POST", body: payload });
    }
    console.log(`\n🎉 部署与自定义域名配置成功: https://${CUSTOM_DOMAIN}`);
  } else {
    console.log(`\n🎉 部署成功: https://${PAGES_PROJECT_NAME}.pages.dev`);
  }
}

main().catch(err => {
  console.error("❌ 部署流程失败:", err);
  process.exit(1);
});
```

---

## 5. 故障排查与自愈指引 (Troubleshooting)

当遇到报错时，AI 应当自主按照以下方案进行排查和修复：

| 错误信息 / 异常表现 | 根本原因 | AI 自愈动作 |
|---|---|---|
| `Domain is already in use on another Pages project` | 域名挂载在旧的 Pages 项目上 | 调用 `DELETE /accounts/{accountId}/pages/projects/{oldProject}/domains/{domain}` 强制解绑旧项目，再重新绑定 |
| `HTTP 525 SSL Handshake Failed` | DNS 刚刚创建，边缘 SSL 证书正在签发中 | 告知用户这是正常边缘签发延迟，确认 DNS 开启了 `proxied: true`（橙色小云朵），等待 1-3 分钟即可恢复 |
| SPA 前端刷新页面报 404 | 边缘 CDN 找不到对应的静态 HTML 文件 | 在构建输出目录中写入 `_routes.json` 或 `_redirects` 规则：`/*  /index.html  200` |
| `Authentication error (code: 10000)` | API Token 无效或过期 | 检查请求头 `Authorization: Bearer <TOKEN>` 是否包含多余空格，确认 Token 未被撤销 |
| `Project not found` | 项目尚未创建 | Wrangler 会自动创建项目，若报错则先调用 `POST /accounts/{accountId}/pages/projects` 创建空项目 |
