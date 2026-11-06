# git-pull 概念设计

状态：概念设计（已定稿，待实施） · 归属：任务流层（版本由用户维护） · 上游方案：`scratch/notes/remote-repository-policy.md`（关闭「pull 自动化边界未定」开放项）；命名空间区分见 `scratch/designs/release-flow-generic-skills-design.md` 的 git-sync 定义

## 1. 定位

任务流层拉取流程：当前分支从其 upstream（或显式指定目标）拉取更新——委托 fetch 取远端真实状态，做分叉分析，按可配置策略执行快进或合并提交，聚焦本地更新（读入方向），不含 push、不含 rebase。

| 做 | 不做 |
| --- | --- |
| 委托 `/git-fetch --force` 取远端状态、消费返回契约、分叉分析（behind/ahead）、严格落后时无条件快进、分叉时按 `pull_strategy` 合并（merge commit）、`auto_stash` 编排（更新前 stash、成功后 pop）、冲突停机协议、两级出口报告 | rebase（历史改写，留给未来 `/git-rebase` 原语）、push（报告中建议用户运行 `/git-push`）、upstream 设置、stash 冲突自动恢复（保留 stash 如实报告即止）、detached HEAD 支持、远程写入、基准分支 merge 回流（属 git-sync） |

与相近技能/命令的根本区别：

| 对比对象 | 区别 |
| --- | --- |
| `/git-fetch` | 原语层：只更新 remote-tracking ref，不写 `refs/heads/*`、不判定分叉策略、不合并。git-pull 是其委托方，分叉判定与本地更新留在 git-pull（「委托 fetch ≠ 委托分叉判定」）。 |
| `/git-push` | 写入方向（本地 → 远程），持有 force-with-lease 与 upstream 设置；git-pull 是读入方向（远程 → 本地），两者共同构成 push 分叉预警文案指向的「专门的拉取、同步或冲突处理流程」缺口的两侧。 |
| `/git-sync`（未来，release-flow 已定稿） | git-sync 是**显式基准分支 merge 回工作分支**（发布后回流，`--work`/`--base`，目标可与当前分支无关）；git-pull 是**当前分支从其 upstream 拉取**。git-sync 名已被占用且语义不同，git-pull 不得复用该名。 |
| `/git-merge`（未来原语） | 单命令合并 + 机械检查、无停机点。git-pull v1 自己执行 merge 命令，Step 4 预留委托接缝（同 git-push 后来切换委托 git-fetch 的先例）。 |
| 裸 `git pull` | 裸 pull 无策略化：默认行为取决于用户 git 配置（可能是 rebase）、无分叉停机协议、无 stash 编排、无改写信号拦截；git-pull 是显式安全协议（`--force` fetch 取真实状态 + REWRITE_SIGNAL 拦截 + fast-forward/merge 显式策略 + 冲突停机）。 |

**准入论证（不可或缺性）**：按准入标准把基础技能与现有技能视为可自由编排的原子步骤——`/git-fetch` 委托 + 执行 merge 命令本身可由编排临时组合，但以下安全协议不可由编排临时复现：① fetch 委托的 `--force` 与契约消费绑定（否则历史改写时旧 tracking ref 会让 behind/ahead 判定基于过期 OID，分叉分析失去意义）；② REWRITE_SIGNAL 拦截（强制 fetch 后仍停在 pull 层，防止把改写历史静默并入本地分支）；③ 分叉策略询问（inline_persistence 四选一的 ask 协议与写回）；④ auto_stash 与 merge 冲突的停机编排（何时 pop、何时保留、何时恢复工作区）。这些与 git-push 的论证同构（分叉检测、force-with-lease、managed_safe_sync 不可临时复现）。普遍性同时满足：套装完整安装前提下适用于任何 Git 管理方案的项目，差异由参数与规则吸收。git-push 分叉预警文案已将用户指向「专门的拉取、同步或冲突处理流程」——git-pull 填补此缺口。

