---
name: git-push
description: 安全推送 - 解析目标远程与分支，更新远程状态并预警分叉，按统一远程协作策略执行推送并自动设置 upstream，仅提供 force-with-lease 安全强推
argument-hint: "[--remote REMOTE] [--branch BRANCH]"
license: MIT
---

# git-push

安全推送助手。解析目标远程与分支，推送前更新远程状态并预警分叉，在远程协作策略允许时执行推送并自动设置 upstream；推送被拒或远端分叉时仅提供 --force-with-lease 安全强推，从不执行裸 --force。

## Step 1: 环境检查与规则加载

所有命令都必须检查退出码。命令失败时立即停止当前阶段，不把空输出当作成功。

### Step 1a: Git 仓库和操作状态

1. 执行 git rev-parse --show-toplevel，取得仓库根目录；失败时提示原因并终止。
2. 后续 Git 命令从仓库根目录执行。
3. 执行 git status --short --branch，记录当前分支和工作区状态。
4. 检查 MERGE_HEAD、CHERRY_PICK_HEAD、REVERT_HEAD、REBASE_HEAD 是否存在，以及 git status 是否报告 merge、rebase、cherry-pick 或 revert 正在进行。
5. 存在未完成 Git 操作时停止推送流程，报告状态和文件，不自动 continue、abort、reset 或 rebase。
6. 工作区存在未提交更改不阻止推送——推送只涉及已有提交——但必须在最终报告中说明。

### Step 1b: 项目规则加载

配置文件路径为 .claude/git-claude-rules.json。本 skill 只读取远程协作策略；当前 canonical 规则表中没有 git-push 作用域的规则，不读取其他规则，也不得因缺失而自行新增。

1. 配置不存在时，内部调用 /git-rules init --internal。初始化过程不向用户展示；初始化失败时停止当前任务，只报告规则配置无法初始化。
2. 配置存在时，先读取文件。读取失败时立即停止，不覆盖文件。
3. 如果 JSON 解析失败或整体结构无效，内部调用 /git-rules repair --internal。修复失败时停止；修复成功后重新读取和校验。
4. 如果 project repository_relation_mode 缺失或 value 非法，内部调用 /git-rules repair project repository_relation_mode --internal。修复成功后重新读取和校验。
5. 如果配置 version 与基准版本不一致，或规则清单与 canonical 不一致，内部调用 /git-rules calibrate --internal。校准静默完成，不向用户展示过程；配置 version 高于基准版本时不做写入，继续使用现有值。
6. 从精确路径 rules.project.repository_relation_mode.value 读取规则值。
7. 规则缺失或非法时不得只在内存中静默使用默认值；必须先完成对应的内部修复。
8. 计算本次有效远程策略：
   - unconfigured：进入 Step 2 前先向用户说明两种模式的含义，调用 /git-rules ask project repository_relation_mode，按返回的 effective_action 重新计算；
   - independent_repositories：目标不猜测，由用户参数或询问确定；
   - same_repository：使用 git-rules 的 managed_safe_sync 策略，语义见 git-rules 的 Policy inheritance。
9. 所有 /git-rules ask 调用都必须先检查返回的 status：selected 时才使用 effective_action；cancelled 时终止当前任务；error 时报告规则询问或写回失败并终止，不得继续执行依赖该选择的动作。

## Step 2: 解析推送目标

从 $ARGUMENTS 读取原始参数。不得通过 shell eval 解释用户输入；参数中的引号、反斜杠、美元符号和反引号不参与任何命令拼接。

### Step 2a: 参数解析

参数格式为 `[--remote REMOTE] [--branch BRANCH]`，两个选项均可省略且顺序不限：

- 不提供选项：目标由 upstream 或后续询问确定；
- 只提供 `--remote`：覆盖目标 remote，远程分支按 Step 2b 解析；
- 只提供 `--branch`：覆盖远程分支，remote 按 Step 2b 解析；
- 同时提供：使用明确指定的 remote 和远程分支。

`--remote` 和 `--branch` 各自最多出现一次，并且必须紧跟一个非空值。出现未知选项、位置参数、重复选项或缺少值时，报告用法并停止，不猜测用户意图。不得通过拆分或重新解释选项值来兼容旧的 `[remote] [branch]` 位置参数格式。

