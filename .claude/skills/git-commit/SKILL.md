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

### Step 1b: 项目规则加载

配置文件路径为 .claude/git-claude-rules.json。

1. 配置不存在时，内部调用 /git-rules init --internal。初始化过程不向用户展示；初始化失败时停止当前任务，只报告规则配置无法初始化。
2. 配置存在时，先读取文件。读取失败时立即停止，不覆盖文件。
3. 如果 JSON 解析失败或整体结构无效，内部调用 /git-rules repair --internal。修复失败时停止；修复成功后重新读取和校验。
4. 如果单个规则缺失或 value 非法，内部调用 /git-rules repair SCOPE KEY --internal。修复成功后重新读取和校验。
5. 如果配置 version 与基准版本不一致，或规则清单与 canonical 不一致，内部调用 /git-rules calibrate --internal。校准静默完成，不向用户展示过程；配置 version 高于基准版本时不做写入，继续使用现有值。
6. 从以下精确路径读取规则值：
   - rules.project.repository_relation_mode.value
   - rules.project.large_file_size_limit.value
   - rules["git-commit"].empty_staging_mode.value
   - rules["git-commit"].commit_message_mode.value
   - rules["git-commit"].post_commit_push_mode.value
   - rules["git-commit"].pre_check_mode.value
7. 规则缺失或非法时不得只在内存中静默使用默认值；必须先完成对应的内部修复。
8. 计算本次有效远程策略：
   - unconfigured：首次进入远程同步时按 Step 5b 询问并保存选择；
   - independent_repositories：使用 git-commit 自身的 post_commit_push_mode；
   - same_repository：使用 git-rules 的 managed_safe_sync 策略，语义见 git-rules 的 Policy inheritance。
9. 所有 /git-rules ask 调用都必须先检查返回的 status：selected 时才使用 effective_action；cancelled 时终止当前任务；error 时报告规则询问或写回失败并终止，不得继续执行依赖该选择的动作。

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

各分支解析完成后，对暂存区执行以下检查：

1. 识别新增、修改、删除、重命名、复制和二进制文件。
2. 删除文件不扫描删除内容；重命名同时检查旧路径和新路径的敏感文件名。
3. 二进制文件跳过文本内容扫描，但继续执行文件名和大小检查，并显示文件名、状态和大小。
4. 无法读取暂存 blob、无法识别编码或扫描器执行失败时停止提交，不把失败视为未发现问题。

#### 大文件检测

large_file_size_limit 必须是正数，格式为数字加 B、KB、MB 或 GB，单位不区分大小写，按 1024 进位计算。例如 1MB 等于 1048576 字节。

- 暂存 blob 大小超过阈值：警告并询问是否继续；
- 所有非删除暂存 blob 总大小超过阈值的 10 倍：警告并询问是否继续；
- 使用 git check-attr filter -- PATH 检查实际路径是否配置 Git LFS；
- 如果匹配 LFS，提示应由 LFS 管理；不自动转换文件或修改 .gitattributes；
- 用户拒绝或取消时终止，保留当前暂存状态。

#### 敏感文件名检查

按不区分大小写的完整路径或文件名匹配：

- 直接阻止：私钥文件（*.key、*.pem、*.p12、*.pfx、*.jks）、凭据文件（credentials*）、环境文件（.env 及其变体）、id_rsa、id_ed25519；
- 警告并确认：文件名包含 secret 或 password 的普通文件、*.pub 公钥文件；
- 删除敏感文件不按“新增敏感文件”阻止，但仍在报告中说明删除动作。

直接阻止时不提供“确认后继续”选项；警告级别必须由用户明确选择继续或终止。

#### 敏感信息内容扫描

内容扫描必须针对 git show :PATH 输出的暂存版本，使用 rg --pcre2 -n -I 或等价的 PCRE2 扫描器。正则不得再经过 Markdown 表格转义。rg 返回 0 表示匹配，1 表示无匹配，2 或更高值表示扫描失败。执行器必须丢弃原始匹配行，只保留文件路径、行号和规则名称。

