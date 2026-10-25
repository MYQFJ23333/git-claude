# 块：任务初始化（task-init）

本块的引用、放置与编写约定见根 CLAUDE.md「跨技能协议块」节。本块为只读初始化流程，不修改任何仓库状态与配置文件（写入一律经 `/git-rules` 内部调用）。

## 执行与跳过

本节是跨块通用跳过机制的唯一定义处，其他所有块遵循同样的规则，不过不再做说明：
- 首次执行：按下方步骤执行，完成后在任务上下文中留下执行事实（例如向用户注明「初始化完成」），使后续技能可见；
- 本任务已存在本块的完整执行记录，且引用方所需 value（见步骤二）均可从已有快照取得时：跳过整块，不重复执行步骤、不重复读取本文件；
- 本块内容已在本会话上下文中时：直接使用已读内容，不再次读取文件；
- 无法从已有快照取得所需 value（如会话压缩导致快照内容丢失）时：重新执行整块；
- 会话压缩（/compact）导致执行记录不可见时：重新执行本块。本块只读幂等，重执行无副作用；

## 步骤

### 一、git 环境与操作状态

1. 执行 git rev-parse --show-toplevel，取得仓库根目录；失败时报告原始错误并终止当前任务。
2. 后续所有 Git 命令从仓库根目录执行。
3. 执行 git status --short --branch 并检查退出码；失败时报告原始错误并终止当前任务。成功后记录当前分支和工作区状态。
4. 检查未解决的暂存冲突：执行 git diff --cached --name-only --diff-filter=U 并检查退出码；失败时报告原始错误并终止当前任务；输出非空表示存在未解决冲突，按第 6 项处理。
5. 检查 MERGE_HEAD、CHERRY_PICK_HEAD、REVERT_HEAD、REBASE_HEAD 是否存在：对每个文件名执行 git rev-parse --git-path NAME 取得实际路径（退出码非 0 时报告原始错误并终止当前任务），再检查该路径下文件是否存在；同时确认 git status 是否报告 merge、rebase、cherry-pick 或 revert 正在进行。
6. 存在未解决暂存冲突或未完成 Git 操作时，报告状态和文件并终止当前任务，不自动 continue、abort、reset 或 rebase。

### 二、配置快照

引用方所需 value 清单取自调用行（JSON Pointer 列表；可为空，为空时只做版本与配置健康检查、不缓存 value）。

1. 读取 `.claude/skills/git-rules/VERSION`，只接受单行 `MAJOR.MINOR.PATCH`；文件缺失、不可读、为空或格式非法时停止，不从其他文件、配置或记忆猜测基准版本。
2. 读取 `.claude/git-claude-rules.json` 一次并解析 JSON：
   - 文件不可读时停止，不覆盖；
   - 文件不存在或 JSON 无法解析时，内部调用 `/git-rules repair --internal`，成功后只重新读取一次，失败或仍不可用时停止。
3. 比较配置 version 与 VERSION：
   - 相等时继续；
   - 不相等、缺失或格式非法时，内部调用 `/git-rules calibrate --internal`，成功后只重新读取一次；配置版本高于基准、校准失败或重读后仍不一致时停止。
4. 缓存清单中每条 JSON Pointer 指向的 value；缺失值也按缺失状态缓存，本步骤不校验或修复它们。清单为空时跳过本项。

版本一致的正常路径不读取 `git-rules/SKILL.md` 正文，也不检查完整规则清单、规则元数据或清单外的 value。每个 value 只在引用方流程实际使用时验证；目标 scope、key 或 value 缺失、类型错误或非法时，由引用方内部调用 `/git-rules repair SCOPE KEY --internal`，成功后只重新读取一次，失败或仍不可用时停止。任何 `/git-rules ask` 或其他配置写入成功后都丢弃旧快照，并一次性重新读取 VERSION、配置 version 和引用方所需 value。不得静默使用内存默认值。
