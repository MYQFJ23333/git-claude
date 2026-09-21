# Git Skills 设计方案

> 帮助使用者用 Claude Code 管理 Git 项目，无需手动敲 Git 命令

---

## 📊 状态与查看

### `/git-status`
**智能状态概览**
- 分析工作区、暂存区、未跟踪文件
- 识别变更类型（新增/修改/删除/重命名）
- 评估风险点（大文件、二进制文件、敏感文件）
- 提供下一步操作建议

### `/git-log`
**可视化提交历史**
- 按时间范围过滤（今天/本周/本月）
- 按作者筛选
- 按文件/目录过滤
- 图形化分支合并历史
- 统计提交频率和贡献分布

### `/git-diff`
**智能差异分析**
- 显示工作区/暂存区/分支间差异
- 解释变更的实际含义
- 识别潜在问题（调试代码残留、格式问题）
- 支持按文件类型/目录过滤

---

## ✍️ 提交与同步

### `/git-commit`
**智能提交**
- 自动分析暂存文件内容
- 生成符合 Conventional Commits 规范的 message
- 识别关联的 issue/ticket 编号
- 预检：大文件警告、敏感信息检测
- 支持交互式选择提交范围

### `/git-push`
**推送与远程处理**
- 推送到指定远程/分支
- 自动处理 upstream 设置
- 检测远程冲突并预警
- 支持 force-with-lease 安全强推

### `/git-pull`
**拉取与合并**
- 智能选择 rebase 或 merge
- 自动 stash 未提交更改
- 冲突检测与解决引导
- 支持多远程源

### `/git-sync`
**一站式同步**
- fetch + rebase + push 流水线
- 自动跟踪远程分支变化
- 清理已合并的远程分支引用
- 同步前后状态对比报告

---

## 🌿 分支管理

### `/git-branch`
**分支操作中心**
- 列出本地/远程分支，标注状态
- 智能命名建议（基于当前任务/jira ticket）
- 快速创建/切换/删除分支
- 批量清理已合并分支
- 分支关系可视化

### `/git-merge`
**智能合并**
- 合并前冲突预检
- 生成合并说明（汇总被合并提交）
- 支持 fast-forward / no-ff 策略选择
- 合并后自动运行测试（可选）

### `/git-rebase`
**交互式变基**
- 压缩（squash）多个提交
- 重排提交顺序
- 修改提交信息
- 自动处理冲突并继续
- 安全中断与恢复

---

## ↩️ 撤销与修复

### `/git-undo`
**智能撤销助手**
- 分析要撤销的内容类型
- 支持多种撤销方式：
  - `restore` - 撤销工作区修改
  - `reset` - 撤销暂存/提交
  - `revert` - 安全撤销已推送提交
- 撤销前创建备份点
- 操作历史可追溯

### `/git-stash`
**工作区暂存**
- 暂存当前更改（带描述）
- 列出所有 stash 记录
- 应用/弹出指定 stash
- 清理过期 stash
- 支持部分文件 stash

### `/git-resolve`
**冲突解决助手**
- 识别所有冲突文件
- 分析冲突原因（并行修改/重构/移动）
- 提供解决建议
- 支持选择 ours/theirs/手动编辑
- 解决后验证完整性

---

## 🔍 历史与审计

### `/git-blame`
**代码责任追踪**
- 逐行显示最后修改者和提交
- 追踪特定代码块的演变历史
- 忽略空白/格式变更
- 生成作者贡献报告

### `/git-bisect`
**Bug 二分查找**
- 标记好/坏提交
- 自动选择下一个测试点
- 结合测试脚本自动化
- 定位引入问题的精确提交
- 生成 bisect 日志报告

### `/git-cherry-pick`
**提交拣选**
- 从其他分支选择特定提交
- 批量拣选（范围选择）
- 冲突处理引导
- 保持原始作者信息

---

## 🧹 维护与清理

### `/git-clean`
**仓库清理**
- 删除已合并的本地分支
- 清理过期远程分支引用
- 移除未跟踪文件（预览+确认）
- 清理空目录
- 生成清理报告

