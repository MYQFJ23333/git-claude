# 发布流程抽象为四个通用基础技能（0.4.0 设计方案）

设计日期：2026-10-11

状态：方案已讨论定稿，等待正式实施指令；本文件属本地私有设计文档（`workspace-sync` 跟踪、`main` 忽略），不是运行时依赖。

来源：从根 CLAUDE.md:41-47 约定的五步发布流程中抽象拆分，目标是产出四个能在大部分项目通用的基础技能——与现有 `/git-commit`、`/git-push` 同构的多步流程技能，而不是为本项目流程定制的新技能。

## Context

git-claude 是一整套装件（git-rules + 全部技能），会整体安装到其他项目；「通用」指套装装入正经工作项目后通过 git-rules 的 init/ask/set 适配该项目。因此新技能**深度接入 git-rules**（快照协议、ask 协议、canonical 规则表），与现有技能完全一致，不存在无配置环境的降级分支。

套装版本 0.3.1 → **0.4.0**（新增 skill 属 minor，CLAUDE.md:31）。

**已拍板决策**（不再改动）：

1. 粒度：四技能（准备 / PR 合并 / tag+Release / 回流），每个多步流程；
2. 配置：深度接入 git-rules，整套装件一起分发；
3. 平台：先 Git + GitHub（gh CLI），其他平台后期再议；
4. 本项目自身发布流程**暂不迁移**（CLAUDE.md:41-47 保持文字描述）；
5. A 命名 `git-extract`（弃 git-prepare/git-release-branch）；
6. D 命名 `git-sync`（吸收蓝图名，语义重定义为 merge 回流）；
7. A 应用补丁后**内嵌委托 `/git-commit`** 完成提交（接受 same_repository 下顺带推送的既有行为，报告中如实说明）。
8. A 的选择逻辑采用**三类文件模型**（A 类 base 已有 / B 类分叉前有 base 已删 / C 类完全新增）与**三规则**（`selection_mode` / `new_files_mode` / `extra_excludes`），**推翻原「A 零新规则」**；base 的 `.gitignore` 为 C 类排除信号，`.gitignore` 自身变更单独询问——详见 `scratch/designs/git-extract-design.md` 第 3/6 章。

## 总架构原则

1. **既有职责一律委托，不内建**：提交 → `/git-commit`（A）；分支推送 → `/git-push`（B、D）。分支级远程写入的完整实现只保留 git-push 一处，杜绝 managed_safe_sync 出现第四处副本（反面教材：git-commit Step 5c、git-push Step 3b/4、git-rules Policy inheritance 三处重复，scratch/reviews/git-commit-skill-review.md:88-92）。
2. **新语义单点定义**：tag 推送、diff/apply 转移、PR 生命周期、回流合并是无既有归属的新操作，在新技能内单点定义。
3. **四技能彼此不调用**：仅有的跨技能委托为 A→`/git-commit`、B→`/git-push`、D→`/git-push`。状态靠参数与 git 检测传递（B 默认 head=当前分支即 A 的产物、C 默认 target=基准分支 HEAD 即 B 的 squash 提交、D 默认 work=当前分支）。未来编排属 CLAUDE.md 层，本次不做。

## 四技能设计

全部遵循既有模板：frontmatter 4 字段（name / 两段式 description / argument-hint / license: MIT）；线性步骤流（仿 git-push 五步粒度，Step 下分 `### Step Na`）；文末「错误处理原则」节只收跨切面不变量。Step 1a 环境检查与 Step 1b 规则快照以 git-push/SKILL.md:12-43 为**同构模板基**——协议文字照搬，**各技能必须显式声明自身差异**（如 A 的 tracked 工作区变更即停、各技能缓存自身规则值），不得以「逐字同构」一笔带过；漂移靠 /review-skill 兜底。

### A：`git-extract`

