# 子 agent 的构筑与编排方法（学习手册）

> 来源：Anthropic《How we built our multi-agent research system》、Claude Code subagents 官方文档、AWS 多 agent 编排模式文章。2026-11-02 由临时父 agent 三路并行学习浓缩带回。

## 一、orchestrator-worker 本质

- 结构：lead 规划 → 拆解 → 生成并行 subagent → 各自独立上下文探索 → 浓缩结果回传 → lead 合成判断是否继续。
- **本质是「压缩」**：subagent 是智能过滤器，并行探索后把海量语料压成关键 token 回给 lead。对比静态 RAG（取相似 chunk），多 agent 是动态多步搜索、随发现调整。
- 性能：Opus4 做 lead + Sonnet4 做 subagent，比单 Opus4 高 **90.2%**。
- 成本：agent ≈ chat 的 **4×**，多 agent ≈ chat 的 **15×**。派活前先问「值不值这个账」。

## 二、八条可操作设计原则（Anthropic）

1. 像你的 agent 一样思考：复刻 prompt+工具逐步看它干活，直接暴露失败模式。
2. **教 orchestrator 怎么委派**：每个 subagent 必须给「目标 / 输出格式 / 来源 / 任务边界」四条。模糊委派 → 重复劳动（Anthropic 踩过：3 个 subagent 2 个做重复调查）。
3. 按复杂度缩放投入：简单查询 1 agent + 3-10 次调用；直接对比 2-4 个 subagent 各 10-15 次；复杂研究 10+ 分工。别 spawn 50 个。
4. 工具设计 = 人机界面：每个工具要有明确目的和清晰描述，坏描述把 agent 带沟里。
5. 让 agent 自我改进（能当 prompt 工程师，建工具测试 agent）。
6. 先宽后窄：先短宽查询评估，再逐步收窄。
7. 引导思考：extended thinking 当可控草稿纸。
8. 并行工具调用：lead 一次起 3-5 个 subagent + 每个 subagent 用 3+ 工具并行，复杂查询时间砍 90%。

## 三、可靠性（错误会复利）

- 不能从头重启 → **resume + checkpoint + 重试**（从出错点恢复，不是每轮从头）。
- 计划存外部 Memory（上下文超 20 万 token 会截断）；临近上限 spawn 全新上下文 subagent、靠握手连续。
- **rainbow 部署**：新旧 agent 并存渐进切流量，别一刀切。
- 同步 orchestrator 是瓶颈：lead 等 subagent 全完才继续、无法中途 steering。
- 可观测性：全量 tracing + 监控决策模式（**不监控对话内容**保隐私）。

## 四、评估（非确定性）

- 多 agent 非确定性 → 不查「走了预设步骤」，查「结果对不对 + 过程合不合理」。
- ~20 条真实查询即可开测；LLM-as-judge 单一调用输出 0.0-1.0 分 + pass/fail 最稳定（维度：事实准确/引用准确/完整性/来源质量/工具效率）。
- 人评能抓到自动化漏掉的（早期偏爱 SEO 内容农场，加来源质量启发式解决）。

## 五、Claude Code subagents 定义清单

- 定义 = Markdown + YAML frontmatter；只有 `name`（≤256 字符、不含 `:`）和 `description`（决定何时委派，写 "use proactively" 鼓励主动）必填。
- frontmatter 字段：`tools` / `disallowedTools` / `model` / `permissionMode` / `maxTurns` / `autoCompactWindow` / `skills` / `mcpServers` / `hooks` / `memory` / `background` / `omitClaudeMd` / `effort` / `isolation: worktree` / `color` / `initialPrompt` / `experimental.cacheTtl`。
- scope 优先级：managed settings(1) → `--agents` CLI(2) → `.claude/agents/`(3) → `~/.claude/agents/`(4) → plugin(5)。
- **上下文隔离**：非 fork subagent 冷启动，看不到主会话历史/已读文件；初始 = 自己 system prompt + 委派消息 + CLAUDE.md + git status + 预载 skill + sibling roster。
- **resume**：`SendMessage` 用 agent ID/name 续，保留完整历史。**fork**：继承父对话全部上下文，拿到主会话一模一样的工具池。
- 嵌套：默认**最多 3 层**；并发默认**同时 20 个**；description 之和超 15,000 token 启动告警。
- 输出扫描防注入：反斜杠插入 + 标记行，不替代权限检查+沙箱。

## 六、8 种通用编排模式 + 判断

1. Sequential（固定步骤）2. Parallel（独立并行）3. Router（单域分类分发）4. Broadcast（同题多视角）5. Hierarchical（中央分解委派综合 = orchestrator-worker）6. Graph-based（显式控制信息流）7. Team of agents（无 supervisor 协作）8. Dynamic worker pool（运行时定 worker 数）。

- supervisor/sub-agent 是**角色**不是模式。
- 判断：固定步骤→Sequential；独立→Parallel；单域分类→Router；多视角→Broadcast；跨域协调→Hierarchical；显式控制→Graph；分阶段专长→Team；任务定 worker 数→Dynamic pool。
- 确定性（sequential/parallel/broadcast/graph）可预测省钱；模型驱动（router/hierarchical/team/dynamic）灵活但贵。生产系统混合。

## 七、坑清单

1. 模糊委派 → 重复劳动/漏项。2. 简单任务过度投入（spawn 50 个）。3. token 爆炸（agent 4×、多 agent 15×）。4. 同步 orchestrator 是瓶颈。5. 错误复利不能从头重启 → resume + checkpoint。6. 部署要 rainbow。7. subagent 报告可能带注入 → 输出扫描 + 权限兜底。8. 后台 subagent 内置工具集比前台小。9. 太多 subagent 回传冗长结果反而淹上下文。10. 非 fork 冷启动慢、看不到主对话已读文件 → 频繁迭代/共享上下文的活别派。
