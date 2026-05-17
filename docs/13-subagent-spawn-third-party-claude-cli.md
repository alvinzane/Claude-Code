# 13 · 用 Subagent 拉起带第三方 API 的 Claude Code CLI 子进程

> 主会话保留官方 Anthropic API（Opus/Sonnet）做规划与决策；通过 Subagent + Bash 在 terminal 里**再拉起一个独立的 `claude` 进程**，把这个进程指向**第三方 API**（GLM、Kimi、DeepSeek、本地 Ollama via claude-code-router 等），用便宜或本地的算力消化机械任务。

延伸阅读：

- `docs/09-plan-mode-and-subagents.md` — Subagent / Plan Mode 内部机制。
- `docs/10-opus-plan-sonnet-execute.md` — 单进程内 Opus 规划 / Sonnet 执行的模型分工。本文是它的**跨进程跨供应商**版本。
- `docs/12-plan-mode-workflow-and-subagent-prompts.md` — 自定义 subagent 提示词的设计参考。

---

## 0. 适用场景与不适用场景

✅ 适合：

- 大量同质机械工作（批量改 import、批量补测试、grep + 改、跑 lint 修 lint、生成样板代码）。
- 离线 / 内网 / 保密代码不便发往官方 API 的场景：把任务下放到本地 Ollama。
- 需要并发跑多个独立任务，又不想抢占主会话上下文窗口。

❌ 不适合：

- 需要持续利用主会话上下文的任务 —— 子进程是**独立的 `claude` 进程**，不共享 transcript，主会话只拿到 stdout。
- 跨文件深度重构、需要架构判断的任务 —— 第三方模型工具调用质量参差，容易留半成品，得不偿失。
- 想省 prompt cache 钱的任务 —— 跨进程跑等于放弃 cache 复用，单次成本反而可能更高。

---

## 1. 原理：三件套（subagent + Bash + headless claude）

整个机制就三件东西凑起来：

1. **Subagent**：主会话调 `Agent` 工具开一个 subagent，subagent 有独立上下文与受限工具集（`src/tools/AgentTool/`）。
2. **Bash 工具**：subagent 默认就能用 Bash（默认 general-purpose subagent `tools: ['*']`，见 `src/tools/AgentTool/built-in/generalPurposeAgent.ts:29`）。
3. **headless claude**：在 Bash 里执行 `claude -p ...`，开一个**完全独立的子进程**，它不复用主会话任何状态，跑完通过 stdout 把结果交回 subagent。

关键事实：

- Anthropic SDK 在创建客户端时读 `process.env.ANTHROPIC_BASE_URL`（`src/main.tsx:1329`），不设就走 OAuth 配置 URL。
- 鉴权回退链以 `process.env.ANTHROPIC_AUTH_TOKEN` 起头（`src/services/api/client.ts:323`）。
- Bash 子进程默认**全量继承** `process.env`，只在 `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`（GitHub Actions 专用）时才会从一个 scrub 列表里剔除 `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN` 等敏感变量（`src/utils/subprocessEnv.ts:15-21,86-99`）。

合起来一句话：**只要在 Bash 命令前面把这两个 env 改掉，子进程就用第三方 API**，主会话毫发无伤：

```bash
ANTHROPIC_BASE_URL=https://<provider-endpoint> \
ANTHROPIC_AUTH_TOKEN=<provider-token> \
claude -p --bare --permission-mode bypassPermissions ...
```

---

## 2. 路由第三方 API 的环境变量

