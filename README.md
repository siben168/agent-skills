# Agent Skills

统一维护可跨项目使用的 Agent 技能。先安装到所用工具能够发现的技能目录，再在真实任务中调用；各工具的安装路径和发现机制分别确认。

## 复制给 AI，完成安装（推荐）

**在 Codex 中安装 agile-docs：复制下面整段话发送给 AI 即可。**

```text
使用 skill-installer，从 siben168/agent-skills 安装 skills/agile-docs，固定使用 agile-docs-v0.1.0。
请安装完整技能目录，并确认安装路径、版本及工具是否能识别；如果已有安装，先检查版本和本地修改，避免直接覆盖。
```

适用于具有本地文件操作和网络访问能力的 Codex 环境。安装后可在后续对话调用；如果未出现在技能列表中，重启 Codex 再检查。安装方法见 [Codex 官方说明](https://learn.chatgpt.com/docs/build-skills#install-curated-skills-for-local-use)。

**在 Claude Code、Cursor 或其他支持技能的工具中：** `skill-installer` 是 Codex 的安装技能，不要求其他工具具有同名能力。复制以下通用文案：

```text
请为当前 AI 工具安装 https://github.com/siben168/agent-skills 中的 skills/agile-docs，固定到标签 agile-docs-v0.1.0。
先确认当前工具支持的技能安装方式与目录，再安装完整目录（包括 SKILL.md、references 和 assets）。
完成后报告安装位置、版本及识别结果；已有安装时先检查本地修改。
如果当前环境无法安装，请说明缺少的能力并提供该工具可执行的安装步骤。
```

同一本地环境中的用户级安装可跨项目复用；项目级安装只在相应范围生效。换电脑、远程环境或工具后，应重新确认安装。只克隆源码仓库不代表工具已加载技能。喜欢终端操作的用户也可使用 [skills CLI](https://skills.sh/docs/cli)，按其当前说明选择技能、工具及安装范围。

## 技能目录与发布版本

| 技能 | 用途 | 入口 |
|---|---|---|
| **agile-docs** | 敏捷研发文档协作：PRD、设计、任务台账、文档一致性与归档 | [技能](skills/agile-docs/SKILL.md) · [v0.1.0 发布版](https://github.com/siben168/agent-skills/releases/tag/agile-docs-v0.1.0) |
| **cloudflare-pages-deploy** | Cloudflare Pages 部署、自定义域名与 DNS 配置 | [技能及前置条件](skills/cloudflare-pages-deploy/SKILL.md) |

安装其他技能时替换仓库内路径，并选择该技能实际存在的版本或提交；`agile-docs-v0.1.0` 是 agile-docs 的发布标签。

## 日常使用 agile-docs

安装后，在项目中直接发送具体任务。以下文案可按实际范围替换：

```text
使用 agile-docs，整理这个需求的 PRD，检查验收条件是否清晰；有必要时再补技术设计。
```

```text
使用 agile-docs，接续这份任务台账，确认当前可执行的任务并继续完成。
```

```text
使用 agile-docs，检查指定目录的文档，处理重复内容、过期说明和失效引用，保持范围不扩散。
```

首次接入其他项目：

```text
使用 agile-docs，先识别本项目已有的文档约定和权威入口，再整理指定需求。
沿用有效约定，不批量迁移整个项目；项目特有的目录、编号和流程留在本项目。
```

在 Codex CLI / IDE 中也可用 `$agile-docs` 明确调用，或通过 `/skills` 查看技能。自然语言调用与自动匹配机制见 [官方说明](https://learn.chatgpt.com/docs/build-skills#how-chatgpt-and-codex-use-skills)。

## 不点名技能，也能主动参考吗？

可以。支持自动选择的工具会根据技能的名称与描述匹配任务。agile-docs 的描述已覆盖需求、架构与实现设计、任务台账、文档一致性和归档，无需每次点名。自动匹配取决于工具是否发现并启用技能，以及具体上下文，不能视为每次必定触发。

**希望项目内稳定采用时，将以下约定合并进现有 `AGENTS.md` 的文档协作部分；其他工具放到它实际读取的项目规则文件中：**

```markdown
编写、修改或整理本项目研发文档（PRD、架构与实现设计、Guide / Runbook、任务台账及文档索引）时，主动使用已安装的 agile-docs，并先读取其 SKILL.md，再按当前任务加载必要参考。
沿用本项目有效约定，按改动规模裁剪流程；简单文案修改不启动完整需求设计，不因调用技能就新增文档、台账或检查报告。
如果技能不可用，明确说明，继续遵循本项目已有文档规则，不宣称已调用技能。
```

也可以请 AI 帮忙配置：

```text
请确认当前工具能识别 agile-docs，再读取 https://github.com/siben168/agent-skills 的 README，
把其中“主动参考”的文档协作约定合并进当前项目已有的 Agent 规则。
只补必要的调用入口，避免重复规则或复制整份 skill；报告修改位置。
```

项目规则负责提示何时使用，技能负责通用方法，项目文档约定负责本地细节。添加这条约定不会自动安装技能，也不等于自动检查所有历史文档。

## 更新已安装版本

**源码发布新版本，不会自动更新通过上述方式安装的固定版本副本。** 在需要升级的环境中复制：

```text
请检查 siben168/agent-skills 中 agile-docs 的最新正式发布版本，与本机已安装版本比较。
确认规则差异及对当前项目约定的影响后，升级完整技能目录并验证工具能识别。
保留已有本地修改；若与新版冲突，先说明冲突。报告升级前后版本和验证结果。
```

使用原安装工具或方式完成升级；需要可复现时指定发布标签或提交，不直接跟随可变的 main。GitHub 源码仓库、已安装副本和项目文档是不同的维护位置，更新其中一个不会自动同步另外两个。

## 发现优化点，如何回流？

在正在应用技能的项目中，直接告诉 AI：

```text
这里使用 agile-docs 出现了这个问题：[填写现象、预期和最小案例]。
先判断属于项目特有约定、执行偏差还是 skill 的通用缺陷。
如果是通用缺陷，在 siben168/agent-skills 的源码中提出最小修订，并用真实任务验证。
优先澄清、替换或合并已有规则，避免扩充成大而全的技能；给出修改与验证结果。
```

- 项目约定与原始证据留在当前项目，优先使用已有文档、台账或 Issue/PR；不为每次使用另写报告。
- 通用改进回到本技能仓库，携带最小的通用化案例；先判断现有规则是否已覆盖，避免把项目局部经验升级为全局要求。
- 在源码中修订并验证，通过 PR 评审后发布新版本，再更新需要采用新版的安装环境。仅在本机安装副本修改不会回流 GitHub。

详细判断门槛与验证方式以 [演进与回流规则](skills/agile-docs/references/evolution.md) 为准。维护者修改 README 等安装说明时，无需因此改动运行时规则或重发技能版本。

## 包内容与验证范围

agile-docs 只分发 `skills/agile-docs/`：16 个运行时文件，入口 122 行。`references/` 提供按需规则与案例，`assets/` 提供模板；仓库 `docs/` 是维护资料，不安装为运行时必读材料。不要只复制入口文件。

v0.1.0 已完成来源项目全量文档治理及 Codex 本地显式调用回归，运行时与已验证候选 `agile-docs-v0.1.0-rc.1` 一致。自动触发、跨项目、跨工具及长期维护收益尚未验证；上面的自动选择说明描述工具机制，不代表这些场景已完成验证。

维护依据：[设计共识](docs/agile-docs-design.md) · [场景试用与限制](docs/agile-docs/scenario-trials.md)。

## 使用 cloudflare-pages-deploy

安装 `skills/cloudflare-pages-deploy` 后，可以发送：

```text
使用 cloudflare-pages-deploy，帮我把当前前端项目部署到 Cloudflare Pages，并绑定域名 my-app.example.com。
先按项目约定确认构建命令、产物和部署目标。
```

该技能需要 Cloudflare 凭据；agile-docs 不需要。部署前按 [技能的凭据说明](skills/cloudflare-pages-deploy/SKILL.md) 配置 `CLOUDFLARE_ACCOUNT_ID`、`CLOUDFLARE_API_TOKEN`，使用自定义域名时另需 `CLOUDFLARE_ZONE_ID`。凭据保存在受保护的环境变量或已忽略的本地配置中，不粘贴进对话或提交仓库；具体部署步骤由技能维护，不在 README 重复一份。