**detached HEAD 评估（v1 直接终止任务）**：git-pull 的核心语义是「更新当前分支引用」，detached HEAD 无分支可更新、无 upstream 语义；`--branch` 覆盖的是**远程分支**而非本地检出，不提供「创建/切换分支」的路径。支持 detached 需要引入未设计的分支管理语义，超出 v1 范围。故 Step 2b 检出 detached 时报告原因并**终止任务**（归类于目标解析失败——这是状态级不兼容，非可部分完成的操作；与 git-push 对 detached 的宽容处理不同，因 push 有定义良好的 `LOCAL_REF=HEAD` 路径）。

## 2. 参数接口

```text
[--remote REMOTE] [--branch BRANCH]
```

| 参数 | 形式 | 默认 | 约束 |
| --- | --- | --- | --- |
| `--remote` | 单值 | 按 Step 2c 优先级解析 | 必须与 `git remote` 输出完全一致 |
| `--branch` | 单值 | 按 Step 2c 优先级解析 | `git check-ref-format --branch` 通过；**覆盖的是远程分支**，本地始终为当前分支 |

解析规则（严格对齐 git-push Step 2a）：每个选项至多出现一次且必须紧跟非空值；未知选项、位置参数、重复、缺值 → 报告用法并终止任务，不猜测意图。不提供 shell eval，引号/反斜杠/美元符号/反引号不参与任何命令拼接。本步只解析语法，校验在 Step 2c 统一执行。

不提供 `--strategy` / `--stash` 参数（v1 非目标）：策略与 stash 行为一律走规则，避免参数面与规则面双通道漂移。

## 3. 规则集成

### 3.1 task-init value 清单（3 条 JSON Pointer）

```text
/rules/project/repository_relation_mode/value
/rules/git-pull/pull_strategy/value
/rules/git-pull/auto_stash/value
```

- `repository_relation_mode`：**不用于计算远程写策略**（git-pull 不写远程、不受 managed_safe_sync 管辖，见第 6 节）。缓存它的唯一目的是在委托 `/git-fetch` 前完成配置初始化——`unconfigured` 时在 Step 1 说明两种模式含义并 `ask project repository_relation_mode`（initialize），使委托路径上不触发 git-fetch 的配置初始化停机点（对齐 git-push Step 1 先例）。校验白名单：`unconfigured` / `independent_repositories` / `same_repository`；缺失或非法 → `/git-rules repair project repository_relation_mode --internal`，成功后只重读一次，失败即停。
- 两条 git-pull 规则：只在实际使用时验证（task-init 缓存但不预验）；缺失或 value 非法 → `/git-rules repair git-pull <key> --internal`，成功后丢弃快照一次性重读，仍不可用则停。

### 3.2 canonical 规则表新增行（git-rules/SKILL.md 表尾 +2 行）

| scope | key | 默认 value | options（value=标签） | 控制值 | ask_mode | description |
| --- | --- | --- | --- | --- | --- | --- |
| git-pull | pull_strategy | prompt | prompt=每次询问、fast_forward=总是快进、merge=总是合并 | prompt | inline_persistence | 分叉（本地与远端均有独有提交）时的更新策略：快进策略遇分叉停止更新，合并策略创建合并提交；严格落后一律快进 |
| git-pull | auto_stash | prompt | prompt=每次询问、always=总是、never=从不 | prompt | inline_persistence | 工作区有未提交更改时，更新前是否 stash、更新成功后是否 pop |

option token 依据：`auto_stash` 的 `prompt/always/never` 与 `post_commit_push_mode` 完全一致；`pull_strategy` 以 `prompt` 对齐 `empty_staging_mode` 先例（options 含 prompt、控制值同名、set 可写回「每次询问」），持久动作值 `fast_forward` / `merge` 是行为名——标签「总是」承载 always 语义，token 承载行为语义。

### 3.3 inline_persistence 四选项映射表

**`git-pull/pull_strategy`**（问题：「本地与远端已分叉（双方各有独有提交），如何更新当前分支？」）