`--remote` 的值必须与 git remote 输出中的名称完全一致；不一致时列出全部可用 remote 并停止，不自动改用其他 remote。`--branch` 的值必须通过 git check-ref-format --branch BRANCH 校验；校验失败时报告原始错误并停止。

### Step 2b: 确定 remote 和 branch

1. 执行 git remote，命令失败或输出为空时报告未配置远程仓库并结束远程阶段。
2. 执行 git branch --show-current：
   - 为空表示 detached HEAD；此时必须同时提供 `--remote` 和 `--branch` 才能继续，否则报告原因并停止；
   - 非空时继续。
3. 执行 git rev-parse --abbrev-ref --symbolic-full-name '@{u}'：
   - 退出码非 0 且错误信息为 "no upstream configured"（该场景实际退出码为 128）表示没有 upstream，不视为命令失败；
   - 其他失败必须报告并停止，不得把未知错误当作无 upstream；
   - 有 upstream 时，通过 branch.BRANCH.remote 和 branch.BRANCH.merge 取得明确的 remote 和 branch。
4. 目标优先级，自高到低，命中即停：
   - 同时提供 `--remote` 和 `--branch`：直接使用；
   - 只提供 `--remote`：该 remote 与 upstream 的 remote 相同时沿用 upstream 的 branch，否则使用当前分支名作为远程分支候选；
   - 只提供 `--branch` 且存在 upstream：使用 upstream 的 remote，并以参数值覆盖远程分支；
   - 不提供选项且存在 upstream：使用 upstream 的 remote 和 branch；
   - 只提供 `--branch` 且没有 upstream：保留参数指定的远程分支；same_repository 模式下只有一个可用 remote 时使用该 remote，否则只询问 remote；
   - 不提供选项且没有 upstream：same_repository 模式下只有一个可用 remote 时，使用该 remote 和当前分支名；其他情况询问目标 remote 和 branch。
   本 skill 由用户显式发起，缺少 upstream 时通过询问取得目标而非直接停止；询问即取得明确目标，不构成猜测。
5. 询问 remote 时最多展示 4 个选项；可用 remote 超过 4 个时打印完整列表，让用户口述名称。用户取消询问时结束远程阶段。
6. 执行 git remote get-url --push REMOTE 检查 push URL；命令失败或输出为空时停止远程阶段。
7. 后续所有 fetch、比较和 push 都使用明确的 remote、branch 和 refspec，不依赖 git 默认推断。

## Step 3: 更新远程状态与分叉检测

### Step 3a: 远程分支存在性

1. 执行 git ls-remote --heads REMOTE refs/heads/BRANCH；退出码 0 且输出为空表示远程分支不存在，其他失败停止远程阶段。
2. 远程分支不存在（首推）：展示推送目标（remote、branch、本地分支、将推送的提交数）。提交数用 git rev-list --count LOCAL_REF 计数（远端尚无此分支，推送会带上该 ref 的全部提交；LOCAL_REF 为当前分支名，detached HEAD 时为 HEAD），不用 BRANCH..LOCAL_REF——远端分支不存在时该范围会因无法解析 BRANCH 而失败，或在本地同名分支时恒为 0。展示后使用 AskUserQuestion 确认：
   - 确认推送：进入 Step 4，创建远程分支并设置跟踪关系；
   - 取消：结束远程阶段，进入 Step 5。
3. 远程分支存在时进入 Step 3b。

### Step 3b: 更新远端状态

