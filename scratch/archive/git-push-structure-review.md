# git-push 结构审查待办

审查日期：2026-09-28

目标文件：`.claude/skills/git-push/SKILL.md`

本报告只记录本轮未修改的结构建议。detached HEAD 路径无法闭合的问题已在目标文件中修复，不列入待办。

**2026-10-21 修复记录：下列 4 条历史待办已全部修复**（连同子代理全量复审发现的 W-1 退出码缺口、W-2「仅 --remote 无 upstream」路径、W-3 出口词汇、W-4 managed_safe_sync 双重定义一并处理；W-4 按方案甲落地：git-commit Step 5c 改为委托 /git-push，git-push 新增「委托契约」节，git-commit 的 `@{u}` 退出码表述统一为 128）。修复摘要：Step 2 重排为 2a 语法 / 2b 状态读取 / 2c 目标解析与统一校验并建立统一上下文字段表；Step 3b fetch 后记录 EXPECTED_REMOTE_OID，强推改用 `--force-with-lease=refs/heads/BRANCH:EXPECTED_REMOTE_OID`；Step 4c 非快进与 stale lease 均返回整个 Step 3 并要求重新确认；错误处理原则定义「终止任务 / 结束远程阶段」双出口词汇。**剩余待办（批次 4，均为 INFO）见文末「2026-10-21 遗留 INFO」一节。**

2026-10-11 对照现行 SKILL.md 复核：下列 4 条建议**均未实施**——校验顺序未调整（2a 仍引用 2b 才读取的 `git remote` 结果）、强推仍为不带 OID 的 `--force-with-lease`、非快进失败仍只返回 Step 3b、仍未建立统一推送上下文——全部继续有效。

## WARNING：参数校验早于依赖状态读取

位置：Step 2a、Step 2b

现状：Step 2a 要求根据 `git remote` 的结果校验 `--remote`，但远程列表到 Step 2b 才读取；通过询问取得的 branch 也没有明确再次执行 `git check-ref-format --branch`。

影响：语法解析、仓库状态发现和最终目标校验的先后关系不够明确，不同执行者可能采用不同命令顺序。

建议：将 Step 2 拆成“参数语法解析 → 仓库状态读取 → 最终目标解析与统一校验”，所有来源的最终 REMOTE 和 BRANCH 在离开 Step 2 前统一校验。

## WARNING：force-with-lease 未绑定明确的远端提交

位置：Step 3b、Step 4a

现状：fetch 后取得了远端状态，但强推命令只使用不带预期 OID 的 `--force-with-lease`。

影响：remote-tracking ref 若被后台 fetch 等操作更新，lease 所保护的状态可能不再等于用户确认时看到的远端状态。

建议：fetch 后保存 `EXPECTED_REMOTE_OID`，强推时使用：

```text
git push --force-with-lease=refs/heads/BRANCH:EXPECTED_REMOTE_OID REMOTE LOCAL_REF:BRANCH
```

## WARNING：非快进失败只返回 Step 3b

位置：Step 4c

现状：推送因检测后的远端变化而被拒时，只重新执行 fetch 和分叉检测。

影响：远端分支可能已被删除或重新创建，只回到 Step 3b 无法重新判断远程分支是否存在。

建议：返回整个 Step 3，从 `ls-remote` 开始重新确认；重新展示状态并等待用户选择，不自动再次 push。

## INFO：建议建立统一推送上下文

位置：Step 2 至 Step 5

现状：推送目标和状态变量分散在多个章节中。

建议：Step 2 结束时形成统一上下文，后续步骤只读取这些字段：

```text
LOCAL_REF
CURRENT_BRANCH
REMOTE
REMOTE_BRANCH
HAS_UPSTREAM
REMOTE_EXISTS
EXPECTED_REMOTE_OID
PUSH_MODE = normal | force-with-lease
```

这可以减少条件重复，并让首推、普通推送、detached HEAD 和强推路径更容易独立验证。

## 2026-10-21 遗留 INFO（批次 4 待办，复审子代理的 I 群未修项）

**以下 7 条已于 2026-10-21 批次 4 全部修复**，记录保留为处置档案：

1. 首推计数口径：Step 3a 第 2 项改为「提交总数 + 本次新增数」双指标，新增数用 `git rev-list --count LOCAL_REF --not --remotes`。
2. upstream 覆盖语义：Step 5 报告新增「本次目标与现有 upstream 不一致时明确说明未更改跟踪关系」。
3. `--set-upstream` 适用范围：Step 4a 声明常规与强推路径均适用。
4. 操作状态检测：Step 1a 第 4 项指定 `git rev-parse --git-path NAME` 后判文件存在性。
5. 询问交互形式：Step 2c 第 2 项写明 remote 选项化、branch 口述 +「使用当前分支名」快捷项。
6. `branch.BRANCH` 措辞：git-push Step 2b 第 3 项与 git-commit Step 5b 第 4 项统一为 `branch.CURRENT_BRANCH.*`，git-commit 5b 第 2 项补 CURRENT_BRANCH 记录。
7. git-rules delegate remote write 行精确化：委托方可为选择分支读取/缓存 repository_relation_mode，但不复算远程写入安全判定。

新审查发现的 WARNING 群（W-1 退出码、W-2 仅 --remote 无 upstream、W-3 出口词汇、W-4 双重定义）已随本次修复处理；stale lease 归类（原 I-5）已并入 Step 4c 第 3 项。**本审查单至此全部闭环（2026-10-21）。**
