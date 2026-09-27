# git-commit / git-rules SKILL.md 结构与抽取分析

- 分析对象：`.claude/skills/git-commit/SKILL.md`、`.claude/skills/git-rules/SKILL.md`、`.claude/git-claude-rules.json`
- 快照基线：commit `06a32b0`。**本文档引用的原始问题行号均以该版本为准**；状态于 2026-09-27 按 commit `d3a0b45` 复核，当前实现行号不再沿用基线行号。
- 目的：找出文档内的重复步骤，判断哪些部分适合拆分为独立 skill、哪些适合收敛为 hook

## 状态总览

| 条目 | 状态 |
| --- | --- |
| 一 · 1 `Protocol` 段与 Step 1b 重复 | 已删除 |
| 一 · 2 冲突检测执行两遍 | 已删除第二处 |
| 一 · 3 `pre_check_mode` 双向跳转 | 已合并为解析表 |
| 一 · 4 标签表与 JSON 漂移 | 已消除（以 SKILL.md 文案为准） |
| 一 · 5 配置询问模式重复四次 | 已收敛为 `git-rules` 的 `ask` 分支 |
| 一 · 6 `same_repository` 语义两份 | 已归 `git-rules` |
| 一 · 7 错误处理原则与正文重叠 | 已收敛为补充清单 |
| 二 询问定义外置 | **已实施**：按规则拆到 `references/ask/`，并增加缺失/不一致时硬失败校验 |
| 三 三项 PreToolUse hook | **未实施** |
| 五 · 3 attribution 张力 | **未处理** |
| 五 · 5 多 remote 超出 AskUserQuestion 选项上限 | **待决策** |
| 六 版本对齐（version 比对 + 通用对齐算法） | **已实施**：新增 `calibrate`，首次读取配置时校验一次 |

行数变化：

| 文件 | 基线 `06a32b0` | 当前 |
| --- | --- | --- |
| `git-commit/SKILL.md` | 299 | 274 |
| `git-rules/SKILL.md` | 270 | 343 |

`git-rules` 当前比基线增加 73 行，主要来自版本校准协议、按规则分派的 `ask_mode`、映射文件校验和更完整的分支契约。此前“净减 9 行”的统计只适用于 2026-09-21 的中间状态，现已失效。

---

## 一、重复的步骤（七项，均已处理）

### 1. `Protocol: 规则自愈机制` 是 `Step 1b` 的重述

`SKILL.md:288-299` 的 8 步协议（解析根目录 → 检查特殊状态 → init → repair → repair SCOPE KEY → 重读校验 → 计算策略 → 执行）与 `SKILL.md:27-46` 的 `Step 1b` 完全同构，且未提供任何额外信息。

**实际处理**：整段删除。它不承载独立契约，`Step 1b` 已经是完整的执行路径。

### 2. 冲突检测执行了两遍

`SKILL.md:22-25`（`Step 1a`）检查 `git diff --cached --name-only --diff-filter=U` 与 `MERGE_HEAD` / `CHERRY_PICK_HEAD` / `REVERT_HEAD` / `REBASE_HEAD`；随后 `SKILL.md:64`（`Step 2a` 第 6 点）在暂存操作后"重新检查清单为空和未解决冲突"。

第二处实际**不可达**：`Step 1a` 遇到冲突已终止，而 `git add --all` 不会产生冲突。

**实际处理**：只保留空清单检查。空清单检查仍必要——工作区本身干净时 `git add --all` 加不进任何东西。

### 3. `pre_check_mode` 的状态机双向跳转写了两遍

- `SKILL.md:72`：unconfigured 分支声明"用户选择 prompt 时，跳跃到 prompt 流程"
- `SKILL.md:76`：prompt 分支声明"用户选择 unconfigured 时，跳跃到 unconfigured 流程"

两个方向的跳转各写一次，读起来像互相递归，实际是同一模式解析逻辑的两个入口。

**实际处理**：合并为一张「当前值 → 动作」表，五种取值各占一行，跳转关系只在行内出现。

### 4. 标签表与 JSON 配置重复，且已经漂移

`SKILL.md:78-84` 硬编码了一份中文标签，而 `git-claude-rules.json` 的 `labels` 字段才是数据源。两份已经对不上：

| value | JSON `labels` | SKILL.md:78-84 |
| --- | --- | --- |
| unconfigured | 需要初始化 | 设置默认行为 |
| disabled | 不检查 | 不检查 |
| warn | 警告但继续 | 检查仅警告 |
| block | 阻止提交 | 检查并阻止 |
| prompt | 询问 | 每次询问 |

五项里只有 `disabled` 一致。

**原建议**：以 JSON 为准，SKILL.md 改为引用 `labels`。

