# 块委托执行行为分析（2026-10-24）

## 背景

2026-10-24 会话：将版本号更新为 0.3.3 并执行 /git-commit。流程要点：

- git-commit 由 Skill 工具解析到旧副本 `C:\Users\13199\.claude\skills\git-commit`（内联 Step 1「环境检查与规则加载」，不含 task-init 块调用）；git-push 因旧副本缺该技能，解析到仓库权威副本 `.claude/skills/git-push/SKILL.md`——两份副本混跑；
- 流程内完成：calibrate 内部校准（配置 version 0.3.2→0.3.3）、empty_staging_mode=stage_all、custom 风格 message、提交 `fcb2fc3`；
- 远程阶段按权威 git-commit 的委托写法调用 /git-push，无参数；git-push Step 1 读取 task-init.md 与 error-principles.md，随后在 ls-remote 阶段因 github.com:443 连接超时（exit 128）结束远程阶段，本地 commit 保留、分支 ahead 1。

事后用户就两处块读取行为提问（明确为调试 skill 仓库，非质询）：① task-init 有跳过机制，为何在 git-push 时又读了一遍；② error-principles 约定出错后才读，为何预读。本记录为客观分析与可调试结论。

## 事实与归因

### task-init：首次读取是自举成本，跳过机制本身正常

1. 本次会话中 task-init.md 从未被加载：执行 git-commit 的是旧副本（内联 Step 1，无块调用），块内容与「本块执行记录」均不在上下文中。权威副本的块委托写法（git-commit SKILL.md Step 1「执行 task-init 块」）未被执行。
2. 跳过判据（「本块完整执行记录存在 + 所需 value 可从快照取得 → 跳过整块、不重复读取本文件」「内容已在上下文 → 不再读取」）单点保管在块文件内部（task-init.md「执行与跳过」节）；引用方按 CLAUDE.md 约定只写一行「执行 task-init.md」，不携带判据。于是产生自举问题：判断「跳还是不跳」的唯一判据来源就是块文件本身，会话内首次遇到块调用时，不读文件无法按判据判断，只能盲猜。
3. 实际行为：读取一次后命中跳过规则，未重做任何初始化步骤（所需 value `/rules/project/repository_relation_mode/value` 已由 git-commit 内联初始化缓存）。跳过机制按设计工作。
4. 措辞细节：跳过条件写的是「本块的完整执行记录」。旧副本 git-commit 做的是步骤等价的内联初始化，并非「本块」执行；本次按等价性做了宽松解释。该解释空间只在新旧副本混跑时出现，链路正确时记录天然归属本块。

结论：多付的一次读取，根因是旧副本遮蔽致块从未加载 +「判据只在块内」的自举属性（会话内首个引用 task-init 的技能必读一次），非跳过机制失效。设计意图的完整链路下（权威 git-commit 先执行块并留下记录），后续委托应零读取。

### error-principles：执行侧违规预读

- git-push 错误处理节明确写「出错后执行……加载技能时不预读」；实际在 Step 1 与 task-init 并行读入，当时流程无任何错误——直接违反文本指令，属执行侧偏差，非设计含糊。
- 信息来源值得注意：error-principles 的引用只出现在 git-push 正文文末错误处理节，不在 Step 1 附近；预读由「读过仓库 CLAUDE.md、知道块存在」驱动，属「默认把技能引用的块都读全」的习惯性动作。
- CLAUDE.md 会话纠错一节记录过该失败模式（「防止执行侧模型默认预读该块」），SKILL.md 带了防护声明，仍被违反。说明该防护是纯文本约束，依赖执行模型遵循，无法机制化保证。
- 本次后果：无正确性影响（网络错误发生时不变量已在上下文，行为一致）；若全程无错误则为纯上下文浪费，防护目的落空。

## 结论（待处理候选项）

1. **旧副本遮蔽使仓库侧协议修改不生效**：`~/.claude/skills/git-commit` 仍为内联结构，与权威副本（块委托 + error-principles 条件读取调用行）已分叉。真实会话中仓库侧对 git-commit 协议的修改碰不到运行时，直至同步旧副本。这同时解释了 task-init 未在 commit 阶段加载、error-principles 的 git-commit 侧调用行从未执行两件事。
2. **跳过机制的自举成本是设计固有属性**：判据单点保管在块内 + 引用方不带判据 ⇒ 每会话首个引用者必读一次块文件。若不可接受，只能把最小判据摘要上提到总会加载的层（如 CLAUDE.md），但与「单点保管」约定冲突，需权衡。
3. **「加载时不预读」文本防护可被违反**：本次实证。文本层无更强写法可机制化此约束；要机制化只能依赖 hook 等运行时手段。

## 附：会话关键事实

- 提交：`fcb2fc3` `chore: 更新版本号至 0.3.3`（workspace-sync，2 文件 +3/−3）
- 变更：VERSION 0.3.2→0.3.3；git-claude-rules.json 经 calibrate 对齐（version、updated_at）
- message 确认：用户要求只保留第一条细则后使用
- push 未完成：ls-remote 连接 github.com:443 超时，分支领先 origin/workspace-sync 1 个提交
