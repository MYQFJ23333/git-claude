---
name: git-extract
description: 选择性搬移 - 从基准分支拉出目标分支，按三类分类与 .gitignore 推断选择性搬移另一分支的更改并原子应用补丁，委托 git-commit 提交，用于发布准备排除本地私有文件
argument-hint: "[--base BRANCH] [--from BRANCH] [--name BRANCH] [--message MSG] [INCLUDE_PATHSPEC...] [--exclude PATHSPEC...]"
license: MIT
---

# git-extract

选择性搬移助手。从基准分支拉出目标分支，把另一分支的更改按路径选择性搬移进来并原子应用补丁，委托 git-commit 完成提交。主用例是发布准备：将工作分支上的可发布更改移入 `release/vX.Y.Z`，排除本地私有文件。

- 做：建目标分支、分类与推断选择集、原子应用补丁、委托提交；
- 不做：merge / cherry-pick（不建立祖先关系）、推送、创建 PR、暂存分析 / 预检 / 消息规则（全部委托）。

本技能是**搬移**（diff/apply，两分支间无祖先关系），不是合并（建立祖先关系、全量包含）；v1 不提供 `--mode commits`——下游 PR 为 squash 合并，提交粒度会被抹平。

所有命令都必须检查退出码。命令失败时立即停止当前阶段，不把空输出当作成功。

## Step 1: 环境检查与规则快照

### Step 1a: Git 仓库和操作状态

1. 执行 `git rev-parse --show-toplevel`，取得仓库根目录；失败时提示原因并终止。
2. 后续 Git 命令从仓库根目录执行。
3. 执行 `git status --short --branch`，记录当前分支和工作区状态。
4. 检查 MERGE_HEAD、CHERRY_PICK_HEAD、REVERT_HEAD、REBASE_HEAD 是否存在，以及 git status 是否报告 merge、rebase、cherry-pick 或 revert 正在进行。
5. 存在未完成 Git 操作时停止，报告状态和文件，不自动 continue、abort、reset 或 rebase。
6. **差异（与 git-push 模板不同）**：本技能涉及 checkout + apply，工作区存在已跟踪文件的未提交更改时立即停止，提示先提交或自行 stash，不自动执行任何一种；未跟踪文件不拦截——与补丁目标路径冲突时在 Step 4 apply 如实失败。

### Step 1b: 项目规则一次性读取

1. 读取 `.claude/skills/git-rules/VERSION`，只接受单行 `MAJOR.MINOR.PATCH`；文件缺失、不可读、为空或格式非法时停止，不从其他文件猜测基准版本。
2. 读取 `.claude/git-claude-rules.json` 一次并解析 JSON：
   - 文件不可读时停止，不覆盖；
   - 文件不存在或 JSON 无法解析时，内部调用 `/git-rules repair --internal`，成功后只重新读取一次，失败或仍不可用时停止。
3. 比较配置 version 与 VERSION：
   - 相等时继续；
   - 不相等、缺失或格式非法时，内部调用 `/git-rules calibrate --internal`，成功后只重新读取一次；配置版本高于基准、校准失败或重读后仍不一致时停止。
4. 版本一致后缓存以下三个 JSON Pointer 指向的值（**差异**：本技能只缓存自身三规则，不缓存其他 scope）：
   - `/rules/git-extract/selection_mode/value`
   - `/rules/git-extract/new_files_mode/value`
   - `/rules/git-extract/extra_excludes/value`
   缺失值也按缺失状态缓存，本步骤不校验或修复它们。

版本一致的正常路径不读取 `git-rules/SKILL.md`，也不检查完整规则清单、规则元数据或其他 value。value 只在流程实际使用时验证（`selection_mode` 在 Step 2，`new_files_mode` 与 `extra_excludes` 在 Step 3）；目标规则缺失或 value 非法时，内部调用 `/git-rules repair git-extract KEY --internal`，成功后只重新读取一次，失败或仍不可用时停止。任何 `/git-rules ask` 或其他配置写入成功后都丢弃旧快照，并一次性重新读取 VERSION、配置 version 和三个 value。不得静默使用内存默认值，写规则只能内部调用 set，不直接改 JSON。

配置基准版本（决定校准）与被发布物版本（release 分支名中的版本号）不同源，本技能不读取被发布物版本——目标分支名由 `--name` 提供。

