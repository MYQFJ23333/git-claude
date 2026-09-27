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

### Step 1a: Git 仓库和操作状态

1. 执行 git rev-parse --show-toplevel，取得仓库根目录；失败时提示原因并终止。
2. 后续 Git 命令从仓库根目录执行。
3. 执行 git status --short --branch，记录当前分支和工作区状态。
4. 检查以下特殊状态：
   - git diff --cached --name-only --diff-filter=U 是否有未解决的暂存冲突；
   - MERGE_HEAD、CHERRY_PICK_HEAD、REVERT_HEAD、REBASE_HEAD 是否存在；
   - git status 是否报告 merge、rebase、cherry-pick 或 revert 正在进行。
5. 如果存在冲突或未完成 Git 操作，停止普通提交流程，报告状态和文件，不自动 continue、abort、reset 或 rebase。

### Step 1b: 项目规则一次性读取

1. 读取 `.claude/skills/git-rules/VERSION`，只接受单行 `MAJOR.MINOR.PATCH`；文件缺失、不可读、为空或格式非法时停止，不从其他文件猜测基准版本。
2. 读取 `.claude/git-claude-rules.json` 一次并解析 JSON：
   - 文件不可读时停止，不覆盖；
   - 文件不存在或 JSON 无法解析时，内部调用 `/git-rules repair --internal`，成功后只重新读取一次，失败或仍不可用时停止。
3. 比较配置 version 与 VERSION：
   - 相等时继续；
   - 不相等、缺失或格式非法时，内部调用 `/git-rules calibrate --internal`，成功后只重新读取一次；配置版本高于基准、校准失败或重读后仍不一致时停止。
4. 版本一致后一次性缓存以下 value；缺失值也按缺失状态缓存，本步骤不校验或修复它们：

   - `/rules/git-commit/empty_staging_mode/value`；
   - `/rules/git-commit/pre_check_mode/value`；
   - `/rules/project/large_file_size_limit/value`；
   - `/rules/git-commit/commit_message_mode/value`；
   - `/rules/project/repository_relation_mode/value`；
   - `/rules/git-commit/post_commit_push_mode/value`。

版本一致的正常路径不读取 `git-rules/SKILL.md`，也不检查完整规则清单、规则元数据或各 value。每个 value 只在流程实际使用时验证；目标 scope、key 或 value 缺失、类型错误或非法时，内部调用 `/git-rules repair SCOPE KEY --internal`，成功后只重新读取一次，失败或仍不可用时停止。任何 `/git-rules ask` 或其他配置写入成功后都丢弃旧快照，并一次性重新读取 VERSION、配置 version 和后续所需 value；不得静默使用内存默认值。

所有 /git-rules ask 调用都必须先检查返回的 status：selected 时才使用 effective_action；cancelled 时终止当前任务；error 时报告规则询问或写回失败并终止，不得继续执行依赖该选择的动作。

## Step 2: 暂存区分析

### Step 2a: 确定提交范围

1. 执行 git diff --cached --name-status -z，检查命令是否成功。
2. 输出为空才表示暂存区为空。
3. 暂存区为空时按 empty_staging_mode 处理：
   - prompt：调用 /git-rules ask git-commit empty_staging_mode，按返回的 effective_action 执行下面对应分支；询问选项和写回映射由 git-rules 按需读取；
   - stage_all：从仓库根目录执行 git add --all -- .；
   - abort：报告暂存区为空并终止。
4. git add 失败时停止，保留已有暂存状态，报告原始错误。
5. 暂存操作完成后重新执行 git diff --cached --name-status -z，显示实际暂存文件清单。内部创建的规则配置属于项目文件；如果用户选择暂存全部，它会按实际状态进入暂存清单。
6. 重新检查暂存清单是否为空；为空时终止。冲突状态已在 Step 1a 检查并终止，此处不再复查。
7. 执行 git diff --name-only -z 和 git ls-files --others --exclude-standard -z，判断暂存区之外是否还存在未暂存或未跟踪更改；命令失败时停止当前阶段。
8. 暂存区非空且同时存在未暂存或未跟踪更改时，使用 AskUserQuestion 询问用户本次提交的范围。询问前必须提示用户：当前暂存区内容已确认，但询问期间仍然可以继续修改暂存区，最终提交范围以确认后的最新暂存状态为准。选项：
   - 提交全部更改：执行 git add --all -- .，把工作区更改一并纳入本次提交；
   - 只提交已暂存更改：不修改暂存区，仅提交当前已暂存内容。
