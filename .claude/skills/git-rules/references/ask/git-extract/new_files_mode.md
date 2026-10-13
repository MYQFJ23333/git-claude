# new_files_mode ask mapping

- scope：git-extract
- key：new_files_mode
- ask_mode：inline_persistence
- 控制值：prompt

只为 `git-extract.new_files_mode` 生成一个问题：“存在未被忽略的新增文件，如何处理？”严格按下表解析答案：

| 展示选项 | 选项说明 | effective_action | stored_value |
| --- | --- | --- | --- |
| 仅本次添加 | 仅本次将剩余新增文件全部加入移动集 | add | 空 |
| 仅本次不添加 | 仅本次将剩余新增文件全部排除 | exclude | 空 |
| 总是添加 | 本次及以后均自动加入全部剩余新增文件 | add | all |
| 总是不添加 | 本次及以后均排除全部剩余新增文件 | exclude | none |

`stored_value=空` 表示不写回配置。不得把 prompt 作为选项或 effective_action 返回。
