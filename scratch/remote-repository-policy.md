# 远程仓库协作策略待处理记录

## 已确定的设计原则

- `repository_relation_mode` 是远程仓库协作策略的上层控制项。
- 当设置为“视为同一仓库”（`same_repository`）时，应同步控制 `post_commit_push_mode` 及未来其他 Git skill 的远程处理字段。
- 各 Git skill 应自动遵循同一套远程仓库处理策略，帮助 Git 新手使用远程仓库，避免本地与远程两端在不知情的情况下产生分叉或冲突。
- same_repository 模式下，只有一个可用 remote 且当前分支没有 upstream 时，可以自动设置 upstream 并继续安全同步；多个 remote 时必须询问用户。
- 具体的字段继承关系、覆盖优先级，以及 fetch、pull、push 的自动化边界仍需统一设计。

## 待处理：内部 fetch 的调用方式

- 待确认：各 skill 内部需要更新远端状态时，是否统一调用专用 `/git-fetch` skill，替代直接执行 `git fetch`。
- 用户原始表述为 `/git-ferch`，专用 skill 的准确命名需要确认；当前暂不据此创建或调用该 skill。
- 评估内容：专用 skill 是否已存在、是否支持多远程和 upstream、失败与网络中断如何返回、如何避免多个 skill 重复 fetch，以及其行为如何服从 `repository_relation_mode`。
- 状态：待处理，不在本次审查中决定。