9. 用户选择后重新执行 git diff --cached --name-status -z，显示最新暂存文件清单，作为本次提交的确定范围；后续 Step 2b 和 Step 3b 一律使用该清单，不使用询问前的旧结果。选择只提交已暂存更改时，本次重扫用于确定用户在询问期间是否修改过暂存区，并按修改后的内容提交。重新扫描后暂存区为空时终止。暂存区之外只剩未暂存或未跟踪更改时，不在本次提交流程中处理，留待 Step 6 报告。

### Step 2b: 文件类型、大小和安全检查

使用 Git index 中的暂存内容检查，不以工作区未暂存版本代替暂存版本。

pre_check_mode 在所有检查流程前解析。下表描述每种取值对应的动作：

| 当前值 | 动作 |
| --- | --- |
| disabled | 跳过大文件检测、敏感文件名检查和敏感信息内容扫描，直接进入 Step 3 |
| warn | 执行全部检查，发现问题时警告但允许继续 |
| block | 执行全部检查，发现阻止级别问题时终止提交 |
| prompt | 向用户介绍预检功能（大文件检测、敏感文件名检查、敏感信息扫描），调用 /git-rules ask git-commit pre_check_mode，再按返回的 effective_action 执行对应动作；询问选项和写回映射由 git-rules 按需读取 |

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

当 commit_message_mode 为 unconfigured 时，无论用户是否提供 message，都先向用户说明三种风格，再调用 /git-rules ask git-commit commit_message_mode；该初始化型询问只显示三种实际风格并必须写回，调用方按 effective_action 取得生效值：

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

### Step 4b: Pre-commit 检查

1. 执行 git rev-parse --git-path hooks，取得实际 hooks 目录；相对路径按仓库根目录解析。
2. 读取 git config --get core.hooksPath；退出码为 1 表示未配置，其他失败必须停止；若存在，以实际配置为准。
3. 检查实际 hooks 目录中的 pre-commit 文件及当前操作系统下的可执行状态。
4. 检查 .pre-commit-config.yaml 和 package.json 只作为辅助提示，不把文件存在等同于 hook 已安装。
5. 如果检测到 hook，告知用户提交时会自动运行；不因提示重复询问 message。
6. 除非用户或项目要求，否则不要将你自己添加到作者或者合作者一栏中

### Step 4c: 执行并报告

1. 执行提交并检查退出码。
2. 非零退出时立即停止，不进入 Step 5；显示原始错误，不自动 pull、rebase、reset 或重试。
3. 成功后执行 git rev-parse HEAD 和 git show --stat --oneline --summary HEAD，报告实际提交哈希、message 和文件统计。
4. 检查提交后的工作区变化。hook 产生的未暂存或未跟踪变化只报告，不自动加入下一次提交。

## Step 5: 提交后的远程同步

只有 Step 4 成功后才能进入本步骤。

### Step 5a: 选择有效远程策略

进入远程阶段时从 Step 1b 的快照取得并验证 repository_relation_mode。值为 unconfigured 时，先向用户说明两种模式的含义，再调用 /git-rules ask project repository_relation_mode；该初始化型询问只显示两种实际模式并必须写回，调用方按 effective_action 重新计算有效策略：

- 视为独立仓库：本地和远程分别处理，遵守 post_commit_push_mode；
- 视为同一仓库：将远程视为本地仓库协作的一部分，由统一 managed_safe_sync 策略自动处理远程事项。

effective policy：

- independent_repositories：
  - 此时才读取 post_commit_push_mode；
  - prompt：调用 /git-rules ask git-commit post_commit_push_mode；询问选项和写回映射由 git-rules 按需读取。effective_action 为 push 时继续解析远程上下文；为 skip 时不读取远程状态，明确报告尚未同步并进入 Step 6；
  - always：继续解析远程上下文；
  - never：不读取远程状态，明确报告尚未同步并进入 Step 6。
- same_repository：使用 Step 5c 定义的 managed_safe_sync 流程。本次 post_commit_push_mode 的存储值保留，但被该策略覆盖，不参与本次行为。

