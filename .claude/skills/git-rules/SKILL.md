---
name: git-rules
description: 管理 Git Skills 的项目级规则配置，支持查看、设置、重置、初始化、修复、校准、导出和导入 git-claude-rules.json
argument-hint: "[show|set <scope> <key> <value>|reset [scope] [--force]|init [--internal] [--replace]|repair [scope] [key] [--internal]|calibrate [--internal]|ask <scope> <key> [--only <value,...>]|export|import <file>]"
license: MIT
---

# git-rules

Git Skills 规则管理中心。它是一个命令分派器和配置服务，不要求所有操作经过同一条线性流程。先根据命令选择对应的操作分支，再使用该分支所需的状态检查、读写和输出规则。

<!--
特殊结构说明：本 skill 有意采用“命令分派 + 独立操作分支”的非线性结构。
它不是所有 Git skill 都必须套用的 SKILL.md 模板，也不应因为缺少统一的
Step 1 → Step 2 → Step 3 顺序而被判定为结构缺陷。审查时应分别验证各命令
分支的输入、前置条件、动作、失败处理和输出契约；只有跨分支共享的行为才放在
Shared data model、Policy inheritance 或 Call contract 中。
-->

## Command dispatch

从 $ARGUMENTS 获取命令：

- 无参数：只读展示当前配置和用法提示后结束，见「无参数调用」；
- show：显示当前规则；
- set SCOPE KEY VALUE：设置单项规则；
- reset [SCOPE] [--force]：重置指定作用域或全部规则；
- init [--internal] [--replace]：初始化配置文件；
- repair [SCOPE] [KEY] [--internal]：修复配置文件或指定规则；
- calibrate [--internal]：把配置对齐到基准版本；
- ask SCOPE KEY [--only VALUE,...]：向用户询问单项规则的值，统一处理「总是使用 / 仅本次」和写回；
- export：导出规则；
- import FILE：导入规则。

未知命令、参数缺失或参数格式错误时，报告用法并停止，不猜测用户意图。

本 skill 只在这两种明确声明的场景发起询问：ask 分支的单项规则询问、show 分支在配置缺失时的二选一确认。其余分支一律不询问，直接执行或只读展示后结束。

调用分为两种模式：

- 用户模式：显示操作过程；除上述两种场景外不发起询问，执行完直接报告结果；
- 内部模式：由其他 skill 调用，不展示初始化、修复和策略计算过程，只返回成功或失败结果。

--internal 只供内部调用使用。直接用户调用不得使用 --internal。

## Shared data model

### Configuration location and read policy

配置文件固定为 .claude/git-claude-rules.json。

- 需要读取配置的操作，先确认文件存在且可读取；
- 文件不存在时，用户模式可询问是否初始化，内部模式直接执行 init；
- 文件读取失败时停止，不修改、不覆盖原文件；
- JSON 非法或整体结构错误时，只有 repair 分支可以恢复；
- 单个规则缺失或 value 非法时，repair 分支可以只修复该项；
- 所有规则实际值位于 rules.{scope}.{key}.value；scope 为 git-commit 时使用 rules["git-commit"]；
- 修改规则时必须保留 options、labels、description、set_at 等元数据。

### Canonical schema

顶层结构固定为：

    {
      "version": "0.0.2",
      "created_at": "YYYY-MM-DD",
      "updated_at": "YYYY-MM-DD",
      "rules": {
        "<scope>": {
          "<key>": {
            "value": "<当前值>",
            "options": ["<选项>"],
            "labels": { "<选项>": "<中文标签>" },
            "description": "<说明>",
            "set_at": "YYYY-MM-DD"
          }
        }
      }
    }

顶层 version 既是配置文件写入的字段，也是判断是否需要校准的基准值，语义见 Project version。

可选的 type 字段用于类型化规则；没有 options 的规则省略 options 和 labels。

全部规则定义见下表，本表是 init、repair、calibrate 和 import 的唯一规范来源。`默认 value` 是 init 写入的值；`options` 列每项写作 `value=标签`，写入时拆为 labels 对象的键值对，选项顺序即展示顺序；`description` 列对应 description 字段。

