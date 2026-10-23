# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

git-claude 是一组面向 Claude Code 的 Git Skills，全部由 Markdown 指令构成（无源码、无依赖、无构建、无测试、无 CI）。受众是所有同时使用 Git 与 Claude Code 的人。README 标注本项目处于极早期版本，配置格式和目录结构可能变化。

四个 skill 位于 `.claude/skills/`，在本仓库会话中可直接调用：

| skill | 分工 |
| --- | --- |
| `/git-help` | 只读导览：list / detail / advice 三分支，按参数分派，不修改仓库 |
| `/git-rules` | 规则配置中心：show / set / reset / init / repair / calibrate / ask / export / import |
| `/git-commit` | 六步提交流程：环境与规则 → 暂存分析 → message → 提交 → 远程同步 → 报告 |
| `/git-push` | 五步推送流程：环境与规则 → 解析目标 → 分叉检测 → 执行推送 → 报告 |

## 常用命令

没有 build / lint / test 命令。开发方式是直接编辑 `.claude/` 下的文件，验证方式：

- 实际调用 skill 走流程：`/git-help`、`/git-rules show`、`/git-commit`、`/git-push`
- 改动 SKILL.md 后做只读审查：`/review-skill <SKILL.md 路径> [相关文件...]`（用户级 skill，位于 `~/.claude/skills/review-skill`），例如 `/review-skill .claude/skills/git-commit/SKILL.md .claude/git-claude-rules.json`
- 改动规则或协议后用 `/git-rules show` 核对实际配置；技能运行时的 repair / calibrate / set 会改写 `.claude/git-claude-rules.json`，属于正常变更，按双轨分支约定随 `workspace-sync` 提交即可

## 项目约定

### 版本规则

- 版本号格式 `0.主.子`（major.minor.patch）：第一位为 major，表示项目大阶段；项目处于早期预览阶段，故固定为 0。
- 第二位为 minor：每新增一个 skill、hook 等能力后迭代。
- 第三位为 patch：每次修改已有内容后迭代。
- Release tag 不必每个小版本都创建：相邻小版本变更极小时只给最终版本打 tag（例：0.3.0→0.3.1 变更极小，只发了 0.3.1 的 tag）。
- 版本号的唯一基准文件是 `.claude/skills/git-rules/VERSION`。

### 双轨分支

- `workspace-sync`：个人多设备同步的工作分支，始终领先，跟踪全部内容（含本地私有文件，不维护 `.gitignore`）。
- `main`：发布分支，只含可发布内容，由 `.gitignore` 忽略本地私有文件（`scratch/`、`.claude/git-claude-rules.json`、`.claude/standards/commit-message-format.md` 等）。

### 发布流程

1. 从 `main` 创建 `release/vX.Y.Z`；
2. 将 `workspace-sync` 上的非本地（可发布）更改移入该分支，排除本地私有文件；
3. 发起 PR（标题惯例 `vX.Y.Z update`），squash and merge 到 `main`；
4. 按需创建 tag 与 GitHub Release（当前均为 prerelease）；
5. 将 `main` merge 回 `workspace-sync`，保持 `workspace-sync` 领先。

### 提交约定

- commit message 使用完整 Conventional Commits 结构，但不使用 scope（见 `.claude/standards/commit-message-format.md`）。

### 技能准入标准

适用于本库全部技能（既有技能的改造评估与新增技能的立项均须检验）：

- 每个技能必须满足其一，否则不应存在于本库：**普遍性**（即「通用」，含义见会话纠错一节）或**不可或缺性**（无法通过其他基础技能组合得到）。
- 「基础技能」包括：平替 Git 内同名指令的技能，以及流程简单、无法再分的自定义技能。
- 判断不可或缺性时，将基础技能与现有技能视为可自由编排的原子步骤：若某技能的全部作用可由它们组合复现，该技能即不满足标准，应改造或淘汰。

## 架构

### 配置体系（跨文件关系）

一套配置由四类文件共同定义，改动协议时必须同时考虑：

