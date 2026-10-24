# 远程仓库协作策略待处理记录

## 已落地的设计原则（2026-10-11 复核，以现行文件为准）

以下原则均已实现，权威定义在 `.claude/skills/git-rules/SKILL.md` 的 Policy inheritance 与根 CLAUDE.md「远程策略继承」章节，本节不再展开细则：

- `repository_relation_mode` 是远程仓库协作策略的上层控制项，`same_repository` 模式同步控制 `post_commit_push_mode` 及后续接入的远程字段；
- 各 Git skill 遵循同一套远程处理策略（`git-commit`、`git-push` 已接入），避免本地与远程在不知情的情况下分叉；
- `same_repository` 下仅一个可用 remote 且无 upstream 时自动设置并继续安全同步；多个 remote 时询问用户（git-push 已落实单问题 4 选项上限，git-commit 侧仍待决策，见 `scratch/reviews/git-commit-skill-review.md` 五 · 5）；
- 字段继承关系与覆盖优先级已定型：子字段原始配置值保留，策略切换后恢复。

**仍开放**：pull 的自动化边界尚未统一设计；fetch 原语已实施（见下节），但各 skill 内部委托切换未完成（push 已完整实现，见上述权威定义）。

## 已落地：内部 fetch 的调用方式（2026-10-25）

- 专用 skill 已实施：`/git-fetch`（`.claude/skills/git-fetch/SKILL.md`，原语层，命名已定），五项评估回应如下：
  - 是否已存在 → 已实施入暂存区（0.4.0）；
  - 多远程与 upstream → `--remote` 参数 + 目标解析优先级（参数 > upstream > 单 remote > 终止），见设计第 2–3 节；
  - 失败与网络中断返回 → 明确失败出口表，认证/权限/网络错误原样报告不重试，见设计第 6 节；
  - 避免重复 fetch → 委托方消费返回契约、不复制逻辑；task-init 跳过机制覆盖同任务重复调用；
  - 服从 `repository_relation_mode` → prune 默认与模式挂钩（same_repository 自动开）。
- 余留：各 skill 内部尚未切换为委托 `/git-fetch`——git-push Step 3b 仍为内联强制 refspec（设计明确本期不改造、仅预留契约），改造另行评估；pull 自动化边界未定（fetch + 合并组合，合并语义不属 fetch 原语）。
