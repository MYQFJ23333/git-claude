# git-fetch 概念设计

状态：概念设计（已定稿，待实施） · 归属：待定（版本由用户维护） · 上游方案：`scratch/notes/remote-repository-policy.md`（待办五项评估）；原语层预定位见 `scratch/designs/release-flow-generic-skills-design.md`

## 1. 定位

原语层 fetch 原语：机械更新 remote-tracking ref，无流程停机点；既作为其他技能内部 fetch 的统一委托目标，用户也可直接调用。

| 做 | 不做 |
| --- | --- |
| 更新 tracking ref（全分支 / 单分支、双模式 refspec）、prune（按 `repository_relation_mode` 挂钩）、改写信号检测、机械报告与返回契约 | 分叉判定策略化（behind/ahead、祖先关系属调用方）、push、upstream 设置、更新本地分支（`refs/heads/*`）、pull、除配置初始化外的任何询问 |

与相近技能的根本区别：

- **`/git-push`**：分支级远程**写入**的唯一实现入口。其 Step 3b 现为内联 fetch，与 lease 语义绑定（强制 refspec + `EXPECTED_REMOTE_OID` + 分叉三分支判定）。**本期不改造**，仅在第 5 节预留委托契约；「委托 fetch ≠ 委托分叉判定」。
- **`/git-sync`**（未来，任务流层）：负责本地分支快进与合并（写 `refs/heads/*`，属任务流自理）；其 fetch 环节按上游方案「委托而非吸收」调用本技能。
- **pull**：fetch + 合并的组合，合并语义不属本技能。

**准入论证（不可或缺性）**：单看用户直接调用，一条裸 `git fetch` 即可平替——本技能的存在依据是作为**被多个任务流消费的协议单点**：安全 refspec 双模式、prune 与规则的挂钩、改写信号的标准化判定、失败分类与返回契约。这些语义已有两个消费方预留（git-push Step 3b 的强制 refspec 现行实现、git-sync 设计预留的委托点），分散实现必然重复 task-init 快照、ask 协议与失败处理并产生语义漂移——与 `task-init` 块单点保管、`delegate remote write` 同构的架构决策。普遍性同时满足：套装完整安装前提下适用于任何 Git 管理方案的项目，差异由参数与规则吸收。

## 2. 参数接口

```text
[--remote REMOTE] [--branch BRANCH] [--force] [--prune | --no-prune]
```

| 参数 | 形式 | 默认 | 约束 |
| --- | --- | --- | --- |
| `--remote` | 单值 | 按 Step 2c 解析（参数 > upstream > 单 remote > 停止） | 必须与 `git remote` 输出完全一致 |
| `--branch` | 单值 | 缺省 = 全分支 fetch | `git check-ref-format --branch` 通过；指定即只 fetch 该分支 |
| `--force` | 无值开关 | 关 = 仅快进（非强制 refspec） | 开启时 refspec 加 `+` 前缀 |
| `--prune` / `--no-prune` | 无值开关，互斥 | 随规则：`same_repository` 自动开、`independent_repositories` 关 | 显式给出时覆盖规则默认；同现 → 报告用法终止 |

解析规则（仿 git-push Step 2a）：每个选项至多出现一次；单值选项必须紧跟非空值，无值开关不得携带值；未知选项、位置参数、缺值、重复 → 报告用法并终止任务，不猜测意图。不得兼容旧的位置参数格式。

**refspec 构造规则**（Step 3 使用，全部显式、不依赖 git 默认推断）：

- 有 `--branch`：`refs/heads/BRANCH:refs/remotes/REMOTE/BRANCH`，`--force` 时加 `+` 前缀；
- 无 `--branch` 且 `remote.REMOTE.fetch` **未配置**：`refs/heads/*:refs/remotes/REMOTE/*`，同上决定 `+`；
- 无 `--branch` 且 `remote.REMOTE.fetch` **已配置**（用户自定义跟踪范围）：尊重配置，不传 refspec、直接 `git fetch [--prune] REMOTE`——配置是显式意图而非推断；此时配置自带的 `+` 不受双模式控制，报告中注明。

