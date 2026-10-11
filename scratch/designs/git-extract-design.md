# git-extract 概念设计

状态：概念设计（选择逻辑已定稿，待实现） · 归属：0.4.0 发布流水线技能 A · 上游方案：`scratch/designs/release-flow-generic-skills-design.md`

## 1. 定位

从基准分支拉出目标分支，把另一分支的更改**按路径选择性搬移**进来并提交。主用例：发布准备——将工作分支上的可发布更改移入 `release/vX.Y.Z`，排除本地私有文件。

| 做 | 不做 |
| --- | --- |
| 建目标分支、分类与推断选择集、原子应用补丁、委托提交 | merge / cherry-pick（不建立祖先关系）、推送、创建 PR、暂存分析 / 预检 / 消息规则（全部委托） |

与 `/git-sync` 的根本区别：本技能是**搬移**（diff/apply，两分支间无祖先关系）；git-sync 是**合并**（建立祖先关系、全量包含）。语义互斥，不共享实现。

## 2. 参数接口

```text
[--base BRANCH] [--from BRANCH] [--name BRANCH] [--message MSG] [INCLUDE_PATHSPEC...] [--exclude PATHSPEC...]
```

| 参数 | 形式 | 默认 | 约束 |
| --- | --- | --- | --- |
| `--base` | 单值 | 默认分支探测（`symbolic-ref refs/remotes/REMOTE/HEAD`，失败→询问，不猜） | ref 必须存在 |
| `--from` | 单值 | 当前分支 | ref 必须存在；不得与 base 相同 |
| `--name` | 单值 | 无默认，必给或询问 | `git check-ref-format --branch` 通过；分支不得已存在 |
| `--message` | 单值 | 省略时不生成，交由 git-commit 按其规则决定 | — |
| `INCLUDE_PATHSPEC` | 位置参数，可多个 | —（可省略） | 「至少给一个」仅 manual 模式检查，且按 INCLUDE 与 `--exclude` **合计** ≥1 计（见 3.2 / 3.6）；auto/confirm 可省 |
| `--exclude` | 可重复 | —（每项一个 pathspec） | 同为 pathspec 语法 |

解析规则（仿 git-push Step 2a）：单值选项各至多一次且必须紧跟非空值；未知选项、pathspec 之外的多余位置参数、缺值 → 报告用法并停止，不猜测意图。

**边界决策**：pathspec 完备性依赖选择模式——**auto / confirm 模式零 pathspec 合法**（全自动推断，见第 3 章）；**manual 模式零 pathspec → 报告并停止**（提示改用 auto/confirm 或给出 pathspec）。v1 不提供 `--mode commits`（cherry-pick 序列保留提交粒度），因下游 PR 为 squash 合并、粒度会被抹平。

## 3. 选择逻辑

### 3.1 三类文件模型

对 `BASE...FROM`（三点 diff）中的每条路径，按 base / merge-base / from 三参照点分类：

| 类 | 判定 | auto / confirm 下的处理 |
| --- | --- | --- |
| **A 类｜base 已有** | base 跟踪该文件 | 变更内容**转移** |
| **B 类｜分叉前有、base 已删** | merge-base 有、base 现无、from 侧有变更 | **判私有，排除**（判定基准 = merge-base） |
| **C 类｜此次完全新增** | merge-base 无、base 无、from 有 | 依次过两道过滤（base `.gitignore`、持久清单，见 3.3），剩余交三策略（见 3.4） |

三点 diff 天然偏向该模型：B 类中 from 侧未改动的文件、A 类中 from 侧未改动的文件都不会出现在 diff 中——进 diff 的都是 from 侧真实动过的路径，无需额外处理。

### 3.2 selection_mode 三模式

| 值 | 语义 |
| --- | --- |
| `auto` | 三类分类 + base 的 `.gitignore` 双信号推断（3.3），C 类剩余交三策略 |
| `manual` | 不做任何推断，纯参数决定（「pathspec 至少一个」的检查仅此模式执行） |
| `confirm` | 推断同 auto，但每次展示完整分类与推断结果并经确认 |

控制值 `unconfigured` 触发首次询问（initialize，见第 6 节）。选项值用 `confirm` 而非 `prompt`——`prompt` 是全局控制值保留字，不得用作本规则的存储值。

### 3.3 auto / confirm 推断流程

1. 取分离 `.gitignore` 自身变更（见 3.6）后的 `BASE...FROM` 路径清单，按 3.1 三类分类；
2. A 类 → 移动集；B 类 → 排除集（判私有）；
3. C 类 → 过 base 的 `.gitignore` 信号：**base 未跟踪 ∧ 被 base 忽略 → 排除**；
4. C 类剩余 → 过持久清单 `extra_excludes`（任何模式均参与，见 3.5）；
5. 仍剩余的 C 类 → 交三策略（3.4）；
6. 按 3.5 优先级矩阵合成最终 INCLUDE/EXCLUDE，生成补丁。