本技能不涉及远程写入，不解析 `repository_relation_mode`。所有 `/git-rules ask` 调用只传 SCOPE/KEY 并检查返回的 status：selected 时才使用 effective_action；cancelled 时终止任务；error 时报告询问或写回失败并终止，不得继续执行依赖该选择的动作。

## Step 2: 解析参数与引用

从 $ARGUMENTS 读取原始参数。不得通过 shell eval 解释用户输入；参数中的引号、反斜杠、美元符号和反引号不参与任何命令拼接。

### Step 2a: 参数解析

参数格式为 `[--base BRANCH] [--from BRANCH] [--name BRANCH] [--message MSG] [INCLUDE_PATHSPEC...] [--exclude PATHSPEC...]`：

- `--base` / `--from` / `--name` / `--message` 为单值选项，各自最多出现一次且必须紧跟一个非空值；
- `--exclude` 可重复出现，每项一个 pathspec；
- 位置参数均为 INCLUDE pathspec，可多个、可省略（完备性检查见 Step 2b 第 2 条）；
- 出现未知选项、pathspec 之外的多余位置参数、重复的单值选项或缺少值时，报告用法并停止，不猜测用户意图；
- `--message` 省略时不生成，交由 git-commit 按其规则决定。

INCLUDE 与 `--exclude` 同为 git pathspec 语法，不做预校验，diff 阶段失败时如实报告。

### Step 2b: 消费 selection_mode 并确定引用

1. **消费 `selection_mode`**：从快照读取，只接受 auto / manual / confirm；值为 `unconfigured` 时调用 `/git-rules ask git-extract selection_mode`（initialize），selected → 以 effective_action 作为本次模式，cancelled / error → 终止；value 缺失或非法 → `repair git-extract selection_mode --internal` 后按协议重读。ask 成功后按 Step 1b 协议丢弃旧快照并一次性重读。
2. **pathspec 完备性检查仅 manual 模式执行**：INCLUDE 与 `--exclude` 合计为零 → 报告并停止，提示改用 auto / confirm 模式或给出 pathspec；auto / confirm 模式零 pathspec 合法（全自动推断）。
3. **base 缺省 → 默认分支探测**（失败 → 询问，不猜）：执行 `git remote` 取得可用 remote，对每个 remote 执行 `git symbolic-ref --quiet --short refs/remotes/REMOTE/HEAD`，取第一个成功结果作为 BASE；无 remote 或全部失败 → 列出候选分支询问 base（候选超过 4 个时打印完整列表让用户口述），不猜测 main / master，取消询问 → 终止。
4. **from 缺省 → 当前分支**：执行 `git branch --show-current`；为空表示 detached HEAD，此时必须显式给 `--from`，否则报告原因并停止。
5. **`--name` 缺省 → 询问目标分支名**；取消询问或未给出 → 终止。
6. 验证：`git rev-parse --verify` 逐一确认 BASE / FROM 存在；BASE 与 FROM 相同 → 报告并停止；`git check-ref-format --branch` 校验 NAME；`git branch --list NAME` 非空 → 报告目标分支已存在并停止。
7. 记录 BASE / FROM / NAME 与本次模式，进入 Step 3。

## Step 3: 分类与推断，合成选择集

1. 执行 `git merge-base BASE FROM` 取得分叉点；取 `git diff --name-only BASE...FROM` 路径清单，按「选择逻辑协议 · 三类文件模型」分类。命令失败即停止。
2. **先行分离**路径名为 `.gitignore` 的变更（任意层级、含修改与删除）——移出分类输入、暂存待询（见协议「.gitignore 自身变更闸」），不进入后续推断。
3. manual 模式：跳过推断，移动集 / 排除集仅由显式参数决定，仍展示分类供参考；auto / confirm 模式：对剩余路径执行协议「auto / confirm 推断流程」（base 无 `.gitignore` 时信号贡献为空，三类分类照做）。
4. **读取并解析 `extra_excludes`**：逗号分隔字符串 → pathspec 数组，空串 = 空清单；解析失败即停止并报告原字符串。任何模式均参与排除（见优先级矩阵）。
5. **延迟消费 `new_files_mode`**：仅当 auto / confirm 存在「非忽略 C 类文件待定」时读取并验证（只接受 prompt / all / none）：
   - `all` → 剩余 C 类全部进入移动集，不询问；
   - `none` → 剩余 C 类全部排除，不询问；
   - `prompt` → 展示 C 类清单后调用 `/git-rules ask git-extract new_files_mode`，selected → 按 effective_action 决定本次全部添加或全部排除（persisted / stored_value 由 ask 处理，不复制写回逻辑），cancelled / error → 终止；ask 成功后按协议重读快照；
   - manual 模式不消费本规则。