| scope | key | 默认 value | options（value=标签） | description |
| --- | --- | --- | --- | --- |
| project | repository_relation_mode | unconfigured | unconfigured=需要初始化、independent_repositories=分别处理、same_repository=视为同一仓库 | 当前项目的远程仓库协作策略 |
| project | large_file_size_limit | 1MB | 无（type=file_size） | 单个暂存 blob 的大文件警告阈值 |
| git-commit | empty_staging_mode | prompt | prompt=询问、stage_all=暂存全部、abort=终止 | 暂存区为空时的行为 |
| git-commit | commit_message_mode | unconfigured | unconfigured=需要初始化、conventional_simple=简单 Conventional、conventional_full=完整 Conventional、custom=自定义 | commit message 风格 |
| git-commit | post_commit_push_mode | prompt | prompt=询问、always=总是、never=从不 | independent_repositories 模式下 commit 成功后的 push 行为 |
| git-commit | pre_check_mode | unconfigured | unconfigured=设置默认行为、disabled=不检查、warn=检查仅警告、block=检查并阻止、prompt=每次询问 | 提交前的预检行为（大文件检测、敏感文件名检查、敏感信息扫描等） |

### Policy inheritance

repository_relation_mode 是远程协作策略的上层控制项：

- unconfigured：首次涉及远程仓库时由对应 skill 询问；
- independent_repositories：使用各 skill 自己的远程字段；
- same_repository：使用统一 managed_safe_sync 策略。

在 same_repository 模式下，有效策略可以覆盖多个子字段：

- post_commit_push_mode 的有效行为为 managed_safe_sync；
- 远程动作前必须更新远程状态；
- 只有一个可用 remote 且缺少 upstream 时自动设置；
- 远端出现无法证明安全关系的新历史时停止自动写入；
- 未来新增远程字段必须声明是否受 repository_relation_mode 控制。

子字段原始配置值继续保留。切换回 independent_repositories 后，恢复使用这些值。

large_file_size_limit 接受正数加 B、KB、MB 或 GB，单位不区分大小写，按 1024 进位计算。默认值为 1MB。

### Project version

顶层 version 是 Git Skills 项目的版本号。基准值由本文件 Canonical schema 中的 version 字段给出，是本 skill 判断是否需要校准的唯一依据，不需要任何额外文件或外部来源。

配置文件中的 version 由 init 和 calibrate 写入，记录该配置是在哪个项目版本下核对过的。比对只看配置中的 version 与基准是否相等，不解释各级数字的含义。

### Version calibration

校准把配置对齐到基准版本。触发判定与对齐行为都不依赖版本号的语义。

是否需要对齐：

- 配置 version 高于基准版本：视为配置来自更新版本的项目，停止，不写入；
- 配置 version 缺失、非法或低于基准版本：需要对齐；
- 配置 version 等于基准版本：仍需校验规则清单，canonical 的 scope 与 key 集合和配置不一致时，说明规则表已变更而版本号未反映，按需要对齐处理；集合一致时才跳过对齐。

对齐一律直接执行，不发起询问，也不等待用户确认。对齐过程中的异常不中断对齐，改为记入报告：

- 补齐缺失规则、刷新 options、labels、description、type：正常完成，无需单独说明；
- 用户已选的 value 不在新候选集中：写回该规则的默认 value，并作为异常项记入报告，说明原值与新值；
- 配置 version 高于基准版本：无法推断对齐目标，不对齐、不写入，直接报告差异。

用户已选的 value 是否仍然有效，以 canonical 表中该规则的 options 为准；无 options 的规则按 type 校验。配置中存在但 canonical 已无的规则视为废弃，保留原值且不参与策略读取。

版本校验在一次任务中只执行一次，由该任务内首次读取配置的 ensure configuration 承担。校验结束后，本次任务内一律不再重复比对版本，也不再重复触发 calibrate；同一配置在任务中被多次读取时，直接使用既有的校验结果。只有 ensure configuration 和 calibrate 两个入口会比对版本，set、ask、show 等分支不自行比对。

## Operation branches

### 无参数调用

直接调用 /git-rules 而不带命令时，只做一次只读展示，然后结束：

1. 读取并校验配置，展示每条规则的方式与 show 分支一致；
2. 配置不存在、读取失败或整体结构错误时，报告状态和建议命令后结束，不自动执行 init 或 repair；
3. 打印用法提示，至少覆盖 /git-rules show、/git-rules set SCOPE KEY VALUE、/git-rules reset [SCOPE]、/git-rules ask SCOPE KEY 四种；
4. 说明用户可以直接口述要修改的规则，由当前会话代为调用 set，无需自己拼命令；
5. 展示完毕后立即结束，不追加任何确认步骤，也不询问是否要修改。

本分支不调用 AskUserQuestion，不询问是否要修改任何规则，也不进入交互式配置流程。

