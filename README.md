# git-claude

> [!WARNING]
> **本项目目前处于极早期版本。** 功能、配置格式、命令行为和目录结构都可能随时发生较大变化，暂不建议用于关键或生产环境。使用前请确认工作区状态，并自行保留必要备份。

`git-claude` 是一组面向 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 的 Git Skills，尝试把常见的提交操作、项目级偏好和安全检查组织成可复用、可配置的工作流。

## 当前能力

- **`/git-help`**：列出可用 Git Skills、说明具体技能的用法，或根据问题推荐分步操作流程；该技能只提供指导，不修改仓库。
- **`/git-rules`**：管理 `.claude/git-claude-rules.json` 中的项目级规则，支持查看、设置、重置、初始化、修复、校准、导入和导出。
- **`/git-commit`**：分析暂存内容，生成或校验提交信息，按配置执行提交前检查，并在策略允许时处理远程同步。

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
│   └── skills/
│       ├── git-commit/              # 智能提交工作流
│       ├── git-help/                # 只读帮助与技能导览
│       └── git-rules/               # 规则管理及询问映射
├── .gitignore
├── LICENSE
└── README.md
```

## 基本使用

将本仓库中的 `.claude/skills/` 放入目标项目的 `.claude/skills/` 下，随后可在 Claude Code 中使用：

```text
/git-help
/git-help git-commit
/git-rules show
/git-commit
```

也可以向 `/git-commit` 传入提交信息；具体参数和行为请使用 `/git-help <技能名>` 查看。首次使用或配置不完整时，相关技能会按需初始化、修复或询问项目规则。

## 使用前须知

- 执行 Git 操作前，请先检查当前分支、暂存区和工作区内容。
- 自动检查只能降低误操作风险，不能替代人工审查。
- 涉及提交、推送或规则写入时，请认真核对 Claude Code 展示的范围与结果。

## 许可证

本项目采用 [MIT License](LICENSE)。