6. **`.gitignore` 暂存集询问**：非空时展示差异摘要，一次 AskUserQuestion 批量询问（全部转移 / 全不转移 / 取消终止；仅本次、不写回，属流程内交互而非规则询问）；结论直接并入移动集或排除集，该路径不再进入 gitignore 信号、持久清单或三策略；取消 → 终止。
7. 按「优先级矩阵」合成最终 INCLUDE / EXCLUDE，执行 `git diff --binary BASE...FROM -- …` 输出到补丁文件（`git rev-parse --git-path git-extract.patch`，落在 git 目录、不产生未跟踪文件）：
   - pathspec 以进程参数数组传递，`:(exclude)` 为独立参数，不进 shell 字符串；
   - INCLUDE / EXCLUDE 均为空时不传 pathspec（取全部三点变更）；
   - 分类与推断先于 diff：diff 只见合成后的最终 pathspec。
8. 补丁为空 → 报告「无可转移更改」（含排除命中情况），删除补丁文件并正常结束；否则展示选择结果——**confirm 模式必须展示完整分类与推断结果**，auto 模式简要列出移动 / 排除清单与 gitignore 命中——并说明 from 侧未提交更改不在转移范围，再 AskUserQuestion 确认；取消 → 终止（删除补丁文件）。

## Step 4: 创建目标分支并原子应用

1. 执行 `git switch -c NAME BASE`，从基准分支拉出目标分支；失败 → 报告原始错误与现场并停止。
2. 执行 `git apply --index <补丁文件>`，检查退出码。
3. **失败 = 与 base 冲突**：按安全不变量停止——不 `--3way`、不 `--reject`、不回滚；保留已建分支与现场，报告补丁文件路径与清理方式（`git switch -` 切回原分支、`git branch -D NAME` 删除目标分支），由用户处置。
4. 成功 → 删除补丁文件，暂存区就绪，进入 Step 5。

## Step 5: 委托提交

1. 调用 `/git-commit`；带 `--message` 时传入，省略时由 git-commit 按 `commit_message_mode` 等规则自行处理。
2. 本技能不实现暂存分析、预检、消息风格、远程策略——全部由 git-commit 消费其自身规则。
3. 委托返回 cancelled 或失败 → 终止并如实报告：分支已建、补丁已在暂存区，用户可手动完成提交。

## Step 6: 报告

- 分支名、基准（BASE）、来源（FROM）、新提交哈希；
- 转移文件数与清单（超 20 条截断报总数）；排除统计按 B 类 / gitignore 信号 / 持久清单 / 显式参数分列；
- C 类策略与本次选择（含 `new_files_mode` 是否被消费、`.gitignore` 闸结论）；
- 提交是否已被 git-commit 顺带推送——same_repository 模式下 git-commit 可能把目标分支顺带推上远端（其既有行为），本技能不加开关，如实报告；
- 建议下一步：创建 PR 并 squash 合并到基准分支（`/git-pr` 落地后可直接使用，当前阶段可手动创建）。

## 选择逻辑协议

### 三类文件模型

对 `BASE...FROM`（三点 diff）中的每条路径，按 base / merge-base / from 三参照点分类：

| 类 | 判定 | auto / confirm 下的处理 |
| --- | --- | --- |
| **A 类｜base 已有** | base 跟踪该文件 | 变更内容**转移** |
| **B 类｜分叉前有、base 已删** | merge-base 有、base 现无、from 侧有变更 | **判私有，排除**（判定基准 = merge-base） |
| **C 类｜此次完全新增** | merge-base 无、base 无、from 有 | 依次过两道过滤（base `.gitignore`、持久清单），剩余交三策略 |

判定方法：用 `git ls-tree -r --name-only` 分别取 BASE、merge-base、FROM 的文件清单做集合运算，不对单条路径逐个查询。三点 diff 天然偏向该模型：B 类中 from 侧未改动的文件、A 类中 from 侧未改动的文件都不会出现在 diff 中——进 diff 的都是 from 侧真实动过的路径，无需额外处理。

### auto / confirm 推断流程

1. 取分离 `.gitignore` 自身变更后的 `BASE...FROM` 路径清单，按三类分类；
2. A 类 → 移动集；B 类 → 排除集（判私有）；
3. C 类 → 过 base 的 `.gitignore` 信号：**base 未跟踪 ∧ 被 base 忽略 → 排除**；
4. C 类剩余 → 过持久清单 `extra_excludes`（任何模式均参与）；
5. 仍剩余的 C 类 → 交三策略；
6. 按优先级矩阵合成最终 INCLUDE / EXCLUDE。

