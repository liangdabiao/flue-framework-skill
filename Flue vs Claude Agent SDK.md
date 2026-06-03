# 深度调研：Flue vs Claude Agent SDK（`@anthropic-ai/claude-agent-sdk`）——架构、差异与选型

---

## 0. 先定调：它们甚至不完全在同一个分类层上

这是整个对比里**最重要的一句话**：

> **Flue 是 "Agent Harness / 框架层"**（怎么组织 agent、在哪跑、怎么隔离、怎么部署）；**Claude Agent SDK 是 "Claude Code 的可编程运行时"**（你把 Claude Code 当库用，它自带 agentic loop + 内置工具 + 权限，但你被绑定到 Claude 的生态与 CLI 架构）。

所以它们之间**不是简单的竞品关系**，更像两个不同哲学下的解答：

| | **Flue** | **Claude Agent SDK** |
|---|---|---|
| 回答的问题 | *"我怎么用一个框架把 agent 的定义、沙箱、session、部署形态标准化？"* | *"我想直接获得 Claude Code 级别的自主 agent 能力，包进我自己的服务里"* |
| 类比 | Next.js/Astro for Agents（框架 + 约定） | Claude Code as a Library（平台运行时 API 化） |

---

## 1. 两个项目分别是谁、出自哪、什么性质

### Flue

| 项目 | 值 |
|------|----|
| **全称** | *Flue — The Agent Harness Framework / Sandbox Agent Framework* |
| **出处** | https://github.com/withastro/flue（Fred K. Schott，Astro 联合创始人） |
| **License** | Apache-2.0 |
| **语言** | TypeScript（全栈 TS 优先） |
| **包** | `@flue/sdk`（核心构建系统/sessions/tools）、`@flue/cli`、`@flue/connectors` |
| **版本状态** | **Experimental**，core runtime ~0.7.x，README 明确说 **APIs may change / no formal GitHub Release** |
| **仓库活跃度** | 2025年5月发布公开预览，持续高频更新（3000+ stars，活跃 commit） |
| **官方 slogan** | *"像 Claude Code，但 100% 无头（headless）且可编程。No TUI. No GUI. Just TypeScript."* |

Flue 的自我定位原话很关键：

> *"Flue isn't another AI SDK. It's a proper **runtime-agnostic framework** — think Astro or Next.js, **but for agents**. Write once, build, and deploy your agents anywhere (Node.js, Cloudflare, GitHub Actions, GitLab CI/CD, etc)."*

它的核心抽象是一条链：

```
Agent Definition (.flue/agents/*.ts)
  → Skills / Context / AGENTS.md（Markdown 承载行为）
    → Sandbox（virtual just-bash 默认 / 可选容器 via connectors）
      → Session（消息历史 + 状态）
        → 结构化输出（valibot schema → 运行时校验 → 类型安全返回值）
```

---

### Claude Agent SDK

| 项目 | 值 |
|------|----|
| **全称** | *Claude Agent SDK*（`@anthropic-ai/claude-agent-sdk`） |
| **出处** | **Anthropic 官方**（前身叫 *Claude Code SDK*，2025年9月底随 Sonnet 4.5 发布并更名） |
| **License** | npm 包分发（TypeScript 版闭源封装/CLI 是独立二进制；Python 版开源在 GitHub） |
| **语言** | TypeScript/Node 18+ 和 Python 3.10+ 双轨 |
| **版本状态** | 生产可用（但架构上有明确约束，见下） |
| **前身** | Claude Code SDK → 2025年底更名 Agent SDK，伴随实质性能力扩展（subagents、hooks、Skills） |

官方对自己的定位说得很直白：

> *"Build production AI agents with **Claude Code as a library**."*

核心执行模型（这是理解它的一切钥匙）：

```
你的代码
  → SDK query() / ClaudeSDKClient
    → SDK spawns / supervises a claude CLI subprocess
      → CLI 拥有：shell、working directory、JSONL session transcripts on disk
        → agentic loop（模型思考 → tool_call → CLI执行工具 → 结果喂回模型 → 循环）
          → 结果通过 stdio IPC 流回你的代码
```

每一个活跃 session ≈ **一个长住子进程** + 本地磁盘状态（`~/.claude/projects/`）。这不是隐喻，是部署含义：

> *"Hosting it is not like hosting a stateless API wrapper. Every running agent is a long-lived process tied to local state."*

---

## 2. 架构解剖：六维逐层拆

### 维度 ① — Agent Loop 归谁管？

