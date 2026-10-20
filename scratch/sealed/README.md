# 技能封存箱（sealed/）

本目录是技能源文件的**临时封存区**：正式面（`.claude/skills/`、git-help registry、git-rules canonical 表与 Policy inheritance、根 CLAUDE.md、README）中彻底移除的技能及其配套文件，按项目相对路径整套保存在这里——本目录视作一个子项目根目录。启封 = 按路径拷回正式面 + 恢复下方「正式面移除片段」。启封后对应内容移出本目录。

与 `archive/` 的区别：归档只进不出、永久保留；封存是待启封的暂存，启封后内容离开本目录。本目录文件不进正式发布（`main` 由 `.gitignore` 忽略 `scratch/`）。

## 封存清单

### git-extract（封存日期：2026-10-20）

**封存原因**：技能通过技能准入标准（不可或缺性成立——三类分类与推断模型在基础技能组合中无归属；结构判决为「不拆、不并、维持任务流」，manual/auto/confirm 即技能内建的基础/智能分层）。但其最终形态依赖未来基础技能（git-branch / git-apply 等原语的委托钩子）与推断先验修订（B 类判私有的发布场景先验，见 `scratch/notes/git-extract-scope-analysis.md`）。为避免对已知要重构的技能做增量维护，先行封存，待基础技能完成后再启封大修。

**备份内容（按项目相对路径，启封时拷回）**：

| 备份路径 | 来源 |
| --- | --- |
| `.claude/skills/git-extract/SKILL.md` | 技能本体（含 2026-10-20 方案 A 表述泛化：主发布用途 + 次要移植用途） |
| `.claude/skills/git-rules/references/ask/git-extract/selection_mode.md` | ask 映射（initialize） |
| `.claude/skills/git-rules/references/ask/git-extract/new_files_mode.md` | ask 映射（inline_persistence） |
| `.claude/git-claude-rules.json` | 移除 `rules.git-extract` scope 前的本仓库配置完整快照 |

**正式面移除片段（启封时恢复，原文如下）**：

1. `.claude/skills/git-help/SKILL.md` Skill registry 表行：

   ```text
   | git-extract | /git-extract [--base BRANCH] [--from BRANCH] [--name BRANCH] [--message MESSAGE] [--include PATHSPEC...] [--exclude PATHSPEC...] | 从基准分支拉出目标分支，按三类分类与 .gitignore 推断选择性搬移另一分支的更改并原子应用补丁，委托 git-commit 提交；主用于发布准备排除本地私有文件，亦支持跨分支按路径选择性移植 |
   ```

2. `.claude/skills/git-rules/SKILL.md` canonical 规则表三行（表尾）：

   ```text
   | git-extract | selection_mode | unconfigured | auto=自动推断、manual=仅按参数、confirm=展示并确认 | unconfigured | initialize | 路径选择模式（三类分类与 .gitignore 推断） |
   | git-extract | new_files_mode | prompt | prompt=每次询问、all=全添加、none=全不添加 | prompt | inline_persistence | C 类非忽略新文件的处理策略 |
   | git-extract | extra_excludes | "" | 无 | 无 | none | 持久排除清单（逗号分隔 pathspec，运行时解析，任何模式均参与） |
   ```

3. `.claude/skills/git-rules/SKILL.md` Policy inheritance 归属声明第三条：

   ```text
   - `git-extract` 的三规则（selection_mode、new_files_mode、extra_excludes）为本地路径选择语义，同样不受 repository_relation_mode 管辖。
   ```

4. 根 `CLAUDE.md`：分工表行 `/git-extract | 六步搬移流程：……`、常用命令中的 `/git-extract` 调用项、项目概述「五个 skill」计数、scope 列举中的 `git-extract`。

5. `README.md`：能力清单条目、可配置行为条目「路径选择模式、新增文件的处理策略与持久排除清单（`/git-extract`）」、目录树 `git-extract/` 行、使用示例 `/git-extract --name release/vX.Y.Z`、参数说明中「或通过 `--base`、`--from`、`--name` 指定 `/git-extract` 的搬移基准、来源与目标分支」子句。

