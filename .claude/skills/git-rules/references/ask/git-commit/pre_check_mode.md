# pre_check_mode ask mapping

- scope：git-commit
- key：pre_check_mode
- ask_mode：separate_persistence
- 控制值：prompt

为 `git-commit.pre_check_mode` 使用一次 AskUserQuestion 调用、两个问题。

第一问：“本次提交如何执行预检？”

| 展示选项 | 选项说明 | effective_action | stored_value |
| --- | --- | --- | --- |
| 不检查 | 跳过大文件、敏感文件名和敏感信息扫描 | disabled | disabled |
| 仅警告 | 执行全部预检，发现问题时警告但允许继续 | warn | warn |
| 检查并阻止 | 执行全部预检，发现阻止级别问题时终止提交 | block | block |

第二问固定为：

| 展示选项 | 选项说明 | 写回行为 |
| --- | --- | --- |
| 仅本次 | 本次采用第一问所选动作 | 不写回 |
| 始终使用 | 本次及以后采用第一问所选动作 | 写回第一问的 stored_value |

第一问不得显示 prompt 或 unconfigured。无论第二问如何选择，effective_action 都是第一问命中行的值；选择“仅本次”时配置继续保持 prompt。