### `/git-gc`
**仓库优化**
- 垃圾回收（garbage collect）
- 压缩对象数据库
- 清理大文件历史（BFG/Clean filter）
- 仓库体积分析
- 优化 pack 文件

### `/git-hooks`
**Hooks 管理**
- 列出已安装的 hooks
- 安装/卸载 hooks
- 提供常用 hooks 模板：
  - pre-commit: lint/test
  - commit-msg: 格式校验
  - pre-push: 完整测试
- hooks 执行历史查看

---

## 📋 工作流集成

### `/git-flow`
**Git Flow 自动化**
- 初始化 flow 配置
- feature 分支创建/完成
- release 分支管理
- hotfix 处理
- 版本号自动递增

### `/git-pr`
**Pull Request 管理**
- 创建 PR（自动生成描述）
- 关联 issue
- 设置 reviewer/label
- 查看 PR 状态和评论
- 合并 PR（支持策略选择）

### `/git-release`
**版本发布流程**
- 自动计算版本号（semver）
- 生成 CHANGELOG（基于 conventional commits）
- 创建 annotated tag
- 推送 tag 和分支
- 生成 GitHub Release

---

## 🎯 推荐实现优先级

### P0 - 核心高频
1. `/git-status` - 最常用入口
2. `/git-commit` - 提效最明显
3. `/git-undo` - 安全网
4. `/git-branch` - 分支管理
5. `/git-sync` - 一站式同步

### P1 - 常用功能
6. `/git-diff` - 变更分析
7. `/git-log` - 历史查看
8. `/git-resolve` - 冲突解决
9. `/git-stash` - 工作区管理

### P2 - 高级功能
10. `/git-rebase` - 提交整理
11. `/git-blame` - 责任追踪
12. `/git-clean` - 维护清理
13. `/git-pr` - PR 集成

### P3 - 专业场景
14. `/git-bisect` - 调试工具
15. `/git-flow` - 工作流
16. `/git-release` - 发布管理
17. `/git-gc` - 深度优化
18. `/git-hooks` - 自动化配置

---

## 📐 设计原则

1. **智能推断** - 自动分析上下文，减少用户输入
2. **安全优先** - 危险操作前确认，提供撤销路径
3. **渐进披露** - 简单操作一步到位，复杂操作分步引导
4. **一致性** - 统一的交互风格和输出格式
5. **可组合** - skills 之间可以串联使用

---

## 💾 规则存储系统

### 架构设计

```
~/.claude/skills/git-*/          ← Skills 全局安装
    git-commit.md
    git-pull.md
    ...
    
$(pwd)/.claude/rules.json        ← 每个项目独立配置（运行时读取）
```

**核心原则**：Skill 全局安装，配置项目级读取，缺失时自愈创建。

### rules.json 完整结构

```json
{
  "version": 1,
  "created_at": "2026-09-07",
  "updated_at": "2026-09-07",
  "rules": {
    "git-commit": {
      "when_staging_empty": {
        "action": "always_stage_all",
        "options": ["ask", "always_stage_all", "always_error"],
        "description": "暂存区为空时的行为",
        "set_at": "2026-09-07"
      },
      "auto_message": {
        "action": true,
        "options": [true, false],
        "description": "是否自动生成 commit message",
        "set_at": "2026-09-07"
      },
      "message_style": {
        "action": "conventional",
        "options": ["conventional", "simple", "custom"],
        "description": "commit message 风格",
        "set_at": "2026-09-07"
      }
    },
    "git-pull": {
      "strategy": {
        "action": "rebase",
        "options": ["ask", "rebase", "merge"],
        "description": "拉取合并策略",
        "set_at": "2026-09-07"
      },
      "auto_stash": {
        "action": true,
        "options": [true, false],
        "description": "是否自动 stash 未提交更改",
        "set_at": "2026-09-07"
      }
    },
    "git-push": {
      "auto_set_upstream": {
        "action": true,
        "options": [true, false],
        "description": "是否自动设置 upstream",
        "set_at": "2026-09-07"
      },
      "force_mode": {
        "action": "force-with-lease",
        "options": ["ask", "force-with-lease", "force"],
        "description": "强推模式",
        "set_at": "2026-09-07"
      }
    },
    "git-branch": {
      "delete_merged": {
        "action": "ask",
        "options": ["ask", "auto_confirm", "skip"],
        "description": "删除已合并分支时的确认",
        "set_at": "2026-09-07"
      },
      "naming_convention": {
        "action": "slash",
        "options": ["slash", "dash", "underscore"],
        "description": "分支命名风格 (feature/xxx vs feature-xxx)",
        "set_at": "2026-09-07"
      }
    },
    "git-merge": {
      "ff_mode": {
        "action": "no-ff",
        "options": ["ff", "no-ff", "ask"],
        "description": "Fast-forward 策略",
        "set_at": "2026-09-07"
      }
    },
    "git-rebase": {
      "auto_continue": {
        "action": false,
        "options": [true, false],
        "description": "解决冲突后是否自动 continue",
        "set_at": "2026-09-07"
      }
    },
    "git-undo": {
      "create_backup": {
        "action": true,
        "options": [true, false],
        "description": "撤销前是否创建备份分支",
        "set_at": "2026-09-07"
      }
    }
  }
}
```