1. `.claude/skills/git-rules/VERSION` — 唯一版本基准，单行 `MAJOR.MINOR.PATCH`。缺失、不可读或格式非法时停止，不得从 SKILL.md、配置文件或记忆中猜测版本。
2. `.claude/git-claude-rules.json` — 项目规则配置，固定 schema：顶层 `version` + `rules`，每条规则为 `{value, options, labels, description, set_at}`（可选 `type`）。实际值统一用 JSON Pointer `/rules/{scope}/{key}/value` 引用。
3. `.claude/skills/git-rules/references/ask/{scope}/{key}.md` — 每条 `ask_mode != none` 的规则一份询问映射（问题、选项、effective_action、stored_value）。路径中的 scope/key 用 canonical 原值。
4. `.claude/standards/` — `staged-content-checks.md` 是预检标准（pre_check_mode 为 warn/block 时必须完整读取，缺失即停止提交）；`commit-message-format.md` 是 `custom` 消息风格的规范来源。

### 跨技能协议块（blocks/）

`.claude/skills/git-rules/references/blocks/` 存放跨技能共享的协议块。当前两块：

- `task-init.md`（任务初始化：git 环境与操作状态检查 + 配置快照，可执行步骤；配置协议定义权威仍在 git-rules SKILL.md 的 Configuration snapshot and recovery 节）；
- `error-principles.md`（跨技能错误处理不变量，无执行步骤、读取即生效；各技能的出口词汇定义与流程内错误分支仍在技能正文）。

通用「执行与跳过」机制由 task-init 块单点保管：后续新增块只写自身执行内容，跳过语义按其通用机制执行，不各自复制该节。

编写与引用约定（写给技能编写者，不进块文件）：

- 任务流技能（git-commit、git-push 及未来的任务流/原语技能）在 Step 1 调用 task-init 并声明所需 value JSON Pointer 清单（可为空，为空时块只做版本与配置健康检查），并在「错误处理原则」节调用 error-principles（一行调用 + 缺失即终止声明，不复制不变量）；git-help（只读导览，不碰仓库状态）与 git-rules（协议定义权威，正文内嵌）两块均豁免；
- 引用方只写一行调用 + value 清单 + 自身特有语义，不复制块内文字；引用行必须声明「块文件缺失、为空或不可读时终止任务，不自动创建或猜测块内容」；
- 技能特有语义（如 git-push 的「工作区更改不阻止推送」）留在技能内，不并入块；
- 新增协议块时更新本节列举；被委托技能经块内跳过规则自动去重，调用方无需附加声明。

### 配置快照协议（所有 skill 任务开始时）

执行细则单点定义于 `.claude/skills/git-rules/references/blocks/task-init.md`（见上一节）；引用该块的技能按块内规则执行并声明所需 value 清单。

1. 各读一次 VERSION 和配置文件，只比较配置 `version` 与 VERSION，不扫描规则清单。
2. 版本一致：只缓存本任务实际用到的 value，**不读取 git-rules/SKILL.md**。
3. 版本不一致 / 配置缺失 / JSON 损坏：内部调用 `/git-rules calibrate --internal` 或 `repair --internal`；单项 value 缺失或非法：`repair SCOPE KEY --internal`。校准失败或配置版本高于基准即停止。
4. 任何配置写入（ask / set / repair / calibrate）成功后丢弃旧快照，一次性重读；重读仍不可用时停止，**不得用内存默认值继续**。
5. value 只在流程实际使用时才验证，不预先校验其他规则。

### canonical 规则表与 ask 协议

- git-rules SKILL.md 中的规则表（scope / key / 默认值 / options / 控制值 / ask_mode / description）是 init、repair、calibrate、import 的**唯一规范来源**。修改规则模型时以该表为准，不从配置文件反推。
- 控制值（`unconfigured`、`prompt`）只触发询问，不是本次动作：不得作为 ask 的候选项或 `effective_action` 返回。`prompt` 可用 set 写回以恢复“每次询问”，`unconfigured` 只能由 init / reset / repair / calibrate 写入。
- ask_mode 三种：`initialize`（2–4 选项，必须写回）、`inline_persistence`（恰好 4 选项，一次表达动作+是否持久化）、`separate_persistence`（两问：动作 + 仅本次/始终使用）。ask 返回 `status / effective_action / persisted / stored_value`，调用方只消费这些字段，不复制候选项、展开映射或写回逻辑。
- 其他 skill 写规则只能内部调用 set，**不直接改 JSON**；不为 git-push 新增作用域（现有 scope 为 `project`、`git-commit`）。
- 单个 AskUserQuestion 问题最多 4 个选项。

### 远程策略继承