| 变量 | 作用 | 备注 |
|------|------|------|
| `ANTHROPIC_BASE_URL` | API endpoint 重定向 | 必须是供应商提供的**Anthropic 兼容**端点。OpenAI 兼容端点直接喂会 400 |
| `ANTHROPIC_AUTH_TOKEN` | 鉴权 token | 与 `ANTHROPIC_API_KEY` 二选一。普通模式两者都能用 |
| `ANTHROPIC_API_KEY` | 鉴权 key | **`--bare` 模式下严格只用这一个**（OAuth 与 keychain 都不读，见 `src/main.tsx:982` 的 option 文案） |
| `ANTHROPIC_MODEL` | 默认模型 alias | 与 `--model` 二选一，推荐用 `--model` 显式传，避免被 shell 别处覆盖 |
| `ANTHROPIC_SMALL_FAST_MODEL` | "小快模型"替身（Haiku 替身） | 让第三方模型承担背景压缩、轻量任务 |

`--bare` 是配第三方 API 最干净的姿势：跳过 hooks / LSP / 插件 / auto-memory / CLAUDE.md 自动发现 / keychain，不会去摸主会话的 OAuth token，也不会因为项目里有 hook 而把子进程跑挂。

支持的 `--permission-mode` 取值在 `src/types/permissions.ts:16-22`：`default` / `acceptEdits` / `bypassPermissions` / `dontAsk` / `plan`。子进程一般用 `bypassPermissions` 让它一路跑完不卡交互。

---

## 3. 最小可运行 demo

主会话里你给主模型一句话：

```
用 cheap-cli subagent，让它在 /tmp/foo.py 上跑一遍格式化并返回 diff。
```

主模型会调 `Agent`：

```json
{
  "subagent_type": "cheap-cli",
  "prompt": "格式化 /tmp/foo.py，返回 diff，不要解释。"
}
```

`cheap-cli` subagent 拿到任务后在 Bash 里跑：

```bash
ANTHROPIC_BASE_URL=<provider-anthropic-endpoint> \
ANTHROPIC_AUTH_TOKEN=$GLM_TOKEN \
claude -p --bare \
  --model glm-4.6 \
  --permission-mode bypassPermissions \
  --max-turns 10 \
  --max-budget-usd 0.50 \
  --no-session-persistence \
  --add-dir /tmp \
  --append-system-prompt "你是一个只做格式化的工人。输出 unified diff。" \
  "format /tmp/foo.py and print the diff"
```

stdout 就是 subagent 拿到的结果，subagent 摘要后再交回主会话。整段过程主会话**只消耗 subagent 调度本身的 token**，子进程那部分由第三方 API 计费。

---

## 4. 自定义 subagent 模板

`.claude/agents/cheap-cli.md`：

```markdown
---
name: cheap-cli
description: "把机械任务下放到第三方 / 本地模型驱动的 claude -p 子进程"
tools: [Bash, Read]
model: haiku
---
你只能用 Bash 执行 `claude -p ...` 子进程来完成任务，**不要**自己写代码或编辑文件。
每次调用必须显式传：

- `ANTHROPIC_BASE_URL` 与 `ANTHROPIC_AUTH_TOKEN`（覆盖父进程继承值）
- `--bare`（跳过 hooks/OAuth/auto-memory）
- `--permission-mode bypassPermissions`（避免卡在交互）
- `--max-turns N` 与 `--max-budget-usd X`（兜底，防长跑 / 爆预算）
- `--no-session-persistence`（短任务别落盘）
- `--add-dir <dir>` 收紧可写目录

完成后只回传子进程 stdout 的关键摘要，不要复述命令、不要把 chain-of-thought 转回主会话。
```

字段说明：

- `tools: [Bash, Read]` —— subagent 自己只允许调 Bash 与 Read（用来 sanity-check 结果），逼它把所有"动手"都丢给子进程，避免它自作主张用主模型干活。
- `model: haiku` —— 这是**父 subagent 自己的调度模型**，跟子进程模型无关。Haiku 完全够它写一行 Bash 加摘要。
- frontmatter schema 见 `src/tools/AgentTool/loadAgentsDir.ts:73-99,300-393`。

---

## 5. 进阶姿势

