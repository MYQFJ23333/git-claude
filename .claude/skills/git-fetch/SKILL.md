---
name: git-fetch
description: fetch 原语 - 机械更新 remote-tracking 引用（全分支或单分支），按 repository_relation_mode 挂钩 prune，检测远端历史改写信号，输出标准化返回契约供任务流委托，也可由用户直接调用
argument-hint: "[--remote REMOTE] [--branch BRANCH] [--force] [--prune | --no-prune]"
license: MIT
---

# git-fetch

原语层 fetch 原语：机械更新 remote-tracking ref，无流程停机点；既作为其他技能内部 fetch 的统一委托目标，用户也可直接调用。不做分叉判定策略化（behind/ahead、祖先关系属调用方）、push、upstream 设置、更新本地分支（refs/heads/*）、pull，除配置初始化外的任何询问。

## Step 1: 环境检查与规则加载

所有命令都必须检查退出码，不把空输出当作成功——空输出只有在对应步骤明确定义为合法结果（如 detached HEAD、无 upstream、未配置自定义 refspec）时才可采信。命令失败时立即停止：Step 1 内的失败终止任务；进入 Step 2 之后的失败按「错误处理原则」定义的出口词汇处理。

执行 `.claude/skills/git-rules/references/blocks/task-init.md`——初始化步骤（git 环境与操作状态检查、配置快照）与「本任务已执行则跳过」规则均在块内；本技能所需 value：`/rules/project/repository_relation_mode/value`。块文件缺失、为空或不可读时终止任务，不自动创建或猜测块内容。块检查失败或存在未完成 Git 操作时终止任务。

工作区存在未提交更改不阻止 fetch（fetch 不改工作区与本地分支），但必须在最终报告中说明。

在解析 fetch 目标前验证缓存值，只接受 unconfigured、independent_repositories 或 same_repository；规则缺失或 value 非法时，按 task-init 块内修复机制内部调用 `/git-rules repair project repository_relation_mode --internal`。

初始化完成并验证缓存值后，计算本次有效 MODE：

- unconfigured：进入 Step 2 前先向用户说明两种模式的含义，调用 /git-rules ask project repository_relation_mode，按返回的 effective_action 重新计算。配置初始化询问是原语层「无停机点」的唯一例外；
- same_repository：prune 默认开启——远端删除视为统一管理下的正常同步事件，报告列出被清理引用；
- independent_repositories：prune 默认关闭，需显式 --prune 开启。

所有 /git-rules ask 调用都必须先检查返回的 status：selected 时才使用 effective_action；cancelled 时终止当前任务；error 时报告规则询问或写回失败并终止，不得继续执行依赖该选择的动作。

## Step 2: 解析 fetch 目标

从 $ARGUMENTS 读取原始参数。不得通过 shell eval 解释用户输入；参数中的引号、反斜杠、美元符号和反引号不参与任何命令拼接。

### Step 2a: 参数语法解析

参数格式为 `[--remote REMOTE] [--branch BRANCH] [--force] [--prune | --no-prune]`，全部选项均可省略且顺序不限：

- 不提供选项：全分支 fetch，prune 取规则默认，目标 remote 在 Step 2c 解析；
- 只提供 `--remote`：覆盖目标 remote，全分支 fetch；
- 只提供 `--branch`：只 fetch 指定分支，remote 在 Step 2c 解析；
- `--force`：refspec 加 `+` 前缀（强制同步 tracking ref）；默认仅快进；
- `--prune` / `--no-prune`：覆盖规则默认的 prune 策略，两者互斥。

`--remote` 和 `--branch` 各自最多出现一次，并且必须紧跟一个非空值；`--force`、`--prune`、`--no-prune` 为无值开关，不得携带值。出现未知选项、位置参数、重复选项、`--prune` 与 `--no-prune` 同现或缺少值时，报告用法并终止任务，不猜测用户意图。不兼容旧的位置参数格式。

本步骤只解析语法，不读取仓库状态，也不校验 remote 名称或分支名格式——最终校验在 Step 2c 统一执行。

### Step 2b: 读取仓库状态

1. 执行 git remote 并检查退出码；失败时报告原始错误并终止任务；输出为空表示未配置远程仓库，报告后终止任务。
2. 执行 git branch --show-current 并检查退出码：
   - 命令失败时报告原始错误并终止任务，不把空输出当作 detached HEAD；
   - 退出码 0 且输出为空表示 detached HEAD；fetch 不依赖当前分支，不构成 fetch 障碍，跳过 upstream 查询；
   - 退出码 0 且非空时记录 CURRENT_BRANCH，继续第 3 项。
3. 仅在 CURRENT_BRANCH 非空时执行 git rev-parse --abbrev-ref --symbolic-full-name '@{u}'：
   - 退出码 128 且错误信息为 "no upstream configured" 表示没有 upstream，不视为命令失败；
   - 其他失败必须报告并终止任务，不得把未知错误当作无 upstream；
   - 有 upstream 时记录其 remote，供 Step 2c 目标解析使用。
4. REMOTE 解析后（见 Step 2c 第 2 项）执行 git config --get-all remote.REMOTE.fetch：
   - 退出码 0 且输出非空：记录 REMOTE_FETCH_CONFIGURED=true（用户自定义跟踪范围）；
   - 退出码 1：未配置，记录 REMOTE_FETCH_CONFIGURED=false，属合法结果；
   - 其他失败：报告原始错误并终止任务。

### Step 2c: 解析目标并统一校验

1. 目标优先级，自高到低，命中即停：
   - 提供 `--remote`：直接使用参数值；
   - 无参数但存在 upstream：使用 upstream 的 remote；
   - 仓库仅有一个 remote：自动采用（确定默认，非猜测）；
   - 其余情况：终止任务并列出全部可用 remote，要求显式 `--remote`。本原语不做目标询问，委托方一律传显式参数，不会落入本分支。
2. 形成最终目标前统一校验，两项都必须通过：
   - REMOTE 必须与 git remote 输出中的名称完全一致；不一致时列出全部可用 remote 并终止任务，不自动改用其他 remote；
   - `--branch` 给出时必须通过 git check-ref-format --branch BRANCH 校验；校验失败时报告原始错误并终止任务。
   校验通过后执行 Step 2b 第 4 项的 git config --get-all remote.REMOTE.fetch。
3. 形成本次 fetch 的统一上下文，后续步骤只读取这些字段，不再各自推断或重新解析目标：
   - REMOTE：第 1 项解析并经第 2 项校验的最终目标；
   - BRANCH：参数值；为空表示全分支 fetch；
   - FETCH_MODE：fast-forward（默认）或 force（--force）；
   - PRUNE_ENABLED：显式 `--prune` / `--no-prune` 优先，否则取规则默认（same_repository 开、independent_repositories 关）；
   - MODE：Step 1 计算的有效模式；
   - REMOTE_FETCH_CONFIGURED：Step 2b 第 4 项结果。
   后续所有 fetch 命令都使用这些字段拼装明确的 refspec，不依赖 git 默认推断。

## Step 3: 构造 refspec 并执行 fetch

### Step 3a: 构造 refspec

使用进程参数数组，不拼接 shell 命令字符串，不解释用户输入中的特殊字符。全部显式构造、不依赖 git 默认推断：

- 有 BRANCH：refspec 为 refs/heads/BRANCH:refs/remotes/REMOTE/BRANCH，FETCH_MODE=force 时加 `+` 前缀；
- 无 BRANCH 且 REMOTE_FETCH_CONFIGURED=false：refspec 为 refs/heads/*:refs/remotes/REMOTE/*，同上决定 `+`；
- 无 BRANCH 且 REMOTE_FETCH_CONFIGURED=true：尊重用户配置，不传 refspec、直接 git fetch [--prune] REMOTE——配置是显式意图而非推断；此时配置自带的 `+` 不受 FETCH_MODE 控制，报告中注明。

默认仅快进（不传 `+`）：non-fast-forward 错误即「远端历史被改写」的检测信号，改写是异常，应显式暴露而非静默吸收。--force 服务于需要远端真实 OID 的场景（如 lease 检查的委托方）。

### Step 3b: 执行与失败分类

1. 执行 fetch 前快照：git for-each-ref --format='%(refname) %(objectname)' refs/remotes/REMOTE 并检查退出码；失败时终止任务并报告原始错误（结果枚举无法进行）。
2. 执行 git fetch [--prune 当 PRUNE_ENABLED] REMOTE [refspec] 并检查退出码；refspec 按 Step 3a 构造，尊重配置模式时不传。
3. 按退出码与 stderr 分类：
   - 退出码 0：进入 Step 4；
   - 仅快进模式（FETCH_MODE=fast-forward）下 non-fast-forward 被拒（退出码非 0 且 stderr 含 "updates were rejected" 或 "non-fast-forward" 判定特征）：不作为笼统失败——记录 REWRITE_SIGNAL=true 与被拒 ref，其余 ref 可能已正常更新，进入 Step 4 汇总报告；
   - 认证、权限、网络错误：显示原始错误，不自动重试，终止任务；
   - 其他错误（含 `--branch` 指向远端不存在的分支时的 "couldn't find remote ref"）：显示原始错误，终止任务。

## Step 4: 机械检查与报告

1. 结果枚举：执行 fetch 后快照（与 Step 3b 第 1 项同命令）并检查退出码，与前快照对比得出：
   - UPDATED：OID 变化的 ref（old → new）；
   - NEW：新增 ref；
   - PRUNED_REFS：消失的 ref（prune 结果）。
   后快照命令失败时终止任务并如实报告「fetch 已执行完成，但结果枚举失败」——fetch 本身的成功不被掩盖，契约字段缺失也不编造。
2. force 模式的改写可见性：FETCH_MODE=force 时，对 UPDATED 中每个 ref 执行 git merge-base --is-ancestor OLD NEW 并检查退出码；退出码 1 = 非快进更新（远端历史已改写），列入改写信号报告——--force 同步了状态，但改写事件仍须对用户可见；其他失败终止任务并报告原始错误。
3. 报告（即返回契约）：
   - 面向用户：REMOTE、FETCH_MODE、PRUNE_ENABLED 与 PRUNED_REFS 清单、UPDATED/NEW（超过 20 条时截断并报告总数）、改写信号（仅快进模式下被拒清单并建议显式 `--force`；或 force 模式下非快进更新清单）、工作区未提交更改说明、无更新时明确报告「已是最新」；尊重配置模式时注明配置自带的 `+` 不受 FETCH_MODE 控制；PRUNED_REFS 中某 ref 的 OID 与当前 HEAD OID 相同时注明（detached 检出于该 ref 时删除 tracking ref 不影响已检出的工作区）；
   - 面向委托方的契约字段（在任务上下文中保留）：REMOTE、FETCHED_SCOPE（all | BRANCH）、NEW_OID（单分支时）、UPDATED、PRUNED_REFS、REWRITE_SIGNAL、FETCH_MODE、PRUNE_ENABLED。

## 返回契约（委托方视角）

- 委托方式：以 /git-fetch 形式调用并传入显式参数；委托方只消费 Step 4 定义的契约字段，不复制 refspec 构造、失败分类或改写判定逻辑，也不重新 rev-parse 取 OID（避免重复命令与 TOCTOU）。
- 委托 fetch ≠ 委托分叉判定：behind/ahead（rev-list --left-right --count）、祖先关系检查、推送决策、确认询问一律留在调用方。
- 同任务内重复调用（委托链中多次 fetch）：本技能无跨调用状态，每次独立执行；task-init 跳过语义按块内通用机制执行。

## 错误处理原则

出错后执行 `.claude/skills/git-rules/references/blocks/error-principles.md`——跨技能错误处理不变量在块内，加载技能时不预读；块文件缺失、为空或不可读时终止任务，不自动创建或猜测块内容。

出口词汇（全文统一）：

- 终止任务：报告原因后整个流程停止，不产出 Step 4 契约。用于 Step 1 的环境与配置失败、Step 2a 参数语法错误、Step 2b 仓库状态读取失败、Step 2c 目标解析与校验失败、规则询问取消或写回失败、Step 3b 前快照失败与认证/权限/网络及其他错误、Step 4 后快照或复检失败；
- 正常结束：Step 4 完整报告（含返回契约）。仅快进模式 non-fast-forward 不属于终止——记录 REWRITE_SIGNAL 后仍正常结束。

原语层不存在 git-push 的「结束远程阶段」中间态。
