# commit_message_mode ask mapping

- scope：git-commit
- key：commit_message_mode
- ask_mode：initialize
- 控制值：unconfigured

只为 `git-commit.commit_message_mode` 生成一个问题：“提交信息使用哪种风格？”严格按下表解析答案：

| 展示选项 | 选项说明 | effective_action | stored_value |
| --- | --- | --- | --- |
| 简单 Conventional | 使用单行 TYPE[(SCOPE)]: DESCRIPTION | conventional_simple | conventional_simple |
| 完整 Conventional | 使用 Conventional 标题、正文和可选 footer | conventional_full | conventional_full |
| 自定义 | 使用 .claude/sample/commit-message.md 中的项目规范 | custom | custom |

选择后必须写回 stored_value。不得显示或返回 unconfigured。