| 展示选项 | 选项说明 | effective_action | stored_value |
| --- | --- | --- | --- |
| 仅本次快进 | 本次停止更新，不合并、不改写历史 | fast_forward | 空 |
| 仅本次合并 | 本次创建合并提交更新当前分支 | merge | 空 |
| 总是快进 | 本次停止更新，以后分叉时一律停止更新 | fast_forward | fast_forward |
| 总是合并 | 本次创建合并提交，以后分叉时一律合并 | merge | merge |

（「仅本次快进」的 effective_action = fast_forward = 遇分叉停止，与持久 fast_forward 行为一致，只是不写回。）

**`git-pull/auto_stash`**（问题：「工作区有未提交更改，更新前是否暂存？」）

| 展示选项 | 选项说明 | effective_action | stored_value |
| --- | --- | --- | --- |
| 是 | 仅本次更新前 stash、成功后 pop | stash | 空 |
| 否 | 仅本次不 stash，直接更新 | skip | 空 |
| 总是 | 本次及以后更新前 stash、成功后 pop | stash | always |
| 从不 | 本次及以后均不 stash，直接更新 | skip | never |

**需创建的 ask 映射文件（2 个，实施时）**：

- `.claude/skills/git-rules/references/ask/git-pull/pull_strategy.md`
- `.claude/skills/git-rules/references/ask/git-pull/auto_stash.md`

格式对齐 `references/ask/git-commit/post_commit_push_mode.md` 先例：头四行元数据（scope / key / ask_mode / 控制值）+ 单问题 + 恰好 4 行选项表 + 尾注（stored_value=空 表示不写回；不得把 prompt 作为选项或 effective_action 返回）。

## 4. 步骤流（5 步，与未来 SKILL.md 同构）

开头声明（同 git-push Step 1）：所有命令检查退出码，不把空输出当作成功（空输出仅在明确定义为合法结果处可采信）；Step 1 内的失败终止任务，Step 2 起的失败按第 5 节出口词汇处理。

### Step 1: 环境检查与规则快照

1. 执行 task-init 块调用（引用行 + 第 3.1 节 value 清单 + 「块文件缺失、为空或不可读时终止任务，不自动创建或猜测块内容」声明）；块检查失败或存在冲突 / 未完成 merge / rebase / cherry-pick / revert 时终止任务。
2. 工作区存在未提交更改**不阻止 pull**（与 fetch/push 先例一致；stash 策略在 Step 4 编排；此为本技能特有语义，不并入块）。
3. 验证 `repository_relation_mode` 白名单 → 非法走 repair；`unconfigured` → 说明含义 + ask + status 检查（见第 3.1 节）。两条 git-pull 规则此时只缓存不验证。
4. 所有 `/git-rules ask` 调用先检查 `status`：`selected` 才用 `effective_action`；`cancelled` / `error` 报告并**终止任务**。

### Step 2: 解析拉取目标

- **2a 参数语法解析**：按第 2 节规则；本步只解析语法，不读仓库状态、不校验名称。
- **2b 读取仓库状态**：
  1. `git remote`：失败 → 终止并报告原始错误；输出为空 → 终止并报告「未配置远程仓库」。
  2. `git branch --show-current`：失败 → 终止（不把空输出当作 detached）；退出码 0 且输出为空 = detached HEAD → 报告原因（见第 1 节评估）并**终止任务**；非空记 CURRENT_BRANCH。
  3. `git rev-parse --abbrev-ref --symbolic-full-name '@{u}'`：退出码 128 且含 "no upstream configured" 判定特征 = 无 upstream（合法，非失败）；其他失败 → 终止并报告，不得把未知错误当作无 upstream。有 upstream 时通过 `git config --get branch.CURRENT_BRANCH.remote` 与 `git config --get branch.CURRENT_BRANCH.merge` 取得明确 remote 和 branch（**不从 @{u} 输出按斜杠切分**）。
