# 09 · Plan 模式工作流与 Subagent / Skill 的代码检索优化

本文基于 `Claude-Code` 仓库源码（`src/` 下还原版本），系统梳理 Plan Mode 的内部工作流、只读 subagent 在分析代码仓库时的实现细节，并据此提出**自定义 Skill / Subagent 提升检索效率、节省 token** 的工程化做法。所有引用均给出源文件 + 行号，便于按图索骥。

---

## 1. Plan Mode 概览

Plan Mode 是 Claude Code 的"先想后做"模式：模型只能读、研究、向用户提问，并把方案写入**唯一可写的计划文件**，由用户审阅后再退出。

- **进入入口**：`src/tools/EnterPlanModeTool/EnterPlanModeTool.ts:77-102`，调用 `handlePlanModeTransition()` 把 `permissionMode` 切到 `'plan'`。
- **退出入口**：`src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts:243-403`，校验当前确为 plan 模式后还原 `prePlanMode`（自动模式下还会做权限收紧，`:382-393`）。
- **唯一可写文件**：`src/utils/plans.ts:119-129` 决定路径。
  - 主会话：`~/.claude/plans/<slug>.md`（slug 通过 `generateWordSlug()` 生成并缓存）
  - Subagent：在主会话 slug 后追加 `-agent-<agentId>`
- **与 Auto / Bubble 的关系**：Plan 是更严格的上层模式；`ALL_AGENT_DISALLOWED_TOOLS`（`src/constants/tools.ts:36-46`）始终禁用 `EnterPlanMode` 与 `ExitPlanMode`，只有主会话能切换。

> 一句话：Plan Mode = 系统级"只读 + 一份草稿"约束，由 `permissionMode` 与 `filterToolsForAgent()` 共同把守。

---

## 2. 五阶段工作流（含 Interview 变体）

工作流提示词在 `src/utils/messages.ts:3235-3292` 注入到每次 plan-mode turn 的 system reminder 中。

| 阶段 | 目标 | 关键约束 |
|------|------|----------|
| Phase 1 — Initial Understanding | 通过 Explore agent **并行**检索代码 | 只允许使用 Explore subagent；上限由 `getPlanModeV2ExploreAgentCount()` 控制（默认 3） |
| Phase 2 — Design | 调 Plan agent 设计实现 | 上限由 `getPlanModeV2AgentCount()` 控制；可同时多视角 |
| Phase 3 — Review | 主会话精读关键文件，必要时 `AskUserQuestion` | `AskUserQuestion` 只用于澄清，**不能用于求批准** |
| Phase 4 — Final Plan | 把方案写入计划文件 | 仅一次写入；变体（trim/cut/cap）由 `getPewterLedgerVariant()` 决定，见 `src/utils/planModeV2.ts:88-95` |
| Phase 5 — ExitPlanMode | 把控制权交回用户 | 一回合**只能**以 `AskUserQuestion` 或 `ExitPlanMode` 收尾 |

可选的 **Interview Phase**（`messages.ts:3323-3383`，开关 `isPlanModeInterviewPhaseEnabled()`，`planModeV2.ts:50-62`）把流程改成"探索 → 写草稿 → 提问 → 迭代"，更适合需求不清晰的场景。

`AskUserQuestion` 在 plan mode 下的语义被特别约束（`src/tools/AskUserQuestionTool/prompt.ts:43`）：**禁止提及"the plan"** —— 因为用户在 `ExitPlanMode` 之前看不到计划文件。

---

## 3. 只读 Subagent 在 Plan Mode 下的实现细节

### 3.1 Explore Agent（轻量检索）

定义：`src/tools/AgentTool/built-in/exploreAgent.ts:13-83`

- 系统提示词显式标注 **READ-ONLY MODE**，禁止 `Write/Edit/Delete/mkdir/redirect`。
- `disallowedTools = [Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit]`（`:67-72`）。
- 外部用户默认 **Haiku**（更快、更便宜）；ant 用户走 inherit，可由 GrowthBook flag `tengu_explore_agent` 翻盘。
- `omitClaudeMd: true`（`:81`）—— 不注入项目级 `CLAUDE.md`，按源码注释每周可省 5–15 Gtok（34M+ 次 spawn）。

### 3.2 Plan Agent（设计代理）

定义：`src/tools/AgentTool/built-in/planAgent.ts:14-92`

