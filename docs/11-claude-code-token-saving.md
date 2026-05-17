# Claude Code CLI 省 Token 综合指南

> 本文整理使用 Claude Code CLI 时降低 token 消耗的全面技巧，覆盖模型选择、上下文管理、配置优化、文件读取、输出控制等多个维度。

---

## 1. 理解 Token 从哪里来

每次请求的 token 由四部分构成：

| 来源 | 说明 | 优化空间 |
|------|------|----------|
| 系统提示 + CLAUDE.md | 每轮自动注入 | 高 |
| 对话历史（上下文） | 随会话增长 | 高 |
| 工具调用结果（文件、命令输出） | 按需注入 | 中 |
| Claude 的输出（包括 thinking） | 按任务复杂度变化 | 中 |

> 关键认知：token 不只是你"输入的问题"——历史对话、文件内容、工具返回值都在持续消耗配额。

---

## 2. 模型选择：默认 Sonnet，按需升 Opus

**原则**：从最便宜的够用的模型出发。

| 模型 | 适用场景 |
|------|----------|
| Sonnet（默认） | 日常编码、文件修改、跑命令 |
| Opus（按需） | 复杂架构推理、多方案权衡、深度调试 |
| Haiku（极简） | 高频简单查询、日志分析、批量操作 |

- 每次新会话开头运行 `/model` 确认当前模型，避免用 Opus 做机械操作
- 使用 `/model opusplan` 自动混合模式：Plan 阶段用 Opus，执行阶段自动切 Sonnet（详见文档 10）

---

## 3. 上下文管理：四个核心命令

上下文随对话增长是最大的隐性消耗源。

| 命令 | 作用 | 何时使用 |
|------|------|----------|
| `/clear` | 彻底清空对话历史 | 切换到不相关的新任务 |
| `/compact` | 压缩对话历史为摘要 | 同一任务已进行较长，想保留结论但减少 token |
| `/rename <name>` | 重命名当前会话 | 多任务并行，方便管理 |
| `/resume` | 恢复已命名会话 | 回到某个历史任务继续 |

**使用节奏**：
- 每个独立任务结束后用 `/clear` 开新会话
- 长会话中段用 `/compact` 一次，保留结论，丢掉中间过程
- 不要让会话无限积累——即便配额充足，超长上下文也会降低 Claude 的输出质量

---

## 4. CLAUDE.md 瘦身

CLAUDE.md 在每次请求前自动全量注入，是固定的 token 底座。

**优化目标**：控制在 **200 行以内 / 5k token 以内**

**保留内容**：
- 项目简介（2–3 行）
- 技术栈（列表，不超过 10 项）
- 编码风格约定（精简，只写不明显的规则）
- 当前活跃任务或已知 Bug（保持最新）

**移出内容**：
- 详细架构说明 → 拆到 `docs/` 子文件按需引用
- 完整 API 文档 → 另存，用时再 Read
- 历史变更记录 → 属于 git log，不属于 CLAUDE.md

> 原则：CLAUDE.md 是"任何任务都用得上的最小公共知识"，不是项目百科全书。

---

## 5. Plan 模式控成本

先规划后执行，避免多次来回消耗 token：

- 进入 Plan 模式（`Shift+Tab`），Claude 只读不写，输出结构化计划
- 审阅无误再批准，一次性执行，减少反复修改的往返轮次
- 复杂任务用 `/model opusplan` 让 Opus 只做规划，Sonnet 做执行

> 详细用法见文档 10：`10-opus-plan-sonnet-execute.md`

---

## 6. Thinking Budget：控制深度推理消耗

Thinking token 按输出 token 计费，默认预算可达数万 token/次。

### 通过 `/effort` 控制

```
/effort low      # 轻量任务：关闭或最小化 thinking
/effort medium   # 默认
/effort high     # 架构设计、复杂调试
```

### 通过环境变量控制（更精确）

```bash
MAX_THINKING_TOKENS=8000 claude   # 限制单次 thinking 预算
```

适合将 thinking 预算设为 8k 作为日常上限，复杂任务临时提高。

**建议**：80% 的日常任务用 `low` 或 `medium` 就够；只在需要深度推理时才开 `high`。

---

## 7. MCP 服务器精简

每个已连接的 MCP 服务器会在每次消息时注入其全部工具定义——单个 MCP 服务器可增加约 **18,000 token/消息**。