- **职责**：从基准分支拉出目标分支，把另一分支的更改按路径选择性搬移进来（显式 pathspec 或 auto/confirm 自动推断）。主用例：发布准备。
- **argument-hint**：`"[--base BRANCH] [--from BRANCH] [--name BRANCH] [--message MSG] [INCLUDE_PATHSPEC...] [--exclude PATHSPEC...]"`（pathspec 完备性**仅 manual 模式强制**；auto/confirm 可零参数自动推断）。
- **步骤（6 步）**：1 环境与快照（缓存 git-extract 三规则，延迟消费）→ 2 解析参数与引用（base=默认分支探测、from=当前分支、name 必给或询问；selection_mode 解析后消费、pathspec 完备性仅 manual 检查；同名分支已存在即停）→ 3 分类与推断（三类分类 → `.gitignore` 自身变更先行分离、批量单独询问 → base `.gitignore` 信号 → `extra_excludes` → 三策略，按优先级矩阵合成 pathspec 后三点 diff；空差异即报告结束，confirm 模式展示分类与推断结果并确认）→ 4 建分支 + `git apply --index` 原子应用（失败=冲突，按安全不变量停止，不自动 --3way/--reject/回滚）→ 5 **委托 `/git-commit`** 提交（传 --message；省略时由 git-commit 按其规则生成）→ 6 报告（分支、提交哈希、转移/排除分类统计、C 类策略与本次选择、release 分支是否已被顺带推送、建议下一步 `/git-pr`）。
- **机制选型（已定）**：三点 diff（`BASE...FROM` 只含 from 侧改动，两点 diff 会反向还原 base 侧改动——不能用）+ pathspec 排除（私有文件根本不进补丁，零泄漏）+ `git apply --index`（全有或全无）。**否决 merge 后还原排除路径**（私有内容先进历史，事后 revert 消不掉）；cherry-pick 序列留作未来 `--mode commits`，v1 不做（squash 合并会抹平提交粒度）。

### B：`git-pr`

- **职责**：创建 PR（标题按模板生成）并 squash 合并到基准分支。
- **argument-hint**：`"[--head BRANCH] [--base BRANCH] [--title TITLE] [--delete-branch|--keep-branch]"`。
- **步骤（6 步）**：1 环境与快照 → 2 解析目标（标题优先级 `--title` > 从 head 分支名按 `release/vX.Y.Z` 模式解析版本套模板 > 询问；`gh --version` + `gh auth status` 探测；`gh pr list --head` 查已存在 PR）→ 3 检出 head 分支（工作区必须干净）并**委托 `/git-push`** 推送 → 4 `gh pr create`（已存在则报链接跳 5；认证/权限/网络错误原样报告不重试）→ 5 展示合并计划并确认后 `gh pr merge --squash`（删分支按参数或当场询问）→ 6 报告（PR 链接、merge 哈希、建议下一步 `/git-release`）。
- **注意**：git-push 的 LOCAL_REF 恒为当前分支（git-push/SKILL.md:113），故必须先检出才能委托——该约束写进步骤。分叉检测、首推确认、upstream、force-with-lease 全归 git-push，B 不碰。

### C：`git-release`

- **职责**：按需创建 annotated 版本 tag 并创建 GitHub Release（prerelease 属性可配置）。蓝图的 semver 自动计算与 CHANGELOG **不吸收**（版本由参数提供）。
- **argument-hint**：`"[--version VERSION] [--target REF] [--tag TAG] [--notes TEXT] [--no-tag]"`。
- **步骤（6 步）**：1 环境与快照（**缓存 `/rules/git-release/prerelease_mode/value`**，缺失/非法走 `repair git-release prerelease_mode --internal`，只缓存不预先验证）→ 2 解析发布目标（tag 默认 `v{version}`、目标默认基准分支 HEAD，展示计划并确认）→ 3 按需建 annotated tag（本地已存在→询问复用或停止，不覆盖）→ 4 推送 tag（`git ls-remote --tags` 查远端，**已存在即停止，不覆盖、无任何 force 路径**；`git push REMOTE refs/tags/TAG`）→ 5 创建 Release（此处才解析 prerelease_mode：控制值触发 `/git-rules ask git-release prerelease_mode`；`gh release create TAG --title --generate-notes [--prerelease]`；gh 缺失/认证失败→tag 已推送则如实报告后停止）→ 6 报告（tag、Release 链接、prerelease 属性来源、建议下一步 `/git-sync`）。
- **不委托 git-push 的理由**：git-push 整套流程（heads 存在性、upstream、ahead/behind、force-with-lease）对 tag 全部失义；C 内置的是更弱、无 force 的单 ref 推送，单点定义。