阻止级别模式：

    AWS Access Key: \b(?:AKIA|ASIA)[0-9A-Z]{16}\b
    AWS Secret Key: (?i)\b(?:aws_secret_access_key|aws_secret)\s*[:=]\s*['"]?[A-Za-z0-9/+=]{40}['"]?
    GitHub Token: \b(?:ghp|gho|ghu|ghs|ghr)_[A-Za-z0-9_]{36,}\b
    Private Key: -----BEGIN (?:RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----
    Stripe Secret Key: \b(?:sk|rk)_(?:test|live)_[0-9A-Za-z]{10,}\b
    Slack Token: \bxox[bapors]-[0-9A-Za-z-]{10,}\b

警告级别模式：

    Generic API Key: (?i)\b(?:api[_-]?key|apikey)\s*[:=]\s*['"][0-9A-Za-z]{32,}['"]
    Password in Code: (?i)\b(?:password|passwd|pwd)\s*[:=]\s*['"][^'"]{8,}['"]
    Connection String: (?i)\b(?:mysql|postgres(?:ql)?|mongodb(?:\+srv)?)://\S{20,}
    JWT Token: \beyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\b
    Generic Secret: (?i)\b(?:secret|token)\s*[:=]\s*['"][0-9A-Za-z_-]{16,}['"]
    Stripe Publishable Key: \bpk_(?:test|live)_[0-9A-Za-z]{10,}\b

扫描结果只显示文件路径、行号和类型，不显示完整匹配值；不得直接向用户展示扫描器原始输出。

- 阻止级别：立即终止，不允许用户绕过；
- 警告级别：显示类型和位置，询问是否继续；
- 用户拒绝或取消：终止并保留暂存状态。

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
- custom：读取 .claude/sample/commit-message.md。

Conventional type 使用 feat、fix、docs、style、refactor、perf、test、build、ci、chore 或 revert。标题必须有合法 type、冒号和非空描述。

custom 文件缺失、为空或无法读取时停止，并提示需要配置该文件、提示用户可通过文本描述引导会话创建对应文件；不在提交流程中自动创建或猜测 custom 规范。

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

### Step 5a: 解析远程上下文

1. 执行 git remote，命令失败时报告错误并结束远程阶段；输出为空时报告未配置远程仓库并进入 Step 6。
2. 执行 git branch --show-current：
   - 为空表示 detached HEAD；报告本地 commit 成功，跳过自动远程操作；
   - 非空时继续。
3. 执行 git rev-parse --abbrev-ref --symbolic-full-name '@{u}'；退出码为 1 表示没有 upstream，其他失败必须报告。存在 upstream 时，再通过当前分支的 branch.BRANCH.remote 和 branch.BRANCH.merge 配置取得明确的 remote 和 branch，并执行 git remote get-url --push REMOTE 检查对应 push URL；push URL 缺失或命令失败时停止远程阶段。
4. 有 upstream 时使用 upstream 的 remote 和 branch。
5. 没有 upstream 时：
   - same_repository：若只有一个可用 remote 且存在 push URL，使用当前分支名作为远程分支候选，并按 Step 5c 自动设置 upstream；多个可用 remote 时询问用户，用户取消则停止远程阶段；没有可用 remote 时跳过远程阶段；
   - independent_repositories：不猜测目标；需要 push 时报告缺少 upstream 并停止自动 push。
6. 所有后续 fetch 和 push 都必须使用明确的 remote、branch 和 refspec。

### Step 5b: 选择有效远程策略

repository_relation_mode 为 unconfigured 时，先向用户说明两种模式的含义，再调用 /git-rules ask project repository_relation_mode；该初始化型询问只显示两种实际模式并必须写回，调用方按 effective_action 重新计算有效策略：

- 视为独立仓库：本地和远程分别处理，遵守 post_commit_push_mode；
- 视为同一仓库：将远程视为本地仓库协作的一部分，由统一 managed_safe_sync 策略自动处理远程事项。

effective policy：

- independent_repositories：
  - prompt：调用 /git-rules ask git-commit post_commit_push_mode；询问选项和写回映射由 git-rules 按需读取。effective_action 为 push 时使用明确的 REMOTE HEAD:BRANCH refspec，失败时停止；为 skip 时不推送并明确报告尚未同步；
  - always：有明确 upstream 时自动 push；没有 upstream 时停止并报告；
  - never：不自动 push，明确报告尚未同步。
- same_repository：使用 git-rules 的 managed_safe_sync 策略，语义见 git-rules 的 Policy inheritance。本次 post_commit_push_mode 的存储值保留，但被该策略覆盖，不参与本次行为。

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
