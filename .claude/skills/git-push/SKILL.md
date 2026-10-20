---
name: git-push
description: 安全推送 - 解析目标远程与分支，更新远程状态并预警分叉，按统一远程协作策略执行推送并自动设置 upstream，仅提供 force-with-lease 安全强推
argument-hint: "[--remote REMOTE] [--branch BRANCH]"
license: MIT
---

# git-push

安全推送助手。解析目标远程与分支，推送前更新远程状态并预警分叉，在远程协作策略允许时执行推送并自动设置 upstream；推送被拒或远端分叉时仅提供 --force-with-lease 安全强推，从不执行裸 --force。

## Step 1: 环境检查与规则加载

所有命令都必须检查退出码，不把空输出当作成功——空输出只有在对应步骤明确定义为合法结果（如首推、无 upstream、detached HEAD）时才可采信。命令失败时立即停止：Step 1 内的失败终止任务；进入 Step 2 之后的失败按「错误处理原则」定义的出口词汇处理。

执行 `.claude/skills/git-rules/references/blocks/task-init.md`——初始化步骤（git 环境与操作状态检查、配置快照）与「本任务已执行则跳过」规则均在块内；本技能所需 value：`/rules/project/repository_relation_mode/value`。块文件缺失、为空或不可读时终止任务，不自动创建或猜测块内容。块检查失败或存在未完成 Git 操作时终止任务。

工作区存在未提交更改不阻止推送——推送只涉及已有提交——但必须在最终报告中说明。

在解析推送目标前验证缓存值，只接受 unconfigured、independent_repositories 或 same_repository；目标规则缺失或 value 非法时，内部调用 `/git-rules repair project repository_relation_mode --internal`，成功后只重新读取一次，失败或仍不可用时停止。不得新增 git-push 作用域。

初始化完成并验证缓存值后，计算本次有效远程策略：
   - unconfigured：进入 Step 2 前先向用户说明两种模式的含义，调用 /git-rules ask project repository_relation_mode，按返回的 effective_action 重新计算；
   - independent_repositories：目标不猜测，由用户参数或询问确定；
   - same_repository：使用本 skill 在 Step 3 和 Step 4 定义的 managed_safe_sync 流程。

所有 /git-rules ask 调用都必须先检查返回的 status：selected 时才使用 effective_action；cancelled 时终止当前任务；error 时报告规则询问或写回失败并终止，不得继续执行依赖该选择的动作。

## Step 2: 解析推送目标

从 $ARGUMENTS 读取原始参数。不得通过 shell eval 解释用户输入；参数中的引号、反斜杠、美元符号和反引号不参与任何命令拼接。

### Step 2a: 参数语法解析

参数格式为 `[--remote REMOTE] [--branch BRANCH]`，两个选项均可省略且顺序不限：

- 不提供选项：目标由 upstream 或后续询问确定；
- 只提供 `--remote`：覆盖目标 remote，远程分支在 Step 2c 解析；
- 只提供 `--branch`：覆盖远程分支，remote 在 Step 2c 解析；
- 同时提供：使用明确指定的 remote 和远程分支。

`--remote` 和 `--branch` 各自最多出现一次，并且必须紧跟一个非空值。出现未知选项、位置参数、重复选项或缺少值时，报告用法并终止任务，不猜测用户意图。不得通过拆分或重新解释选项值来兼容旧的 `[remote] [branch]` 位置参数格式。

本步骤只解析语法，不读取仓库状态，也不校验 remote 名称或分支名格式——最终 REMOTE 和 BRANCH 在 Step 2c 离开目标解析前统一校验。

### Step 2b: 读取仓库状态

1. 执行 git remote 并检查退出码；失败时报告原始错误并结束远程阶段，进入 Step 5；输出为空表示未配置远程仓库，报告后结束远程阶段，进入 Step 5。
2. 执行 git branch --show-current 并检查退出码：
   - 命令失败时报告原始错误并结束远程阶段，进入 Step 5，不把空输出当作 detached HEAD；
   - 退出码 0 且输出为空表示 detached HEAD；此时必须同时提供 `--remote` 和 `--branch` 才能继续，否则报告原因并结束远程阶段，进入 Step 5。参数完整时设置 LOCAL_REF=HEAD，直接采用参数中的 REMOTE 和 BRANCH，跳过 upstream 查询和目标推断，直接进入 Step 2c；
   - 退出码 0 且非空时记录 CURRENT_BRANCH，设置 LOCAL_REF=CURRENT_BRANCH，继续第 3 项。