| | Flue | Claude Agent SDK |
|---|---|---|
| **Loop 所有权** | Flue 框架层自己实现 harness（调度 session → 调模型 → 处理工具 → 更新状态）。模型通过标准 API（Anthropic / OpenRouter / …）调用，loop 在 **你的 Node 进程内** | Loop 在 **Claude Code CLI 子进程内**。SDK 只是 IPC 门面：发 prompt → 收流式消息。你不直接控制 loop 节奏 |
| **可控性** | 更高——你能看到 session 对象、能插逻辑在 prompt() 前后、能决定沙箱形态 | 更低——CLI 的黑盒里跑了三阶段循环（Gather Context → Take Action → Verify Work），你通过 `allowedTools` / `permissionMode` / `hooks` / `canUseTool` 做边界控制，但 loop 本身不在你代码里 |
| **代价** | 你得信任 Flue 的 harness 实现（尚 Experimental） | 你得接受子进程架构（进程开销、磁盘状态、冷启动形态） |

**这是两者最本质的架构分歧**：Flue 的 loop 是 TypeScript 框架级构造；Claude SDK 的 loop 是 Claude Code CLI 的内部行为。

---

### 维度 ② — 模型绑定（Model Agnostic vs Claude-Only）

| | Flue | Claude Agent SDK |
|---|---|---|
| **设计立场** | **模型无关倾向**：它不绑你到某家 API。示例里直接写了 `anthropic/claude-sonnet-4-6` 和 `openrouter/moonshotai/kimi-k2.6` 两种写法 | **Claude-only by architecture**：模型必须是 Claude（通过 Anthropic 直连 / Bedrock / Vertex / Foundry），工具调用的 message 格式对齐 Anthropic API 的 tool_use 协议 |
| **能不能接别家？** | 可以——只要你能把模型访问接进它的 provider 抽象层（OpenRouter 已经是一个现成路径） | 技术上 prompt 协议也是 standard tool calling，但系统 prompt、能力调优、上下文窗口预期都针对 Claude；Anthropic 不阻止你理论上试，但效果和支持都明确不在承诺范围内 |

选型含义：

- 你需要 **switch model / 多模型 fallback / 成本套利**（Haiku for cheap tasks → Sonnet for hard）→ **Flue 更符合**
- 你坚定押注 Claude（特别是 Opus 编码/agentic 任务质量对你最重要）→ Claude SDK 的「Claude Code 调好的 loop」就是护城河

---

### 维度 ③ — 沙箱与执行隔离（这是 Flue 最大的差异化牌）

#### Flue 的三层沙箱策略

| 层级 | 机制 | 适用场景 |
|------|------|---------|
| **Virtual Sandbox（默认）** | `just-bash`：在宿主环境模拟一个受限的文件视图 + bash 命令子集（grep/glob/read），**不开容器** | 高并发、短命任务、知识库检索、翻译、分类——**大幅更快更便宜**（无镜像 pull、无容器冷启） |
| **Local Sandbox** | mount 宿主机文件系统，把特权 CLI（gh/git/npm）连进来，但不泄漏密钥 | CI/CD triage agent |
| **Container Sandbox（opt-in）** | 通过 connectors 接 **Daytona / E2B / Modal** 等 | 需要编译、浏览器、强隔离、原生二进制的任务 |

Flue 的核心论点：

> *"虚拟沙箱比每请求起容器 **快得多、便宜得多、可扩展性高得多**，非常适合高流量 agent。"*

#### Claude Agent SDK 的沙箱/隔离

| 机制 | 描述 |
|------|------|
| **CLI 的 cwd + permissions** | 你传 `cwd:` 隔离工作目录；`allowedTools:` 白名单限制工具；`permissionMode:` 控制放行策略 |
| **本质** | 沙箱 = 受控的 shell 进程树 + 目录约束 + 权限钩子——**不是容器级隔离** |
| **更强的隔离路线** | 自己包 Docker（每个 session 一个容器）或用 Managed Agents（Anthropic 替你管） |

所以：

> *"For security hardening beyond basic sandboxing, including network controls, credential management, and isolation options, see Secure Deployment."*

**一句话**：Flue 把「沙箱策略」做成了框架的一等可换层（virtual → container 渐进升级）；Claude SDK 的隔离主要靠 Claude Code 的 permission + 你在外层包容器。

---

### 维度 ④ — 工具（Tools）与扩展机制

#### Flue 的工具观

- Tools 是框架级概念：agent 的 sandbox 自带一组 bash/shell-facing 能力（grep/glob/read/exec），agent 通过 **skills（Markdown）+ 可选自定义 tool 定义**扩展
- 扩展路径：写 `.flue/roles/` 和 skill md 文件让 agent "看到"更多能力；或通过 SDK 级 tool registry 注册
- MCP：有支持但 README 自述 **第一版、能力有限**（不自动探测 transport、不 spawn 本地 stdio MCP、不处理 OAuth 回调）——所以目前 MCP 集成是「能用但不完整」的状态

#### Claude Agent SDK 的工具观

