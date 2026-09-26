# empty_staging_mode ask mapping

- scope：git-commit
- key：empty_staging_mode
- ask_mode：inline_persistence
- 控制值：prompt

只为 `git-commit.empty_staging_mode` 生成一个问题：“暂存区为空，是否暂存全部更改？”严格按下表解析答案：

| 展示选项 | 选项说明 | effective_action | stored_value |
| --- | --- | --- | --- |
| 是 | 仅本次暂存全部更改 | stage_all | 空 |
| 否 | 仅本次终止提交 | abort | 空 |
| 总是 | 本次及以后遇到空暂存区时暂存全部更改 | stage_all | stage_all |
| 从不 | 本次及以后遇到空暂存区时均终止提交 | abort | abort |

`stored_value=空` 表示不写回配置。不得把 prompt 作为选项或 effective_action 返回。