**回退说明**：base 无 `.gitignore` 时信号贡献为空——三类分类照做、C 类直接过清单与三策略。原「无知识回退」问题已被模型吸收，不存在独立回退分支。

gitignore 信号只作用于 C 类：A 类已被 base 跟踪（gitignore 对已跟踪文件无意义）、B 类已判私有。本项目 `scratch/` 等私有文件落在 C 类，由该信号自动挑出，每次发布无需重复选择。

### 3.4 C 类三策略（new_files_mode）

| 值 | 语义 |
| --- | --- |
| `prompt`（默认 + 控制值） | 每次选择——展示 C 类清单后经规则询问决定本次动作（恰好 4 选项，见第 6 节映射）；需要部分选择时取消本次，改用显式 INCLUDE/EXCLUDE 重跑 |
| `all` | 全添加——剩余 C 类全部进入移动集 |
| `none` | 全不添加——剩余 C 类全部排除 |

**延迟消费**：仅当 auto/confirm 实际走到第 5 步（存在非忽略 C 类文件）时才读取本规则；值为 all/none 直接执行不问，值为 ask 才发起询问。manual 模式不消费本规则。

### 3.5 优先级矩阵

1. **显式 `INCLUDE` 覆盖一切自动推断**——gitignore 信号、B 类判私有、持久清单均可被显式包含推翻（参数是显式意图，推断是启发式）；
2. **显式 `--exclude` 与 `INCLUDE` 同路径 → 排除优先**（沿用 git pathspec 语义，不报错）；
3. **`extra_excludes` 不受 selection_mode 门控**——任何模式下都参与排除；与显式 INCLUDE 同路径时服从第 1 条（本次参数覆盖持久配置）。

### 3.6 情况矩阵与 .gitignore 自身变更

| selection_mode \ 情况 | base 有 `.gitignore` | base 无 `.gitignore` |
| --- | --- | --- |
| `auto` | 三类分类 + gitignore 信号 + 清单 + 三策略（全自动） | 三类分类 + 清单 + 三策略（信号空） |
| `confirm` | 同 auto，完整展示分类与推断结果并确认 | 同左（信号空） |
| `manual` | 纯参数；gitignore 不参与（pathspec 至少一个） | 纯参数（pathspec 至少一个） |

**`.gitignore` 自身的变更**（任意层级、含修改与删除）不走常规推断——它是知识源文件本身：Step 3 第 2 步将其**先行分离**出分类输入，第 6 步展示差异摘要后**批量一次 AskUserQuestion**（全部转移 / 全不转移 / 取消终止；仅本次、不写回，属流程内交互而非规则询问），结论直接并入移动集或排除集——该路径不再进入 gitignore 信号、持久清单或三策略，避免重复询问与推断覆盖。本项目 workspace-sync 对 `.gitignore` 的删除即由此闸拦住。

## 4. 步骤流（6 步）

### Step 1 环境检查与规则快照

- **1a 环境**：同 git-push Step 1a 模板（仓库根、`status --short --branch`、四种 `*_HEAD` 未完成操作检测）。**差异**：本技能涉及 checkout + apply，tracked 工作区变更非空即停止（提示先提交或自行 stash，不自动执行）；未跟踪文件不拦截（与补丁冲突时在 Step 4 如实失败）。
- **1b 快照**：同 git-push Step 1b 模板（读 VERSION + 配置、版本比对、repair/calibrate 恢复协议）。**缓存本任务三规则 value**（`selection_mode` / `new_files_mode` / `extra_excludes`，对齐「只缓存本任务实际用到的 value」），不缓存其他 scope；value 只在实际使用时验证，缺失/非法走 `repair git-extract KEY --internal`。

### Step 2 解析参数与引用

1. 按第 2 节规则解析 $ARGUMENTS（选项成对、语法校验照旧）。
2. **消费 `selection_mode`**：unconfigured → 调 `/git-rules ask git-extract selection_mode`（initialize，写回后继续）；cancelled/error → 终止。得到本次模式 auto / manual / confirm。
3. **pathspec 完备性检查仅 manual 模式执行**（零 pathspec → 报告并停止，提示见第 2 节）。
4. base 缺省 → 默认分支探测；from 缺省 → 当前分支（detached HEAD 时必须显式给 `--from`，否则停止）。
5. `git rev-parse --verify` 逐一验证 base / from；`git branch --list NAME` 检查目标分支不存在。
6. 记录 `BASE / FROM / NAME` 与本次模式，进入 Step 3。

### Step 3 分类与推断，合成选择集