适用本 skill 全部分支的约束：规则集包含 6 条规则、每条 3 到 5 个选项，无法用单次 AskUserQuestion 表示，因为单个问题最多 4 个选项。需要改动多条规则时，由用户逐条调用或口述后由会话代为执行。

### show

1. 读取并校验配置；
2. 显示基准版本与配置中的 version，并在两者不一致时提示可用 calibrate 对齐；
3. 按作用域显示每个规则的当前 value、可选值、描述和默认状态；
4. 不显示未知字段中的原始内容；
5. 配置缺失时按用户模式询问是否进入 init。

### set

1. 确认配置存在、可读取且结构有效；
2. 校验 SCOPE 和 KEY；
3. 有 options 时校验 VALUE 在选项中；
4. type 为 file_size 时按统一单位规则校验；
5. 只更新对应规则的 value 和 set_at；
6. 更新 updated_at；
7. 使用临时文件写入并原子替换；
8. 重新读取并校验，失败时报告错误。

### reset

1. 确认配置存在且结构有效；
2. 用户模式下，除非使用 --force，否则先确认；
3. 指定 SCOPE 时只重置该作用域；
4. 未指定 SCOPE 时重置全部规则；
5. 保留当前 version 与 created_at，只更新被重置规则的 value、set_at 和文件的 updated_at；
6. 使用 set 的原子写入和重新校验规则；
7. 内部调用不得用 reset 代替单项 repair。

### init

1. 确认 .claude 目录存在，不存在则创建；
2. 文件已存在时不覆盖并报告已存在；
3. --replace 只允许 repair 在完成恢复记录后内部使用，直接用户调用必须拒绝；
4. 生成包含全部 canonical 默认规则的配置，其中 version 写入 Canonical schema 中的基准版本；
5. --internal 时不提示用户进入交互式配置；
6. 用户模式下可在初始化后询问是否配置常用规则；
7. 写入后重新读取并校验。

### repair

repair 是配置修复分支，不是所有命令的必经步骤。

#### 文件不存在

文件不存在时，repair 等价于内部 init，生成默认配置并重新读取校验；不存在旧内容时不创建恢复记录。

#### 单项缺损或非法

当整体 JSON 和 rules 结构可读取，但规则缺失、value 缺失、类型错误或 value 不在 options 中时：

1. 读取 canonical 默认定义；
2. 只重新初始化指定的 SCOPE/KEY；
3. 保留其他规则和元数据；
4. 更新该项 set_at 及文件 updated_at；
5. 写入后重新读取并校验。

未指定 SCOPE/KEY 时，扫描所有已知规则并逐项修复。

#### 整体结构错误

当 JSON 无法解析，或 rules、scope、rule object 等整体结构错误时：

1. 先确认原文件可读取；读取失败立即停止，不做任何写入；
2. 使用容错分析提取已知 scope/key 中结构清晰且 value 合法的候选值；
3. 将原始内容、错误原因、时间和候选值保存到由 git rev-parse --git-path git-claude-rules-recovery 取得的 Git 私有目录下的 repair-TIMESTAMP.json；目录不存在时先创建；
4. 恢复记录无法保存时停止，不覆盖原配置；
5. 使用内部 init --internal --replace 初始化全新的默认配置；
6. 将恢复候选值逐项加载到新配置；
7. 重新读取并校验最终配置。

恢复记录保留原始文本和候选值，但普通输出不得打印原始内容。未知字段不自动迁移，只有 canonical 结构中的合法值可以恢复。

任意读取、分析、恢复记录写入、初始化、覆盖或最终校验失败，都报告具体阶段并停止。

### calibrate

把配置对齐到基准版本，只处理结构完好但版本或规则清单与当前项目不一致的情况。结构损坏由 repair 处理，本分支不自行恢复。

1. 确认配置存在、可读取且结构有效；结构无效时转 repair，不在本分支内恢复；
2. 按 Version calibration 判定是否需要对齐；不需要对齐时不写入，直接报告当前版本已是最新；
3. 配置 version 高于基准版本时停止，报告差异，不写入；
4. 需要对齐时，逐条处理 canonical 中的规则：
   - 配置中缺失：按该规则的默认 value 与完整元数据补齐，写入新的 set_at；
   - 配置中存在且 value 仍然有效：保留原有 value 和 set_at，只刷新 options、labels、description 和 type；
   - 配置中存在但 value 不在新候选集中：写回该规则的默认 value，并记入报告的异常项，不中断对齐；
