# Agensi.io 拉新上架邮件与平台调查记录

日期：2026-10-21
性质：外部平台拉新事件 + 背景调查记录（暂无行动项，候选事项备查）

## 事件

收到 Agensi.io 的创作者拉新邮件（2026-10 前后），邀请把 git-claude 技能上架其市场。署名 **Joeri Brons, Agensi.io**——公开网络查无此人足迹（无 LinkedIn / 新闻 / 公司页面），无法确认是真人增长岗还是 outreach agent 的人设名；邮件短促、全事实句、模板+插槽个性化特征明显，大概率由 agent 生成或至少 agent 辅助筛选目标。

邮件原文要点：

> Saw git-claude, a set of Git skills for Claude Code … 2 stars. Developers search Google for this, not GitHub, and that is where Agensi's 65,000+ monthly visitors come from. Would you list it on Agensi.io? … sign up, upload each skill as a zip with its SKILL.md … Live within 24 hours after a security scan and review. Free listings are fine … Non exclusive, you keep 70% on paid sales and we cover the Stripe fees.
> https://www.agensi.io/sell

爬取是真的读了仓库：项目名、功能概括（对应 git-rules + 安全预检）、GitHub star 数均准确；以「2 star + Google 搜索流量」为痛点切入。

## 平台背景调查摘要（2026-10-21 检索）

- **是什么**：AI agent 技能交易市场，基于开放标准 `SKILL.md`，自我定位「AI agent 的 App Store」。支持 Claude Code / Cursor / Codex CLI / Gemini CLI 等 20+ agent。
- **归属**：Agensi B.V.，荷兰阿姆斯特丹，2026 年 3 月成立（Terms 生效日 2026-03-02；Reddit 发帖 2026-03-17 前后上线）；创始人 & CEO **Samuel Rose**（Samuel Lachlan Rose，LinkedIn ~2,100 粉）；早期投资方 **Antler**；团队约 2 人。
- **商业模式**：创作者分成 70%（平台抽 30%），一次性买断或订阅；Stripe 或 USDC（Solana/Zoneless）付款；Agensi 为 Merchant of Record；MCP 连接器（mcp.agensi.io）让 agent 自主搜索/推荐/经确认购买；技能经 8 点自动安全扫描 + 人工审核后上架；付费技能带买家指纹防泄漏。
- **数据（注意口径）**：
  - 官方/PR（2026-07-09 Newsfile 付费分发，Yahoo Finance / Globe and Mail 等转载）：上线 3 个月 2,000+ 技能、3,000+ 用户、50,000+ 月访、200+ 创作者，自称零付费广告自然增长；
  - 官网 2026-10 自述：10,000+ 技能、10,000+ 用户、1,000+ 创作者；邮件自称 65,000+ 月访；
  - 第三方 TrustMRR（Stripe API 验证）：近 30 天收入仅 **~$826**，~5,500 用户，DR 76/100，排名 #2669（2026-10-20）；
  - Trustpilot 4.6/5 但评价少且部分疑似模板；Gridinsoft 信任分 69/100（非确认诈骗，有谨慎信号）。
- **流量来源**：官网挂大量 `/learn/` SEO 文章（市场对比、如何卖技能等），邮件「developers search Google not GitHub」印证其主打 SEO 流量。
- **媒体**：2026-07 被 Business Insider 等报道；KDnuggets 2026-04 列为 Top 5 技能市场；已入 Anthropic Claude MCP 连接器目录。
- **竞争格局**：Skills.sh（Vercel 系，2026-01）、SkillsMP（80 万+ 抓取）、LobeHub、ClaudeSkills.info 等；Agensi 差异点 = 付费创作者经济 + 安全扫描。
- **行业风险背景**：Snyk ToxicSkills 研究——36% 被测技能含提示注入；技能 = 带代码执行权的依赖，扫描规则不公开、外部无法验证。

## 对本项目的适配分析

倾向：**当乐子，暂不上架**；若将来顺手，按下列约束试水（免费 bundle）。

1. **技能是套装不是原子件**：本仓库设计假设「git-rules + 全部技能完整安装，不设计降级路径」；Agensi 上架单位是「每技能一个 zip」，单抽 `/git-commit` 得到的是残缺品。要挂就整套挂，用其 **Bundles** 功能。
2. **打包边界 = `main` 可发布内容**：`.claude/git-claude-rules.json`、`scratch/`、`.claude/standards/commit-message-format.md` 等本地私有文件绝不能进 zip；须带全 `references/ask/`、`VERSION` 等附属文件，并确认剥离私有文件后跨文件依赖仍自洽。
3. **语言吃亏**：全中文 vs 其英文 SEO 流量；商店容许中文技能但搜不到。**英文 README 的优先级高于上架**。
4. **权威来源约定**：非独家、GitHub 版照旧免费，不冲突；但「本仓库是 skill 的权威来源」从此多一个需同步的分发渠道，且项目尚未出稳定正式版（无英文版计划在先）。
5. **收益预期**：平台月收入 ~$826，付费上架无利可图；免费上架 = 零风险试水，代价是维护同步。

## 关键链接

- 官网 / 上架页：https://www.agensi.io · https://www.agensi.io/sell
- 安全机制：https://www.agensi.io/security
- 关于页（创始人/分成）：https://www.agensi.io/about
- 里程碑 PR（2026-07-09）：Newsfile / Yahoo Finance / Globe and Mail 同题转载
- 第三方收入：https://trustmrr.com/startup/agensi
- 创始人 LinkedIn：nl.linkedin.com/in/samuel-lachlan-rose
