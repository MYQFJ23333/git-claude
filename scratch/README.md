# scratch/ 草稿箱索引

本目录是本地私有草稿区（`workspace-sync` 跟踪、发布时排除），存放设计方案、审查记录与待处理便签。本 README 是文档清单与状态的**唯一权威**；新增、移动、删除文档时必须同步更新本文件。

## 文档清单

| 文档 | 类型 | 状态 | 说明 |
| --- | --- | --- | --- |
| `designs/git-skills-design.md` | 设计蓝图 | 蓝图（部分过时） | 项目初期愿景清单；头部有现状对照（已实现 / 0.4.0 计划中技能），「规则存储系统」「Skill 兼容性设计」两章已标注过时仅供历史参考 |
| `designs/release-flow-generic-skills-design.md` | 设计方案 | 已定稿，阶段 2 已实施后封存 | 发布流程抽象为 git-extract / git-pr / git-release / git-sync 四个通用技能（套装 0.4.0）；含共享语义单一定义表与技能分层/演化吸收分析；git-extract 已实施后封存（2026-10-20，见 `sealed/`）、git-release 规则配套已封存（2026-10-21，见 `sealed/`，技能本体未实施），git-pr、git-sync 待实施 |
| `designs/git-extract-design.md` | 设计方案 | 已实施后封存（待启封修订） | 技能 A 的具体设计：参数接口、选择逻辑（三类分类 / .gitignore 推断 / 优先级矩阵 / 三规则）、6 步流程、diff+apply 机制选型、委托边界与失败出口；曾于 0.4.0-beta.1 实施，2026-10-20 封存，启封时以此为设计基准 |
| `designs/git-fetch-design.md` | 设计方案 | 已定稿待实施 | 原语层 fetch 技能：双模式 refspec（默认仅快进，`--force` 强制）、prune 挂钩 `repository_relation_mode`（same_repository 自动）、返回契约供委托方消费；本期不改造 git-push 仅预留契约，回应 `notes/remote-repository-policy.md` 五项评估（2026-10-23 定稿） |
| `reviews/git-commit-skill-review.md` | 审查记录 | 已复核（2026-10-11 二次更新） | git-commit / git-rules 结构与抽取分析；行数统计、失效路径已修正，含未实施待办（hooks、技能抽取） |
| `archive/git-push-structure-review.md` | 审查记录 | 已归档（2026-10-21 全单闭环） | git-push 结构审查（2026-09-28）：4 条历史待办、复审 WARNING 群与 7 条 INFO 已全部修复并经批次 1–4 落地，处置档案完整保留；detached HEAD 问题更早已修复不列入 |
| `notes/remote-repository-policy.md` | 待处理便签 | 部分已落地 | 原则部分已实现（指向 git-rules Policy inheritance），余 fetch/pull 自动化边界与 `/git-fetch` 设想待处理 |
| `notes/git-extract-scope-analysis.md` | 分析记录 | 已决策已落地 | git-extract 适用范围分析（2026-10-14）：机制层通用、推断层窄（发布场景先验）；已按方案 A 泛化 description（2026-10-20），技能随后封存，议题 1（B 类先验）留待启封时决策 |
| `notes/agensi-listing-outreach.md` | 记录 | 待处理（暂无行动，候选备查） | Agensi.io 拉新上架邮件（2026-10，署名 Joeri Brons 疑为 agent outreach）与平台调查：归属/商业模式/数据口径/适配分析；若将来试水，约束为英文 README 优先、整套 bundle、打包边界限 `main` 可发布内容 |
| `sealed/README.md` | 封存记录 | 启封待定 | 技能封存箱：git-extract 已封存（2026-10-20，技能本体）；git-release 已封存（2026-10-21，规则配套——ask 映射、canonical 行、Policy inheritance 行与配置 scope，技能本体未实施）；按项目相对路径备份，含正式面移除片段与启封清单 |

## 管理约定

1. 文档按类型入对应子目录，不放在 `scratch/` 根目录：
   - `designs/` — 设计方案（待实施、已定稿、蓝图）
   - `reviews/` — 对现有文件的审查记录与待办
   - `notes/` — 短便签、待处理记录
   - `sealed/` — 技能封存箱（临时储存的技能源文件与封存记录，启封后移出；区别于 `archive/` 的只进不出）
   - `archive/` — 已过时或已落地的归档
2. 状态只在本索引维护（进行中 / 已定稿待实施 / 已实施 / 待处理 / 已过时等），文档本身不强制加状态行，避免双重维护。
3. 文档变为已过时、或其方案已落地实施时，移入 `archive/` 并更新本索引；归档**只进不出，永久保留**。
4. 不在本索引中的文档视为未登记，先登记再引用。
5. 引用归档文档时注意其内容可能已被现行文件取代，一切以 `.claude/skills/` 下的技能文件与根 CLAUDE.md 为准。
