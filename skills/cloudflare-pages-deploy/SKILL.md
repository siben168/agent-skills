---
name: cloudflare-pages-deploy
description: 通用 Cloudflare Pages 自动化无头部署与自定义域名绑定技能。适用于任何 Web 项目（静态站点、SPA、SSR/全栈 Pages Functions），提供基于 API Token 的非交互式 Wrangler CLI 与 REST API 自动化部署、环境变量注入、域名自动绑定与 DNS 解析、以及零停机蓝绿换代机制。
---

# Cloudflare Pages 通用自动化部署与自定义域名绑定指南

本技能提炼自生产级 Cloudflare Pages 自动化部署实践，旨在为**任何 Web 项目**（如 Vite、React、Vue、Next.js 导出、Astro、纯静态页面或包含 `_worker.js`/`functions` 的全栈应用）提供无头（Headless）、免浏览器交互、自动化配置自定义域名及 DNS 解析的端到端方案。

---

## 1. 适用场景与支持架构

| Web 项目类型 | 构建命令示例 | 发布目录 (Upload Dir) | 说明 |
|---|---|---|---|
| **纯静态 / HTML** | 无 / 简单复制 | `.` 或 `public/` | 直接上传静态文件 |
| **Vite (React/Vue/Svelte)** | `npm run build` | `dist/` | 标准 SPA 应用 |
| **Next.js (Static Export)** | `next build` (`output: 'export'`) | `out/` | 纯静态导出的 Next.js |
| **Astro / Nuxt (SSG)** | `npm run build` | `dist/` 或 `.output/public/` | 静态生成站点 |
| **全栈 Pages (Functions)** | 自定义构建 | 含 `_worker.js` 或 `functions/` 的目录 | 支持服务端边缘计算 |

---

## 2. 账号连接与前置凭据 (Keys & Permissions)

自动化部署**完全无需**手动执行 `wrangler login` 网页登录，仅需配置以下非交互式环境变量：

### 2.1 API Token 最小权限配置
前往 **Cloudflare 控制台 -> My Profile -> API Tokens -> Create Custom Token**，创建具备以下最小权限的 Token：

| 作用域 (Scope) | 资源 (Resource) | 权限 (Permissions) | 必选/可选 | 用途 |
|---|---|---|---|---|
| **Account** | 指定 Account | **Cloudflare Pages: Edit** | **必选** | 创建/更新 Pages 项目、上传部署、挂载自定义域名 |
| **Account** | 指定 Account | **Workers KV Storage: Edit** | 可选 | 若应用使用 KV 存储时必选 |
| **Zone** | 指定域名 Zone | **DNS: Edit** | 自定义域名必选 | 自动在 Cloudflare DNS 中配置 CNAME 记录 |

### 2.2 环境变量定义清单

在部署机或 CI/CD 环境中准备以下环境变量：

```env
# 平台连接凭据 (必填)
CLOUDFLARE_ACCOUNT_ID="<32位十六进制 Account ID>"
CLOUDFLARE_API_TOKEN="<上一步生成的 Custom API Token>"

# 自定义域名与 DNS (若使用自定义域名时必填)
CLOUDFLARE_ZONE_ID="<32位十六进制 Zone ID，位于根域名 Overview 页>"
CUSTOM_DOMAIN="app.yourdomain.com"

# 项目配置
PAGES_PROJECT_NAME="my-web-app"
PRODUCTION_BRANCH="main"
COMPATIBILITY_DATE="2026-01-20"
```

---

## 3. 两种部署模式选择

### 模式 A：标准就地覆盖发布（适合绝大多数通用 Web 项目）
* **特点**：项目名固定，每次发布自动产生新的 deployment 并成为线上版本。
* **适用**：个人博客、企业官网、内部后台、常规 Web 站点。

### 模式 B：零停机蓝绿换代模式（One-Shot 随机发布，汲取自高可用生产实践）
* **特点**：每次发布新建随机项目名（如 `web-a1b2c3d4`），新项目 deployment 验证成功后，再调用 API 将自定义域名从旧项目热迁移到新项目，保留上一代项目用于秒级回退。
* **适用**：核心业务、边缘高可用服务、对部署容灾回滚要求极高的系统。