- **内置工具**是杀手锏：`Read | Write | Edit | Bash | Glob | Grep | WebSearch | WebFetch | Skill`——开箱即有，通过 `allowedTools` 白名单控制
- **MCP 是一等公民**：`mcp_servers:` 可拉起外部 MCP server 进程（Playwright、Postgres…），也有 in-process SDK MCP server（`tool()` + `createSdkMcpServer()`）
- **Hooks**：`PreToolUse / PostToolUse / PermissionRequest` 等 lifecycle hook 让你审计/拦截每次工具调用

**对比小结**：Claude SDK 的内置工具生态 **更成熟更完整**；Flue 的工具思路是 **更声明式/文档驱动**（skills/AGENTS.md），但底层工具能力要看你选哪种 sandbox。

---

### 维度 ⑤ — Session / State / 持久化

| | Flue | Claude Agent SDK |
|---|---|---|
| **Session 形态** | `agent.session()` 返回 session 对象；在 Cloudflare 部署时 message history & state 自动持久化（平台存储绑定）；其他地方你自己管 store | `~/.claude/projects/` 下 JSONL transcript 在**本地磁盘**；可配 `SessionStore` adapter 持久化到跨节点存储；`resume:` / `continue:` 恢复会话 |
| **跨重启生存** | 取决于你后端是否有持久层；Flue 设计上期望你接存储 | 默认不存活（本地磁盘丢失 = session 丢失），必须配 adapter 或挂载卷 |
| **多租ant 隔离** | 框架层设计了 agent-per-trigger 的形态（webhook trigger / CLI trigger），沙箱层隔离；但仍要靠你部署拓扑保证 | 官方文档专有一节讲 multi-tenant：每 session 一个子进程，cwd 隔离是基本手段，强隔离要外层容器/网络策略 |

---

### 维度 ⑥ — 部署形态与运维复杂度

#### Flue

```
设计目标：write once → deploy anywhere
  ✅ Node.js 服务
  ✅ Cloudflare Workers/Edge（virtual sandbox 无容器依赖时特别匹配）
  ✔️  GitHub Actions / GitLab CI（local sandbox / CLI runner）
  ⚠️  容器模式 → 依赖外部 provider（Daytona/E2B/Modal）→ 外部成本
```

运维画像：更像 **跑一个 TypeScript 应用框架**。virtual sandbox 模式下甚至不需 Docker。

#### Claude Agent SDK

```
每活跃 session = 一个 claude CLI 子进程 + 磁盘状态
  → 你需要：
    - 进程监督（不崩、不僵尸）
    - 磁盘策略（volume / SessionStore）
    - 容器镜像含 Claude Code CLI
    - 多租户时要 nets/username 隔离或每租户容器
  → 或者走 Managed Agents Beta（Anthropic 替你管基础设施）
```

官方自己给了四种 session pattern：

| Pattern | 说明 |
|---------|------|
| **Ephemeral（一次性）** | 每任务一容器，完事销毁 → 最适合 bug fix / 翻译 / 提取 |
| **Sticky session** | 容器活得比一次 query 长，session resume 继续 |
| **Pool** | 预热池减冷启 |
| **Managed Agents** | Anthropic 代运维（Beta） |

---

## 3. 一张表总览

| 维度 | **Flue** | **Claude Agent SDK** |
|------|----------|---------------------|
| **本质** | Agent Harness / 框架（怎么定义/隔离/部署 agent） | Claude Code CLI 的可编程包装（获得 Claude 级自主 agent 运行时） |
| **开源/厂商** | ✅ Apache-2.0，社区（withastro） | Anthropic 官方；TS 包封装闭源 CLI（Python 版开源） |
| **模型绑定** | 倾向模型无关（Anthropic/OpenRouter/…） | **Claude-only** |
| **Agent Loop** | TypeScript 框架内（你能看到 session 对象） | Claude Code CLI 子进程内部（你通过 IPC 观察） |
| **沙箱** | 一等抽象：virtual（just-bash）/ local / container（connectors） | cwd + permission + hooks；强隔离靠外层容器或 Managed |
| **内置工具** | bash/shell 子集（virtual 模式）；靠 sandbox 能力扩展 | 极其完整：Read/Write/Edit/Bash/Grep/Glob/WebSearch/WebFetch/Skill |
| **Skills / 行为组织** | ✅ AGENTS.md + skills in Markdown（repo-as-control-plane） | ✅ Skills 系统（CLAUDE.md memory + project settings） |
| **MCP** | 有，但 V1 不完整（README 自述限制） | ✅ 原生、成熟、多模式 |
| **Session 持久化** | 设计成可插存储；CF 自动；自管要自己接 | JSONL on disk → SessionStore adapter；或 resume |
| **部署广度** | Node / Cloudflare Edge / CI（尤其 virtual 模式轻量） | 需要能跑 CLI 的宿主；或 Managed Agents |
| **成熟度** | ⚠️ Experimental（0.7.x，API 会变） | ✅ 生产可用（但架构约束明确） |
| **最适合的 you** | TS 团队、想框架级控制、高并发轻量 agent、多模型灵活、edge/CI 部署 | 你押注 Claude、你需要最强开箱 agentic 能力、你接受子进程/宿主约束 |