**实际处理**：方向相反——**以 SKILL.md 的文案为准**，更新了 JSON 的 `labels`，SKILL.md 删除硬编码表改为从 `rules["git-commit"].pre_check_mode.labels` 读取。理由是 SKILL.md 那套文案是专为 AskUserQuestion 写的，比 JSON 里的更具体（「检查仅警告」比「警告但继续」更清楚地说明动作）。两份副本的问题由「引用而非复制」解决，与选哪份为准无关。

### 5. "询问 → 是否保存 → set → 重算"模式重复四次

分别出现在 `SKILL.md:56-59`（`empty_staging_mode`）、`:72`（`pre_check_mode`）、`:172`（`commit_message_mode`）、`:239`（`repository_relation_mode`）。每次都是"介绍选项 → AskUserQuestion → 判断是否'总是使用' → 内部调用 `/git-rules set` → 重新读取并计算"。

**原建议**：抽为文档内的一次性共享小节。

**实际处理**：抽为 `git-rules` 的独立分支 `### ask SCOPE KEY`。它比文档内小节更好，因为 `git-rules` 已经持有 rules 数据模型、`labels` 和写回契约，共享小节仍要反过来调它。调用点只负责检查返回状态并使用 `effective_action`；候选项和映射由 `git-rules` 按规则加载，不再使用早期的 `--only` 参数。

**边界（后补）**：`ask` 只在「首次配置」或「允许用户改变持久行为」时适用。语义上要求「每次都询问」的取值不得用它——它提供的「总是使用」会破坏那个语义。

### 6. `same_repository` 的语义在两份文档中定义

`SKILL.md:247-252`（`Step 5b`）展开了 `managed_safe_sync` 的行为规则，而 `git-rules/SKILL.md:136-152` 的 Policy inheritance 已经定义了同一套语义，且 `git-rules` 的 Call contract 已包含 `resolve remote policy` 这一职责。

**实际处理**：语义归 `git-rules`。`git-commit` 的 `Step 5b` 和 `Step 1b` 都改为一行引用（`Step 1b` 原本也重复了覆盖语义，是同一个问题的第二处）。`Step 5c` 的**执行步骤**保留在 `git-commit`——那是动作，不是语义。

### 7. 错误处理原则与正文内联规则重叠

`SKILL.md:280-286` 的"不自动 reset / rebase / pull""用户取消则终止""规则写入失败则停止"等条目，已分别在 `:100`、`:138`、`:211` 等处单独声明过。

**原建议**：保留汇总作为索引，正文各处改为引用它。

**实际处理**：方向相反——**保留正文内联声明，把末尾汇总收敛**为只含「跨切面、不被任何分支覆盖」的三条（认证/网络/hook 错误、规则写入失败、远程不安全时停止写入），并加一句指回正文。让正文去引用一个远处的汇总会损害就地可读性，而汇总的价值恰恰在于它是「无法绕过的约束清单」。

---

## 二、结构抽取的取舍

原分析列出了四个抽取候选。**这部分的前提在 2026-09-21 被推翻**，先记结论，再看候选。

### 前提修正：拆分 skill 不省 token

原分析把「拆成独立 skill」和「省 token」绑定在一起，这是错的。核实后的机制见第三节，结论是：

- 拆成更多 skill **不减少**一次完整流程的上下文总量——总量是「被加载的正文之和」，拆开后是「编排壳 + 各子 skill 正文」，与单文件相当甚至略增；
- 反而增加常驻成本：每多一个 skill 就多一条 description，而 description 是**每一轮**都付费的；
- 整个 skill 列表共享模型上下文 1% 的预算，超出后按「调用最少」顺序**整条丢弃** description——丢掉的正是 Claude 判断该不该用这个 skill 的关键词。

所以下方候选的**正当理由是复用性和可维护性，不是 token**。

### 已采用的路径：按规则外置询问定义

v0.2.0 已采用 `references/ask/{scope}/{key}.md`，但只外置每条规则的提问文案、选项和动作/持久值映射；canonical schema、命令分派和通用询问协议仍保留在主 `SKILL.md`。

此前担心的“附属文件缺失后模型静默补写”已通过硬校验解决：`ask` 只读取当前规则对应的单个映射文件；文件缺失、不可读，或其中的 scope、key、ask_mode、控制值与 canonical 不一致时，返回 `status=error`，不询问也不写配置。新增可询问规则时，也明确要求同时创建对应映射文件。

当前共有 5 个询问定义文件，分别覆盖 `repository_relation_mode`、`empty_staging_mode`、`commit_message_mode`、`post_commit_push_mode` 和 `pre_check_mode`。这一实现接受“复制时必须携带完整 skill 目录”的分发约束，换取主文档与逐规则交互细节解耦。

### 候选清单（保留，理由已修正）