**默认仅快进的依据**：Git 默认配置 `remote.origin.fetch` 自带 `+`（裸 fetch 即强制同步 tracking ref）；本技能显式传非强制 refspec 是**有意比裸 git fetch 更保守**——non-fast-forward 错误成为「远端历史被改写」的检测信号（见 Step 3b），改写是异常，应显式暴露而非静默吸收。`--force` 服务于需要远端真实 OID 的 lease 场景。

## 3. 规则集成

task-init 引用行（格式同 git-push Step 1，含「块文件缺失、为空或不可读时终止任务，不自动创建或猜测块内容」声明），本技能所需 value 仅一条：

```text
/rules/project/repository_relation_mode/value
```

- **校验白名单**：仅接受 `unconfigured`、`independent_repositories`、`same_repository`；缺失或非法时内部调 `/git-rules repair project repository_relation_mode --internal`，成功后只重新读取一次，失败或仍不可用时停止。
- **unconfigured**：进入 Step 2 前先向用户说明两种模式含义，调 `/git-rules ask project repository_relation_mode`（initialize），按返回的 `effective_action` 继续；`status` 为 `cancelled` 或 `error` 时终止任务。配置初始化询问是原语层「无停机点」的唯一例外。
- **模式语义（唯一行为差异 = prune 挂钩）**：

| 模式 | prune 默认 | 目标解析 |
| --- | --- | --- |
| `same_repository` | **自动开**（远端删除视为统一管理下的正常同步事件，报告列出被清理引用） | 统一规则，见下 |
| `independent_repositories` | 关，需显式 `--prune` | 统一规则，见下 |

- **目标解析（两模式统一，仿 git-push Step 2c 但无询问出口）**：优先级自高到低、命中即停——① `--remote` 参数；② 当前分支 upstream 的 remote；③ 仓库仅有一个 remote 时自动采用（确定默认，非猜测）；④ 其余情况**终止任务**并列出可用 remote，要求显式 `--remote`。原语层无停机点，不做目标询问；委托方（git-push 未来改造、git-sync）一律传显式参数，不会落入 ④。
- **声明**：不新增 `git-fetch` 作用域、不持有自有远程字段、不创建 ask 映射文件；其他 skill 写规则只能内部调 set，本技能不直接改 JSON。
- 值只在实际使用时验证；任何 ask / 写入成功后丢弃旧快照一次性重读（task-init 既有协议）。

## 4. 步骤流（4 步，与未来 SKILL.md 同构）

开头声明（同 git-push Step 1）：所有命令检查退出码，空输出不等于成功；Step 1 内的失败终止任务，Step 2 之后的失败按第 6 节出口处理。

### Step 1: 环境检查与规则加载

1. 执行 task-init 块调用（引用行 + value 清单 + 缺失声明）；块检查失败或存在冲突 / 未完成 merge / rebase / cherry-pick / revert 时终止任务。
2. 工作区存在未提交更改**不阻止 fetch**（fetch 不改工作区与本地分支），但必须在最终报告中说明。
3. 验证缓存值（白名单）→ 非法走 repair（见第 3 节）。
4. `unconfigured` → 说明含义 + ask + status 检查（见第 3 节）。
5. 记录本次有效 `MODE`，进入 Step 2。

### Step 2: 解析 fetch 目标

- **2a 参数语法解析**：按第 2 节规则解析 `$ARGUMENTS`；本步只解析语法，不读仓库状态、不校验名称。
- **2b 读取仓库状态**：
  1. `git remote`：失败 → 终止并报告原始错误；输出为空 → 终止并报告「未配置远程仓库」。
  2. `git branch --show-current`：失败 → 终止（不把空输出当作 detached）；空输出 = detached HEAD，**不构成 fetch 障碍**（fetch 不依赖当前分支），跳过 upstream 查询。
  3. 当前分支非空时执行 `git rev-parse --abbrev-ref --symbolic-full-name '@{u}'`：退出码 128 且信息为 "no upstream configured" = 无 upstream（合法，非失败）；其他失败 → 终止并报告，不得把未知错误当作无 upstream。
  4. `git config --get-all remote.REMOTE.fetch` 在 REMOTE 解析后执行（见第 2 节 refspec 规则）：退出码 1 = 未配置（合法）；其他失败 → 终止。