### D：`git-sync`

- **职责**：把基准分支 merge 回工作分支并安全推送，使工作分支包含基准分支全部内容。主用例：发布后回流。
- **argument-hint**：`"[--work BRANCH] [--base BRANCH]"`。
- **步骤（6 步）**：1 环境与快照 → 2 解析目标（work=当前/参数、base=默认分支探测/参数，必须不同且存在；检出 work 且工作区干净）→ 3 更新基准分支（非强制 refspec `git fetch REMOTE refs/heads/BASE:refs/heads/BASE` 仅快进；分叉即停，不写入）→ 4 `git merge BASE`（base 已是祖先→报告无需合并；冲突→停止，不自动 continue/abort/reset/rebase）→ 5 **委托 `/git-push`** 推送（被取消/失败→报告「本地合并已保留、未推送」）→ 6 报告（`merge-base --is-ancestor` 判定、合并提交、推送结果）。
- **边界**：merge 提交不走 git-commit（git 默认消息的机械操作，无暂存分析/预检诉求）；D 不实现分叉检测/强推/upstream。

## git-rules 集成（最小必要集）

**新增 4 条规则**（canonical 表 git-rules/SKILL.md:99-106 表尾 **+4 行**）；ask 映射 **+3 个**（C 1 个 + A 2 个，`extra_excludes` ask_mode=none 无映射）。

**C：git-release**

| scope | key | 默认 value | options（value=标签） | 控制值 | ask_mode | description |
| --- | --- | --- | --- | --- | --- | --- |
| git-release | prerelease_mode | prompt | prompt=每次询问、prerelease=标记为预发布、stable=正式发布 | prompt | inline_persistence | 创建 GitHub Release 时的 prerelease 属性 |

- ask_mode 取 `inline_persistence`：恰好 4 选项（仅本次预发布/仅本次正式/总是预发布/总是正式），对齐 `post_commit_push_mode` 先例；控制值 prompt 触发询问、不入候选。

**A：git-extract（草案，规则定义与格式依据详见 `scratch/designs/git-extract-design.md` 第 6 节）**

| scope | key | 默认 value | options（value=标签） | 控制值 | ask_mode | description |
| --- | --- | --- | --- | --- | --- | --- |
| git-extract | selection_mode | unconfigured | auto=自动推断、manual=仅按参数、confirm=展示并确认 | unconfigured | initialize | 路径选择模式（三类分类与 .gitignore 推断） |
| git-extract | new_files_mode | prompt | prompt=每次询问、all=全添加、none=全不添加 | prompt | inline_persistence | C 类非忽略新文件的处理策略 |
| git-extract | extra_excludes | （空字符串） | 无 | 无 | none | 持久排除清单（逗号分隔 pathspec，运行时解析，任何模式均参与） |

- **B/D 零新规则**：分支名、PR 标题、删分支偏好全由参数与 git 检测决定；远程策略经 git-push 解析，B/D 不缓存 repository_relation_mode。A 的三规则持久化的是**模式、策略与显式清单**，不是推断结果——推断结果由仓库自描述（base 的 `.gitignore`）动态产生，不落配置。

**配套文件**：

