# git-push 结构审查待办

审查日期：2026-09-28

目标文件：`.claude/skills/git-push/SKILL.md`

本报告只记录本轮未修改的结构建议。detached HEAD 路径无法闭合的问题已在目标文件中修复，不列入待办。

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