- 运行 `/mcp` 查看当前连接列表
- 断开当前任务用不到的 MCP 服务器
- 优先用 CLI 工具替代 MCP（如用 `gh` CLI 代替 GitHub MCP）

---

## 8. 文件读取经济：两步法

Claude 读文件靠 Glob/Grep + Read 工具。低效做法会产生 **80k–150k token/小时** 的导航开销。

### 低效 vs. 高效

| 做法 | token 代价 |
|------|-----------|
| 让 Claude 自由探索目录结构 | 高（多次 Glob + 多次 Read） |
| 直接给文件路径 + 行号 | 低（一次 Read） |
| 先 Glob/Grep 定位，再 Read 目标段落 | 中（两步最优） |

**实践建议**：
- 在 prompt 中直接写文件路径（如 `src/auth/middleware.ts:42`），跳过搜索步骤
- 需要搜索时先用 Grep 定位行号，再用 Read 只读相关段落（用 `offset` + `limit` 参数）
- 不要让 Claude 读完整大文件——超过 500 行的文件按需分段读

---

## 9. 输出冗长度控制

Claude 的每个输出 token 也计费，避免不必要的冗长。

**在 prompt 里明确限制**：
```
简洁回答，不要解释显而易见的内容
不要重复我的问题
只给出修改后的代码片段，不要整个文件
```

**关闭不需要的功能**：
- 不需要 thinking 时：`/effort low`
- 不需要步骤说明时：直接说"只输出结果"

---

## 10. Subagent 与 Skills：隔离上下文

### Subagent（子代理）

- 用 Agent 工具把独立的探索/搜索任务派给子代理
- 子代理的工具调用结果不会污染主会话上下文
- 适合：大量文件搜索、独立调研、并行任务

### Skills（技能）

- Skills 只在调用时按需加载，不常驻上下文
- 将重复工作封装为 Skill，避免每次重写长提示词

---

## 11. Prompt Caching：理解缓存机制

Claude Code 内置 prompt caching，命中时缓存读取费用约为正常价格的 10%。

**如何提高命中率**：
- 保持 CLAUDE.md 稳定（内容不变则缓存不失效）
- 同一会话内连续操作比频繁开新会话更省
- 缓存 TTL 约 5 分钟：`/compact` 后的新上下文需重新建立缓存

**已知注意事项**：
- 2026 年 3 月曾有 prompt caching bug，导致 token 消耗异常膨胀（10–20x）
- 如发现 token 消耗突然异常飙升，可通过账单面板核查缓存命中率

---

## 12. 关键命令速查

| 操作 | 命令 / 快捷键 |
|------|--------------|
| 清空上下文 | `/clear` |
| 压缩上下文 | `/compact` |
| 恢复会话 | `/resume` |
| 查看/切换模型 | `/model` |
| 启用 Opus Plan 混合模式 | `/model opusplan` |
| 设置推理深度 | `/effort low \| medium \| high` |
| 查看 MCP 连接 | `/mcp` |
| 设置 thinking 预算 | `MAX_THINKING_TOKENS=8000 claude` |

---

## 参考来源

- [Manage costs effectively — Claude Code Docs](https://code.claude.com/docs/en/costs)
- [Best practices for Claude Code — Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [7 Practical Ways to Reduce Claude Code Token Usage — KDnuggets](https://www.kdnuggets.com/7-practical-ways-to-reduce-claude-code-token-saving)
- [How to Manage Claude Code Token Usage: 10 Techniques — MindStudio](https://www.mindstudio.ai/blog/how-to-manage-claude-code-token-usage)
- [18 Claude Code Token Management Hacks — MindStudio](https://www.mindstudio.ai/blog/claude-code-token-management-hacks)
- [Claude Code Token Optimization: Stop the $1,600 Bill — Build to Launch](https://buildtolaunch.substack.com/p/claude-code-token-optimization)
- [12 Proven Techniques to Save Tokens in Claude Code — Aslam Doctor](https://aslamdoctor.com/12-proven-techniques-to-save-tokens-in-claude-code/)
- [23 Tips for Smart Claude Code Token Saving — Analytics Vidhya](https://www.analyticsvidhya.com/blog/2026/05/tips-for-claude-code-token-saving/)
- [Claude Code Context Window: Optimize Your Token Usage — ClaudeFast](https://claudefa.st/blog/guide/mechanics/context-management)
- [Effort — Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/effort)