- 工具集**继承自 Explore**（`:85` `tools: EXPLORE_AGENT.tools`）—— 保持只读。
- 提示词强调：彻查 → 罗列权衡 → 输出"关键文件清单"。
- 同样 `omitClaudeMd: true`。

### 3.3 双重工具门控

1. **静态层** —— `src/constants/tools.ts:36-46` 的 `ALL_AGENT_DISALLOWED_TOOLS` 永远屏蔽 `TaskOutput / ExitPlanMode / EnterPlanMode / AskUserQuestion / TaskStop`。
2. **运行时层** —— `src/tools/AgentTool/agentToolUtils.ts:70-116` 的 `filterToolsForAgent()`：
   - plan-mode 下对 `ExitPlanMode` 单独放行（`:88-92`）—— 但仅限主会话。
   - MCP 工具默认通过（`:83`）。
   - 自定义 agent 额外受 `CUSTOM_AGENT_DISALLOWED_TOOLS` 约束（`:97`）。

### 3.4 上下文裁剪

`src/tools/AgentTool/runAgent.ts:385-410` 在构造子会话上下文时：
- 跳过 `gitStatus`（约 40 KB 且常常已过期）。
- 跳过 `CLAUDE.md`（取决于 `omitClaudeMd`）。
- 由 `tengu_slim_subagent_claudemd` 总开关守护（默认 true）。

### 3.5 结果裁剪 —— 父会话只看到"最后一句话"

`src/tools/AgentTool/agentToolUtils.ts:227-258, 276-357` 的 `finalizeAgentTool()`：

- 仅提取 subagent **最后一条 assistant 消息中的 text block** 返回给父会话。
- 中间所有 `tool_use` / `tool_result` / `thinking` 全部丢弃。
- 附带 `totalToolUseCount / totalTokens / totalDurationMs` 三项指标。

> 这是 subagent 节省 token 的**核心机制**：父会话从不被 grep 输出、文件正文、ls 列表"灌肠"。

---

## 4. 代码检索效率 & Token 节省的杠杆点

把上面这些机制翻译成可操作的优化原则：

| 杠杆 | 做法 | 原理 |
|------|------|------|
| **上下文隔离** | 凡需要 grep/find/读多文件，先派 Explore subagent | 父会话只吃到一段总结，原始噪声被关进子会话 |
| **并行** | 同一条消息里发多个 Agent / Bash 调用 | 子会话间互不阻塞，墙钟更短，缓存命中更连续 |
| **缓存窗口** | 把"重头戏"集中在 5 分钟 TTL 内 | Anthropic prompt cache TTL 5 分钟；30 分钟空闲后再开会全量重建 |
| **延迟工具加载** | 大量 MCP / 自定义工具配 ToolSearch | 见 `src/tools/ToolSearchTool/prompt.ts:54-108`；可减少 ~10% 缓存创建浪费 |
| **裁剪上下文** | 自定义 agent 也设 `omitClaudeMd` 等价开关 | 与内置 Explore 同源，单次 spawn 直接少几 KB 系统提示 |
| **选小模型** | 检索类 subagent 显式 `model: haiku` | 检索/分类任务对 Opus 是浪费 |

---

## 5. 编写自定义 Subagent

加载入口：`src/tools/AgentTool/loadAgentsDir.ts`（`.claude/agents/*.md`，项目优先于用户目录）。

### 5.1 frontmatter 字段速查

| 字段 | 含义 |
|------|------|
| `name` | agent 标识 / 显示名 |
| `description` | "何时用我"——用于父模型的路由决策 |
| `model` | `haiku` / `sonnet` / `opus` / `inherit` |
| `tools` | 允许的工具白名单；省略=全部 |
| `disallowedTools` | 黑名单；与白名单求差集 |
| `effort` | `low` / `high` / 0–5（控推理预算） |
| `permissionMode` | `plan` / `bubble` / `auto` |
| `maxTurns` | 子会话最大回合数 |
| `skills` | 预加载的自定义 skill |
| `memory` | `user` / `project` / `local`（自动注入 Read/Write/Edit） |
| `background` | 后台异步执行 |
| `isolation` | `worktree`（git worktree 隔离） |
| `color`, `hooks`, `mcpServers` | UI 颜色 / 会话钩子 / MCP 限定 |

### 5.2 示例：项目级"检索专家"

`.claude/agents/repo-scout.md`：