| 候选 skill | 基线位置 | 体量 | 可复用场景 |
| --- | --- | --- | --- |
| `/git-precheck` | `SKILL.md:66-138`（`Step 2b`） | ~70 行 | git-push / git-amend / PR 创建 / security-review |
| `/git-sync` | `SKILL.md:215-268`（`Step 5`） | ~55 行 | git-pull / git-push / git-fetch |
| `/git-message` | `SKILL.md:140-187`（`Step 3`） | ~48 行 | git-amend / squash / PR 描述生成 |
| `/git-context` | `SKILL.md:16-25`（`Step 1a`） | ~10 行 | 所有 git skill 的前置检查 |

`/git-sync` 是基线 `SKILL.md:256` 自己记录的 TODO（是否改为调用专用 `/git-fetch`）。那行已删除——当时它引用的 `scratch/remote-repository-policy.md` 尚不存在，且该行本身是开发期旁白而非执行指令。该文件目前已作为独立设计记录纳入仓库，但不再被 skill 运行时引用。

`/git-precheck` 的额外理由：它的输入输出契约干净（暂存集合 → 问题列表），且正则表与提交流程的变更理由不同。

### 已实施的 schema 原地压缩

`git-rules` 的 canonical schema 原本是一份 78 行的 pretty-printed JSON。其中绝大部分体积来自重复结构：每条规则的 `options` 各占一行、每个标签再各占一行。换成「一个通用骨架 + 一张六行的规则表」后减少 46 行，**且不新增任何文件**。

骨架部分特意保留字面形式不压缩——模型要据此写出精确的 JSON，可照抄比省几行重要。

**当前状态补充**：这项 schema 表格压缩仍然保留，但它不再是“外置”的替代方案；v0.2.0 同时采用了 schema 原地压缩和询问定义外置。由于新增校准及询问契约，`git-rules/SKILL.md` 的总行数已增至 343 行。

---

## 三、上下文与 token 的实际机制（核实结果）

依据官方文档（code.claude.com）核对，以下为文档明确的内容：

1. **会话启动时只有 name + description 进上下文**，正文不加载。未调用时，两个 skill 各只贡献约 55 字符。
2. **正文被调用时作为单条消息进入并常驻**；内容相同的重复调用只加一条短提示，**不会追加第二份副本**（v2.1.202+）。参数变化或动态注入输出变化时才会再追加。
   - 推论：`git-commit` 一个会话里调用 `git-rules` 多次（init / repair / ask），`git-rules` 的正文**只付一次**。此前担心的「反复调用累积消耗」不成立。
3. **正文常驻本身就是持续成本**——压缩（`/compact`）后每个被调用过的 skill 会重新附加，但**只保留前 5,000 tokens**，超出部分截断时**保留文件开头**。因此越靠后的内容在长会话里越可能被截掉，关键指令应前置。
4. **附属文件不随 SKILL.md 加载**：`reference.md` 按需读取，`scripts/` 只执行、内容不进上下文（但脚本 stdout 会进）。
5. **description 每一轮都付费**，单条与 `when_to_use` 合并后截断于 1,536 字符；整个列表的预算是模型上下文的 1%。诊断手段是 `/context` 的 Skills 行与 `/doctor`。

文档未明确、不要当结论的三处：skill→skill 嵌套调用的注入/计费语义（第 2 条的推论依赖通用生命周期规则）；附属文件由哪个工具读取；常驻正文与 prompt caching 的交互。

**可操作的判据**：文档给的上限是 SKILL.md 不超过 500 行，两个文件都在限内。真正该做的是——不重复（第一节）、把「只在特定分支才用得到」的大块数据压紧、关键指令前置。

---

## 四、适合做成 hook 的部分（未实施）

判据：hook 由 harness 执行，**不依赖模型记得去读 skill**。当前文档里所有"不自动执行 X"都是提示词层面的约束，模型换个说法就能绕过。hook 的文档成本为零（除非它返回内容），也天然不受本节的 token 讨论影响。

### 1. PreToolUse 守卫危险 Git 命令（建议优先）

拦截 `git reset --hard`、`git push --force`（放行 `--force-with-lease`）、`git clean -f[d]`、`git checkout -- .`，以及改写历史的 `git rebase`。

文档在 `SKILL.md:25`、`:211`、`:283` 反复承诺绝不自动改写历史，但没有任何机制保证这一点。**hook 才是那个保证。**

### 2. PreToolUse 拦截 `git commit --no-verify` / `-n`

目前完全没有防护。绕过 pre-commit hook 是明确的降级行为，适合硬拦。

### 3. PreToolUse 在 `git commit` 前执行 block 级别预检

把 `Step 2b` 中**阻止级别**的敏感文件名规则与 6 条阻止级别正则做成 hook，命中即以非零退出码阻止并把文件名与行号回传。