### Step 5b: 解析远程上下文

1. 执行 git remote，命令失败时报告错误并结束远程阶段；输出为空时报告未配置远程仓库并进入 Step 6。
2. 执行 git branch --show-current：
   - 为空表示 detached HEAD；报告本地 commit 成功，跳过自动远程操作；
   - 非空时继续。
3. 执行 git rev-parse --abbrev-ref --symbolic-full-name '@{u}'；退出码为 1 表示没有 upstream，其他失败必须报告。存在 upstream 时，再通过当前分支的 branch.BRANCH.remote 和 branch.BRANCH.merge 配置取得明确的 remote 和 branch，并执行 git remote get-url --push REMOTE 检查对应 push URL；push URL 缺失或命令失败时停止远程阶段。
4. 有 upstream 时使用 upstream 的 remote 和 branch。
5. 没有 upstream 时：
   - same_repository：若只有一个可用 remote 且存在 push URL，使用当前分支名作为远程分支候选，并按 Step 5c 自动设置 upstream；多个可用 remote 时询问用户，用户取消则停止远程阶段；没有可用 remote 时跳过远程阶段；
   - independent_repositories：不猜测目标；报告缺少 upstream 并停止自动 push。
6. 所有后续 fetch 和 push 都必须使用明确的 remote、branch 和 refspec。
7. independent_repositories 使用明确的 git push REMOTE HEAD:BRANCH 执行已在 Step 5a 决定的推送，失败时停止；same_repository 进入 Step 5c。

### Step 5c: same_repository 的安全同步

1. 使用 git ls-remote --heads REMOTE refs/heads/BRANCH 确认远程分支是否存在；退出码 0 且输出为空表示分支不存在，其他失败停止远程阶段。
2. 如果远程分支不存在，使用明确的 git push --set-upstream REMOTE HEAD:BRANCH 创建跟踪关系；这适用于当前分支发布到唯一可用 remote。
3. 如果远程分支存在，执行 git fetch REMOTE refs/heads/BRANCH:refs/remotes/REMOTE/BRANCH；失败时停止 push 并报告原始错误。
4. 使用 git rev-parse REMOTE/BRANCH 取得远端提交，使用 git merge-base --is-ancestor REMOTE/BRANCH HEAD 验证远端提交是否为本地 HEAD 的祖先，并使用 git rev-list --count REMOTE/BRANCH..HEAD 取得本地领先提交数。git merge-base --is-ancestor 返回 0 表示祖先关系成立，返回 1 表示不成立，其他退出码表示比较失败。
5. 仅当以下条件同时满足才允许自动 push：
   - 远端提交是本地 HEAD 的祖先，不存在远端独有提交；
   - REMOTE/BRANCH..HEAD 的提交数大于 0；
   - 比较命令全部成功。
6. 条件满足时执行明确的 git push REMOTE HEAD:BRANCH；无 upstream 时使用 --set-upstream。允许一次推送一个或多个本地领先提交。
7. 祖先关系不成立或本地领先提交数不大于 0 时，使用 git rev-list --left-right --count REMOTE/BRANCH...HEAD 取得 ahead/behind 数量，停止自动远程写入，报告本地分支、远程分支、本地 HEAD、远端提交及数量。
8. fetch、比较或 push 任一命令失败时停止远程阶段；本地 commit 保留。push 期间远端再次变化导致失败时，不自动重试，重新报告远程状态。

## Step 6: 提交后报告

根据最新 git status 和远程结果报告：

- 本地 commit 是否成功；
- 是否已同步；
- 是否存在未暂存、未跟踪或未推送提交；
- 如无 upstream，说明需要明确设置目标后才能推送；
- 如因远端分叉停止，建议使用专门的 pull、sync 或冲突处理流程，不自动执行。

## 错误处理原则

以下约束适用于全流程，不被任何分支覆盖：

- 认证、权限、网络和 hook 错误：显示原始错误和结果，不自动重试；
- 任何内部规则初始化、修复或写入失败：停止当前任务，不继续执行依赖该规则的动作；
- 远程状态无法证明可安全合并时停止远程写入，不自动执行 git pull --rebase，避免改写本地 commit。

其余错误行为（用户取消确认、暂存区为空、nothing to commit、提交失败）已在对应步骤声明。