`project.repository_relation_mode` 是远程协作的上层开关：

- `independent_repositories`：各 skill 使用自己的远程字段（如 `post_commit_push_mode`）。
- `same_repository`：统一 `managed_safe_sync` 策略覆盖子字段——远程动作前先 fetch 更新状态；验证远端提交是本地 HEAD 的祖先且本地领先才允许 push；无 upstream 且只有一个可用 remote 时自动设置；远端出现独有提交即停止写入。子字段原始值保留，切回独立模式后恢复。

## 维护约定与安全不变量

### 跨文件同步义务

- 新增 `ask_mode != none` 的规则时，必须同时创建 `references/ask/{scope}/{key}.md`，否则 ask 返回 error。
- git-help 的 Skill registry 表必须与实际存在的 skill 目录同步；`scratch/` 设计文档中提到但未实现的技能（如 `/git-status`）**不得**写入 registry 或假定存在。
- git-help 和 git-rules 有意采用“命令分派 + 独立分支”的非线性结构（正文有 HTML 注释声明），不要把它们“重构”成线性 Step 1 → 2 → 3。
- `scratch/` 是设计方案与审查记录，按双轨分支约定属本地私有内容（`workspace-sync` 跟踪、`main` 忽略），不是运行时依赖。

### 安全不变量（适用于全部 skill，不被任何分支覆盖）

- 每条 git 命令检查退出码；空输出不等于成功。
- 用户输入不经 shell eval，不拼接 shell 命令字符串；commit message 用参数数组或 `git commit --file=-` 传入。
- 存在冲突或未完成的 merge / rebase / cherry-pick / revert 时停止，不自动 continue / abort / reset / rebase。
- 远端状态无法证明安全时停止远程写入，不自动 pull / rebase / 改写本地历史；强推仅限 `--force-with-lease` 且必须用户明确选择，永不使用裸 `--force`。
- 认证、权限、网络、hook 错误原样报告，不自动重试。
- `/git-rules ask` 返回 `cancelled` 或 `error` 时终止依赖该选择的后续动作。
- 除非用户或项目要求，不把自己（Claude）加入 author 或 collaborator 栏。

### 其他

- 本仓库是 skill 的权威来源；`~/.claude/skills/` 下存在较旧副本（缺 git-push 和 VERSION），不要反向同步。
- 所有面向用户的内容为中文；README 使用前须知：自动检查不能替代人工审查，执行 Git 操作前先确认分支、暂存区和工作区状态。

## 会话纠错与澄清

记录会话任务中出现过的错误理解或违规操作，以抽象原则形式供后续会话遵循。

- **误解「通用」的含义**：错误在于把「对大部分项目通用」理解为技能需要兼容无 git-rules 的环境并设计降级分支。正确理解：「通用」指每个技能都足够抽象，可应用于采用不同 Git 管理方案的项目，用于简化甚至自动化该项目的 Git 管理流程。做法：技能设计一律假设套装（git-rules + 全部技能）完整安装，不设计无配置降级路径；目标项目之间的差异由技能参数与 git-rules 规则吸收。
- **写入非项目路径**：错误在于把计划、草稿等过程产物写到项目目录之外的路径。正确做法：过程产物一律写入本仓库 `scratch/`（草稿箱）；除非用户明确要求，禁止创建、修改或删除项目目录之外的文件（**尤其是 C:/Users/13199/.claude 等全局路径**）。
- **主动维护版本号**：错误在于把迭代 VERSION patch 号当作修改 skill 内容后的自动义务。正确理解：版本号由用户维护，无需 Claude Code 维护。做法：除非用户明确要求，不修改 `.claude/skills/git-rules/VERSION`，也不为内容变更主动迭代版本号；「版本规则」一节仅作项目背景知识，不构成 Claude 的操作指令。
- **把运行时语义写进开发侧文档**：错误在于把「出错后执行」这一运行时触发时机写进 error-principles 块自述和本仓库 CLAUDE.md 的块枚举。正确理解：本项目的 Claude 负责开发这个库，不是执行这个库的工具——何时读取块属于运行时语义，应写在面向目标项目的产品文件（技能正文的调用行）；块文件只承载约束内容本身，本仓库 CLAUDE.md 只承载开发与维护约定。做法：调用行写明「出错后执行」并声明加载技能时不预读，防止执行侧模型默认预读该块。
