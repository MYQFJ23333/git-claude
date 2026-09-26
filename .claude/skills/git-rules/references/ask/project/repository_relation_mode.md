# repository_relation_mode ask mapping

- scope：project
- key：repository_relation_mode
- ask_mode：initialize
- 控制值：unconfigured

只为 `project.repository_relation_mode` 生成一个问题：“当前项目中的本地仓库与远程仓库是什么协作关系？”严格按下表解析答案：

| 展示选项 | 选项说明 | effective_action | stored_value |
| --- | --- | --- | --- |
| 分别处理 | 各 Git skill 按自己的远程配置决定是否同步 | independent_repositories | independent_repositories |
| 视为同一仓库 | 使用统一的 managed_safe_sync 策略安全同步 | same_repository | same_repository |

选择后必须写回 stored_value。不得显示或返回 unconfigured。