- 新建 `.claude/skills/git-rules/references/ask/git-release/prerelease_mode.md`（否则 ask 返回 error，CLAUDE.md:91）——格式参照现有 `references/ask/git-commit/post_commit_push_mode.md`：头四行元数据 + 单问题 + 恰好 4 行选项表（展示选项/说明/effective_action/stored_value）+ 尾注。
- 新建 `.claude/skills/git-rules/references/ask/git-extract/selection_mode.md`（initialize，恰好 3 选项；选项值 auto/manual/confirm，**不得使用 `prompt`**）与 `.claude/skills/git-rules/references/ask/git-extract/new_files_mode.md`（inline_persistence，恰好 4 行——仅本次添加/仅本次不添加 stored 空、总是添加→all、总是不添加→none）。`extra_excludes` ask_mode=none，**不需要**映射文件。
- git-rules/SKILL.md 另改两处：
  - **Policy inheritance**（:110-126）补三行声明：① 远程分支写入（含 git-pr/git-sync 的推送）一律经 `/git-push` 解析有效策略，新技能不持有自有远程字段；② `git-release` 的 **tag 推送不受 repository_relation_mode 管辖**（无 upstream/分叉语义），安全判定为无条件规则（远端同名 tag 即停、永不 force）；③ `git-extract` 三规则为本地路径选择语义，同样不受 repository_relation_mode 管辖。
  - **Call contract**（:330-342）补一句：委托 `/git-push` 执行远程写入即视为遵循有效远程策略，委托方不自行缓存 repository_relation_mode。
- init/repair/calibrate/import 从 canonical 表自动生效，分支正文不改。

## 共享语义单一定义（防第四处重复）

| 语义 | 唯一位置 | 引用方式 |
| --- | --- | --- |
| 配置快照协议 | git-rules/SKILL.md:61-71；版本权威 :130-132 | 新技能 Step 1b 同构执行（协议文字照搬 git-push/SKILL.md:27-36，缓存的 value 集合按各技能自行声明并显式标注差异） |
| ask 调用契约 | git-rules/SKILL.md:330-342 | 只传 SCOPE/KEY、检查 status、cancelled/error 终止 |
| 分支远程安全写入（managed_safe_sync） | 语义 git-rules:110-126；执行 git-push Step 3b/4 | B/D 委托不复制；A 经 git-commit 间接获得 |
| tag 推送安全判定 | git-release Step 4（新语义单点） | 仅 C 使用；Policy inheritance 声明不适用 |
| 发布分支命名 `release/vX.Y.Z` | git-extract Step 2（创建者定义） | B 从分支名逆解析版本时同构 ≤2 行 |
| PR 标题模板 `{version} update` | git-pr Step 2（唯一 PR 创建点） | 其他技能不涉及 |
| 默认基准分支探测 | git-extract Step 2（首个定义处） | B/C/D 同构内联 ≤4 行（symbolic-ref 探测，失败→询问不猜） |
| gh 探测（--version + auth status） | git-pr Step 2 | C 同构内联；A/D 不用 gh |
| 环境检查 Step 1a | 模板权威 git-push/SKILL.md:16-23 | 同构模板；差异须在技能内显式声明（如 A：tracked 工作区变更即停，与 git-push 的「不阻止推送」不同） |
| 版本号读取 | `.claude/skills/git-rules/VERSION` | **各 SKILL.md 写一句防混淆**：配置基准版本（决定校准）≠ 被发布物版本 `--version`（决定 tag/分支名/标题），不同源 |
| 路径三类分类与 base `.gitignore` 推导 | git-extract Step 3（设计期锚点 = git-extract-design.md 第 3 章） | 仅 A 使用；B/D 不做路径选择，其他技能需要路径过滤时引用本行，不复制推断逻辑 |

## 技能分层与演化吸收

套件技能分三层：**原语**（单命令 + 机械检查：未来的 git-branch / git-merge / git-fetch / git-tag）、**任务流**（多步编排 + 委托 + 安全停机点：git-commit、git-push 与本方案四技能）、**编排**（跨技能串联，属 CLAUDE.md 层）。四技能的不可吸收核心不是命令（已委托或可委托），而是编排与停机点——原语层不含停机点，故被吸收的唯一可能是某未来技能把整个任务实现为自己的一个模式。