---

## 4. 怎样选择：决策树（实用版）

### 走 **Claude Agent SDK** 当——

```
✅ 你的核心诉求是："我要 Claude 级别的自主编码/文件操作 agent 嵌进我服务"
✅ 你接受 / 愿意处理 子进程 + 磁盘状态 的部署模型
✅ 模型绑定 Claude 不是问题（甚至是优势——Claude Opus 的 agentic 质量）
✅ 你需要内置 Read/Write/Bash/Grep/Glob/WebSearch 全套开箱
✅ 你需要成熟 MCP 集成 + hooks + permission 粒度
✅ 或者你直接走 Managed Agents（Beta），让 Anthropic 管沙箱/基础设施
```

典型场景匹配：
- **内部 dev tools**：代码库问答、自动 PR review、issue triage、refactor agent
- **合规/审计场景**：你需要 hooks 审计每一次 tool use
- **原型→生产快速通路**：不想自己写 harness，宁可用官方做好的

⚠️ 前提你要接受：**不是模型无关、宿主要有 CLI 能力、（自托管时）要管子进程生命周期**。

---

### 走 **Flue** 当——

```
✅ 你在 TypeScript 全栈团队，agent 要成为你产品的一部分（不是玩票）
✅ 你需要 agent 跑在 Cloudflare Edge / 轻量 Node，不是每台机器起容器
✅ 你的很多 agent 任务是"轻量自主"（检索、翻译、分类、KB问答）
   → virtual sandbox 的成本/延迟优势是实打实的
✅ 你想 agent 行为用 Markdown（skills/AGENTS.md）管理，靠近代码库版本控制
✅ 模型灵活（今天 Claude，明天 Kimi/K2/GPT，走 OpenRouter 切换）
✅ 你能接受 Experimental（0.7.x，API 会变，升级要跟）
```

典型场景匹配：
- **SaaS 产品内嵌 agent**：support agent（R2 挂载 KB + 虚拟沙箱 grep）、内容翻译 agent
- **CI/CD agent**：triage、release note 生成（local sandbox，挂 gh/git CLI）
- **高并发短任务**：每请求不值得起一个容器

⚠️ 前提：**你接受早期项目风险**（breaking change、生态不完整、MCP 还弱、文档在演变）。

---

### 走 **第三条路（两者都不/混合）** 当——

很多时候最务实的架构是：

```
轻量任务   → Vercel AI SDK / 自己写 ReAct loop / 纯 Messages API + 工具
重编码任务 → Claude Agent SDK（获得最强 Claude agentic 能力）
未来迁移   → 等 Flue 稳定后评估能否统一 harness
```

或者如果你真正需要的是 **LangGraph 级的控制 + 持久化 + 多 agent 协作**，那 Flue 和 Claude SDK 都不是最优解——它们是不同象限的东西（LangGraph = 你控制 graph/runtime；Claude SDK = 你用 Claude 的 runtime；Flue = 你在 TS 里建 harness 约定）。

---

## 5. 关键坑位预警（省你踩雷时间）

| 坑 | Flue | Claude Agent SDK |
|----|------|-----------------|
| **以为 agent 不用沙箱** | virtual sandbox 不是真隔离——敏感任务要升 container 模式 | cwd 隔离 + permission 不是容器——多租户必须外层网路/用户隔离 |
| **高估成熟度** | Flue 自标 Experimental，升级 breaking change 预期 | Claude SDK 稳但架构约束（子进程/磁盘）容易被低估直到你做容器编排 |
| **Session 跨重启** | 默认内存态，要配持久层 | 默认 `~/.claude/projects/` 在本地磁盘，容器重启蒸发，必须 SessionStore/卷 |
| **成本失控** | virtual 模式便宜；但 container 模式的外包（Daytona/E2B/Modal）是外部账单 | Claude Sonnet agentic loop 每任务可能几十~上百 K tokens；量大会贵，要设 max_turns + budget guard |

---

如果你把**你的具体使用场景**（agent 做什么任务？跑在哪？并发多大？要不要用 Claude 模型？有没有合规/多租户要求？团队是纯 TS 还是混 Python？）回我四个问题，我可以把上面的对比**收敛成一个明确的推荐 + 落地架构草图**（含每个组件你实际要写什么、不写什么）。