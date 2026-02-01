# OpenCode 智能体设计学习指南

本文档提供**源码分析路径**和**设计方法提炼**，用于系统学习本项目的智能体（Agent）设计。

---

## 一、整体架构：智能体在系统中的位置

```
┌─────────────────────────────────────────────────────────────────────────┐
│  TUI / CLI / ACP / Server                                                │
│  （选择/切换 Agent，发起会话）                                            │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ currentAgent / session.agent
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Agent 层 (agent/agent.ts)                                               │
│  • 定义：name, mode, permission, prompt, model, options                  │
│  • 内置：build / plan / general / explore / compaction / title / summary │
│  • 配置合并：Config.agent + 用户/项目配置                                 │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ Agent.Info
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Session + LLM 层                                                        │
│  • prompt.ts：按 Agent 选 system prompt、组 tools、插入 reminder         │
│  • llm.ts：按 Agent.permission 过滤工具，传入 temperature/topP            │
│  • processor.ts：流式处理 tool call，权限不足时触发 ask                   │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Tool + Permission 层                                                    │
│  • ToolRegistry：所有工具列表，init(agent) 可做 Agent 相关分支            │
│  • PermissionNext：ruleset(allow/deny/ask) 决定工具是否可用/需询问        │
└─────────────────────────────────────────────────────────────────────────┘
```

**核心结论**：智能体 = **身份 + 权限 + 提示 + 模型参数**；会话层和工具层都**依赖 Agent.Info**，不做重复的“角色”定义。

---

## 二、推荐阅读顺序与要点

### 阶段 1：智能体“是什么”（定义与配置）

| 顺序 | 路径 | 关注点 |
|------|------|--------|
| 1 | `packages/opencode/src/agent/agent.ts` | `Agent.Info` 结构；内置 Agent（build/plan/general/explore）的 permission、prompt、mode；`state()` 与 Config 的 `agent` 合并逻辑；`get/list/defaultAgent/generate` |
| 2 | `packages/opencode/src/agent/prompt/*.txt` | explore/compaction/summary/title 等 prompt 文本，理解“不同 Agent 说不同的话” |
| 3 | `packages/opencode/src/config/config.ts`（agent 相关） | 用户/项目里 `agent` 配置如何被读取、合并到 Agent.state |

**设计要点**：
- **单一数据源**：所有 Agent 定义来自 `Agent.state()`，内置 + 配置覆盖/扩展。
- **mode**：`primary`（主 Agent，可作默认）/ `subagent`（被 @ 调用）/ `all`；hidden 的不出现在选择列表。
- **permission**：与 PermissionNext 规则一致，见下一阶段。

---

### 阶段 2：权限模型（Agent 能做什么）

| 顺序 | 路径 | 关注点 |
|------|------|--------|
| 1 | `packages/opencode/src/permission/next.ts` | `Rule`（permission + pattern + action）；`fromConfig` 把配置转成 Ruleset；`merge`；`disabled(tools, ruleset)` 如何算出被禁用的工具；`ask` 时如何发请求 |
| 2 | `packages/opencode/src/agent/agent.ts`（再次） | 每个内置 Agent 的 `PermissionNext.fromConfig({ ... })`，例如 plan 的 `edit: { "*": "deny", ... }`，build 的 `plan_enter/plan_exit: "allow"` |
| 3 | `packages/opencode/src/session/llm.ts` | `resolveTools` 里 `PermissionNext.disabled(toolIds, input.agent.permission)`，以及被禁用工具如何从传给模型的 tools 中移除 |

**设计要点**：
- **规则即数据**：permission 是 (permission_key, pattern, action) 的列表，不是硬编码 if/else。
- **分层合并**：默认规则 → 内置 Agent 规则 → 用户 Config.agent.permission，用 `PermissionNext.merge` 叠加。
- **粒度**：既有“整类工具”（如 `edit: "deny"`），也有“某路径”（如 `read: { "*.env": "ask" }`）。

---

### 阶段 3：工具与 Agent 的绑定

| 顺序 | 路径 | 关注点 |
|------|------|--------|
| 1 | `packages/opencode/src/tool/registry.ts` | `ToolRegistry.tools(model, agent)`；`all()` 工具列表；每个工具 `init({ agent })`；plugin 工具的加载与封装 |
| 2 | `packages/opencode/src/tool/tool.ts` | `Tool.Info`、`init(ctx?)` 的签名，理解 agent 如何传入工具 |
| 3 | 示例：`packages/opencode/src/tool/plan.ts` 或 `packages/opencode/src/tool/bash.ts` | 若存在 `init({ agent })` 分支，看如何按 Agent 改变参数/描述 |

**设计要点**：
- **工具注册与过滤分离**：Registry 负责“所有工具 + init(agent)”；LLM 层用 permission 再过滤一遍“当前 Agent 可见的工具”。
- **Agent 只做参数/描述**：工具内部可读 `agent` 做差异化，但“能不能用”由 permission 统一决定。

---

### 阶段 4：会话流中如何使用 Agent