- **2c 解析目标并统一校验**：按第 3 节优先级解析 REMOTE；`--branch` 给出时经 `git check-ref-format --branch` 校验，失败终止；REMOTE 必须与 `git remote` 输出完全一致，不一致列出可用 remote 并终止。
- **形成本次统一上下文**，后续步骤只读字段不再推断：`REMOTE`、`BRANCH`（可空 = 全分支）、`FETCH_MODE`（fast-forward | force）、`PRUNE_ENABLED`（显式参数 > 规则默认）、`MODE`、`REMOTE_FETCH_CONFIGURED`（是否尊重自定义 refspec 配置）。

### Step 3: 构造 refspec 并执行 fetch

1. **3a 构造**（进程参数数组，不拼接 shell 字符串，不解释用户输入中的特殊字符）：
   - `git fetch [FETCH_MODE=force 时的 refspec 依第 2 节规则] [--prune 当 PRUNE_ENABLED] REMOTE [refspec]`；
   - 有 `--branch` 或未配置自定义时显式传 refspec；尊重配置模式不传 refspec。
2. **3b 执行并按退出码分类**：
   - 退出码 0 → Step 4；
   - **仅快进模式下 non-fast-forward**（退出码非 0 且 stderr 含 non-fast-forward 判定特征，如 "updates were rejected" / "non-fast-forward"）：**不作为笼统失败**——记录 `REWRITE_SIGNAL=true` 与被拒 ref，其余 ref 可能已正常更新，进入 Step 4 汇总报告；
   - 认证、权限、网络错误：原样显示原始错误，不自动重试，终止任务；
   - 其他错误：原样显示原始错误，终止任务。

### Step 4: 机械检查与报告

1. **结果枚举**（fetch 前后各执行一次 `git for-each-ref --format='%(refname) %(objectname)' refs/remotes/REMOTE`，diff 得出）：
   - `UPDATED`：OID 变化的 ref（old → new）；`NEW`：新增 ref；`PRUNED_REFS`：消失的 ref（prune 结果）；
   - 快照命令失败 → 终止任务并如实报告「fetch 已执行完成，但结果枚举失败」（fetch 本身成功不被掩盖，契约字段缺失也不编造）。
2. **force 模式的改写可见性**：对 `UPDATED` 中每个 ref 执行 `git merge-base --is-ancestor OLD NEW`；失败 = 非快进更新（远端历史已改写），列入改写信号报告——`--force` 同步了状态，但改写事件仍须对用户可见。
3. **报告（即返回契约）**：
   - 面向用户：REMOTE、FETCH_MODE、PRUNE_ENABLED 与 `PRUNED_REFS` 清单、`UPDATED`/`NEW`（超 20 条截断报总数）、改写信号（仅快进被拒清单 + 建议显式 `--force`；或 force 下非快进更新清单）、工作区未提交更改说明、无更新时明确报告「已是最新」；
   - 面向委托方的契约字段（任务上下文中保留）：`REMOTE`、`FETCHED_SCOPE`（all | BRANCH）、`NEW_OID`（单分支时）、`UPDATED`、`PRUNED_REFS`、`REWRITE_SIGNAL`、`FETCH_MODE`、`PRUNE_ENABLED`。

## 5. 返回契约（委托方视角）

- 委托方式：以 `/git-fetch` 形式调用并传入显式参数；委托方**只消费第 4 节契约字段**，不复制 refspec 构造、失败分类或改写判定逻辑，也不重新 `rev-parse` 取 OID（避免重复命令与 TOCTOU）。
- **委托 fetch ≠ 委托分叉判定**：behind/ahead（`rev-list --left-right --count`）、祖先关系检查、推送决策、确认询问一律留在调用方。
- **git-push 预留契约（本期不改造）**：未来 Step 3b 可改为委托 `/git-fetch --remote R --branch B --force`，从 `NEW_OID` 取得 `EXPECTED_REMOTE_OID`；其分叉三分支判定与强推确认仍留 git-push。现行内联实现（强制 refspec）与本技能 `--force` 模式语义一致，改造是等价替换而非行为变更。
- **git-sync（未来）**：委托本技能取得更新状态后，自行处理本地分支的快进 / 合并（写 `refs/heads/*` 不属本技能）。

## 6. 失败出口