---

## 4. 自动化部署实操流水线 (AI Agent 执行手册)

### 步骤 1：执行项目构建
检测项目构建工具并生成生产产物：
```bash
# 检测 package.json 中的 build 脚本
npm run build
```

### 步骤 2：通过 Wrangler CLI 无头部署 (Headless Deploy)

设置 Wrangler 静默环境变量并触发部署：

```bash
export CLOUDFLARE_API_TOKEN="<YOUR_API_TOKEN>"
export CLOUDFLARE_ACCOUNT_ID="<YOUR_ACCOUNT_ID>"
export CLOUDFLARE_LOAD_DEV_VARS_FROM_DOT_ENV="false"
export WRANGLER_SEND_METRICS="false"

# 执行非交互式上传
npx wrangler pages deploy <构建产物目录，如 ./dist> \
  --project-name "${PAGES_PROJECT_NAME}" \
  --branch "${PRODUCTION_BRANCH:-main}" \
  --commit-dirty true \
  --commit-message "Automated headless deployment" \
  --force
```
> **参数技巧**：
> - `--force`：跳过交互式确认。
> - `--commit-dirty true`：允许在没有 clean git 状态下构建部署（适合 CI 或动态产物环境）。
> - 如果产物是已经打包好的单文件 `_worker.js`，必须追加 `--no-bundle` 防止 Wrangler 二次编译。

---

### 步骤 3：注入环境变量与加密 Secrets (可选)

若该 Web 项目需要在 Cloudflare Pages 边缘运行时读取环境变量，可通过 Cloudflare REST API v4 进行配置：

```bash
curl -s -X PATCH "https://api.cloudflare.com/client/v4/accounts/${CLOUDFLARE_ACCOUNT_ID}/pages/projects/${PAGES_PROJECT_NAME}" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "deployment_configs": {
      "production": {
        "compatibility_date": "'"${COMPATIBILITY_DATE}"'",
        "env_vars": {
          "API_ENDPOINT": { "type": "plain_text", "value": "https://api.example.com" },
          "AUTH_SECRET": { "type": "secret_text", "value": "your-encrypted-secret-key" }
        }
      }
    }
  }'
```

---

### 步骤 4：自动化配置自定义域名与 DNS 解析

自动实现将 `CUSTOM_DOMAIN` 绑定到 Pages，并配置 Cloudflare DNS 记录。

#### 4.1 将域名关联至 Pages 项目
```bash
curl -s -X POST "https://api.cloudflare.com/client/v4/accounts/${CLOUDFLARE_ACCOUNT_ID}/pages/projects/${PAGES_PROJECT_NAME}/domains" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{ "name": "'"${CUSTOM_DOMAIN}"'" }'
```

#### 4.2 自动更新/创建 Cloudflare DNS CNAME 解析
使自定义域名的 DNS 记录无缝指向 Pages 的默认地址 `<PAGES_PROJECT_NAME>.pages.dev`：

```bash
# 1. 查找是否存在已有的 CNAME 记录
RECORD_ID=$(curl -s -X GET \
  "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records?type=CNAME&name=${CUSTOM_DOMAIN}" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" | jq -r '.result[0].id // empty')

TARGET="${PAGES_PROJECT_NAME}.pages.dev"

# 2. 如果存在则更新，不存在则创建
if [ -n "$RECORD_ID" ]; then
  curl -s -X PATCH "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records/${RECORD_ID}" \
    -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
      "type": "CNAME",
      "name": "'"${CUSTOM_DOMAIN}"'",
      "content": "'"${TARGET}"'",
      "proxied": true,
      "ttl": 1
    }'
else
  curl -s -X POST "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records" \
    -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{
      "type": "CNAME",
      "name": "'"${CUSTOM_DOMAIN}"'",
      "content": "'"${TARGET}"'",
      "proxied": true,
      "ttl": 1
    }'
fi
```

---

## 5. 跨项目开箱即用的自动化部署脚本模版

