---
name: git-rules
description: 管理 Git Skills 的项目级规则配置，支持查看、设置、重置、初始化、修复、导出和导入 git-claude-rules.json
argument-hint: "[show|set <scope> <key> <value>|reset [scope] [--force]|init [--internal] [--replace]|repair [scope] [key] [--internal]|export|import <file>]"
license: MIT
---

# git-rules

Git Skills 规则管理中心。它是一个命令分派器和配置服务，不要求所有操作经过同一条线性流程。先根据命令选择对应的操作分支，再使用该分支所需的状态检查、读写和输出规则。

<!--
特殊结构说明：本 skill 有意采用“命令分派 + 独立操作分支”的非线性结构。
它不是所有 Git skill 都必须套用的 SKILL.md 模板，也不应因为缺少统一的
Step 1 → Step 2 → Step 3 顺序而被判定为结构缺陷。审查时应分别验证各命令
分支的输入、前置条件、动作、失败处理和输出契约；只有跨分支共享的行为才放在
Shared data model、Policy inheritance 或 Call contract 中。
-->

## Command dispatch

从 $ARGUMENTS 获取命令：

- 无参数：进入面向用户的交互式规则管理；
- show：显示当前规则；
- set SCOPE KEY VALUE：设置单项规则；
- reset [SCOPE] [--force]：重置指定作用域或全部规则；
- init [--internal] [--replace]：初始化配置文件；
- repair [SCOPE] [KEY] [--internal]：修复配置文件或指定规则；
- export：导出规则；
- import FILE：导入规则。

未知命令、参数缺失或参数格式错误时，报告用法并停止，不猜测用户意图。

调用分为两种模式：

- 用户模式：可以询问确认并显示操作过程；
- 内部模式：由其他 skill 调用，不展示初始化、修复和策略计算过程，只返回成功或失败结果。

--internal 只供内部调用使用。直接用户调用不得使用 --internal。

## Shared data model

### Configuration location and read policy

配置文件固定为 .claude/git-claude-rules.json。

- 需要读取配置的操作，先确认文件存在且可读取；
- 文件不存在时，用户模式可询问是否初始化，内部模式直接执行 init；
- 文件读取失败时停止，不修改、不覆盖原文件；
- JSON 非法或整体结构错误时，只有 repair 分支可以恢复；
- 单个规则缺失或 value 非法时，repair 分支可以只修复该项；
- 所有规则实际值位于 rules.{scope}.{key}.value；scope 为 git-commit 时使用 rules["git-commit"]；
- 修改规则时必须保留 options、labels、description、set_at 等元数据。

### Canonical schema

配置使用以下规范结构：

    {
      "version": "0.0.1",
      "created_at": "YYYY-MM-DD",
      "updated_at": "YYYY-MM-DD",
      "rules": {
        "project": {
          "repository_relation_mode": {
            "value": "unconfigured",
            "options": ["unconfigured", "independent_repositories", "same_repository"],
            "labels": {
              "unconfigured": "需要初始化",
              "independent_repositories": "分别处理",
              "same_repository": "视为同一仓库"
            },
            "description": "当前项目的远程仓库协作策略",
            "set_at": "YYYY-MM-DD"
          },
          "large_file_size_limit": {
            "value": "1MB",
            "type": "file_size",
            "description": "单个暂存 blob 的大文件警告阈值",
            "set_at": "YYYY-MM-DD"
          }
        },
        "git-commit": {
          "empty_staging_mode": {
            "value": "prompt",
            "options": ["prompt", "stage_all", "abort"],
            "labels": {
              "prompt": "询问",
              "stage_all": "暂存全部",
              "abort": "终止"
            },
            "description": "暂存区为空时的行为",
            "set_at": "YYYY-MM-DD"
          },
          "commit_message_mode": {
            "value": "unconfigured",
            "options": ["unconfigured", "conventional_simple", "conventional_full", "custom"],
            "labels": {
              "unconfigured": "需要初始化",
              "conventional_simple": "简单 Conventional",
              "conventional_full": "完整 Conventional",
              "custom": "自定义"
            },
            "description": "commit message 风格",
            "set_at": "YYYY-MM-DD"
          },
          "post_commit_push_mode": {
            "value": "prompt",
            "options": ["prompt", "always", "never"],
            "labels": {
              "prompt": "询问",
              "always": "总是",
              "never": "从不"
            },
            "description": "independent_repositories 模式下 commit 成功后的 push 行为",
            "set_at": "YYYY-MM-DD"
          },
          "pre_check_mode": {
            "value": "unconfigured",
            "options": ["unconfigured", "disabled", "warn", "block", "prompt"],
            "labels": {
              "unconfigured": "需要初始化",
              "disabled": "不检查",
              "warn": "警告但继续",
              "block": "阻止提交",
              "prompt": "询问"
            },
            "description": "提交前的预检行为（大文件检测、敏感文件名检查、敏感信息扫描等）",
            "set_at": "YYYY-MM-DD"
          }
        }
      }
    }

### Policy inheritance

repository_relation_mode 是远程协作策略的上层控制项：

- unconfigured：首次涉及远程仓库时由对应 skill 询问；
- independent_repositories：使用各 skill 自己的远程字段；
- same_repository：使用统一 managed_safe_sync 策略。

