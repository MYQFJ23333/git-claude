# 远程仓库协作策略待处理记录

## 已落地的设计原则（2026-10-11 复核，以现行文件为准）

以下原则均已实现，权威定义在 `.claude/skills/git-rules/SKILL.md` 的 Policy inheritance 与根 CLAUDE.md「远程策略继承」章节，本节不再展开细则：

- `repository_relation_mode` 是远程仓库协作策略的上层控制项，`same_repository` 模式同步控制 `post_commit_push_mode` 及后续接入的远程字段；
- 各 Git skill 遵循同一套远程处理策略（`git-commit`、`git-push` 已接入），避免本地与远程在不知情的情况下分叉；
- `same_repository` 下仅一个可用 remote 且无 upstream 时自动设置并继续安全同步；多个 remote 时询问用户（git-push 已落实单问题 4 选项上限，git-commit 侧仍待决策，见 `scratch/reviews/git-commit-skill-review.md` 五 · 5）；
- 字段继承关系与覆盖优先级已定型：子字段原始配置值保留，策略切换后恢复。

**仍开放**：fetch、pull 的自动化边界尚未统一设计（push 已完整实现，见上述权威定义）。

## 待处理：内部 fetch 的调用方式

- 待确认：各 skill 内部需要更新远端状态时，是否统一调用专用 `/git-fetch` skill，替代直接执行 `git fetch`。
- 用户原始表述为 `/git-ferch`，专用 skill 的准确命名需要确认；当前暂不据此创建或调用该 skill。
- 评估内容：专用 skill 是否已存在、是否支持多远程和 upstream、失败与网络中断如何返回、如何避免多个 skill 重复 fetch，以及其行为如何服从 `repository_relation_mode`。
- 状态：待处理，不在本次审查中决定。