1. 取 `BASE...FROM` 路径清单，按 3.1 三类分类。
2. **先行分离**路径名为 `.gitignore` 的变更（任意层级、含删除）——移出分类输入、暂存待询（3.6 闸），不进入后续推断。
3. 对剩余路径按 3.3 执行推断（manual 跳过推断，仅用显式参数，仍展示分类供参考）。
4. **读取并解析 `extra_excludes`**（逗号分隔 → pathspec 数组；解析失败即停止并报告原字符串）。
5. 存在「非忽略 C 类文件待定」且模式为 auto/confirm → **消费 `new_files_mode`**：all/none 直接执行；prompt → 展示 C 类清单后调 `/git-rules ask git-extract new_files_mode`（cancelled/error → 终止）。
6. `.gitignore` 暂存集展示差异摘要后**单独 AskUserQuestion**（一次批量询问：全部转移 / 全不转移 / 取消终止；仅本次、不写回；结论直接并入移动集或排除集，不再进入三策略）。
7. 按 3.5 合成最终 INCLUDE/EXCLUDE，执行 `git diff --binary BASE...FROM -- …` 输出到临时文件（命令细节见第 5 节）。
8. 补丁为空 → 报告「无可转移更改」（含排除命中情况）并结束；否则展示——**confirm 模式必须展示完整分类与推断结果**，auto 模式简要列出移动/排除清单与 gitignore 命中，**并说明 from 侧未提交更改不在转移范围**——再 AskUserQuestion 确认；取消 → 终止。

### Step 4 创建目标分支并原子应用

1. `git switch -c NAME BASE`（从基准分支拉出）。
2. `git apply --index <补丁文件>`，检查退出码。
3. **失败 = 与 base 冲突**：按安全不变量停止——不 `--3way`、不 `--reject`、不回滚；保留已建分支与现场，报告补丁文件路径与清理方式（如 `git switch -` / `git branch -D NAME`），由用户处置。
4. 成功 → 暂存区就绪，进入 Step 5。

### Step 5 委托提交

1. 调用 `/git-commit`（带 `--message` 时传入；省略时由 git-commit 按 `commit_message_mode` 等规则自行处理）。
2. 本技能不实现暂存分析、预检、消息风格、远程策略——全部由 git-commit 消费其自身规则。
3. 委托返回 cancelled / 失败 → 终止并如实报告：分支已建、补丁已在暂存区，用户可手动完成提交。

### Step 6 报告

- 分支名、基准、来源、新提交哈希；
- 转移文件数与清单（超 20 条截断报总数）、排除统计（B 类 / gitignore / 持久清单 / 显式参数分列）、C 类策略与本次选择；
- 提交是否已被 git-commit 顺带推送（见第 6 节副作用）；
- 建议下一步：`/git-pr`。

## 5. 核心机制选型（已定）

| 候选 | 判定 |
| --- | --- |
| merge 后还原排除路径 | **否决**——排除内容先进入历史，事后 revert 消不掉，违反「排除本地私有文件」初衷 |
| cherry-pick 序列（`-n` + 撤出排除） | **备选留待未来** `--mode commits`——保留提交粒度但逐提交处理排除、冲突点多，且 squash 下粒度无意义 |
| **三点 diff + pathspec 排除 + `git apply --index`（选定）** | 排除路径根本不进补丁（零泄漏）；全有或全无，失败即停 |

关键细节：

- `BASE...FROM`（三点）只含 from 侧改动；两点 diff 会把 base 侧新改动反向还原，**不能用**。
- `:(exclude)` 是 pathspec magic，作为独立参数传递（参数数组，不进 shell 字符串）；同路径冲突与覆盖语义统一见 3.5 优先级矩阵。
- `--binary` 覆盖二进制文件；diff 默认开启重命名检测，apply 可应用。
- `--index` 同时写暂存区——成功即 git-commit 可直接处理。
- 分类与推断先于 diff 执行（Step 3 第 6 步）：diff 命令只见合成后的最终 pathspec，模型与命令解耦。

## 6. 委托与配置

| 事项 | 设计 |
| --- | --- |
| 提交 | **委托 `/git-commit`**；已知副作用：same_repository 模式下 git-commit 可能把目标分支顺带推上远端（其 Step 5c 既有行为），本技能不加开关，Step 6 如实报告 |
| 推送 / PR | 不涉及；后续由 `/git-pr` 委托 git-push 完成 |
| 语义复用 | 默认分支探测、快照模板、ask 契约按上游方案「共享语义单一定义」表引用，不重写 |

### git-rules 集成：三规则草案

（草案——0.4.0 实施时写入 canonical 表；本文档不改 git-rules/SKILL.md，实施阶段见上游方案文件清单。）