```markdown
---
name: repo-scout
description: 在大型仓库里高速定位文件与符号；不读过大文件；不做修改。
model: haiku
tools: [Glob, Grep, Read, Bash]
disallowedTools: [FileEdit, FileWrite, NotebookEdit, Agent]
effort: low
maxTurns: 8
---

你是只读检索代理。规则：
1) 优先用 Glob 收敛文件集合，再 Grep 关键字。
2) 单次 Read 不超过 200 行；超过则汇报路径与行号让父会话决定。
3) 输出格式：`path:line — 解释`。
4) 永远不要 cat/head/tail；不要尝试写入。
5) 最终消息只汇总 5–15 条最相关命中，不复述源码大段内容。
```

要点：白名单严格 → 关掉所有写操作；`model: haiku` → 检索很便宜；最终消息约束 → 主会话只吃精炼输出。

---

## 6. 编写自定义 Skill

加载入口：`src/skills/loadSkillsDir.ts:185-265, 407-480`（`.claude/skills/<name>/SKILL.md`）。

### 6.1 常用 frontmatter

| 字段 | 用途 |
|------|------|
| `name` | 显示名 |
| `description` | 决定何时被路由 |
| `when_to_use` | 触发指引（更人话，给模型读） |
| `allowed-tools` | 子集化工具 |
| `arguments` / `argument-hint` | 作为 `/slash` 命令的参数 |
| `user-invocable` | 暴露为 `/skillname` |
| `model` | skill 自己的模型 |
| `paths` | gitignore 风格的激活路径白名单 |
| `context` | `fork` 表示子会话隔离 |
| `agent` | 路由到指定 subagent 执行 |
| `disable-model-invocation` | 仅允许用户主动触发 |
| `effort`, `hooks`, `shell` | 推理预算 / 会话钩子 / 默认 shell |

### 6.2 示例：`/scout` 检索工作流

`.claude/skills/scout/SKILL.md`：

```markdown
---
name: scout
description: 用 repo-scout 子代理快速回答"X 在哪、谁调用 Y"类问题。
when_to_use: 当用户问"在哪里 / 谁调用 / 实现位于"等代码定位问题时。
user-invocable: true
arguments: [query]
argument-hint: "<symbol or keyword>"
agent: repo-scout
context: fork
model: haiku
allowed-tools: [Agent, Read]
---

# scout

把用户的查询交给 `repo-scout` 子代理。返回不超过 15 条 `path:line — 解释`。
若候选超过 30，先用更窄的 Glob 收敛。不要在主会话里 grep。
```

组合关键点：`agent: repo-scout` 把执行体替换成自定义检索 subagent；`context: fork` 让历史不污染父会话；`paths:`（按需）可让 skill 仅在编辑特定目录时激活，避免被无关任务"召唤"。

---

## 7. 组合范式：Skill 调度 + Subagent 检索

一次"读懂模块 X"的最佳路径：

```
用户在主会话 →
   /scout "<keyword>"  (skill, fork 上下文)
       └─ Agent(repo-scout, haiku, 只读)
              ├─ Glob → Grep → Read（在子会话里产生大量噪声）
              └─ 最后 assistant message：15 条 path:line
   父会话只接到 ~1.5 KB 总结
```

Token 经济学：
- 子会话内 grep/read 的 ~50 KB 原始上下文 **不进入** 父会话。
- 父会话保留 prompt cache 不被打破，后续多轮仍命中缓存。
- 模型选用 Haiku，spawn 成本远低于 Sonnet/Opus。
- 若主会话本身处于 plan mode，Phase 1 限定只能用 Explore/Plan agent —— 自定义 `repo-scout` 需挂在常规执行模式下；plan mode 中改用内置 Explore + 在最终方案里再调它。

---

## 8. 验证

```bash
ls -la docs/09-plan-mode-and-subagents.md
# 抽样校验源码行号是否对得上
grep -n "READ-ONLY MODE" src/tools/AgentTool/built-in/exploreAgent.ts
grep -n "filterToolsForAgent" src/tools/AgentTool/agentToolUtils.ts
grep -n "ALL_AGENT_DISALLOWED_TOOLS" src/constants/tools.ts
grep -n "getPlanFilePath" src/utils/plans.ts
```

如果上述 grep 命中，文档与代码一致；后续若内部代码重构（例如把 Explore agent 从 Haiku 切回 inherit），按本文章节顺序逐一更新即可。