| 触发 | 动作 |
| --- | --- |
| 非 git 仓库；存在冲突或未完成 merge/rebase/cherry-pick/revert（task-init） | 终止任务 |
| task-init 块文件缺失、为空或不可读 | 终止任务 |
| 规则 value 缺失/非法且 repair 失败；配置校准失败或配置版本高于基准 | 终止任务 |
| ask 返回 `cancelled` 或 `error` | 终止任务 |
| 参数语法错误（未知选项、缺值、重复、`--prune` 与 `--no-prune` 同现） | 终止任务，报告用法 |
| 无 remote；REMOTE 不匹配；BRANCH 校验失败 | 终止任务，列出可用值 |
| 目标解析落入 ④（多 remote、无 upstream、无参数） | 终止任务，要求显式 `--remote` |
| `git fetch` 认证 / 权限 / 网络错误 | 终止任务，原样报告，不重试 |
| `git fetch` 仅快进模式 non-fast-forward | **不终止**：记 `REWRITE_SIGNAL`，进 Step 4 报告（部分 ref 已更新时一并列出） |
| `git fetch` 其他未知错误 | 终止任务，原样报告 |
| fetch 前后快照命令失败 | 终止任务，如实报告「fetch 已完成但结果枚举失败」 |

原语层只有两种出口词汇：**终止任务**（报告原因后停止）与**正常结束**（Step 4 完整报告）。不存在 git-push 的「结束远程阶段」中间态。全局安全不变量（退出码、不 eval、不自动 continue/abort、ask cancelled/error 终止、不自动 pull/rebase）见根 CLAUDE.md「安全不变量」，本文不重述。

## 7. 边界案例

- **远端分支被改写**：仅快进 → 该 ref 拒绝更新 + `REWRITE_SIGNAL` 报告（建议显式 `--force`）；`--force` → 接受更新，Step 4 以 `merge-base --is-ancestor` 复检并报告非快进事件。两模式下改写都可见。
- **单分支 fetch + prune**：`--prune` 只清理匹配显式 refspec 范围的失效 tracking ref（依赖 git ≥ 2.29 语义；不做运行时版本探测，低版本下单分支 prune 可能不完整——实施时如需可在报告中附 `git --version` 提示）。
- **用户自定义 `remote.REMOTE.fetch`**：全分支模式尊重配置（不传 refspec），配置自带 `+` 不受双模式控制，报告注明；有 `--branch` 时参数意图明确，覆盖配置。
- **detached HEAD**：照常 fetch；无 upstream 可查时按「参数 > 单 remote > 终止」解析。
- **无 remote / 空仓库无远端分支**：前者终止；后者 fetch 返回 0 且无更新 → 报告「已是最新 / 无更新」。
- **`--branch` 指向远端不存在的分支**：fetch 失败（"couldn't find remote ref"）→ 按其他错误终止，原样报告。
- **`--force` 与 `--prune` 同用**：合法，各自独立生效。
- **HEAD 处于将被 prune 的 tracking ref**（detached at `origin/x`）：删除 `refs/remotes/origin/x` 不改变已检出的工作区（对象仍在）；报告 `PRUNED_REFS` 时如其 OID 与 HEAD 当前 OID 相同则注明。
- **同任务内重复调用**（委托链中多次 fetch）：本技能无跨调用状态，每次独立执行；task-init 跳过语义按块内通用机制执行。

## 8. 落地同步义务（实施时，非本次）

- git-help Skill registry 表加行（调用方式含完整 argument-hint，作用取自技能自身 description）；
- 根 CLAUDE.md 分工表加 `/git-fetch` 行；
- README「当前能力」加条目、目录树加行；
- `scratch/notes/remote-repository-policy.md`：回应五项评估（已存在 / 多 remote 与 upstream 支持见第 2–3 节 / 失败返回见第 6 节 / 去重见第 1 节准入论证 / `repository_relation_mode` 服从见第 3 节），关闭或更新待办状态；
- release-flow 方案中 git-sync Step 3 的委托点在实施 git-sync 时对接；
- git-push Step 3b 改造另行评估（不在本技能实施范围内）；
- VERSION 由用户维护，不主动迭代；不新增规则 scope 与 ask 映射（已定）。

## 非目标

- 不实施 `.claude/skills/git-fetch/` 本体（本次仅设计文档）；
- 不改造 git-push（仅预留第 5 节契约）；
- 不实现 pull、本地分支更新、分叉判定策略化、upstream 设置；
- 不新增规则 scope、不创建 ask 映射文件；
- 不修改 `.claude/skills/git-rules/VERSION`。