在 same_repository 模式下，有效策略可以覆盖多个子字段：

- post_commit_push_mode 的有效行为为 managed_safe_sync；
- 远程动作前必须更新远程状态；
- 只有一个可用 remote 且缺少 upstream 时自动设置；
- 远端出现无法证明安全关系的新历史时停止自动写入；
- 未来新增远程字段必须声明是否受 repository_relation_mode 控制。

子字段原始配置值继续保留。切换回 independent_repositories 后，恢复使用这些值。

large_file_size_limit 接受正数加 B、KB、MB 或 GB，单位不区分大小写，按 1024 进位计算。默认值为 1MB。

## Operation branches

### show

1. 读取并校验配置；
2. 按作用域显示每个规则的当前 value、可选值、描述和默认状态；
3. 不显示未知字段中的原始内容；
4. 配置缺失时按用户模式询问是否进入 init。

### set

1. 确认配置存在、可读取且结构有效；
2. 校验 SCOPE 和 KEY；
3. 有 options 时校验 VALUE 在选项中；
4. type 为 file_size 时按统一单位规则校验；
5. 只更新对应规则的 value 和 set_at；
6. 更新 updated_at；
7. 使用临时文件写入并原子替换；
8. 重新读取并校验，失败时报告错误。

### reset

1. 确认配置存在且结构有效；
2. 用户模式下，除非使用 --force，否则先确认；
3. 指定 SCOPE 时只重置该作用域；
4. 未指定 SCOPE 时重置全部规则；
5. 使用 set 的原子写入和重新校验规则；
6. 内部调用不得用 reset 代替单项 repair。

### init

1. 确认 .claude 目录存在，不存在则创建；
2. 文件已存在时不覆盖并报告已存在；
3. --replace 只允许 repair 在完成恢复记录后内部使用，直接用户调用必须拒绝；
4. 生成包含全部 canonical 默认规则的配置；
5. --internal 时不提示用户进入交互式配置；
6. 用户模式下可在初始化后询问是否配置常用规则；
7. 写入后重新读取并校验。

### repair

repair 是配置修复分支，不是所有命令的必经步骤。

#### 文件不存在

文件不存在时，repair 等价于内部 init，生成默认配置并重新读取校验；不存在旧内容时不创建恢复记录。

#### 单项缺损或非法

当整体 JSON 和 rules 结构可读取，但规则缺失、value 缺失、类型错误或 value 不在 options 中时：

1. 读取 canonical 默认定义；
2. 只重新初始化指定的 SCOPE/KEY；
3. 保留其他规则和元数据；
4. 更新该项 set_at 及文件 updated_at；
5. 写入后重新读取并校验。

未指定 SCOPE/KEY 时，扫描所有已知规则并逐项修复。

#### 整体结构错误

当 JSON 无法解析，或 rules、scope、rule object 等整体结构错误时：

1. 先确认原文件可读取；读取失败立即停止，不做任何写入；
2. 使用容错分析提取已知 scope/key 中结构清晰且 value 合法的候选值；
3. 将原始内容、错误原因、时间和候选值保存到由 git rev-parse --git-path git-claude-rules-recovery 取得的 Git 私有目录下的 repair-TIMESTAMP.json；目录不存在时先创建；
4. 恢复记录无法保存时停止，不覆盖原配置；
5. 使用内部 init --internal --replace 初始化全新的默认配置；
6. 将恢复候选值逐项加载到新配置；
7. 重新读取并校验最终配置。

恢复记录保留原始文本和候选值，但普通输出不得打印原始内容。未知字段不自动迁移，只有 canonical 结构中的合法值可以恢复。

任意读取、分析、恢复记录写入、初始化、覆盖或最终校验失败，都报告具体阶段并停止。

### export

1. 读取并校验配置；
2. 输出 canonical JSON；
3. 配置读取或校验失败时停止。

### import

1. 读取导入文件；读取失败停止；
2. 解析并校验完整结构；
3. 对每个规则校验 value；
4. 询问采用合并还是覆盖；
5. 写入前保存当前配置的恢复副本；
6. 原子写入并重新校验。

## Call contract for other Git skills

其他 skill 不需要复制 git-rules 的章节结构，也不需要执行与自身无关的命令。只需在需要规则时调用相应分支，并遵守以下结果契约：

- ensure configuration：
  - 缺失 → 内部 init；
  - 读取失败 → 停止调用方任务；
  - 整体结构错误 → 内部 repair；
  - 单项缺失或非法 → 内部 repair SCOPE KEY；
  - 修复后重新读取并校验。
- read rule：读取 rules.{scope}.{key}.value；git-commit scope 使用 rules["git-commit"]。
- write rule：内部调用 set，不能直接改 JSON；
- resolve remote policy：读取 repository_relation_mode，并计算本次有效远程策略；
- local-only skill：不涉及远程操作时，不必解析远程策略；
- remote-aware skill：执行远程操作前必须遵循有效远程策略。

内部调用只返回成功或失败状态，不把初始化、修复和策略继承过程展示给用户。

## Output contract

- 成功：返回操作结果和必要的变更摘要；
- 失败：返回失败阶段、原始错误和建议；
- 涉及文件变更：报告实际路径；
- 用户取消：结束当前分支，不继续执行其他操作；
- 任何写入后：重新读取并校验最终配置。