| 技能 | 被吸收概率 | 不可吸收核心 | 最可能的吸收者 |
| --- | --- | --- | --- |
| git-extract | 低 | diff+pathspec+apply 搬移机制、三类分类与推断模型及其规则消费；确认与失败停机编排 | 无——方向相反：它未来可吸收 `--mode commits`，分支创建一步可委托未来 git-branch |
| git-pr | 高（四个中最薄） | 标题模板、已存在 PR 检测、合并确认 | 未来平台适配层（GitHub/GitLab 统一「提议合并」技能），本技能成为其 GitHub 前端 |
| git-release | 中 | 远端同名 tag 即停、无 force 路径（刻意拒绝 `gh release create` 一步带过） | 未来通用 git-tag 安全原语吸收 Step 3-4，prerelease 规则随迁 |
| git-sync | 低 | 非强制 fetch 策略、祖先判定幂等、回流验收标准 | Step 3 的 fetch 可委托未来 `/git-fetch`（见 `scratch/notes/remote-repository-policy.md`）——委托而非吸收 |

规律：越接近「原语组合」越易被吸收（git-pr），越接近「新机制任务流」越难（git-extract）。

**当前有意的收窄（演化钩子，非缺陷）**：git-pr 固定 squash 策略——适配 no-ff / rebase-merge 项目时加 `--strategy` 参数，不加规则；git-release 绑定 GitHub——多平台时重组为「平台无关的 tag 安全推送 + 平台前端」。

**吸收代价为何可控**：① 委托制使重资产已单点外置，搬家的都是薄编排；② 共享语义单一定义表使吸收只改引用不改语义；③ canonical 规则表 + calibrate 已内置规则迁移机制（废弃规则保留原值、不参与策略读取，ask 映射随 scope 搬迁）。

## 文件清单

**新增（7）**：

1. `.claude/skills/git-extract/SKILL.md`
2. `.claude/skills/git-pr/SKILL.md`
3. `.claude/skills/git-release/SKILL.md`
4. `.claude/skills/git-sync/SKILL.md`
5. `.claude/skills/git-rules/references/ask/git-release/prerelease_mode.md`
6. `.claude/skills/git-rules/references/ask/git-extract/selection_mode.md`
7. `.claude/skills/git-rules/references/ask/git-extract/new_files_mode.md`

（第 6–7 项为**设计先行登记**，实际创建属 0.4.0 实施阶段（阶段 1）；`extra_excludes` ask_mode=none 无映射文件。）

**修改（6）**：

1. `.claude/skills/git-rules/SKILL.md` — canonical 表 +4 行（:99-106）、Policy inheritance 三声明（:110-126）、Call contract 委托说明（:330-342）
2. `.claude/skills/git-rules/VERSION` — 0.3.1 → 0.4.0
3. `.claude/git-claude-rules.json` — 顶层 version → 0.4.0、补 `rules.git-release.prerelease_mode` 与 `rules.git-extract` 三条（经 `/git-rules calibrate` 生成核对，属正常变更）
4. `.claude/skills/git-help/SKILL.md` — registry 表追加 4 行（:45-49，**仅在对应 SKILL.md 落地后写入**，遵守 :51-57）；advice 分支问题类别（:107）补发布类归类
5. `README.md` — 技能清单（:10-13）、可配置行为（:15-21）、目录树（:25-38）、使用示例（:47-61）
6. `CLAUDE.md` — 「四个 skill」计数（:9）、分工表 +4 行（:11-16）、:77 scope 列举改「现有 scope 为 project、git-commit、git-extract、git-release」、常用命令验证清单（:20-24）。**不动** :41-47 发布流程节；:38-39 双轨分支表述已经 2026-10-11 核实与现实一致（workspace-sync 无 `.gitignore`，main 的 `.gitignore` 存在且含 `scratch/` 等私有文件条目），**不得以「.gitignore 已删除」为前提修改它**。

