# prerelease_mode ask mapping

- scope：git-release
- key：prerelease_mode
- ask_mode：inline_persistence
- 控制值：prompt

只为 `git-release.prerelease_mode` 生成一个问题：“创建 GitHub Release 时是否标记为预发布？”严格按下表解析答案：

| 展示选项 | 选项说明 | effective_action | stored_value |
| --- | --- | --- | --- |
| 仅本次预发布 | 本次创建的 Release 标记为预发布 | prerelease | 空 |
| 仅本次正式 | 本次创建的 Release 不标记为预发布 | stable | 空 |
| 总是预发布 | 本次及以后创建的 Release 均标记为预发布 | prerelease | prerelease |
| 总是正式 | 本次及以后创建的 Release 均不标记为预发布 | stable | stable |

`stored_value=空` 表示不写回配置。不得把 prompt 作为选项或 effective_action 返回。