判定「被 base 忽略」必须基于 base 树中的 `.gitignore`（各层级）而非当前工作区；可用临时 worktree checkout BASE 后执行 `git check-ignore`，用完清理。无法判定时按「未忽略」处理——不误排除，交后续流程。base 无 `.gitignore` 时信号贡献为空：三类分类照做、C 类直接过清单与三策略，不存在独立回退分支。gitignore 信号只作用于 C 类：A 类已被 base 跟踪（gitignore 对已跟踪文件无意义）、B 类已判私有。

### C 类三策略（new_files_mode）

| 值 | 语义 |
| --- | --- |
| `prompt`（默认 + 控制值） | 每次选择——展示 C 类清单后经规则询问决定本次动作；需要部分选择时取消本次，改用显式 INCLUDE / EXCLUDE 重跑 |
| `all` | 全添加——剩余 C 类全部进入移动集 |
| `none` | 全不添加——剩余 C 类全部排除 |

延迟消费见 Step 3 第 5 条；manual 模式不消费本规则。

### 优先级矩阵

1. **显式 `INCLUDE` 覆盖一切自动推断**——gitignore 信号、B 类判私有、持久清单均可被显式包含推翻（参数是显式意图，推断是启发式）；
2. **显式 `--exclude` 与 `INCLUDE` 同路径 → 排除优先**（沿用 git pathspec 语义，不报错）；
3. **`extra_excludes` 不受 selection_mode 门控**——任何模式下都参与排除；与显式 INCLUDE 同路径时服从第 1 条（本次参数覆盖持久配置）。

### .gitignore 自身变更闸

路径名为 `.gitignore` 的变更（任意层级、含修改与删除）是知识源文件本身，不走常规推断：Step 3 第 2 步先行分离，第 6 步展示差异摘要后批量一次 AskUserQuestion（全部转移 / 全不转移 / 取消终止；仅本次、不写回），结论直接并入移动集或排除集——该路径不再进入 gitignore 信号、持久清单或三策略，避免重复询问与推断覆盖。

### 核心机制（diff + apply 要点）

- `BASE...FROM`（三点）只含 from 侧改动；两点 diff 会把 base 侧新改动反向还原，**不能用**。
- `:(exclude)` 是 pathspec magic，作为独立参数传递（参数数组，不进 shell 字符串）；同路径冲突与覆盖语义统一见优先级矩阵。
- `--binary` 覆盖二进制文件；diff 默认开启重命名检测，apply 可应用。
- `--index` 同时写暂存区——成功即 git-commit 可直接处理。
- 排除路径根本不进补丁（零泄漏）；全有或全无，失败即停。
- 不采用 merge 后还原排除路径（排除内容先进入历史，事后 revert 消不掉，违反「排除本地私有文件」初衷）；cherry-pick 序列留待未来 `--mode commits`。

## 错误处理原则

以下约束适用于全流程，不被任何分支覆盖：

| 触发 | 动作 |
| --- | --- |
| 存在未完成 merge / rebase / cherry-pick / revert | 停止，报告状态 |
| tracked 工作区变更非空 | 停止，提示自行提交或 stash |
| 参数非法 / 引用不存在 / base 与 from 相同 / 目标分支已存在 | 停止，报告原因 |
| manual 模式零 pathspec | 停止，报告并提示改用 auto / confirm 或给出 pathspec |
| `selection_mode` / `new_files_mode` 的 ask 返回 cancelled 或 error | 终止 |
| 三规则 value 缺失 / 非法且 repair 失败 | 停止（快照协议既有） |
| `extra_excludes` 运行时解析失败 | 停止，报告原字符串 |
| `.gitignore` 自身变更询问被取消 | 终止 |
| 补丁为空 | 报告无事可做，正常结束 |
| 用户取消确认 | 终止 |
| `git apply` 失败 | 停止，保留分支与现场，报告清理方式（不自动回滚） |
| git-commit 委托取消或失败 | 停止，报告「分支已建、补丁在暂存区」 |

其余：认证、权限、网络、hook 错误原样报告，不自动重试；本技能不涉及强推与远程写入，永不执行裸 `--force`、自动 continue / abort / reset。全局安全不变量见根 CLAUDE.md「安全不变量」。
