---
name: git-commit
description: 智能提交 - 分析暂存内容，按项目规则生成和校验 commit message，执行安全预检，并根据统一远程协作策略处理同步
argument-hint: "[message]"
license: MIT
---

# git-commit

智能提交助手。只提交经过明确确定的暂存内容，生成或校验 commit message，执行安全预检，并在远程协作策略允许时处理远程同步。

## Step 1: 环境检查与规则加载

所有命令都必须检查退出码。命令失败时立即停止当前阶段，不把空输出当作成功。

执行 `.claude/skills/git-rules/references/blocks/task-init.md`。块文件缺失、为空或不可读时终止任务，不自动创建或猜测块内容。本技能所需 value 清单：

- `/rules/git-commit/empty_staging_mode/value`；
- `/rules/git-commit/pre_check_mode/value`；
- `/rules/project/large_file_size_limit/value`；
- `/rules/git-commit/commit_message_mode/value`；
- `/rules/project/repository_relation_mode/value`；
- `/rules/git-commit/post_commit_push_mode/value`。

所有 /git-rules ask 调用都必须先检查返回的 status：selected 时才使用 effective_action；cancelled 时终止当前任务；error 时报告规则询问或写回失败并终止，不得继续执行依赖该选择的动作。

## Step 2: 暂存区分析

### Step 2a: 确定提交范围

1. 执行 git diff --cached --name-status -z，检查命令是否成功。
2. 输出为空才表示暂存区为空。
3. 暂存区为空时按 empty_staging_mode 处理：
   - prompt：调用 /git-rules ask git-commit empty_staging_mode，按返回的 effective_action 执行下面对应分支；
   - stage_all：执行 git add --all -- .；
   - abort：报告暂存区为空并终止。
4. git add 失败时停止，保留已有暂存状态，报告原始错误。
5. 暂存操作完成后重新执行 git diff --cached --name-status -z，显示实际暂存文件清单。若用户选择暂存全部，本任务内部写入的规则配置文件会按实际状态进入暂存清单。
6. 重新检查暂存清单是否为空；为空时终止。
7. 执行 git diff --name-only -z 和 git ls-files --others --exclude-standard -z，判断暂存区之外是否还存在未暂存或未跟踪更改；命令失败时停止当前阶段。
8. 暂存区非空且同时存在未暂存或未跟踪更改时，使用 AskUserQuestion 询问用户本次提交的范围。询问前必须提示用户：当前暂存区内容已确认，但询问期间仍然可以继续修改暂存区，最终提交范围以确认后的最新暂存状态为准。选项：
   - 提交全部更改：执行 git add --all -- .，把工作区更改一并纳入本次提交；
   - 只提交已暂存更改：不修改暂存区，仅提交当前已暂存内容。
9. 用户选择后重新执行 git diff --cached --name-status -z，显示最新暂存文件清单，作为本次提交的确定范围；后续 Step 2b 和 Step 3b 一律使用该清单，不使用询问前的旧结果。重新扫描后暂存区为空时终止。暂存区之外只剩未暂存或未跟踪更改时，不在本次提交流程中处理，留待 Step 6 报告。

### Step 2b: 文件类型、大小和安全检查

pre_check_mode 在所有检查流程前解析。下表描述每种取值对应的动作：

| 当前值 | 动作 |
| --- | --- |
| disabled | 跳过大文件检测、敏感文件名检查和敏感信息内容扫描，直接进入 Step 3 |
| warn | 执行全部检查，发现问题时警告但允许继续 |
| block | 执行全部检查，发现阻止级别问题时终止提交 |
| prompt | 调用 /git-rules ask git-commit pre_check_mode，再按返回的 effective_action 执行对应动作 |

最终有效模式为 disabled 时不读取检查标准，直接进入 Step 3。最终有效模式为 warn 或 block 时，完整读取 `.claude/standards/staged-content-checks.md` 并按其中标准检查暂存内容；文件缺失、为空或不可读时停止提交，不自动创建或猜测检查标准。

## Step 3: 生成或校验 Commit Message

从 $ARGUMENTS 读取原始参数。不得通过 shell eval 解释用户输入。

### Step 3a: 用户提供 message

如果 $ARGUMENTS 非空，将其作为用户指定的完整 message。只跳过自动生成，不跳过：

- 标题、body、footer 结构解析；
- Conventional 或 custom 规则校验；
- 最终 message 展示；
- 用户确认。

多行内容必须原样保留，除非用户在“修改后使用”中明确修改。

### Step 3b: 自动分析变更

仅当用户没有提供 message 时：

1. 执行 git diff --cached 获取暂存差异，命令失败则停止。
2. 分析变更类型、影响范围和摘要。
3. 识别暂存路径和差异中的 issue、ticket 和 PR 引用。
4. 使用 #[0-9]+ 识别 issue/PR 编号，使用 [A-Z][A-Z0-9]+-[0-9]+ 识别 ticket 编号；结果去重后只作为关联信息展示。自动生成的 conventional_full message 可以追加中性的 Refs footer，不自动生成 Fixes、Closes 或 Resolves。

### Step 3c: Message 风格

当 commit_message_mode 为 unconfigured 时，无论用户是否提供 message，提交信息风格尚未配置，先调用 /git-rules ask git-commit commit_message_mode，按返回的 effective_action 使用对应风格。

三种风格定义（生成与校验 message 时按生效值选用）：

- conventional_simple：单行 TYPE[(SCOPE)]: DESCRIPTION；
- conventional_full：Conventional 标题、空行、body 和可选 footer；
- custom：读取 `.claude/standards/commit-message-format.md`。

Conventional type 使用 feat、fix、docs、style、refactor、perf、test、build、ci、chore 或 revert。标题必须有合法 type、冒号和非空描述。