- **2c 解析目标并统一校验**：
  1. 优先级（自高到低，命中即停；语义对齐 git-push Step 2c，`--branch` 一律覆盖远程分支）：① 双参数 → 直接使用；② 仅 `--remote`：upstream 存在且其 remote 相同则沿用 upstream branch，否则用当前分支名；③ 仅 `--branch` 且有 upstream：upstream remote + 参数 branch；④ 无参数且有 upstream：upstream 对；⑤ 仅 `--branch` 无 upstream：仓库仅一个 remote 时自动采用，否则**询问 remote**（选项展示，超 4 个打印列表口述）；⑥ 无参数无 upstream：仅一个 remote 时用该 remote + 当前分支名，否则**询问 remote 和 branch**（branch 口述为主，选项含「使用当前分支名」）。用户取消询问 → 终止任务。
  2. 统一校验：REMOTE 必须与 `git remote` 输出完全一致（不一致列出可用 remote 并终止任务）；BRANCH 经 `git check-ref-format --branch`（失败报告原始错误并终止任务）。
  3. 形成统一上下文（后续步骤只读字段，不再各自推断）：LOCAL_BRANCH（= CURRENT_BRANCH）、REMOTE、BRANCH（校验后的最终目标）、HAS_UPSTREAM、STRATEGY_RAW、AUTO_STASH_RAW（缓存的规则原始值，使用时验证并经 ask 解析为 effective 值）。

### Step 3: 委托 fetch 与分叉分析

1. **委托**：以 `/git-fetch` 形式调用 `/git-fetch --remote REMOTE --branch BRANCH --force`（一行调用 + 声明只消费契约字段）。`--force` 的依据：取远端**真实**状态——否则远端历史改写时旧 tracking ref 会让 behind/ahead 判定基于过期 OID（与 git-push lease 绑定使用 `--force` 的理由同构）。单分支委托下 `REWRITE_SIGNAL` 必然关乎目标分支。git-fetch 终止（含其内部错误或用户取消）时，报告其原因并结束更新阶段进入 Step 5。只消费契约字段 `NEW_OID` / `FETCHED_SCOPE` / `UPDATED` / `PRUNED_REFS` / `REWRITE_SIGNAL`，不复制 refspec 构造、不重新 rev-parse 取 OID。
2. **契约完整性**：NEW_OID 缺失或不可用 → 报告契约不完整，结束更新阶段，不自行补取。
3. **REWRITE_SIGNAL=true → 停止**（安全立场，明确声明）：强制 fetch 已把 tracking ref 同步到改写后的远端历史，此时快进或合并等于**把改写历史静默并入本地分支**。处理：展示契约 `UPDATED` 中该 ref 的 old → new 与改写说明，不执行任何本地分支更新，建议用户审查远端历史后显式重跑 `/git-pull` 或改用专门流程，结束更新阶段。
4. **远端分支已删除**：NEW_OID 为空，或 `PRUNED_REFS` 包含 `REMOTE/BRANCH` → 报告远端分支已删除（列出 PRUNED_REFS 中相关项），无更新可做，建议用户需要时用 `/git-push` 重建或清理本地分支，结束更新阶段。
5. **分叉分析**：`git rev-list --left-right --count REMOTE/BRANCH...LOCAL_BRANCH`（左侧远端独有 = BEHIND，右侧本地独有 = AHEAD）；失败 → 结束更新阶段。同时采集将并入的提交清单 `git log --oneline LOCAL_BRANCH..REMOTE/BRANCH`（超 20 条截断报总数；失败 → 结束更新阶段）。
6. **四分支处理**：
   - **BEHIND=0 且 AHEAD=0**：已同步，结束更新阶段（明确报告无需操作）。
   - **BEHIND=0 且 AHEAD>0**（本地领先）：无需拉取；报告领先提交（`git log --oneline REMOTE/BRANCH..LOCAL_BRANCH`，截断规则同上），建议需要时运行 `/git-push`，结束更新阶段。
   - **BEHIND>0 且 AHEAD=0**（严格落后）：**无条件快进**，不询问（策略只作用于分叉——快进策略在此同样快进），进入 Step 4。
   - **BEHIND>0 且 AHEAD>0**（分叉）：解析 STRATEGY——值为 `prompt` 时调用 `/git-rules ask git-pull pull_strategy`（status 检查同 Step 1）；`fast_forward` → 展示双方独有提交（两条 log，截断规则同上），报告「快进策略遇分叉停止更新」，建议改用合并策略（`/git-rules set git-pull pull_strategy merge` 或下次询问时选择）或专门冲突处理流程，结束更新阶段；`merge` → 进入 Step 4。