6. 配置文件 `rules.git-extract` scope（从本目录快照恢复，或经 `/git-rules init` 与 canonical 表恢复后对齐）。

**启封清单**：

1. 将 `.claude/skills/git-extract/` 与 `.claude/skills/git-rules/references/ask/git-extract/` 拷回正式面；
2. 恢复 canonical 表三行与 Policy inheritance 声明；
3. 恢复 git-help registry 行、CLAUDE.md 与 README 各处；
4. 恢复配置中的 git-extract scope，`/git-rules show` 核对；
5. 按启封时的基础技能形态修订：机械步委托原语（git-branch / git-apply 等）、B 类判私有先验的语义决策、描述与受众表述复核；
6. 全套一致性核对（registry ↔ 目录、CLAUDE.md / README / canonical 表交叉），按 `/review-skill` 复查。

**关联记录**：`designs/release-flow-generic-skills-design.md` 的四技能链与阶段划分仍有效（阶段 2 已实施后封存，启封属该方案的后续实施）；`designs/git-extract-design.md` 为启封时的设计基准；`notes/git-extract-scope-analysis.md` 为定位分析来源。

### git-release（封存日期：2026-10-21）

**封存原因**：技能本体从未实施——0.4.0-beta.1 仅落地 canonical 规则行与 ask 映射，正式面长期处于「有规则无技能」的半成品状态。其形态同样依赖未来基础技能：tag 推送安全（设计中 C 技能的 Step 3-4）预计由未来 git-tag 原语吸收、prerelease 规则随迁（见 `designs/release-flow-generic-skills-design.md` 技能分层表）。为与 git-extract 的封存决策一致（原语技能完成前不对已知要重构的语义做增量维护），将规则配套一并封存，待原语技能完成后随四技能链一并实施或启封。

**备份内容（按项目相对路径，启封时拷回）**：

| 备份路径 | 来源 |
| --- | --- |
| `.claude/skills/git-rules/references/ask/git-release/prerelease_mode.md` | ask 映射（inline_persistence） |
| `rules.git-release` scope | 配置范围——完整快照见本目录 git-extract 条目的 `.claude/git-claude-rules.json`（其中含 git-release scope），启封时从中提取或经 `/git-rules init` 按 canonical 表恢复 |

**正式面移除片段（启封时恢复，原文如下）**：

1. `.claude/skills/git-rules/SKILL.md` canonical 规则表行（表尾）：

   ```text
   | git-release | prerelease_mode | prompt | prompt=每次询问、prerelease=标记为预发布、stable=正式发布 | prompt | inline_persistence | 创建 GitHub Release 时的 prerelease 属性 |
   ```

2. `.claude/skills/git-rules/SKILL.md` Policy inheritance 归属声明第二条：

   ```text
   - `git-release` 的 tag 推送不受 repository_relation_mode 管辖（无 upstream / 分叉语义），安全判定为无条件规则：远端同名 tag 即停、永不 force。
   ```

3. 配置文件 `rules.git-release` scope（从 git-extract 快照恢复，或经 `/git-rules init` 与 canonical 表恢复后对齐）。

**启封清单**：

1. 将 ask 映射拷回 `.claude/skills/git-rules/references/ask/git-release/`；
2. 恢复 canonical 表行与 Policy inheritance 归属声明；
3. 恢复配置中的 git-release scope，`/git-rules show` 核对；
4. 与 git-release 技能本体实施合并进行（设计基准见 `designs/release-flow-generic-skills-design.md` C 节）；实施时复核 tag 推送安全判定的归属——若 git-tag 原语先行，按吸收分析归位到原语层。

**关联记录**：`designs/release-flow-generic-skills-design.md` 四技能链仍有效（C=git-release 技能本体待实施，本次仅封存其规则配套）；与 git-extract 封存同属「原语技能完成前不做增量维护」决策（git-extract 2026-10-20，git-release 2026-10-21）。