| 顺序 | 路径 | 关注点 |
|------|------|--------|
| 1 | `packages/opencode/src/session/prompt.ts` | 搜索 `agent`：如何选 system prompt（agent.prompt vs provider）；如何组装 tools（ToolRegistry.tools + 过滤）；`insertReminders` 与 agent/session permission 的合并；maxSteps 等与 agent 的关系 |
| 2 | `packages/opencode/src/session/llm.ts` | `StreamInput` 含 `agent`；`resolveTools` 用 `agent.permission` 禁用工具；temperature/topP 用 `agent` 覆盖 |
| 3 | `packages/opencode/src/session/processor.ts` | 处理 tool call 时，权限为 ask 时的流程（发 PermissionNext 请求、等待用户、再继续）；doom_loop 等与“拒绝”相关的逻辑 |
| 4 | `packages/opencode/src/server/routes/session.ts` | 创建/恢复会话时如何取 `currentAgent`（defaultAgent / info.agent），并传给后续 session 逻辑 |

**设计要点**：
- **会话持有“当前 Agent”**：从创建/恢复时确定，一直到 prompt/llm 使用同一 `Agent.Info`。
- **Prompt 分层**：Agent 专属 prompt（如 plan.txt）> Provider 默认；再叠 instruction、reminder 等。
- **权限在“调用前”统一过滤**：LLM 只看到当前 Agent 允许的工具；执行时再按 ask 弹窗。

---

### 阶段 5：入口与多端一致

| 顺序 | 路径 | 关注点 |
|------|------|--------|
| 1 | `packages/opencode/src/server/routes/session.ts` | 会话创建参数、agent 的传递；与 `Agent.defaultAgent()` 的配合 |
| 2 | `packages/opencode/src/cli/cmd/tui/routes/session/header.tsx` 或 `index.tsx` | TUI 里切换 Agent 的 UI 与状态 |
| 3 | `packages/opencode/src/cli/cmd/tui/component/dialog-agent.tsx`、`dialog-subagent.tsx` | 主 Agent 选择、子 Agent 调用的交互 |
| 4 | `packages/opencode/src/acp/agent.ts` | ACP 协议下 session 的 mode/agent 如何与 `Agent.get/defaultAgent` 对应 |

**设计要点**：
- **多端共用同一套 Agent 定义与 API**：TUI/Server/ACP 都依赖 `agent/agent.ts` 和 Config，不各自维护一份“角色”列表。
- **子 Agent**：通过 mode=subagent 和 @general / @explore 等调用，会话侧区分“主会话 Agent”和“子任务 Agent”。

---

## 三、设计方法提炼（可复用到自己的项目）

1. **Agent 作为“配置实体”**  
   用一张表/一个状态描述：name、mode、permission、prompt、model、options；内置 + 配置合并，避免散落 if/else。

2. **权限与工具解耦**  
   工具列表由 Registry 管理；权限用 Ruleset(permission, pattern, action) 描述；“当前 Agent 可见工具”= 工具列表 + disabled(ruleset)。

3. **会话绑定当前 Agent**  
   会话创建/恢复时确定 Agent；之后 prompt、LLM、processor 都只读“当前会话的 Agent”，不在此层再做角色分支。

4. **Prompt 分层与覆盖**  
   Agent.prompt > Provider 默认；再叠 instruction、reminder；不同 Agent 可完全不同的 system prompt（如 plan vs build）。

5. **子 Agent 与主 Agent 同一套定义**  
   用 mode=subagent 和 permission 限制能力；通过 @mention 或协议字段切换“当前执行的 Agent”，实现并行/专用子任务。

6. **可扩展性**  
   新 Agent = 在 Config.agent 里加一项（或用 Agent.generate 生成）；新工具 = 在 ToolRegistry 注册；新权限粒度 = 在 PermissionNext 里加 pattern/action，无需改 Agent 枚举。

---

## 四、可画的图（帮助记忆）

- **数据流**：Config + 内置 → Agent.state() → Session 持有 currentAgent → prompt/llm 使用 Agent.Info → ToolRegistry.tools(model, agent) + PermissionNext.disabled → LLM 可见工具列表。
- **权限链**：默认 Ruleset → 内置 Agent Ruleset → 用户 agent.*.permission → merge；然后 disabled(tools, ruleset) → 传给模型的 tools。
- **文件依赖**：agent/agent.ts ← config；session/prompt.ts, session/llm.ts ← agent, tool/registry, permission/next；server/routes/session.ts, cli/…/session/* ← agent。

---

## 五、如何验证理解

1. **改 plan Agent**：只允许 `read` 和 `grep`，禁止 `bash`，看 permission 如何改、llm 的 resolveTools 是否生效。  
2. **加一个自定义 Agent**：在 opencode 配置里加 `agent.my_agent`，设 permission、prompt，在 TUI 里能否选中并生效。  
3. **追踪一次请求**：从 TUI 发一条消息，在 session 创建、prompt 组装、llm.stream、processor 里各打 log，确认同一 Agent.Info 贯穿全程。

按上述阶段顺序读源码，再配合本总结，即可系统掌握本项目的智能体设计方法。
