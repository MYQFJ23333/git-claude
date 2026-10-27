# git-claude

## 当前能力

- **`/git-help`**：列出可用 Git Skills、说明具体技能的用法，或根据问题推荐分步操作流程；该技能只提供指导，不修改仓库。
- **`/git-rules`**：管理 `.claude/git-claude-rules.json` 中的项目级规则，支持查看、设置、重置、初始化、修复、校准、询问、导入和导出。
- **`/git-commit`**：分析暂存内容，生成或校验提交信息，按配置执行提交前检查，并在策略允许时处理远程同步。
- **`/git-fetch`**：机械更新远程跟踪引用（全分支或单分支），按仓库协作关系挂钩 prune，检测远端历史改写信号；`/git-push` 等任务流的远程状态更新经此委托，也可直接调用。
- **`/git-push`**：解析目标远程与分支，更新远程状态并预警分叉，按远程协作策略安全推送并自动设置 upstream；需要强推时仅使用 `--force-with-lease`。

目前的可配置行为包括：

- 暂存区为空时如何处理；
- commit message 的格式；
- 大文件、敏感文件名和敏感信息的提交前检查；
- 本地仓库与远程仓库的协作关系；
- 提交完成后的推送策略。

## 目录结构

```text
.
├── .claude/
│   ├── skills/
│   │   ├── git-commit/              # 智能提交工作流
│   │   ├── git-fetch/               # fetch 原语，更新远程跟踪引用
│   │   ├── git-help/                # 只读帮助与技能导览
│   │   ├── git-push/                # 安全推送工作流
│   │   └── git-rules/               # 规则管理、协议块、询问映射与 VERSION
│   └── standards/
│       ├── commit-message-format.md # 自定义 commit message 格式规范
│       └── staged-content-checks.md # 项目级暂存内容检查标准
├── .gitignore
├── LICENSE
└── README.md
```

## 规则与标准

- `.claude/skills/git-rules/VERSION` 是 Git Skills 使用的唯一基准版本文件。
- 任务流技能在开始时执行 `.claude/skills/git-rules/references/blocks/task-init.md` 初始化块（git 环境与操作状态检查、配置快照）：版本一致时只缓存本次任务需要的配置值，不预先扫描或校验完整规则清单；同一任务内的委托链不会重复执行该块。
- 配置缺失、损坏或版本不一致时，初始化块会按需调用 `/git-rules repair` 或 `/git-rules calibrate`。
- 跨技能错误处理不变量统一定义在 `.claude/skills/git-rules/references/blocks/error-principles.md`，各任务流技能出错时读取并遵循。
- 暂存内容检查标准统一存放在 `.claude/standards/staged-content-checks.md`；自定义 commit message 格式规范存放在 `.claude/standards/commit-message-format.md`。

## 基本使用

将本仓库中的 `.claude/skills/` 和 `.claude/standards/` 放入目标项目的对应路径，随后可在 Claude Code 中使用：

```text
/git-help
/git-help git-commit
/git-help git-push
/git-rules show
/git-commit
/git-fetch
/git-push
/git-push --remote origin --branch main
```

也可以向 `/git-commit` 传入提交信息，通过 `--remote`、`--branch` 明确指定 `/git-push` 的推送目标；具体参数和行为请使用 `/git-help <技能名>` 查看。首次使用或配置不完整时，相关技能会按需初始化、修复、校准或询问项目规则。

## 使用前须知

- 执行 Git 操作前，请先检查当前分支、暂存区和工作区内容。
- 自动检查只能降低误操作风险，不能替代人工审查。
- 涉及提交、推送或规则写入时，请认真核对 Claude Code 展示的范围与结果。

## 许可证

本项目采用 [MIT License](LICENSE)。