5. 配置中存在但 canonical 已无的规则：不删除，保留原样，在报告中列出；
6. 全部处理成功后，把 version 更新为基准版本，并更新 updated_at；
7. 使用 set 的原子写入和重新校验规则；
8. 用户模式下只输出一行变更摘要，说明补齐、刷新的规则数量；存在值失效改写或版本高于基准时，单独报告这些异常项，列出规则名与原值、新值；内部模式只返回状态。

本分支不发起任何询问，也不等待用户确认。正常对齐不需要用户介入，只有值失效改写和版本高于基准这两类异常才需要在报告中呈现。

### ask

供其他 skill 在需要用户选择可配置项时调用。本分支统一负责展示选项、处理「总是使用 / 仅本次」和写回，调用方只接收最终生效值。

本分支只适用于首次配置或允许用户改变持久行为的场景。语义上要求「每次都询问」的取值不得使用本分支，因为它提供的「总是使用」会破坏该语义。

1. 确认配置存在、可读取且结构有效，按需完成修复；
2. 读取 rules.{scope}.{key} 的 options、labels 和当前 value；
3. 确定候选值：默认使用该规则的全部 options；调用方以 --only VALUE,... 收窄范围时，只使用列出的候选值，且候选值必须是该规则 options 的子集；
4. 使用一次 AskUserQuestion 调用、两个问题完成询问：问题一选择候选值，问题二选择「总是使用」或「仅本次」；
5. 候选值的选项标签取 labels 中对应文案，无 labels 时使用 value 原文；选项顺序沿用 options 的顺序；
6. 候选值超过 4 个时不得使用本分支；调用方必须先用 --only 收窄到 4 个以内；
7. 用户选「总是使用」时，内部调用 set 分支写回问题一选中的值并更新 set_at；
8. 用户选「仅本次」时不写入规则；
9. 用户取消任一问题时返回取消状态，调用方据此终止当前任务；
10. 返回生效值以及是否已写回。

候选值集合排除当前 value 时，必须由调用方通过 --only 显式指定，本分支不猜测调用方的意图。

--internal 调用不展示询问和写回过程，只返回生效值与写回状态。

### export

1. 读取并校验配置；
2. 输出 canonical JSON；
3. 配置读取或校验失败时停止。

### import

1. 读取导入文件；读取失败停止；
2. 解析并校验完整结构；
3. 导入文件的 version 高于基准版本时拒绝导入，报告差异，不作任何写入；
4. 对每个规则校验 value；
5. 询问采用合并还是覆盖；
6. 写入前保存当前配置的恢复副本；
7. 原子写入；
8. 按 calibrate 对齐到基准版本，再重新读取并校验最终配置。

## Call contract for other Git skills

其他 skill 不需要复制 git-rules 的章节结构，也不需要执行与自身无关的命令。只需在需要规则时调用相应分支，并遵守以下结果契约：

- ensure configuration：
  - 缺失 → 内部 init；
  - 读取失败 → 停止调用方任务；
  - 整体结构错误 → 内部 repair；
  - 单项缺失或非法 → 内部 repair SCOPE KEY；
  - 版本或规则清单与基准版本不一致 → 内部 calibrate，静默完成，不询问、不等待确认；
  - 配置 version 高于基准版本 → 不写入，调用方按现有值继续执行，不视为错误；
  - 版本校验在一次任务中只执行一次：任务内首次读取配置时完成比对和必要的校准，其后不再重复比对，也不重复触发 calibrate；
  - 修复或校准后重新读取并校验。
- read rule：读取 rules.{scope}.{key}.value；git-commit scope 使用 rules["git-commit"]。
- ask rule：需要用户选择可配置项时调用 ask 分支，由它统一展示 labels、处理「总是使用 / 仅本次」并写回；调用方不要把标签文案和写回逻辑复制到自己的流程里；
- write rule：内部调用 set，不能直接改 JSON；
- resolve remote policy：读取 repository_relation_mode，并计算本次有效远程策略；
- local-only skill：不涉及远程操作时，不必解析远程策略；
- remote-aware skill：执行远程操作前必须遵循有效远程策略。

内部调用只返回成功或失败状态，不把初始化、修复和策略继承过程展示给用户。

## Output contract

- 成功：返回操作结果和必要的变更摘要；
- 失败：返回失败阶段、原始错误和建议；
- 涉及文件变更：报告实际路径；
- 用户取消：结束当前分支，不继续执行其他操作；
- 任何写入后：重新读取并校验最终配置。