在任何目标 Web 项目中，可直接创建并运行如下 Node.js 脚本（例如 `deploy-cf.mjs`）：

```javascript
import { spawn } from "node:child_process";

const {
  CLOUDFLARE_ACCOUNT_ID,
  CLOUDFLARE_API_TOKEN,
  CLOUDFLARE_ZONE_ID,
  CUSTOM_DOMAIN,
  PROJECT_NAME = "my-web-app",
  BUILD_DIR = "dist"
} = process.env;

if (!CLOUDFLARE_ACCOUNT_ID || !CLOUDFLARE_API_TOKEN) {
  console.error("Missing CLOUDFLARE_ACCOUNT_ID or CLOUDFLARE_API_TOKEN");
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
    throw new Error(`CF API Error: ${JSON.stringify(data.errors)}`);
  }
  return data.result;
}

async function main() {
  console.log(`[1/3] Deploying ${BUILD_DIR} to Pages project: ${PROJECT_NAME}...`);
  await new Promise((resolve, reject) => {
    const child = spawn("npx", [
      "wrangler", "pages", "deploy", BUILD_DIR,
      "--project-name", PROJECT_NAME,
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
    child.on("exit", code => code === 0 ? resolve() : reject(new Error(`Wrangler exit ${code}`)));
  });

  if (CUSTOM_DOMAIN && CLOUDFLARE_ZONE_ID) {
    console.log(`[2/3] Associating custom domain: ${CUSTOM_DOMAIN}...`);
    try {
      await cfApi(`/accounts/${CLOUDFLARE_ACCOUNT_ID}/pages/projects/${PROJECT_NAME}/domains`, {
        method: "POST",
        body: JSON.stringify({ name: CUSTOM_DOMAIN })
      });
    } catch (e) {
      console.log(`Domain association note: ${e.message}`);
    }

    console.log(`[3/3] Upserting DNS CNAME...`);
    const records = await cfApi(`/zones/${CLOUDFLARE_ZONE_ID}/dns_records?type=CNAME&name=${encodeURIComponent(CUSTOM_DOMAIN)}`);
    const target = `${PROJECT_NAME}.pages.dev`;
    const payload = JSON.stringify({ type: "CNAME", name: CUSTOM_DOMAIN, content: target, proxied: true, ttl: 1 });

    if (records.length > 0) {
      await cfApi(`/zones/${CLOUDFLARE_ZONE_ID}/dns_records/${records[0].id}`, { method: "PATCH", body: payload });
    } else {
      await cfApi(`/zones/${CLOUDFLARE_ZONE_ID}/dns_records`, { method: "POST", body: payload });
    }
    console.log(`Deployment & Custom Domain complete: https://${CUSTOM_DOMAIN}`);
  } else {
    console.log(`Deployment complete: https://${PROJECT_NAME}.pages.dev`);
  }
}

main().catch(err => {
  console.error(err);
  process.exit(1);
});
```

---

## 6. 常见问题排查 (Troubleshooting)

1. **域名冲突 (Domain Conflict)**：
   - *现象*：`Domain is already in use on another Pages project`。
   - *排查*：调用 `DELETE /accounts/{accountId}/pages/projects/{oldProject}/domains/{domain}` 先从旧项目解绑，再绑定到新项目。
2. **CNAME 解析生效与 SSL 证书**：
   - *现象*：页面报 `Error 525 SSL Handshake Failed` 或 `Pending Verification`。
   - *排查*：确保 DNS 记录的 `proxied` 为 `true`（启用橙色小云朵），等待 1-3 分钟让 Cloudflare 边缘节点签发 SSL 证书。
3. **SPA 路由刷新 404 (Single Page Apps)**：
   - *排查*：在构建输出根目录下放置一个 `_routes.json` 或配置重定向规则，让未知路径回退至 `index.html`。
4. **Secret 未生效**：
   - *排查*：通过 API 配置的 `secret_text` 仅在下一次新 deployment 或运行时生效；在静态前端中，客户端变量应在构建期（如 `VITE_APP_*`）烘焙进代码。
