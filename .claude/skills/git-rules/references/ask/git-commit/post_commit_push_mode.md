# post_commit_push_mode ask mapping

- scope：git-commit
- key：post_commit_push_mode
- ask_mode：inline_persistence
- 控制值：prompt

只为 `git-commit.post_commit_push_mode` 生成一个问题：“是否推送本次提交？”严格按下表解析答案：

| 展示选项 | 选项说明 | effective_action | stored_value |
| --- | --- | --- | --- |
| 是 | 仅推送本次提交 | push | 空 |
| 否 | 仅本次跳过推送 | skip | 空 |
| 总是 | 推送本次提交，并在独立仓库模式下始终自动推送 | push | always |
| 从不 | 跳过本次推送，并在独立仓库模式下始终不自动推送 | skip | never |

`stored_value=空` 表示不写回配置。不得把 prompt 作为选项或 effective_action 返回。