### Step 4: 执行更新

预留接缝：本节 merge 命令的执行与机械检查未来可切换为委托 `/git-merge` 原语（同 git-push 切换委托 git-fetch 的先例）；v1 内联执行。

1. **解析 stash 行为**：仅当工作区存在未提交更改（`git status --porcelain` 非空）时处理 AUTO_STASH——`always` → stash；`never` → 不 stash；`prompt` → `/git-rules ask git-pull auto_stash`（status 检查同 Step 1）。需要 stash 时执行 `git stash push -m "git-pull auto stash before updating CURRENT_BRANCH from REMOTE/BRANCH"` 并检查退出码，成功记 STASHED=true；**失败 → 报告原始错误，结束更新阶段**（更新未发生，本地分支未动）。
2. **快进路径**（BEHIND>0, AHEAD=0）：`git merge --ff-only REMOTE/BRANCH` 并检查退出码。
   - 成功：UPDATE_KIND=fast_forward，MERGE_OID=NEW_OID（快进无新合并提交），进入第 4 项。
   - 非零：属竞态或工作区更改被拒（分析后状态已变化）。**若 STASHED=true 且本地分支未发生任何变化，先执行 `git stash pop` 恢复工作区**（失败恢复路径，非「更新成功后 pop」；pop 失败 → stash 保留、如实报告）；然后展示原始错误，结束更新阶段（UPDATE_KIND=failed）。
3. **合并路径**（分叉）：`git merge REMOTE/BRANCH`（git 默认合并消息，机械操作不走 `/git-commit`——对齐 git-sync 先例）并检查退出码。
   - 成功：UPDATE_KIND=merge，`git log --oneline -n 1` 记 MERGE_OID（读取失败时如实说明合并已成功但日志读取失败，不视为合并失败），进入第 4 项。
   - **冲突（非零退出，MERGE_HEAD 存在）→ 冲突停机协议**：不自动 continue / abort / reset / rebase；**不 pop stash**（工作区已冲突，pop 会加剧混乱）；报告冲突文件清单（`git diff --name-only --diff-filter=U`）、UPDATE_KIND=merge_conflict、stash 仍在栈中及指引（解决 merge 冲突后手动 `git stash pop`），结束更新阶段。
4. **更新成功后的 stash 恢复**（STASHED=true 时）：`git stash pop` 并检查退出码。
   - 成功：STASH_POP=applied。
   - **冲突或失败：stash 保留、如实报告、停止**（不自动继续、不自动 resolve）：报告更新已成功、pop 失败的原始错误与冲突文件（如有）、stash 条目仍在栈中及手动处理指引，STASH_POP=failed，结束更新阶段。
5. 不执行任何 push；报告中建议用户需要时运行 `/git-push`。

### Step 5: 更新后报告

根据本次结果与最新状态报告：

- 是否更新、更新方式（快进 / 合并提交，MERGE_OID）、并入的提交（Step 3 采集的清单）；
- 目标 REMOTE / BRANCH 与 upstream 关系（本次目标与 upstream 不一致时明确说明未更改跟踪关系）；
- 各类停止原因与建议后续流程（REWRITE_SIGNAL、远端分支删除、快进策略遇分叉、快进/合并失败、merge 冲突、stash 失败）；
- 本地领先时建议 `/git-push`；
- 委托 fetch 顺带清理的 remote-tracking ref（PRUNED_REFS 非空时列出）；
- stash 状态四态（未发生 / applied / failed 保留 / merge 冲突时保留）及对应指引；
- 工作区未提交或未跟踪更改现状。

### 统一上下文对象字段清单（Step 2c 形成，Step 3–5 只读）