| scope | key | 默认 value | options（value=标签） | 控制值 | ask_mode | description |
| --- | --- | --- | --- | --- | --- | --- |
| git-extract | selection_mode | unconfigured | auto=自动推断、manual=仅按参数、confirm=展示并确认 | unconfigured | initialize | 路径选择模式（三类分类与 .gitignore 推断） |
| git-extract | new_files_mode | prompt | prompt=每次询问、all=全添加、none=全不添加 | prompt | inline_persistence | C 类非忽略新文件的处理策略 |
| git-extract | extra_excludes | （空字符串） | 无 | 无 | none | 持久排除清单（逗号分隔 pathspec，运行时解析，任何模式均参与） |

格式依据：

- `selection_mode`：unconfigured + initialize 对齐 `commit_message_mode` / `repository_relation_mode` 先例；initialize 映射恰好 3 选项（2–4 合法）；选项值用 `confirm` 避开全局控制值保留字 `prompt`。
- `new_files_mode`：`prompt` 同时是默认值、options 成员与控制值，**逐格对齐 `empty_staging_mode` 先例**——用套件既有的「每次询问」控制值 `prompt` 而非自造 `ask`，使 CLAUDE.md 的控制值枚举（`unconfigured`、`prompt`）无需扩写；映射文件为**恰好 4 行选项**——「仅本次添加」「仅本次不添加」stored_value 为空（不写回，value 保持 prompt，下次再问，这正是「每次选择」的可实现形态）、「总是添加」→ all、「总是不添加」→ none（写回后不再询问）。若用 initialize 则必写回，选一次即永久离开 prompt，与语义不符。
- `extra_excludes`：无 options、无控制值、ask_mode=none → **不需要** ask 映射文件；set 不做内容校验，运行时解析（解析失败即停，见 Step 3）。

ask 触发时机与延迟消费：

- `selection_mode`：Step 2 解析后即消费——unconfigured 时调 ask（initialize 写回后续用）；
- `new_files_mode`：仅当 auto/confirm 走到「C 类非忽略文件待定」且 value=prompt 时调 `/git-rules ask`；
- `extra_excludes`：Step 3 合并排除集时读取，永不 ask。

实施阶段配套（本次不创建）：`references/ask/git-extract/selection_mode.md`（initialize，3 选项）、`references/ask/git-extract/new_files_mode.md`（inline_persistence，恰好 4 行）。

## 7. 失败出口

| 触发 | 动作 |
| --- | --- |
| 存在未完成 merge/rebase/cherry-pick/revert | 停止，报告状态 |
| tracked 工作区变更非空 | 停止，提示自行提交或 stash |
| 参数非法 / 引用不存在 / 目标分支已存在 | 停止，报告原因 |
| manual 模式零 pathspec | 停止，报告并提示改用 auto/confirm 或给出 pathspec |
| selection_mode / new_files_mode 的 ask 返回 cancelled 或 error | 终止 |
| 三规则 value 缺失/非法且 repair 失败 | 停止（快照协议既有） |
| `extra_excludes` 运行时解析失败 | 停止，报告原字符串 |
| `.gitignore` 自身变更询问被取消 | 终止 |
| 补丁为空 | 报告无事可做，正常结束 |
| 用户取消确认 | 终止 |
| `git apply` 失败 | 停止，保留分支与现场，报告清理方式（不自动回滚） |
| git-commit 委托取消或失败 | 停止，报告「分支已建、补丁在暂存区」 |

全局安全不变量（退出码、不 eval、不自动 continue/abort、远端证明安全才写入、ask cancelled/error 终止）见根 CLAUDE.md「安全不变量」，本文不重述。

## 8. 边界案例

- **base 自分叉点后前进且同位置已改** → Step 4 apply 失败，走冲突出口（「无法证明安全即停止」的预期行为）。
- **未跟踪文件与补丁目标路径冲突** → 同上，apply 失败如实报告。
- **from 工作区有未提交更改** → 不包含在转移范围（diff 只取提交内容），Step 3 展示时说明。
- **base 无 `.gitignore`** → 信号空、三类分类照做（回退问题已吸收，见 3.3）。
- **B 类文件** → 自动排除（判私有）；显式 INCLUDE 可按 3.5 推翻。
- **C 类被 base 忽略** → 排除；未忽略 → 交三策略。
- **`.gitignore` 自身变更** → 先行分离出推断输入，批量一次询问（任意层级、含删除），不走常规推断与三策略（3.6）。
- **INCLUDE 与 `--exclude` 同路径** → 排除优先（3.5 第 2 条）。
- **INCLUDE 与 `extra_excludes` 同路径** → INCLUDE 覆盖（3.5 第 1 条）。
- **只有 `--exclude`、无 INCLUDE** → manual 下合计 ≥1 满足「至少一个」检查，允许；auto/confirm 零 pathspec 亦允许。
- **auto 零参数** → 全自动推断运行（`--name` 仍必给或询问）。
- **manual 零 pathspec** → 停止。
