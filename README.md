# Agent Skills

统一维护可跨项目使用的 Agent 技能。选择需要的技能，把安装文案复制给 AI；安装、调用、升级和改进都由本仓库提供统一入口。

## 选择技能

| 技能 | 适用任务 | 安装版本 | 使用说明 |
|---|---|---|---|
| **agile-docs** | PRD、架构与实现设计、任务台账、文档一致性与归档 | [agile-docs-v0.1.1](https://github.com/siben168/agent-skills/releases/tag/agile-docs-v0.1.1) | [日常文档协作](#agile-docs) · [技能源码](skills/agile-docs/SKILL.md) |
| **cloudflare-pages-deploy** | Cloudflare Pages 部署、自定义域名与 DNS 配置 | 尚无独立发布版，安装时固定 main 对应的具体提交 | [部署使用](#cloudflare-pages-deploy) · [技能源码](skills/cloudflare-pages-deploy/SKILL.md) |

## 复制给 AI，完成安装（推荐）

### 安装全部技能

```text
请从 https://github.com/siben168/agent-skills 为当前 AI 工具安装以下两个技能：
- skills/agile-docs：固定使用标签 agile-docs-v0.1.1。
- skills/cloudflare-pages-deploy：解析 main 当前的具体提交 SHA，并固定到该提交。
如果是 Codex，使用 skill-installer；其他工具采用其支持的技能安装方式。
请安装各自的完整目录，包括其中的参考和模板；已有安装时先检查版本及本地修改，避免直接覆盖。
完成后分别报告安装路径、标签或提交，以及工具是否能识别。安装技能本身不执行部署。
```

### 只安装 agile-docs

```text
请从 siben168/agent-skills 安装 skills/agile-docs，固定使用 agile-docs-v0.1.1。
如果是 Codex，使用 skill-installer；其他工具采用其支持的技能安装方式。
安装完整技能目录；已有安装时先检查版本及本地修改。完成后报告安装位置、版本及识别结果。
```

### 只安装 cloudflare-pages-deploy

```text
请从 siben168/agent-skills 安装 skills/cloudflare-pages-deploy，解析 main 当前的具体提交 SHA，并固定到该提交。
如果是 Codex，使用 skill-installer；其他工具采用其支持的技能安装方式。
安装完整技能目录；已有安装时先检查版本及本地修改。完成后报告安装位置、提交及识别结果，不执行部署。
```

这些文案适用于具有文件操作和网络访问能力、支持技能的 AI 工具。`skill-installer` 是 Codex 的安装技能，其他工具不要求具有同名能力；无法安装时，应说明缺少的能力并提供相应步骤。

同一本地环境中的用户级安装可跨项目复用，项目级安装只在相应范围生效。换电脑、远程环境或工具后需重新确认。只克隆仓库不代表工具已加载技能；Codex 安装后可在后续对话调用，未识别时重启再检查。参考 [Codex 安装说明](https://learn.chatgpt.com/docs/build-skills#install-curated-skills-for-local-use)；偏好终端操作的用户也可使用 [skills CLI](https://skills.sh/docs/cli)。

## 使用技能

### agile-docs

按当前任务选择一条发送，并补充需求、台账路径或目录范围：

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

### cloudflare-pages-deploy

```text
使用 cloudflare-pages-deploy，帮我把当前前端项目部署到 Cloudflare Pages，并绑定域名 my-app.example.com。
先按项目约定确认构建命令、产物和部署目标。
```

部署需要按 [技能的凭据说明](skills/cloudflare-pages-deploy/SKILL.md) 配置 `CLOUDFLARE_ACCOUNT_ID`、`CLOUDFLARE_API_TOKEN`，使用自定义域名时另需 `CLOUDFLARE_ZONE_ID`。凭据保存在受保护的环境变量或已忽略的本地配置中，不粘贴进对话或提交仓库。agile-docs 不需要这些凭据。具体部署步骤由技能维护，不在 README 重复一份。

## 不点名技能，也能主动参考吗？

支持自动选择的工具会根据技能名称与描述匹配任务：研发文档协作匹配 agile-docs，Cloudflare Pages 部署任务匹配 cloudflare-pages-deploy。是否触发取决于工具是否发现并启用技能及具体上下文，不能视为每次必定触发；选择部署技能也不意味着无关任务获准自动部署。

在 Codex CLI / IDE 中，可用 `$agile-docs` 或 `$cloudflare-pages-deploy` 明确调用，通过 `/skills` 查看技能。机制见 [官方说明](https://learn.chatgpt.com/docs/build-skills#how-chatgpt-and-codex-use-skills)。

### 让项目文档工作主动参考 agile-docs

将以下约定合并进现有 `AGENTS.md` 的文档协作部分；其他工具放到它实际读取的项目规则文件中：

```markdown
编写、修改或整理本项目研发文档（PRD、架构与实现设计、Guide / Runbook、任务台账及文档索引）时，主动使用已安装的 agile-docs，并先读取其 SKILL.md，再按当前任务加载必要参考。
沿用本项目有效约定，按改动规模裁剪流程；简单文案修改不启动完整需求设计，不因调用技能就新增文档、台账或检查报告。
如果技能不可用，明确说明，继续遵循本项目已有文档规则，不宣称已调用技能。
```

也可以请 AI 帮忙配置：

```text
请确认当前工具能识别 agile-docs，再读取 https://github.com/siben168/agent-skills 的 README，
把其中“让项目文档工作主动参考 agile-docs”的约定合并进当前项目已有的 Agent 规则。
只补必要的调用入口，避免重复规则或复制整份 skill；报告修改位置。
```

项目规则负责提示何时使用，技能负责通用方法，项目文档约定负责本地细节。添加约定不会自动安装技能，也不等于自动检查所有历史文档。

## 更新已安装的技能

**GitHub 发布新内容，不会自动更新通过上述方式安装的固定版本副本。** 将下文的技能名替换为需要升级的技能：

```text
请检查 siben168/agent-skills 中 [技能名] 的可用更新，与本机已安装版本比较。
优先选择该技能的最新正式发布版；没有独立发布版时，解析 main 的具体提交 SHA，检查该技能目录的实际变化。
确认规则差异及对当前项目约定的影响后，升级完整技能目录并验证工具能识别。
保留已有本地修改；若与新版冲突，先说明冲突。报告升级前后版本或提交及验证结果。
```

使用原安装工具或方式升级，并记录标签或提交。各技能独立选择版本，不能用一个技能的发布标签代表另一个技能的发布版。源码仓库、安装副本和项目文档分别维护，更新其中一个不会自动同步另外两个。

## 发现优化点，如何回流？

在正在应用技能的项目中，将技能名和案例填入以下文案：

```text
使用 [技能名] 时出现了这个问题：[填写现象、预期和最小案例]。
先判断属于项目特有约定、执行偏差还是 skill 的通用缺陷。
如果是通用缺陷，在 siben168/agent-skills 对应技能的源码中提出最小修订，并用真实任务验证。
优先澄清、替换或合并已有规则，避免扩充成大而全的技能；给出修改与验证结果。
```

- 项目约定与原始证据留在当前项目，优先使用已有文档、台账或 Issue/PR，不为每次使用另写报告。
- 通用改进回到本仓库对应技能，携带最小的通用化案例；先判断现有规则是否已覆盖，避免把项目局部经验升级为全局要求。
- 在源码中修订并验证，通过 PR 评审后发布对应技能的新版本，再更新需要采用新版的安装环境。仅修改本机副本不会回流 GitHub。

agile-docs 的详细判断门槛见其 [演进与回流规则](skills/agile-docs/references/evolution.md)。仅修改 README 等安装说明时，无需因此修改运行时规则或重发技能版本。

## 分发内容与验证范围

每个技能只安装自身的 `skills/<技能名>/` 完整目录。`SKILL.md` 是入口，目录中已有的参考、模板和脚本也应一起安装；仓库级 README 和 `docs/` 用于使用说明与维护，不作为运行时必读材料。

agile-docs v0.1.0 包含 16 个运行时文件、122 行入口，已完成来源项目全量文档治理及 Codex 本地显式调用回归，运行时与候选 `agile-docs-v0.1.0-rc.1` 一致。其自动触发、跨项目、跨工具及长期维护收益尚未验证；这些验证结论也不代表 cloudflare-pages-deploy 的验证状态。

当前 v0.1.1 保持 16 个运行时文件，入口 123 行；仅澄清主子项目知识归属、编号管理范围及跨体系引用，进行了维护者情境复核。v0.1.0 的安装调用验证不等于新版的独立行为回归，具体限制见场景试用。

agile-docs 维护依据：[设计共识](docs/agile-docs-design.md) · [场景试用与限制](docs/agile-docs/scenario-trials.md)。