custom 标准文件缺失、为空或无法读取时停止，并提示需要配置该文件、提示用户可通过文本描述引导会话创建对应文件；不在提交流程中自动创建或猜测 custom 规范。

### Step 3d: 展示并确认

展示最终完整 message、检测到的关联信息以及是否包含 body/footer，然后使用AskUserQuestion工具询问：

- 直接使用；
- 修改后使用；
- 重新生成（仅自动生成的 message 可用）；
- 取消。

修改后必须重新校验；取消则终止。

## Step 4: 执行提交

### Step 4a: 构建提交输入

不得拼接 shell 命令字符串，也不得手工转义 message。

- 优先使用进程参数数组；
- 需要保留多行 message 时使用 git commit --file=-，通过标准输入传入完整 message；
- message 中的引号、反斜杠、美元符号、反引号、感叹号和换行都按原文传递。

不设置 git 的 author/committer（不使用 --author、GIT_AUTHOR_*、GIT_COMMITTER_*），也不在提交信息中加入指向 Claude 的 Co-authored-by trailer，除非用户或项目明确要求。

### Step 4b: Pre-commit 检查

1. 执行 git rev-parse --git-path hooks，取得实际 hooks 目录；相对路径按仓库根目录解析。
2. 读取 git config --get core.hooksPath；退出码为 1 表示未配置，其他失败必须停止；若存在，以实际配置为准。
3. 检查实际 hooks 目录中的 pre-commit 文件及当前操作系统下的可执行状态。
4. 如果检测到 hook，告知用户提交时会自动运行；不因提示重复询问 message。

### Step 4c: 执行并报告

1. 执行提交并检查退出码。
2. 非零退出时立即停止，不进入 Step 5；显示原始错误，不自动 pull、rebase、reset 或重试。
3. 成功后执行 git rev-parse HEAD 和 git show --stat --oneline --summary HEAD，报告实际提交哈希、message 和文件统计。
4. 检查提交后的工作区变化。hook 产生的未暂存或未跟踪变化只报告，不自动加入下一次提交。

## Step 5: 提交后的远程同步

只有 Step 4 成功后才能进入本步骤。

### Step 5a: 选择有效远程策略

进入远程阶段时从 Step 1 建立的配置快照取得并验证 repository_relation_mode。值为 unconfigured 时，远程协作模式尚未配置，先调用 /git-rules ask project repository_relation_mode，按返回的 effective_action 重新计算有效策略。

有效策略：

- independent_repositories：
  - 此时才读取 post_commit_push_mode；
  - prompt：调用 /git-rules ask git-commit post_commit_push_mode。effective_action 为 push 时继续解析远程上下文；为 skip 时不读取远程状态，明确报告尚未同步并进入 Step 6；
  - always：继续解析远程上下文；
  - never：不读取远程状态，明确报告尚未同步并进入 Step 6。
- same_repository：按统一 managed_safe_sync 策略处理，远程写入委托 Step 5c 调用 /git-push 执行，不读取 post_commit_push_mode。

### Step 5b: 解析远程上下文

1. 执行 git remote 并检查退出码；命令失败时报告原始错误并结束远程阶段；输出为空时报告未配置远程仓库并进入 Step 6。
2. 执行 git branch --show-current 并检查退出码；命令失败时报告原始错误并结束远程阶段；退出码 0 且输出为空表示 detached HEAD，报告本地 commit 成功、未执行远程操作，进入 Step 6；退出码 0 且非空时记录 CURRENT_BRANCH。
3. same_repository：直接进入 Step 5c 委托执行。
4. independent_repositories：
   - 执行 git rev-parse --abbrev-ref --symbolic-full-name '@{u}' 并检查退出码；退出码非 0 且错误信息为 "no upstream configured" 时视为没有 upstream，不视为命令失败；其他失败必须报告并结束远程阶段，不得把未知错误当作无 upstream；
   - 存在 upstream 时，通过 branch.CURRENT_BRANCH.remote 和 branch.CURRENT_BRANCH.merge 取得明确的 remote 和 branch，进入 Step 5c 委托执行；
   - 没有 upstream 时：不猜测目标，报告缺少 upstream 并跳过自动推送，进入 Step 6。

### Step 5c: 委托 /git-push 执行远程写入

1. 本 skill 不自行执行任何远程写入：fetch、ls-remote、分叉检测、push 与 upstream 设置一律委托 /git-push 完成，不复算远程写入安全判定。
2. 调用方式：
   - same_repository：调用 /git-push 不带参数，目标由 git-push 按其 Step 2 规则解析；
   - independent_repositories：调用 /git-push --remote REMOTE --branch BRANCH，目标来自 Step 5b 已解析的 upstream；
   自动策略（post_commit_push_mode=always、managed_safe_sync）仅表示自动启动推送流程：本 skill 不代替 git-push 询问，首推确认与分叉询问不因委托而豁免。
3. 结果处理：
   - 推送成功：进入 Step 6；
   - git-push 结束远程阶段（用户取消、目标无法确定、远端分叉放弃或错误）：本地 commit 保留，按其报告如实转述放弃或失败原因，进入 Step 6；
   - /git-push 不可用或委托调用本身失败：报告原始错误并结束远程阶段，本地 commit 保留。

## Step 6: 提交后报告

根据最新 git status 和远程结果报告：

- 本地 commit 是否成功；
- 是否已同步；
- 是否存在未暂存、未跟踪或未推送提交；
- 如无 upstream，说明需要明确设置目标后才能推送。

## 错误处理原则

出错后执行 `.claude/skills/git-rules/references/blocks/error-principles.md`——跨技能错误处理不变量在块内，加载技能时不预读；块文件缺失、为空或不可读时终止任务，不自动创建或猜测块内容。