- **并发**：在主会话发起多个 `Agent` 调用、`run_in_background: true`，配合 Bash 的 `run_in_background` 让多个 `claude -p` 同时跑（每个走各自供应商）。适合批量任务。
- **结构化输出**：`--output-format json` 让子进程 stdout 是单个 JSON 结果对象，subagent 直接 jq 解析。`stream-json` 适合需要边流边处理。
- **inline settings**：`--settings '<json-string>'` 直接喂配置，不污染全局 `~/.claude/settings.json`。
- **嵌套 agent**：`--agents '{"foo": {"description": "...", "prompt": "..."}}'` 给子进程注册临时 agent 定义。
- **计费打 tag**：`--workload <tag>` 给计费 header 打 tag，事后能从供应商账单里识别"哪段是 cheap-cli 花的"。
- **超时兜底**：`timeout 600 claude -p ...`，第三方限流时不会让 subagent 永远挂住。

---

## 6. 完整 demo：以 GLM-4.6（智谱开放平台）为例

挑 GLM 作主例的原因：有官方维护的 Anthropic 兼容端点（不用外挂代理）、模型质量在第三方里相对靠前、国内可直连无 VPN。

### 6.1 前置准备

在智谱开放平台拿到 API key，写入 shell：

```bash
export GLM_TOKEN=...
```

端点 URL 以**智谱开放平台官方"Anthropic 兼容接口"文档**为准 —— 这里不硬编码地址，因为供应商改路径文档就过时了。下面命令里用 `<GLM Anthropic 兼容端点>` 占位，使用时替换成官方文档当下给出的地址。

### 6.2 创建 subagent 定义

按第 4 节模板把 `cheap-cli.md` 放到 `.claude/agents/`，注意它存在的级别（项目级 / 用户级）决定了哪些会话能用到。

### 6.3 单测子进程能跑通

绕过 subagent，直接在终端验证三件套（base URL / token / model alias）：

```bash
ANTHROPIC_BASE_URL=<GLM Anthropic 兼容端点> \
ANTHROPIC_AUTH_TOKEN=$GLM_TOKEN \
claude -p --bare \
  --model glm-4.6 \
  --max-turns 1 \
  --no-session-persistence \
  "用一个词回答：你是哪个模型？"
```

预期 stdout 一个词（实际返回会带 GLM 模型名特征）。如果报 401 → token 错；报 404 / "not found" → base URL 错；返回是 Claude 自我介绍 → env 没生效，检查是不是被 shell 别处覆盖了。

### 6.4 在主会话里派活

派活提示词由使用者自行设计 —— 这取决于具体任务是什么、希望 subagent 用多少自由度。文档不强行示范一种写法。

### 6.5 结果回传与摘要

subagent 拿到子进程 stdout 后，**只把 diff 或关键结论传回主会话**。子进程的 chain-of-thought、命令重复、运行日志一律丢弃，否则主会话上下文很快被噪音撑满，省下来的钱又得交回去。

---

### 其他供应商一句话点名

接入思路完全一致：

- **Kimi K2（Moonshot）**：官方有 Anthropic 兼容端点，照 GLM 流程换 base URL / token / model alias 即可。
- **DeepSeek / Qwen3-Coder / 本地 Ollama**：原生端点是 OpenAI 兼容，不是 Anthropic 协议。需要外挂 `claude-code-router` 一类的翻译层，把 OpenAI 协议转成 Anthropic 协议再喂给 `claude -p`。`ANTHROPIC_BASE_URL` 此时填的是**翻译层暴露的本地端点**，不是供应商直链。

具体端点、模型 ID、限流策略以各供应商当时的官方文档为准。

---

## 7. 安全与坑

