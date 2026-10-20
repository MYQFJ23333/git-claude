# selection_mode ask mapping

- scope：git-extract
- key：selection_mode
- ask_mode：initialize
- 控制值：unconfigured

只为 `git-extract.selection_mode` 生成一个问题：“路径选择采用哪种模式？”严格按下表解析答案：

| 展示选项 | 选项说明 | effective_action | stored_value |
| --- | --- | --- | --- |
| 自动推断 | 按三类分类与 base .gitignore 自动推断选择集 | auto | auto |
| 仅按参数 | 只使用显式 --include 与 --exclude 参数，不做推断 | manual | manual |
| 展示并确认 | 推断后展示完整分类与结果，经确认后执行 | confirm | confirm |

选择后必须写回 stored_value。不得显示或返回 unconfigured。