3. 仅在 CURRENT_BRANCH 非空时执行 git rev-parse --abbrev-ref --symbolic-full-name '@{u}'：
   - 退出码非 0 且错误信息为 "no upstream configured"（该场景实际退出码为 128）表示没有 upstream，不视为命令失败；
   - 其他失败必须报告并结束远程阶段，进入 Step 5，不得把未知错误当作无 upstream；
   - 有 upstream 时，通过 branch.CURRENT_BRANCH.remote 和 branch.CURRENT_BRANCH.merge 取得明确的 remote 和 branch。

### Step 2c: 解析目标并统一校验

1. 目标优先级，自高到低，命中即停：
   - 同时提供 `--remote` 和 `--branch`：直接使用参数值；
   - 只提供 `--remote`：remote 使用参数值；存在 upstream 且其 remote 与参数相同时沿用 upstream 的 branch，否则（无 upstream 或 upstream 属于其他 remote）使用当前分支名作为远程分支；
   - 只提供 `--branch` 且存在 upstream：使用 upstream 的 remote，以参数值覆盖远程分支；
   - 不提供选项且存在 upstream：使用 upstream 的 remote 和 branch；
   - 只提供 `--branch` 且没有 upstream：远程分支使用参数值；same_repository 模式下只有一个可用 remote 时使用该 remote，否则只询问 remote；
   - 不提供选项且没有 upstream：same_repository 模式下只有一个可用 remote 时，使用该 remote 和当前分支名；其他情况询问目标 remote 和 branch。
   本 skill 由用户显式发起，缺少 upstream 时通过询问取得目标而非直接停止；询问即取得明确目标，不构成猜测。直接采用当前分支名的分支同样有明确依据——用户显式给出的 remote 参数或环境中唯一的 remote——属于确定默认，不是猜测。
2. 询问交互形式：remote 用选项展示——最多 4 个选项，可用 remote 超过 4 个时打印完整列表让用户口述名称；branch 为自由分支名，以口述输入为主，并在选项中提供「使用当前分支名」快捷项（CURRENT_BRANCH 非空时）。用户取消询问时结束远程阶段，进入 Step 5。
3. 形成最终目标前统一校验，两项都必须通过：
   - REMOTE 必须与 git remote 输出中的名称完全一致；不一致时列出全部可用 remote 并终止任务，不自动改用其他 remote；
   - BRANCH 必须通过 git check-ref-format --branch BRANCH 校验；校验失败时报告原始错误并终止任务。
4. 执行 git remote get-url --push REMOTE 检查 push URL；命令失败或输出为空时结束远程阶段，进入 Step 5。
5. 形成本次推送的统一上下文，后续步骤只读取这些字段，不再各自推断或重新解析目标：
   - LOCAL_REF：当前分支名；detached HEAD 时为 HEAD；
   - CURRENT_BRANCH：当前分支名；detached HEAD 时为空；
   - REMOTE、BRANCH：第 1、2 项解析并经第 3 项校验的最终目标；
   - HAS_UPSTREAM：Step 2b 解析时是否存在 upstream；
   - PUSH_MODE：初始为 normal，仅 Step 3b 用户选择强推后改为 force-with-lease；
   - REMOTE_EXISTS、EXPECTED_REMOTE_OID：在 Step 3 填充。
   后续所有 fetch、比较和 push 都使用这些字段拼装明确的 refspec，不依赖 git 默认推断。

## Step 3: 更新远程状态与分叉检测

### Step 3a: 远程分支存在性

1. 执行 git ls-remote --heads REMOTE refs/heads/BRANCH；退出码 0 且输出为空表示远程分支不存在（REMOTE_EXISTS=false），其他失败结束远程阶段，进入 Step 5。
2. 远程分支不存在（首推）：展示推送目标（remote、branch、本地分支、新分支将包含的提交总数与其中的本次新增数）。提交总数用 git rev-list --count LOCAL_REF 计数（远端尚无此分支，推送会带上该 ref 的全部提交；LOCAL_REF 为当前分支名，detached HEAD 时为 HEAD），不用 BRANCH..LOCAL_REF——远端分支不存在时该范围会因无法解析 BRANCH 而失败，或在本地同名分支时恒为 0；本次新增数用 git rev-list --count LOCAL_REF --not --remotes 计数（不存在其他远程分支时等于提交总数）。两条命令任一失败时结束远程阶段，进入 Step 5。展示后使用 AskUserQuestion 确认：
   - 确认推送：进入 Step 4，创建远程分支并设置跟踪关系；
   - 取消：结束远程阶段，进入 Step 5。
