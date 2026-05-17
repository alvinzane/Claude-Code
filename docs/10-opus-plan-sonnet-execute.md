# Claude Code CLI 最佳实践：用 Opus 写计划，用 Sonnet 实现代码

> 本文整理 Claude Code CLI 中"使用 Opus 模型做规划、使用 Sonnet 模型做实现"的最佳实践，目标是让团队在日常使用中既保留 Opus 的推理质量，又显著降低 token 消耗与成本。

---

## 1. 核心理念：分离"思考"与"执行"

Claude Code 中的两种典型工作类型：

| 阶段 | 任务性质 | 推荐模型 | 理由 |
|------|----------|----------|------|
| Plan（规划） | 架构设计、边界推理、拆解步骤 | **Opus** | 复杂推理、深度思考显著优于 Sonnet |
| Execute（实现） | 写样板、套模式、改文件、跑命令 | **Sonnet** | 机械性工作中与 Opus 表现相近，但成本/配额低很多 |

> 核心结论：把 Opus 的算力花在"写计划"这个最值钱的环节，剩下的执行交给 Sonnet。在典型的大型构建任务中，约 80% 的 token 消耗在执行阶段，把执行路由到 Sonnet 可将执行阶段成本降低约 75–80%。

---

## 2. 推荐方式：`/model opusplan` 自动混合模式（首选）

这是官方提供的"一键混合"模式，无需手动切换。

### 启用方式

在 Claude Code 会话内输入：

```
/model opusplan
```

或使用 `/model` 命令在选择器中选择 "Opus Plan Mode"。

### 行为

- 进入 **Plan 模式**（Shift+Tab 切换）→ 自动使用 **Opus** 进行架构推理
- 退出 Plan 模式开始执行 → 自动切换到 **Sonnet** 写代码、跑命令
- 执行过程中如出现意料之外的复杂问题需要重新规划，会**自动回到 Opus**

### 建议

**每次新会话开头就设一次 `/model opusplan`**，把它当作默认工作模式。

---

## 3. 手动切换方式（备选/精细控制）

当需要更精细地控制每一步用哪个模型时，可手动操作：

1. 会话开始：`/model opus`（或保持 opus 默认）
2. 进入 Plan 模式：`Shift+Tab` 切到 plan mode
3. 用 Opus 完成详细计划，确认无误
4. 切换到 Sonnet：`/model claude-sonnet-4-6`（或当前 Sonnet 别名）
5. 退出 Plan 模式开始实现
6. 中途遇到架构性疑问 → 临时切回 Opus 解决 → 再切回 Sonnet 继续

---

## 4. Plan 模式的使用要点

### 进入与退出

- `Shift+Tab` 在 Permission Mode 间循环：Interactive → Auto-Accept → **Plan Mode**
- Plan Mode 下 Claude 只读不写，专注产出方案
- 满意后批准计划即开始执行

### 提示词怎么写最划算

**好的写法**：把目标、约束、上下文一次性给清楚，让 Opus 一次性产出可用的计划：

> "我要给 Express 应用加 JWT 认证。当前只有 user 路由，没有 auth 中间件。请先规划要改什么，再开始实现。"

**坏的写法**：缺关键约束或上下文 → Opus 基于假设规划 → 你在 review 阶段发现问题 → 计划要重写 → 白白消耗 Opus token。

> 规则：如果你能用一句话精确描述要改的 diff，就**跳过计划**直接做（来自 Anthropic 官方建议）；只有当任务需要权衡或拆解时，才进 Plan 模式。

---

## 5. 推荐工作流

### 标准流程

1. **开会话**：`/model opusplan`
2. **描述需求**：把目标 + 约束 + 现状一次写清
3. **进 Plan 模式**：`Shift+Tab` → Opus 自动接管，输出结构化计划
4. **审阅计划**：确认拆解、文件路径、测试方法都对
5. **批准 → 自动切到 Sonnet 执行**：写代码、改文件、跑测试
6. **遇到架构疑点**：回到 Plan Mode，Opus 自动重新介入
7. **完成 → 验证**：跑测试、看 diff、UI 任务在浏览器里手测

### 何时该跳过 Plan 模式

- 改一个 typo
- 单文件、单函数的明确小改动
- diff 一句话能说清的工作

### 何时必须用 Plan 模式

- 多文件重构
- 新功能/新模块
- 有多个可行方案需要权衡
- 牵涉架构决策或外部接口

---

## 6. 收益总结

| 维度 | 收益 |
|------|------|
| 成本 | 执行阶段 token 消耗下降约 75–80% |
| 配额 | 订阅计划下，Sonnet 占用周配额远低于 Opus，会话能跑得更久 |
| 质量 | 推理决策仍由 Opus 掌控，质量不下降 |
| 体验 | `opusplan` 一键启用，全程不用手动切换模型 |

---

## 7. 关键命令速查

| 操作 | 命令 / 快捷键 |
|------|--------------|
| 启用混合模式 | `/model opusplan` |
| 手动切到 Opus | `/model opus` |
| 手动切到 Sonnet | `/model claude-sonnet-4-6` |
| 进入 / 退出 Plan 模式 | `Shift+Tab` 循环切换 |
| 查看 / 改模型 | `/model` |

---

## 参考来源

- [Model configuration — Claude Code Docs](https://code.claude.com/docs/en/model-config)
- [How to use Opus for planning and Sonnet for implementing in Claude Code — Codely](https://codely.com/en/blog/how-to-use-opus-for-planning-and-sonnet-for-implementing-in-claude-code)
- [How to Save Tokens in Claude Code Using the Opus Plan Mode — MindStudio](https://www.mindstudio.ai/blog/save-tokens-claude-code-opus-plan-mode)
- [Claude Code has a new /model option: Opus Plan Mode — AI Automation Society](https://www.skool.com/ai-automation-society/claude-code-has-a-new-model-option-opus-plan-mode)
- [Stop Wasting Tokens: Pick the Right Model in Claude Code — wmedia.es](https://wmedia.es/en/tips/claude-code-choose-right-model)
- [What Is Claude Code's Advisor Strategy? — MindStudio](https://www.mindstudio.ai/blog/claude-code-advisor-strategy-opus-sonnet-haiku)
- [Claude Code Plan Mode — ClaudeLog](https://claudelog.com/mechanics/plan-mode/)
- [Why '/model opusplan' in Claude Code Is My Ideal Workflow — Zenn](https://zenn.dev/takibilab/articles/claude-code-model-opusplan?locale=en)
- [Claude Code power user tips — Claude Help Center](https://support.claude.com/en/articles/14554000-claude-code-power-user-tips)
- [Models, usage, and limits in Claude Code — Claude Help Center](https://support.claude.com/en/articles/14552983-models-usage-and-limits-in-claude-code)