价值在于覆盖面：即使模型跳过 skill、或用户直接让 Claude 手工提交，密钥也进不去。

**边界**：仅 `block` 级别进 hook（`warn`、`prompt` 需要交互，留在 skill 层）；hook 是同步的，扫描须针对 `git show :PATH`（暂存版本），大型仓库需注意耗时。

### 4. SessionStart 校验规则文件

启动时检查 `git-claude-rules.json` 的合法性并提示。这样 `Step 1b` 中"配置损坏"的修复路径基本不会被走到。

### 不适合做 hook 的部分

- commit message 的生成与校验（需要模型判断）
- 所有 AskUserQuestion 交互点
- 规则配置询问

`PostToolUse` 自动执行 `git show --stat --oneline --summary HEAD` 并报告：技术可行但价值低，该步骤成本很低且模型会主动执行，不值得增加维护成本。

---

## 五、其他发现的问题

1. **死引用**（已解决）：基线中的 `SKILL.md:256` 曾引用当时不存在的 `scratch/remote-repository-policy.md`，执行指令中的该引用已删除。该文件目前已经存在，但只是独立的设计记录，不再是 skill 运行时依赖。
2. **拼写错误**（已解决）：`SKILL.md:78` 的 `AskUserQuesion` 已随标签表删除。
3. **attribution 规则张力**（未处理）：`SKILL.md:206` 要求"不要把你自己添加到作者或合作者一栏"，而 Claude Code 的默认 attribution 指引要求在 commit 末尾添加 `Co-Authored-By: Claude Code`。`Co-Authored-By` 与"作者栏"严格说不是同一回事，但两条规则存在冲突解读空间，建议写清究竟禁止哪一种。
4. **交互点未统一**（部分处理）：`Step 3a` 以 `$ARGUMENTS` 非空作为分支条件，`Step 5b` 又在 `unconfigured` 时插入询问，全流程没有交互入口清单。已在 `git-rules` 侧补上 AskUserQuestion 的适用约束，`git-commit` 侧未动。
5. **多 remote 超出选项上限**（待决策）：`SKILL.md:216` 的"多个可用 remote 时询问用户"没有数量上限，而 AskUserQuestion 单个问题最多 4 个选项。选项：(a) 最多列 4 个、其余让用户口述；(b) 改为打印 remote 列表让用户输入名称。

### 一条来自实践的教训

`git-rules` 无参数调用时，模型曾直接弹出 AskUserQuestion——因为 `Command dispatch` 里只写了"进入面向用户的交互式规则管理"，没有定义终止边界。已改为显式的只读分支「无参数调用」：展示配置 → 打印用法提示 → 说明可以口述修改 → 立即结束，并写明不调用 AskUserQuestion。

同一次排查还发现，早期 `ask` 分支本身也有同类缺陷：`pre_check_mode` 的状态值与持久化选择无法安全塞进单个问题。v0.2.0 已改为由 canonical 的 `ask_mode` 决定询问协议，并从当前规则专属的映射文件加载定义：`initialize` 使用一个问题，`inline_persistence` 把持久化意图放进动作选项，`separate_persistence` 使用一次调用中的两个问题。`pre_check_mode` 采用最后一种模式；调用方不再传 `--only`，也不复制候选值或映射。

**通用教训**：AskUserQuestion 的单问题 4 选项上限仍是硬约束。当前实现把它落实为 `git-rules` 的全局规则：每次只询问一条配置规则，并由该规则的 `ask_mode` 与映射文件保证每个问题不超过上限；多条规则不合并询问。

---

## 六、建议的落地顺序（修订）

### 已完成

1. **版本对齐**：v0.2.0 已新增 `calibrate` 分支和 `Version calibration` 通用算法。配置版本较低、缺失或非法时按当前 canonical 对齐；配置版本较高时不写入；未知旧规则保留；失效值回退默认值并报告。`git-commit` 的 Step 1b 已接入内部校准，并规定一次任务只在首次读取配置时比对一次。
2. **询问分派重构**：`ask_mode`、逐规则映射文件、文件一致性校验和统一返回契约均已落地，旧的 `--only` 收窄方案已移除。

### 剩余优先级

1. **两项 PreToolUse hook**：危险命令守卫、`--no-verify` 拦截。把提示词承诺变成机制约束。
2. **block 级别预检 hook**：依赖 `/git-precheck` 的抽取或至少共享同一套 patterns。
3. **`/git-sync` 与 `/git-message` 的抽取**：互相独立；理由是可复用性，不是 token。
4. **收尾项**：澄清 attribution 规则；为多 remote 选择设计不超过 4 个选项的交互；整理 `git-commit` 仍然内联的 AskUserQuestion 入口。