- **鉴权泄露**：父进程的 `ANTHROPIC_AUTH_TOKEN` 默认会透传到子进程的 env。如果第三方 endpoint 行为不可控（极端情况下它会日志化或回显 header），等于把官方 token 暴露给了第三方。**解决**：在 Bash 命令里**显式覆盖**而不是依赖继承；高危场景用 `env -i ANTHROPIC_BASE_URL=... ANTHROPIC_AUTH_TOKEN=... PATH=$PATH claude -p ...` 起一个干净 env。`src/utils/subprocessEnv.ts:15-21` 的 GHA scrub 列表只在 GHA 模式生效，本地默认是不 scrub 的。
- **递归爆炸**：子进程理论上也能再起 Agent / Bash / 再 spawn `claude -p`。在 `cheap-cli` 的系统提示里**明确禁止再 spawn**，并且 `--max-turns N` 和 `--max-budget-usd X` 一定要带，至少有兜底。
- **权限模式**：`--permission-mode bypassPermissions` 跳过所有权限交互；这在父进程里**等价于自动批准**子模型对文件系统 / 网络的所有操作。务必配合 `--add-dir` 收紧可写目录，或在 git worktree 里跑（参考主仓库的 `EnterWorktree` 工具）。
- **session 落盘**：默认 `claude -p` 会落 session 文件。短任务必加 `--no-session-persistence`，否则 `~/.claude/projects/...` 很快堆出几百个孤儿 session。
- **第三方稳定性**：第三方 endpoint 限流 / 抖动比官方厉害。子进程被卡时 subagent 的 Bash 调用就一直 block。**对策**：`timeout 600 claude -p ...`，或给 subagent 的 Bash 调用本身加超时。
- **prompt cache 收益归零**：跨进程意味着完全放弃 cache 复用。批量同质任务可以接受，但**不要**把一次性的任务无脑拆成 100 次单独 `claude -p` 调用，反而更贵。
- **CLAUDE.md / hooks 行为不一致**：`--bare` 不读 CLAUDE.md、不跑 hooks、不装 auto-memory。如果任务依赖项目级 CLAUDE.md 提供的上下文，要么用 `--append-system-prompt-file <path>` 显式注入相关片段，要么干脆别加 `--bare` —— 但这样又得自己处理 OAuth / keychain 冲突。两难时优先选 `--bare` + 显式注入。
- **审计**：子进程的所有动作（包括它写的文件、跑的命令）**不会**出现在主会话 transcript 里。如果团队对操作审计有要求，建议让子进程把命令历史和 diff 显式打到 stdout，subagent 在回传摘要时一并附带。

---

## 8. 验证步骤

按这套顺序自检，每一步都过了才算"机制跑通"：

1. 准备一个第三方 token：
   ```bash
   export GLM_TOKEN=...   # Ollama 走本地翻译层时不需要 token，但 ANTHROPIC_AUTH_TOKEN 仍要给个占位符
   ```
2. 在仓库根创建 `.claude/agents/cheap-cli.md`（第 4 节模板）。
3. **单测子进程**（绕过 subagent，直接 shell 跑）：
   ```bash
   ANTHROPIC_BASE_URL=<endpoint> ANTHROPIC_AUTH_TOKEN=$GLM_TOKEN \
   claude -p --bare --model glm-4.6 --max-turns 1 --no-session-persistence \
     "say hello in one word"
   ```
   预期 stdout 是一个词。
4. **在主会话里调 subagent**：`subagent_type: cheap-cli, prompt: "用一个 claude -p 子进程返回 'pong'"`，确认主会话只看到 stdout 摘要，并且**官方 token 没有计费**（核对 Anthropic console）。
5. **进程清理**：跑完 `pgrep -fa "claude -p"`，确认子进程退出干净，没有泄漏的常驻进程。
6. **磁盘清理**：检查 `~/.claude/projects/.../session-*.jsonl`，确认子进程没有把会话落到主会话目录（应该有 `--no-session-persistence` 兜着）。
7. **env 隔离**：在子进程命令里临时 `echo $ANTHROPIC_AUTH_TOKEN` 看是不是想要的那个；用 `env -i` 起一次确认彻底隔离也能跑通。

跑通这套之后，剩下的就是按业务需求把任务模板化、把 subagent 系统提示词写细。