LOCAL_BRANCH、CURRENT_BRANCH、REMOTE、BRANCH、HAS_UPSTREAM、STRATEGY_RAW（→ Step 3 分叉分支解析为 STRATEGY_EFFECTIVE）、AUTO_STASH_RAW（→ Step 4 解析为 AUTO_STASH_EFFECTIVE）、NEW_OID、FETCHED_SCOPE、UPDATED、PRUNED_REFS、REWRITE_SIGNAL、BEHIND、AHEAD、INCOMING_LIST、UPDATE_KIND（fast_forward | merge | merge_conflict | up_to_date | local_ahead | remote_deleted | rewritten_stop | ff_stop | failed）、STASHED、STASH_POP（applied | failed | skipped）、MERGE_OID。

## 5. 失败出口

两级出口词汇（对齐 git-push）：

- **终止任务**：报告原因后整个流程停止，不进入 Step 5。用于：Step 1 环境与配置失败、2a 参数语法错误、2b 状态读取失败（无 remote、detached HEAD、upstream 查询未知错误）、2c 校验失败与目标询问取消、任何 `/git-rules ask` 返回 `cancelled` / `error`。
- **结束更新阶段（进入报告）**：停止更新操作，进入 Step 5 产出报告。用于 Step 3 起所有出口：fetch 委托失败、契约不完整、REWRITE_SIGNAL、远端分支删除、已同步、本地领先、分叉分析失败、快进策略遇分叉、快进/合并命令失败（含 stash 恢复）、merge 冲突、stash push 失败（更新未发生）、pop 失败。

全局安全不变量（退出码纪律、不 eval、不自动 continue/abort、ask cancelled/error 处理、不自动 pull/rebase 恢复）见根 CLAUDE.md「安全不变量」与 error-principles 块，本文不重述；本技能是用户显式调用的 pull 流程，与「其他技能错误恢复路径中不自动 pull」的不变量不冲突。

## 6. 配置继承声明

- **两条 git-pull 规则不受 `repository_relation_mode` 管辖**：它们是本地更新语义（分叉时快进还是合并、是否 stash），与「远程仓库如何协作」正交；类比先例——git-release tag 推送、git-extract 三规则均声明不受管辖（release-flow 设计 Policy inheritance 声明）。
- **managed_safe_sync 的「远程动作前先 fetch」已由 pull 天然满足**：pull 的定义即 fetch + 本地更新，且经 `/git-fetch --force` 委托无条件先取远端真实状态，无需策略挂钩。
- **「远程写入一律经 /git-push」声明不需修改**：git-pull 不持有远程写字段、不执行任何远程写入；实施时在 git-rules Policy inheritance 节补一句「git-pull 为本地更新（读入）技能，两条规则为本地操作语义，不受 repository_relation_mode 管辖」，与 git-release / git-extract 声明并列。
- 报告中的 push 建议是**指路**（建议用户运行 `/git-push`），不是委托远程写入，不构成策略继承关系。

## 7. 边界案例

- **本地领先**（ahead>0, behind=0）→ 报告 + 建议 `/git-push`（Step 3.6）。
- **远端分支已删除**（NEW_OID 空 / PRUNED_REFS 命中）→ 报告 + 建议（Step 3.4）。
- **无 upstream 且无法唯一确定目标** → 询问停机点（Step 2c ⑤⑥），取消即终止任务。
- **merge 冲突后** → 不 pop、报告 stash 在栈 + 手动 pop 指引（Step 4.3）。
- **`pull_strategy=fast_forward` 遇分叉** → 停止 + 展示双方提交 + 建议改用合并策略（Step 3.6）。
- **REWRITE_SIGNAL** → 停止、本地分支不变（Step 3.3）。
- **stash pop 失败** → 更新保留、stash 保留、如实报告、停止（Step 4.4）。
- **快进被工作区更改拒绝且已 stash** → 先 pop 恢复再报告（Step 4.2）。
- **竞态**：fetch 后本地被外部修改 → merge --ff-only / merge 失败按原始错误处理，本地分支未变则恢复 stash。
- **同任务内重复委托 fetch** → git-fetch 无跨调用状态，task-init 跳过机制按块内通用机制去重。
- **stash 只含已跟踪更改**：`git stash push` 不带 `-u`，未跟踪文件不入 stash；未跟踪文件与并入更改同名冲突时 git merge 自行拒绝，按原始错误报告。