**不动**：`.claude/standards/*`、LICENSE、scratch/* 既有文件（设计文档的定稿回写属文档更新阶段，已于实施前完成，不计入实施阶段改动）。

## 实施顺序（每阶段独立提交、独立验证；提交用完整 Conventional Commits、无 scope，落 workspace-sync）

1. **阶段 1：git-rules 基座** — canonical +4 行 + ask 映射 +3 个（git-release 1 + git-extract 2）+ Policy inheritance/Call contract 声明 + VERSION 0.4.0 + calibrate 对齐。A、C 都依赖本阶段规则，基线先于消费方；技能未落地，registry 不动。
2. **阶段 2：A `git-extract`** — SKILL.md + registry 行 + README/CLAUDE.md 对应行（技能文件与同步义务同 commit）。纯本地、无 gh、无远程写入，风险最低，且是 B 的输入。
3. **阶段 3：B `git-pr`** — 同上。首次引入「委托 /git-push」与 gh，单独验证嵌套调用。
4. **阶段 4：C `git-release`** — 同上 + prerelease_mode 实际消费验证。依赖阶段 1 规则与阶段 3 的 squash 结果。
5. **阶段 5：D `git-sync`** — 同上。闭环最后。
6. **阶段 6：收尾** — 全套一致性核对（registry↔目录、README/CLAUDE.md 计数与 scope 表述、version=0.4.0）、`/review-skill` 复查全部改动文件。套装 0.4.0 的正式发布按 CLAUDE.md:41-47 **手工**执行（流程不迁移）；是否发布由用户另行决定。

## 验证（本仓库无 build/test）

- **每阶段通用**：`/review-skill <改动 SKILL.md> .claude/skills/git-rules/SKILL.md .claude/git-claude-rules.json` 只读审查——重点盯 Step 1a/1b 与 git-push 模板同构、错误处理原则只收跨切面项、无第二处 managed_safe_sync；对照「共享语义单一定义」表逐项检查。
- **阶段 1**：`/git-rules show` 见新规则（git-release 1 条 + git-extract 3 条）与 0.4.0；`/git-rules ask git-release prerelease_mode` 四选项、prompt 不入候选、写回正确；`/git-rules ask git-extract selection_mode` 三选项（unconfigured 不入候选、选项值为 auto/manual/confirm 不含 prompt）、`/git-rules ask git-extract new_files_mode` 恰好 4 选项且「仅本次」不写回；`set git-extract extra_excludes` 任意字符串可写入且无映射文件；反向验证：配置 version 改低→技能自动 calibrate、临时移走 ask 映射→`status=error`（验后还原）；`/git-help` registry 仍 3 行（未预写）。
- **阶段 2-5**：`/git-help` registry 行数与目录一致、detail 分支可读；**在临时克隆 + 本地 bare 远端 / 专用 GitHub 测试仓库演练**（绝不碰本仓库 main/远端）——A：私有文件零进入补丁（gitignore 信号）、B 类自动排除、`extra_excludes` 生效、显式 INCLUDE 覆盖持久清单、manual 零 pathspec 拒绝、auto 零参数运行、`.gitignore` 自身变更单独询问、apply 冲突即停、委托 git-commit 全链路（含 same_repository 顺带推送副作用）；B：git-push 委托首推确认、gh create/merge、PR 已存在分支；C：远端同名 tag 即停、prerelease 链路 prompt→ask→写回、gh 缺失降级报告；D：非强制 refspec 只快进、分叉/冲突即停、祖先判定为真。
- **阶段 6**：`/git-rules show` + VERSION 确认 0.4.0；三处清单（CLAUDE.md scope 列举、README 技能清单、git-help registry）与 canonical 表交叉核对。

## 非目标

- 不迁移本项目发布流程（CLAUDE.md:41-47 保持文字描述）；
- 不做 `--mode commits`（cherry-pick 序列）、semver 自动计算、CHANGELOG 生成；
- 不做 GitLab 等其他平台适配；
- 不把 delete_branch_mode、tag 前缀、PR 标题模板升格为规则（参数/默认值解决，升格即触发 canonical+ask 映射+版本联动的完整成本）；原「不把 release_exclude_paths 升格为规则」**已撤销**——0.4.0 已采纳为 git-extract 的 `extra_excludes`（连同 `selection_mode`、`new_files_mode`，A 共三规则，`release_exclude_paths` 即其设想前身）。重开理由：原反对仅覆盖「纯路径清单」场景，对「模式规则 + 仓库自描述（base `.gitignore`）推导」不成立，详见 git-extract-design.md 第 3/6 章；
- 不新建共享 reference 文件抽 Step 1 模板或基准分支探测（与「与现有技能完全一致」有张力，靠 /review-skill 防漂移）；
- 不动 git-help/git-rules 的非线性分派结构。