1. 执行 git fetch REMOTE +refs/heads/BRANCH:refs/remotes/REMOTE/BRANCH；refspec 必须带 `+` 前缀，使 remote-tracking ref 无条件反映远端当前状态——否则远端被强制改写后 fetch 会因 non-fast-forward 拒绝更新，导致恰好在分叉场景下无法取得远端状态，且 --force-with-lease 依赖该 ref 与远端一致。失败时停止远程阶段并报告原始错误。
2. 执行 git rev-list --left-right --count REMOTE/BRANCH...HEAD 取得 behind/ahead 数量（左侧为远端独有提交，右侧为本地独有提交）；失败时停止远程阶段。
3. 按结果处理：
   - behind=0 且 ahead>0：快进推送。展示将推送的提交（git log --oneline REMOTE/BRANCH..HEAD，超过 20 条时截断并报告总数），进入 Step 4；
   - behind=0 且 ahead=0：报告本地与远程已同步，无需推送，进入 Step 5；
   - behind>0：远程冲突预警。分别展示双方独有提交（git log --oneline HEAD..REMOTE/BRANCH 和 REMOTE/BRANCH..HEAD，超过 20 条时截断并报告总数），停止自动推送，使用 AskUserQuestion 询问：
     - 放弃推送：报告状态，建议使用专门的拉取、同步或冲突处理流程，进入 Step 5；
     - force-with-lease 强推：明确说明将丢弃远端哪些提交及后果，确认后进入 Step 4 强推路径；
   - 用户取消询问时按放弃推送处理。
4. behind>0 时的强推分支同样适用于 ahead=0 的情况（等于回退远端），说明文案必须如实反映丢弃的提交。

## Step 4: 执行推送

### Step 4a: 构建推送命令

不得拼接 shell 命令字符串；使用进程参数数组，refspec 中的变量按实际值展开，不解释用户输入中的特殊字符。

- 常规路径：git push REMOTE LOCAL_REF:BRANCH；LOCAL_REF 为当前分支名，detached HEAD 时为 HEAD；
- 无 upstream 且 LOCAL_REF 为当前分支名时追加 --set-upstream——目标已由参数或询问明确确定，设置 upstream 不构成猜测；detached HEAD 无法设置 upstream，跳过并在报告中说明；
- 强推路径：git push --force-with-lease REMOTE LOCAL_REF:BRANCH。仅此一种强推形式，永不使用裸 --force，不提供 --force 选项。

### Step 4b: Pre-push 检查

1. 执行 git rev-parse --git-path hooks 取得实际 hooks 目录；相对路径按仓库根目录解析。
2. 读取 git config --get core.hooksPath；退出码为 1 表示未配置，其他失败必须停止；若存在，以实际配置为准。
3. 检查实际 hooks 目录中的 pre-push 文件；检测到时告知用户推送会自动运行该 hook，不因此重复询问。

### Step 4c: 执行与失败处理

1. 执行推送并检查退出码。
2. 成功后执行 git log --oneline -n 1 报告推送结果；再执行 git rev-parse --abbrev-ref --symbolic-full-name '@{u}' 取得跟踪关系，退出码非 0 且错误信息为 "no upstream configured"（detached HEAD 或未设置 upstream 的场景）时跳过跟踪关系报告，不视为失败，其他失败必须报告。
3. 非零退出时按类型处理：
   - 认证、权限、网络和 hook 错误：显示原始错误，不自动重试，停止；
   - 非快进被拒（远端在检测后再次变化）：不自动重试，重新执行 Step 3b 的 fetch 和分叉检测，按最新状态重新走预警流程；
   - 其他错误：显示原始错误并停止。

## Step 5: 推送后报告

根据最新 git status 和推送结果报告：

- 是否推送成功、推送了哪些提交、目标 remote 和分支；
- upstream 是否已设置或新建跟踪关系；detached HEAD 时说明未设置 upstream 的原因；
- 是否放弃推送、远端分叉状态，以及建议的后续流程；
- 已同步时明确报告无需操作；
- 工作区是否存在未提交或未跟踪更改（不随本次推送上传）。

## 错误处理原则

以下约束适用于全流程，不被任何分支覆盖：

- 认证、权限、网络和 hook 错误：显示原始错误和结果，不自动重试；
- 任何内部规则初始化、修复或写入失败：停止当前任务，不继续执行依赖该规则的动作；
- 远程状态无法证明可安全推送（behind>0 且用户未选择强推）时停止推送，不自动执行 git pull、merge、rebase 或任何改写本地历史的操作；
- 强推仅限 --force-with-lease 且必须经用户明确选择，永不使用裸 --force；
- 用户取消询问时结束当前任务，不继续执行其他操作。

其余错误行为（参数错误、无远程、detached HEAD 缺少必要选项、无 upstream 的询问）已在对应步骤声明。