## 8. 落地同步义务（实施时，非本次）

1. `.claude/skills/git-pull/SKILL.md`——新建（argument-hint：`"[--remote REMOTE] [--branch BRANCH]"`；description 一句职责）。
2. `.claude/skills/git-rules/references/ask/git-pull/pull_strategy.md`、`.../auto_stash.md`——新建（缺失则 ask 返回 error）。
3. `.claude/skills/git-rules/SKILL.md`——canonical 规则表 +2 行（第 3.2 节）；Policy inheritance 节补 git-pull 声明（第 6 节，1 句）。
4. git-help Skill registry 表加行（按字母序，调用方式与作用取自技能自身 SKILL.md）。
5. 根 CLAUDE.md——技能清单表加 `/git-pull` 行；scope 枚举处同步 `git-pull`。
6. README「当前能力」加条目、目录树加 `git-pull/` 行。
7. `scratch/notes/remote-repository-policy.md`——新增「已落地：pull 任务流」节，关闭「pull 自动化边界未定」开放项。
8. release-flow 方案 git-sync 实施时指明与 git-pull 的边界（upstream 拉取 vs 基准回流），不改本次文件。
9. VERSION 由用户维护，不主动迭代。

## 9. 验证方案（实施后端到端场景）

| # | 场景构造 | 预期 |
| --- | --- | --- |
| 1 | 干净工作区，本地严格落后 2 个提交 | 无询问，无条件快进；报告并入 2 提交；无 stash 动作 |
| 2 | 分叉（双方各有提交），pull_strategy=prompt，工作区干净 | ask 恰好 4 选项；选「仅本次合并」→ merge commit，配置不写回（`/git-rules show` 核对 value 仍为 prompt） |
| 3 | 同 2，选「总是合并」 | merge 后配置写回 `merge`；重跑场景直接合并不再询问 |
| 4 | 分叉 + `pull_strategy=fast_forward` | 停止、展示双方提交、建议流程；无任何本地分支写入 |
| 5 | 工作区有未提交更改，auto_stash=prompt，选「总是」+ 严格落后 | stash → 快进 → pop 成功；配置写回 `always`；工作区恢复原状 |
| 6 | 同 5 改为分叉合并且合并冲突 | 冲突停机：不 pop、报告 stash 在栈 + 手动 pop 指引；`git stash list` 可见该条目 |
| 7 | 选「是」stash + 合并成功但 pop 冲突 | 更新成功 + STASH_POP=failed 报告；stash 保留；不自动继续 |
| 8 | 远端历史改写（force push 远端后运行） | fetch 委托成功但 REWRITE_SIGNAL=true → 停止，本地分支 OID 不变 |
| 9 | 无 upstream、单 remote、无参数 | 自动采用该 remote + 当前分支名，正常拉取 |
| 10 | 无 upstream、多 remote、无参数 | 询问 remote（及 branch）停机点；取消 → 终止任务 |
| 11 | 远端分支已删除 | NEW_OID 空 / PRUNED_REFS 命中 → 报告 + 建议；本地不变 |
| 12 | 本地领先 | 报告领先提交 + 建议 `/git-push`；无本地写入 |
| 13 | detached HEAD | 终止任务并说明原因 |
| 14 | 未完成 merge（MERGE_HEAD 存在）时调用 | task-init 检出并终止任务 |
| 15 | `/git-rules reset git-pull` 后重跑 | 两条规则恢复 prompt，询问行为回归 |
| 16 | `/git-help git-pull` | registry 含 git-pull 行且与 SKILL.md 一致；detail 分支正常 |

## 非目标

- 不实施 `.claude/skills/git-pull/` 本体（本次仅设计文档）；
- 不实施 git-rules 改动（canonical 行、ask 映射均为实施时义务）；
- 不实现 rebase、push、upstream 设置、stash 冲突自动恢复、detached HEAD 支持；
- 不占用 git-sync 名（边界见第 1 节）；git-merge 原语另行设计，v1 不依赖；
- 不修改 `.claude/skills/git-rules/VERSION`。
