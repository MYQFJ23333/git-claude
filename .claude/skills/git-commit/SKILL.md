---
name: git-commit
description: 智能提交 - 自动分析暂存文件，生成符合 Conventional Commits 规范的 message，支持 issue 关联、预检和交互式选择
argument-hint: "[message]"
license: MIT
---

# git-commit

智能提交助手，自动分析暂存文件内容，生成规范的 commit message，并执行预检确保提交安全。

## Step 1: 环境检查与规则加载

检查当前目录是否为 git 仓库，并加载项目规则配置。

### Step 1a: Git 仓库验证

执行 `git status` 验证当前目录是 git 仓库。如果不是，提示用户并终止。

### Step 1b: 规则加载

检查 `.claude/rules.json` 是否存在：
- **存在**：读取并解析 `git-commit` 相关规则
- **不存在**：读取 `/git-rules` 并创建默认规则文件

提取以下规则（缺失时使用默认值）：
- `when_staging_empty`：暂存区为空时的行为（默认：`"ask"`）
- `auto_message`：是否自动生成 commit message（默认：`true`）
- `message_style`：commit message 风格（默认：`"conventional"`）

## Step 2: 暂存区分析

分析当前暂存区状态，确定提交范围。

### Step 2a: 检查暂存区

执行 `git diff --cached --stat` 获取暂存文件列表。

如果暂存区为空：
- 根据 `when_staging_empty` 规则决定行为：
  - `"ask"`：询问用户是否要 `git add` 所有更改
  - `"always_stage_all"`：自动执行 `git add .`
  - `"always_error"`：提示暂存区为空并终止

### Step 2b: 文件分析

对暂存文件进行分析：
- 识别变更类型（新增/修改/删除/重命名）
- 检测大文件（>1MB）并警告
- 检测二进制文件并提示
- 扫描敏感信息（密钥、密码、token 等）

如果检测到问题，询问用户是否继续。

## Step 3: 生成 Commit Message

如果用户提供了`$message`，则跳过此步
根据暂存内容生成符合规范的 commit message。

### Step 3a: 分析变更内容

执行 `git diff --cached` 获取详细变更，分析：
- 主要变更类型（feat/fix/docs/style/refactor/test/chore）
- 影响范围（模块/组件）
- 变更摘要

### Step 3b: 识别关联信息

扫描暂存文件和变更内容，识别：
- Issue 编号（如 `#123`、`PROJ-456`）
- Ticket 编号（如 `JIRA-789`）
- 相关的 PR 引用

### Step 3c: 生成消息

根据 `message_style` 规则生成消息：

**Conventional Commits 格式**（默认）：
```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

**Simple 格式**：
```
<description>
```

如果用户提供了 `[message]` 参数，使用用户提供的内容作为描述部分。

### Step 3d: 展示并确认

向用户展示生成的 commit message，询问：
- 直接使用
- 修改后使用
- 重新生成

## Step 4: 执行提交

确认消息后执行 git commit。

### Step 4a: 构建提交命令

根据确认的消息构建命令：
- 如果有 body/footer：使用 `git commit -m "title" -m "body"`
- 如果只有一行：使用 `git commit -m "message"`

### Step 4b: 执行并报告

执行提交命令，完成后显示：
- 提交哈希
- 提交消息
- 变更文件统计

## Step 5: 提交远程仓库

### Step 5a：检测远程仓库

检查该项是否连接了远程仓库。
如果没有，跳过此步。
如果有，询问用户是否同步到远程仓库。

## Step 6: 提交后建议

根据当前仓库状态，提供下一步操作建议：
- 如果有未暂存更改：建议 `git add` 或创建新提交
- 如果当前分支有 upstream：建议 `git push`
- 如果没有 upstream：建议设置 upstream 并推送
- 如果有未推送的提交：提醒推送

## Protocol: 规则自愈机制

每个 skill 执行前必须执行以下流程：

```
┌─────────────────────────────────────────────────────┐
│ 1. 检查 .claude/rules.json 是否存在                   │
│    ├─ 存在 → 读取并解析                                │
│    └─ 不存在 → 调用 create_rules_json() 创建           │
├─────────────────────────────────────────────────────┤
│ 2. 提取本 skill 相关规则                               │
│    ├─ 规则存在 → 按规则执行，跳过询问                    │
│    └─ 规则缺失 → 继续步骤 3                            │
├─────────────────────────────────────────────────────┤
│ 3. 询问用户                                            │
│    ├─ 提供选项 + "始终"选项                             │
│    └─ 用户选择"始终" → 更新 rules.json                 │
├─────────────────────────────────────────────────────┤
│ 4. 执行操作                                            │
└─────────────────────────────────────────────────────┘
```

**create_rules_json() 函数逻辑**：
1. 检查 `.claude/` 目录是否存在，不存在则创建
2. 生成包含默认值的 `rules.json`
3. 返回空规则对象（所有规则使用默认行为）

## Protocol: 安全检查

在执行任何提交操作前，必须完成以下安全检查：

1. **敏感信息扫描**：检查 diff 中是否包含密码、密钥、token 等
2. **大文件警告**：单个文件 >1MB 时警告
3. **二进制文件提示**：二进制文件需要特殊处理
4. **调试代码检测**：检查 console.log、print、debugger 等调试语句

如果检测到问题，必须明确警告用户并获得确认才能继续。