3. 远程分支存在（REMOTE_EXISTS=true）时进入 Step 3b。

### Step 3b: 更新远端状态

1. 执行 git fetch REMOTE +refs/heads/BRANCH:refs/remotes/REMOTE/BRANCH；refspec 必须带 `+` 前缀，使 remote-tracking ref 无条件反映远端当前状态——否则远端被强制改写后 fetch 会因 non-fast-forward 拒绝更新，导致恰好在分叉场景下无法取得远端状态，且 --force-with-lease 依赖该 ref 与远端一致。失败时结束远程阶段，进入 Step 5，并报告原始错误。
2. 执行 git rev-parse refs/remotes/REMOTE/BRANCH 取得远端当前提交并记录为 EXPECTED_REMOTE_OID，作为 Step 4 强推路径绑定的期望值；失败时结束远程阶段，进入 Step 5，报告原始错误。
3. 执行 git rev-list --left-right --count REMOTE/BRANCH...LOCAL_REF 取得 behind/ahead 数量（左侧为远端独有提交，右侧为本地独有提交）；失败时结束远程阶段，进入 Step 5。
4. 按结果处理：
   - behind=0 且 ahead>0：快进推送。展示将推送的提交（git log --oneline REMOTE/BRANCH..LOCAL_REF，超过 20 条时截断并报告总数；命令失败时结束远程阶段，进入 Step 5），进入 Step 4；
   - behind=0 且 ahead=0：报告本地与远程已同步，无需推送，进入 Step 5；
   - behind>0：远程冲突预警。分别展示双方独有提交（git log --oneline LOCAL_REF..REMOTE/BRANCH 和 REMOTE/BRANCH..LOCAL_REF，超过 20 条时截断并报告总数；命令失败时结束远程阶段，进入 Step 5），停止自动推送，使用 AskUserQuestion 询问：
     - 放弃推送：报告状态，建议使用专门的拉取、同步或冲突处理流程，进入 Step 5；
     - force-with-lease 强推：明确说明将丢弃远端哪些提交及后果，确认后 PUSH_MODE=force-with-lease，进入 Step 4 强推路径；
   - 用户取消询问时按放弃推送处理。
5. behind>0 时的强推分支同样适用于 ahead=0 的情况（等于回退远端），说明文案必须如实反映丢弃的提交。
6. 本步骤是唯一取得远端状态的位置。由 Step 4c 第 3 项返回时，重新执行本步骤全部内容；重新到达任何推送执行点前，必须向用户重新展示最新状态并获得确认——首推确认与强推确认不因曾经确认过而豁免，不自动再次推送。

## Step 4: 执行推送

### Step 4a: 构建推送命令

不得拼接 shell 命令字符串；使用进程参数数组，refspec 中的变量按实际值展开，不解释用户输入中的特殊字符。

- 常规路径（PUSH_MODE=normal）：git push REMOTE LOCAL_REF:BRANCH；LOCAL_REF 为当前分支名，detached HEAD 时为 HEAD；
- 无 upstream 且 LOCAL_REF 为当前分支名时追加 --set-upstream——目标已由参数或询问明确确定，设置 upstream 不构成猜测；该规则同时适用于常规与强推路径；detached HEAD 无法设置 upstream，跳过并在报告中说明；
- 强推路径（PUSH_MODE=force-with-lease）：git push --force-with-lease=refs/heads/BRANCH:EXPECTED_REMOTE_OID REMOTE LOCAL_REF:BRANCH。EXPECTED_REMOTE_OID 必须取自 Step 3b 第 2 项；缺失或不可用时不得改用不带绑定的 --force-with-lease，返回 Step 3 重新检测。仅此一种强推路径，永不使用裸 --force，不提供 --force 选项。

### Step 4b: Pre-push 检查