### 自愈机制

每个 skill 执行前必须执行以下流程：

```
┌─────────────────────────────────────────────────────┐
│ 1. 检查 .claude/rules.json 是否存在                   │
│    ├─ 存在 → 读取并解析                                │
│    └─ 不存在 → 调用 create_rules_json() 创建           │
├─────────────────────────────────────────────────────┤
│ 2. 提取本 skill 相关规则                               │
│    ├─ 规则存在 → 按规则执行，跳过询问                    │
│    └─ 规则缺失 → 继续步骤 3                            │
├─────────────────────────────────────────────────────┤
│ 3. 询问用户                                            │
│    ├─ 提供选项 + "始终"选项                             │
│    └─ 用户选择"始终" → 更新 rules.json                 │
├─────────────────────────────────────────────────────┤
│ 4. 执行操作                                            │
└─────────────────────────────────────────────────────┘
```

**create_rules_json() 函数逻辑**：
```
1. 检查 .claude/ 目录是否存在，不存在则创建
2. 生成包含默认值的 rules.json
3. 返回空规则对象（所有规则使用默认行为）
```

### `/git-rules` Skill

**规则管理中心**

#### 功能

- **查看规则**：显示当前项目的所有规则配置
- **修改规则**：交互式修改特定规则
- **重置规则**：恢复默认配置
- **初始化规则**：创建 `.claude/rules.json` 文件
- **导出/导入**：备份和恢复规则配置

#### 命令

```bash
/git-rules                    # 交互式规则管理
/git-rules show               # 显示所有规则
/git-rules set <skill> <key> <value>  # 设置规则
/git-rules reset [skill]      # 重置规则
/git-rules init               # 初始化 rules.json
/git-rules export             # 导出规则
/git-rules import <file>      # 导入规则
```

#### 初始化流程

当调用 `/git-rules init` 或其他 skill 发现 rules.json 缺失时：

```
1. 创建 .claude/ 目录（如果不存在）
2. 生成 rules.json 模板（所有规则为默认值）
3. 提示用户："已初始化 rules.json，是否现在配置常用规则？"
4. 如果用户同意，进入交互式配置
```

---

## 🔄 Skill 兼容性设计

### 全局安装支持

每个 skill 必须遵循以下模式以支持全局安装：

```markdown
# Skill 文件结构

## 执行前检查
1. 验证当前目录是 git 仓库
2. 读取 .claude/rules.json（不存在则创建）
3. 提取本 skill 相关规则

## 核心逻辑
- 根据规则决定行为
- 规则缺失时使用默认行为并询问

## 规则写入
- 用户选择"始终"时，更新 rules.json
```

### 默认行为表

当 rules.json 缺失或特定规则缺失时的默认行为：

| Skill | 场景 | 默认行为 |
|-------|------|----------|
| git-commit | 暂存区为空 | 询问用户 |
| git-pull | 有冲突 | 询问策略 |
| git-push | 无 upstream | 自动设置 |
| git-branch | 删除已合并 | 询问确认 |
| git-merge | ff 策略 | no-ff |
| git-rebase | 冲突后 | 手动 continue |
| git-undo | 撤销操作 | 创建备份 |

---