1. 执行 git rev-parse --git-path hooks 取得实际 hooks 目录并检查退出码；失败时结束远程阶段，进入 Step 5，报告原始错误。相对路径按仓库根目录解析。
2. 读取 git config --get core.hooksPath；退出码为 1 表示未配置，其他失败必须结束远程阶段，进入 Step 5，报告原始错误；若存在，以实际配置为准。
3. 检查实际 hooks 目录中的 pre-push 文件；检测到时告知用户推送会自动运行该 hook，不因此重复询问。

### Step 4c: 执行与失败处理

1. 执行推送并检查退出码。
2. 成功后执行 git log --oneline -n 1 并检查退出码，报告推送结果；该命令失败时如实说明推送已成功但日志读取失败，不视为推送失败。CURRENT_BRANCH 非空时，再执行 git rev-parse --abbrev-ref --symbolic-full-name '@{u}' 取得跟踪关系；退出码非 0 且错误信息为 "no upstream configured" 时跳过跟踪关系报告，不视为失败，其他失败必须报告。detached HEAD 不执行 upstream 查询，直接说明此次推送未设置跟踪关系。
3. 非零退出时按类型处理：
   - 认证、权限、网络和 hook 错误：显示原始错误，不自动重试，结束远程阶段，进入 Step 5；
   - 非快进被拒（远端在检测后再次变化），或强推因 lease 过期被拒（"stale info"、cannot lock ref 等）：不自动重试，返回整个 Step 3——从 3a 的 ls-remote 起重新确认远程分支存在性与状态，远端分支可能已被删除或重建——按最新状态重新展示预警并等待用户选择，重新确认前不执行任何推送；
   - 其他错误：显示原始错误并结束远程阶段，进入 Step 5。

## Step 5: 推送后报告

根据最新 git status 和推送结果报告：

- 是否推送成功、推送了哪些提交、目标 remote 和分支；
- upstream 是否已设置或新建跟踪关系；本次推送目标与现有 upstream 不一致时，明确说明未更改跟踪关系；
- detached HEAD 时说明未设置 upstream 的原因；
- 是否放弃推送、远端分叉状态，以及建议的后续流程；
- 已同步时明确报告无需操作；
- 工作区是否存在未提交或未跟踪更改（不随本次推送上传）。

## 委托契约（Call contract）

本 skill 是分支级远程写入的唯一实现入口：

- 其他 skill（git-commit 的提交后同步，未来的 git-pr、git-sync）需要推送分支时，一律委托本 skill 执行远程写入，不自行复制 fetch、分叉检测、upstream 设置与强推判定逻辑；
- 委托方式：以 /git-push 形式调用并传入目标参数；省略参数时由本 skill 按 Step 2 规则解析目标。本 skill 的规则快照、目标解析、确认询问与失败处理照常执行；
- 委托方只消费最终报告（推送结果、跟踪关系、放弃或失败原因），不复制本 skill 的判定逻辑；
- 委托执行即视为遵循有效远程策略（git-rules Call contract 的 delegate remote write）。

## 错误处理原则

以下约束适用于全流程，不被任何分支覆盖：

- 认证、权限、网络和 hook 错误：显示原始错误和结果，不自动重试；
- 任何内部规则初始化、修复或写入失败：停止当前任务，不继续执行依赖该规则的动作；
- 远程状态无法证明可安全推送（behind>0 且用户未选择强推）时停止推送，不自动执行 git pull、merge、rebase 或任何改写本地历史的操作；
- 强推仅限 --force-with-lease 且必须经用户明确选择，永不使用裸 --force；
- 用户取消询问时停止依赖该询问的后续动作：规则询问（/git-rules ask）取消终止当前任务；远程目标与推送确认类询问取消则结束远程阶段，进入 Step 5。

出口词汇与报告义务（全文统一）：

- 终止任务：报告原因后整个流程停止，不进入 Step 5。仅用于 Step 1 的环境与配置失败、Step 2a 参数语法错误、Step 2c 目标校验失败和规则询问取消；
- 结束远程阶段：停止远程操作，进入 Step 5 产出报告。用于其后所有出口——无远程仓库、detached HEAD 缺少必要选项、upstream 查询未知错误、目标询问取消、push URL 缺失、fetch/ls-remote/比较命令失败、推送失败及放弃推送；
- 其余错误行为已在对应步骤按上述词汇声明。
