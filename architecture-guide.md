# PaiCLI 架构全景学习指南

> 写给新加入项目的同学：从零理解这个 Java Agent CLI 的完整架构。

---

## 目录

1. [项目概览](#一项目概览)
2. [从启动到运行：完整数据流](#二从启动到运行的完整数据流)
3. [POJO 与状态机设计](#三pojo-与状态机设计)
4. [三种 Agent 运行模式](#四三种-agent-运行模式)
5. [工具系统与 HITL 审批](#五工具系统与-hitl-审批)
6. [记忆与上下文工程](#六记忆与上下文工程)
7. [LLM 客户端与 Prompt 系统](#七llm-客户端与-prompt-系统)
8. [CLI 命令与 Skill 系统](#八cli-命令与-skill-系统)
9. [设计模式速查](#九设计模式速查)
10. [关键文件索引](#十关键文件索引)

---

## 一、项目概览

PaiCLI 是一个 Java 实现的终端 AI 编程助手（对标 Claude Code），版本 v16.1.0。

### 包结构鸟瞰

```
com.paicli
├── agent/        ─ 三种Agent模式 (ReAct / Plan-Execute / Multi-Agent)
├── browser/      ─ Chrome CDP 浏览器会话管理
├── cli/          ─ CLI入口 Main、命令解析、补全、历史
├── config/       ─ 配置文件 (~/.paicli/config.json)
├── context/      ─ 上下文窗口策略 (ContextProfile)
├── hitl/         ─ Human-in-the-Loop 审批流
├── image/        ─ 剪贴板图片抓取
├── llm/          ─ LLM客户端 (7个Provider实现)
├── lsp/          ─ 语言服务器诊断集成
├── mcp/          ─ MCP协议 (Server管理/JSON-RPC/Transport)
├── memory/       ─ 短期记忆 + 长期记忆 + 压缩器
├── plan/         ─ 执行计划数据结构 (ExecutionPlan/Task/Planner)
├── policy/       ─ PathGuard + CommandGuard + AuditLog
├── prompt/       ─ Prompt模板仓库 (三级覆写)
├── rag/          ─ 代码向量化检索 (SQLite + Embedding)
├── render/       ─ 渲染器 (inline/lanterna/plain)
├── runtime/      ─ 取消令牌、后台任务、HTTP API
├── skill/        ─ Skill三层加载 + SKILL.md注入
├── snapshot/     ─ Side-Git快照 (JGit)
├── tool/         ─ 工具注册表 (13个内置工具 + MCP动态工具)
├── tui/          ─ Lanterna全屏TUI
├── util/         ─ ANSI样式工具
├── web/          ─ SearchProvider + WebFetcher + HtmlExtractor
└── wechat/       ─ 微信iLink通道
```

### 三层架构

```
┌──────────────────────────────────────┐
│  表示层  CLI/TUI/Renderer             │  ← JLine终端 → 命令解析 → 流式渲染
├──────────────────────────────────────┤
│  编排层  Agent                        │  ← ReAct / Plan-Execute / Multi-Agent
├──────────────────────────────────────┤
│  基础层  LLM / Tool / Memory / MCP   │  ← 模型调用 / 工具执行 / 记忆管理
└──────────────────────────────────────┘
```

---

## 二、从启动到运行：完整数据流

### 2.1 Main.main() 启动流程

```
Main.main()
  ├─ 1. PaiCliConfig.load()           → 加载 ~/.paicli/config.json
  ├─ 2. LlmClientFactory.createFromConfig() → 创建 LLM 客户端
  ├─ 3. Terminal (JLine) + LineReader → 终端交互层
  ├─ 4. HitlToolRegistry             → 工具注册 + 审批拦截
  ├─ 5. McpServerManager.startAll()  → 启动MCP子进程
  ├─ 6. RendererFactory.create()     → inline/plain/lanterna
  ├─ 7. SkillRegistry.reload()       → 三层加载skill
  ├─ 8. Agent(llmClient, hitlToolRegistry) → 创建ReAct Agent
  └─ 9. CLI主循环 while(true):
        ├─ readPromptInput()         → JLine读取输入
        ├─ CliCommandParser.parse()  → 判断命令还是对话
        ├─ [slash命令] → switch分发 → 各种Handler
        └─ [对话输入] → Agent.run()  → ReAct循环
```

### 2.2 一次 ReAct 对话的完整路径

```
用户: "帮我读 README.md"

Agent.run(userInput)
  ├─ memoryManager.addUserMessage()       ← 存短期记忆
  ├─ buildContextForQuery("帮我读 README.md") ← 检索长期记忆
  ├─ updateSystemPromptWithMemory()       ← 注入记忆到system prompt
  ├─ conversationHistory.add(userMsg)     ← 用户消息入历史
  │
  ├─ while(true):  ← ReAct主循环
  │   ├─ llmClient.chat(history, tools, streamListener)
  │   │   ├─ 构造HTTP POST → LLM API
  │   │   ├─ 解析SSE流式响应:
  │   │   │   ├─ reasoning_content → onReasoningDelta → "🧠 思考过程"
  │   │   │   └─ content → onContentDelta → 终端实时打字
  │   │   └─ 返回 ChatResponse(content, toolCalls, tokens)
  │   │
  │   ├─ 有 toolCalls? [read_file("README.md")]
  │   │   ├─ conversationHistory.add(assistant+toolCalls)
  │   │   ├─ HitlToolRegistry.executeTools()
  │   │   │   ├─ read_file 不需要审批 → 直接执行
  │   │   │   ├─ Files.readString(path) → 返回内容
  │   │   │   └─ AuditLog.record()
  │   │   ├─ conversationHistory.add(tool result)
  │   │   └─ continue  ← 下一轮
  │   │
  │   └─ 无 toolCalls → 完成
  │       ├─ memoryManager.addAssistantMessage()
  │       ├─ memoryManager.recordTokenUsage()
  │       └─ return response.content()
  │
  └─ ui.println(response)  ← 输出最终结果
```

---

## 三、POJO 与状态机设计

### 3.1 核心 Record / Enum 一览

| 包 | 类 | 类型 | 用途 |
|---|---|---|---|
| agent | AgentMessage | record | Multi-Agent间通信 (TASK/RESULT/FEEDBACK/APPROVAL/REJECTION/ERROR) |
| agent | AgentRole | enum | PLANNER/WORKER/REVIEWER |
| agent | AgentBudget | class | Token预算 + 停滞检测 + 硬轮数上限 |
| plan | ExecutionPlan | class | DAG执行计划: id/goal/tasks/executionOrder/status |
| plan | Task | class | 任务节点: id/描述/类型/状态/依赖双向链表/result |
| llm | LlmClient.Message | record | LLM消息: role/content/reasoningContent/toolCalls/contentParts |
| llm | LlmClient.Tool | record | 工具定义: name/description/parameters(JSON Schema) |
| llm | LlmClient.ToolCall | record | 工具调用: id/function(name+arguments) |
| llm | LlmClient.ChatResponse | record | LLM响应: content/toolCalls/inputTokens/outputTokens/cachedInputTokens |
| llm | LlmClient.ContentPart | record | 多模态片段: text/image_base64/image_url |
| llm | LlmClient.StreamListener | interface | 流式回调: onReasoningDelta() / onContentDelta() |
| hitl | ApprovalRequest | record | 审批请求: toolName/arguments/dangerLevel/riskDescription |
| hitl | ApprovalResult | record | 审批结果: decision(6种)/modifiedArguments/reason |
| memory | MemoryEntry | class | 记忆条目: id/content/type/metadata/createdAt |

### 3.2 关键状态机

#### ExecutionPlan
```
CREATED ──markStarted()──→ RUNNING ──markCompleted()──→ COMPLETED
                                │
                                └──markFailed()──→ FAILED
```

#### Task
```
PENDING ──markStarted()──→ RUNNING ──markCompleted()──→ COMPLETED
    │                         │
    │                         └──markFailed()──→ FAILED
    └──markSkipped()──→ SKIPPED
```

#### Task 可执行判定 (Task.isExecutable)
```java
return status == PENDING
    && 所有依赖的 status == COMPLETED;
```

### 3.3 ExecutionPlan 的 DAG 拓扑排序

使用 **三色 DFS 后序**：

```java
topologicalSort(task, visited, visiting):
  1. if visiting.contains(id) → return false  // 检测环
  2. if visited.contains(id)  → return true   // 已处理
  3. visiting.add(id)
  4. for each dep in task.dependencies:
       topologicalSort(dep, visited, visiting) // 递归处理依赖
  5. visiting.remove(id); visited.add(id)
  6. executionOrder.add(id)  // 后序：依赖在前，自身在后
```

**批次划分** (getExecutionBatches): BFS 分层，每轮取出"依赖全完成"的任务作为一个可并行的 batch。

### 3.4 Java Record 设计模式

项目大量使用 record 作为不可变 DTO：

```java
// 计划审阅决策
public record PlanReviewDecision(PlanReviewAction action, String feedback) {
    public static PlanReviewDecision execute() { ... }
    public static PlanReviewDecision supplement(String feedback) { ... }
    public static PlanReviewDecision cancel() { ... }
}

// 工具执行结果
public record ToolExecutionResult(
    String id, String name, String argumentsJson,
    String result, long elapsedMillis, boolean timedOut,
    List<LlmClient.ContentPart> imageParts
) {}
```

**设计要点**: record不可变 → 线程安全；静态工厂方法 → 语义清晰；volatile字段(Task中) → 多线程可见性。

---

## 四、三种 Agent 运行模式

### 4.1 ReAct（默认模式）

**文件**: `Agent.java:127-260`
**入口**: 直接输入对话

```
while(true):
  调LLM → 有工具调用? → 执行工具 → 结果回灌 → 继续
                     → 无工具调用 → 结束
```

**特点**:
- 一个 conversationHistory 从头到尾累积
- LLM 自己决定下一步
- 适合简单任务、单步操作

### 4.2 Plan-Execute

**文件**: `PlanExecuteAgent.java`
**入口**: `/plan` 命令

```
Planner.createPlan(goal)          ← LLM一次生成JSON计划
  ↓
PlanReviewHandler.review()        ← 用户审阅确认
  ↓
executePlan():
  while(有可执行任务):
    getExecutableTasksInOrder()    ← DAG可执行判断
    executeTaskBatch()             ← 同批并行(≤4线程)
      └─ 每个task内: 独立ReAct循环(≤5轮)
  ↓
失败早期 → planner.replan()       ← 保留已完成，重新规划
```

**关键差异 vs ReAct**:
1. Planner 是独立 LLM 调用生成 JSON 计划
2. 每个 task 有**独立 messages 列表**（上下文隔离）
3. 有用户审阅节点 (PlanReviewHandler)
4. 独立任务可以并行执行

### 4.3 Multi-Agent

**文件**: `AgentOrchestrator.java`
**入口**: `/team` 命令

```
第一阶段: Planner SubAgent → 生成 JSON 计划
第二阶段: 按依赖顺序执行
  runStep() × N:
    WORKER SubAgent → 执行任务 (有工具)
    REVIEWER SubAgent → 审查结果 (无工具)
      ├─ 通过 → 标记完成
      └─ 拒绝 → 重试(最多2次)
```

**三个角色**: PLANNER(规划) / WORKER(执行+有工具) / REVIEWER(审查+无工具)

| 维度 | Agent.java | SubAgent.java |
|------|-----------|---------------|
| 角色 | 无角色 | PLANNER/WORKER/REVIEWER |
| 工具权限 | 始终有 | 仅WORKER有 |
| 记忆管理 | 完整MemoryManager | 无长期记忆 |
| 对话生命周期 | 跨多轮 | 每个任务后clearHistory() |
| 返回值 | String | AgentMessage |
| 设计意图 | 终端用户独立Agent | 编排器的轻量工作单元 |

---

## 五、工具系统与 HITL 审批

### 5.1 工具注册

`ToolRegistry` 构造时注册 13 个内置工具：
- 文件: `read_file`, `write_file`, `list_dir`, `glob_files`, `grep_code`
- Shell: `execute_command`
- 代码: `create_project`, `search_code` (RAG)
- Web: `web_search`, `web_fetch`
- 记忆: `save_memory`
- Skill: `load_skill`
- 快照: `revert_turn`
- 动态: MCP工具 (`mcp__{server}__{tool}` 前缀)

### 5.2 工具调用全链路

```
LLM返回 toolCalls:[{read_file, {path:"README.md"}}]
  │
  ├─ Agent层: 助手消息入history → executeToolCalls()
  │   └─ toolRegistry.executeTools(invocations)
  │       ├─ 单工具 → 同步; 多工具 → 线程池并行(≤4)
  │
  ├─ HitlToolRegistry层: ← 装饰器拦截
  │   ├─ HITL关闭 / 不需要审批 → super.doExecuteTool()
  │   └─ 需要审批 → hitlHandler.requestApproval()
  │       ├─ y/Enter → 批准
  │       ├─ a → 全部放行(工具级/Server级)
  │       ├─ m → 修改参数后执行
  │       ├─ n → 拒绝
  │       └─ s → 跳过
  │
  ├─ ToolRegistry.doExecuteTool():
  │   ├─ PathGuard.checkOutsideProject()  ← 路径围栏
  │   ├─ CommandGuard.evaluate()          ← 命令风险
  │   ├─ 路由到执行器 (read_file→Files.readString)
  │   └─ AuditLog.record()                ← 审计
  │
  └─ 回到Agent: 工具结果回灌history → continue
```

### 5.3 HITL 设计

**装饰器模式**: `HitlToolRegistry extends ToolRegistry`，覆写 executeTool 在调父类前插入审批。

**ApprovalPolicy**: 4个内置危险工具 + 所有MCP工具需要审批。

**三个 Handler 实现**:
- `TerminalHitlHandler`: System.in/out 交互
- `RendererHitlHandler`: 委托给 Renderer (inline/lanterna)
- `SwitchableHitlHandler`: 运行时切换委托

**三层安全**: PathGuard(路径围栏) → CommandGuard(命令黑名单) → HITL(人工审批)。策略拒绝是硬性的，用户无法通过HITL覆盖。

---

## 六、记忆与上下文工程

### 6.1 三层记忆架构

```
System Prompt
  ├─ 角色定义 + 工具列表
  ├─ Project Context (PAI.md)              ← 项目记忆
  ├─ Memory Context (长期记忆检索结果)      ← 注入上限500 tokens
  └─ External Context (MCP resource 索引)

conversationHistory                        ← 短期记忆(累积)
  ├─ user message
  ├─ assistant + toolCalls
  └─ tool result

LongTermMemory (~/.paicli/memory/*.json)   ← 长期记忆(持久化)
  ├─ scope: project / global
  └─ 跨会话保留
```

### 6.2 记忆检索流程

```
用户输入"上次那个登录功能"
  ↓
MemoryManager.buildContextForQuery("登录功能", maxTokens)
  ↓
MemoryRetriever.retrieve():
  ├─ LongTermMemory.search("登录功能")  ← jieba分词+关键词匹配
  ├─ 相关度评分:
  │   ├─ 精确匹配 → 1.0
  │   ├─ 关键词匹配比例
  │   ├─ 时间衰减: max(0.5, 1.0-ageHours/24)
  │   └─ 长期记忆加权 ×1.2
  ├─ 过滤: scope=project才可见
  └─ 返回格式: "## 相关长期记忆\n\n- [FACT] ..."
  ↓
注入 system prompt 的 Memory Context 段
```

### 6.3 两套压缩机制

| | ContextCompressor | ConversationHistoryCompactor |
|---|---|---|
| 压缩对象 | MemoryEntry (内存中) | LlmClient.Message (发LLM的) |
| 触发条件 | 短期记忆 ≥ 窗口×75% | history估计tokens ≥ triggerTokens |
| 策略 | Map-Reduce: 旧条目分片→摘要→合并 | 保留最近3个user round，前面压缩为摘要 |
| 副作用 | 自动提取事实→长期记忆 | 重建 conversationHistory |

---

## 七、LLM 客户端与 Prompt 系统

### 7.1 LlmClient 接口

```java
public interface LlmClient {
    ChatResponse chat(List<Message> messages, List<Tool> tools, StreamListener listener);

    String getModelName();      // 当前模型名
    String getProviderName();   // provider标识
    int maxContextWindow();     // 默认128k
    boolean supportsTools();    // 是否支持function calling
    boolean supportsImageInput(); // 是否支持多模态

    interface StreamListener {
        void onReasoningDelta(String delta);  // 思维链增量
        void onContentDelta(String delta);    // 回复增量
    }
}
```

### 7.2 7个 Provider 实现

**模板方法模式**: `AbstractOpenAiCompatibleClient` 是基类，子类覆写差异。

| Provider | Client | 特殊处理 |
|---|---|---|
| GLM | GLMClient | glm-5v多模态用不同API URL |
| DeepSeek | DeepSeekClient | HTTP/1.1强制; reasoning回传 |
| StepFun | StepClient | StepSearch MCP集成; reasoning_format |
| Kimi | KimiClient | reasoning回传 |
| 讯飞MaaS | XfyunMaaSClient | LoRA支持; 不支持tools |
| Agnes | AgnesClient | — |
| FreeLLMAPI | FreeLlmApiClient | 自定义baseUrl |

**工厂方法**: `LlmClientFactory.createFromConfig()` → 按 defaultProvider 创建

### 7.3 Prompt 组装

6种 PromptMode，对应不同模板文件：

| PromptMode | 模板文件 | 使用场景 |
|---|---|---|
| AGENT | modes/agent.md | ReAct Agent |
| PLAN | modes/plan.md | Plan-Execute 任务执行 |
| PLANNER | modes/planner.md | 生成执行计划 |
| TEAM_PLANNER | modes/team-planner.md | Multi-Agent 规划者 |
| TEAM_WORKER | modes/team-worker.md | Multi-Agent 执行者 |
| TEAM_REVIEWER | modes/team-reviewer.md | Multi-Agent 审查者 |

**三级覆写**: classpath内置 → `~/.paicli/prompts/` → `.paicli/prompts/`

### 7.4 流式渲染分区策略

reasoning_content 和 content 在终端上**分区展示**：

```
🧠 思考过程                 ← reasoning区域
我需要先读取当前文件...

▪                           ← answer marker 分隔
好的，让我帮你分析这个文件...  ← content区域
```

处理边界情况: content开始后收到reasoning → 缓冲为lateReasoning → finish时以"🧠 补充思考"独立展示。

### 7.5 reasoning_content 的 Provider 差异

| Provider | 字段名 | 回传history? | 说明 |
|---|---|---|---|
| DeepSeek | reasoning_content | ✅ | HTTP/1.1强制 |
| GLM | reasoning_content / reasoning_details | ❌ | |
| Step | reasoning_content (deepseek-style) | ❌ | req: reasoning_format |
| Kimi | reasoning_content | ✅ | |
| Agnes | reasoning_content | ❌ | |
| 讯飞MaaS | 取决于底层模型 | ❌ | 不支持tools |
| FreeLLMAPI | 取决于底层模型 | ❌ | |

---

## 八、CLI 命令与 Skill 系统

### 8.1 命令解析路由

`CliCommandParser.parse()` 返回 `ParsedCommand(CommandType, payload)` → `Main` 中 `switch` 分发。

**执行模式切换**: `/plan` 和 `/team` 仅对**下一条任务**生效，执行后自动回 ReAct。

### 8.2 Skill 三层加载

```
builtin (jar内资源 → ~/.paicli/skills-cache/)
  ↓ 同名覆盖
user (~/.paicli/skills/<name>/SKILL.md)
  ↓ 同名覆盖
project (.paicli/skills/<name>/SKILL.md)
```

- SKILL.md 带 YAML frontmatter（自实现极简解析器）
- 启用/禁用状态持久化到 `~/.paicli/skills.json`
- Skill body 通过 `SkillContextBuffer` 注入: LLM调load_skill → push到buffer → 下一轮drain注入user message

### 8.3 终端交互

| 组件 | 实现 | 功能 |
|---|---|---|
| PaiCliCompleter | JLine Completer | slash命令子命令级联补全 + @mention补全 + 本地路径补全 |
| PaiCliHighlighter | JLine Highlighter | 命令(青色)、@mention(蓝色)、危险(sudo红色)、密钥(黄色) |
| PaiCliHistory | JLine DefaultHistory | 安全过滤: 敏感信息/API key/Base64图片 不写入历史 |
| Ctrl+O | 行内绑定 | 展开/收起可折叠工具块 |
| Ctrl+V | 行内绑定 | 抓剪贴板图片→@image:path |
| ESC | 行内绑定 | 清空输入行 / 取消Plan标记 |

### 8.4 Renderer 三种形态

`RendererFactory.resolveMode()`:

| 模式 | 触发条件 | 实现 |
|---|---|---|
| inline | 默认 | InlineRenderer (Claude Code风格流式+状态栏+可折叠块) |
| plain | dumb终端/PAICLI_RENDERER=plain | PlainRenderer (纯println) |
| lanterna | PAICLI_RENDERER=lanterna/PAICLI_TUI=true | 全屏TUI (文件树+代码高亮+对话历史) |

---

## 九、设计模式速查

| 模式 | 应用位置 | 说明 |
|---|---|---|
| **模板方法** | AbstractOpenAiCompatibleClient | chat()核心在基类，子类覆写URL/headers/body |
| **策略** | SearchProvider (GLM/SerpAPI/SearXNG) | web_search三实现可切换 |
| **工厂方法** | LlmClientFactory | 按provider创建对应Client |
| **装饰器** | HitlToolRegistry extends ToolRegistry | 不改工具逻辑插入审批 |
| **门面** | MemoryManager | 短期记忆+长期记忆+压缩的统一入口 |
| **观察者** | StreamListener / StatusInfo | 流式回调 + 状态栏更新 |
| **命令** | CliCommandParser → CommandType switch | 解析→路由→执行 |
| **状态机** | Task / ExecutionPlan | PENDING→RUNNING→COMPLETED/FAILED |
| **Record (值对象)** | AgentMessage, ApprovalRequest, PlanReviewDecision | 不可变纯数据载体 |
| **依赖注入** | 构造器注入 | 所有组件通过构造器注入，方便测试 |

---

## 十、关键文件索引

### 入口与配置
- `src/main/java/com/paicli/cli/Main.java` — 启动入口 (2938行)
- `src/main/java/com/paicli/config/PaiCliConfig.java` — 配置加载
- `src/main/java/com/paicli/cli/CliCommandParser.java` — 命令解析

### Agent 核心
- `src/main/java/com/paicli/agent/Agent.java` — ReAct 主循环
- `src/main/java/com/paicli/agent/PlanExecuteAgent.java` — Plan-Execute 模式
- `src/main/java/com/paicli/agent/AgentOrchestrator.java` — Multi-Agent 编排
- `src/main/java/com/paicli/agent/SubAgent.java` — 子Agent (Planner/Worker/Reviewer)
- `src/main/java/com/paicli/agent/AgentBudget.java` — Token预算+停滞检测
- `src/main/java/com/paicli/agent/AgentRole.java` — 角色枚举
- `src/main/java/com/paicli/agent/AgentMessage.java` — 通信消息

### 计划系统
- `src/main/java/com/paicli/plan/Planner.java` — LLM计划生成
- `src/main/java/com/paicli/plan/ExecutionPlan.java` — DAG计划数据结构
- `src/main/java/com/paicli/plan/Task.java` — 任务节点

### 工具与安全
- `src/main/java/com/paicli/tool/ToolRegistry.java` — 工具注册表
- `src/main/java/com/paicli/hitl/HitlToolRegistry.java` — HITL装饰器
- `src/main/java/com/paicli/hitl/ApprovalPolicy.java` — 危险工具判定
- `src/main/java/com/paicli/policy/PathGuard.java` — 路径围栏
- `src/main/java/com/paicli/policy/CommandGuard.java` — 命令黑名单

### 记忆系统
- `src/main/java/com/paicli/memory/MemoryManager.java` — 记忆门面
- `src/main/java/com/paicli/memory/ConversationMemory.java` — 短期记忆
- `src/main/java/com/paicli/memory/LongTermMemory.java` — 长期记忆(JSON持久化)
- `src/main/java/com/paicli/memory/ConversationHistoryCompactor.java` — 对话压缩
- `src/main/java/com/paicli/memory/ContextCompressor.java` — Map-Reduce压缩
- `src/main/java/com/paicli/memory/MemoryRetriever.java` — 记忆检索
- `src/main/java/com/paicli/context/ContextProfile.java` — 上下文策略

### LLM 客户端
- `src/main/java/com/paicli/llm/LlmClient.java` — LLM接口
- `src/main/java/com/paicli/llm/AbstractOpenAiCompatibleClient.java` — 模板基类
- `src/main/java/com/paicli/llm/LlmClientFactory.java` — 工厂
- `src/main/java/com/paicli/llm/LlmTraceLogger.java` — 推理日志
- `src/main/java/com/paicli/prompt/PromptAssembler.java` — Prompt组装

### 渲染器
- `src/main/java/com/paicli/render/Renderer.java` — 渲染器接口
- `src/main/java/com/paicli/render/inline/InlineRenderer.java` — inline流式渲染
- `src/main/java/com/paicli/render/PlainRenderer.java` — 纯文本兜底

### Skill 系统
- `src/main/java/com/paicli/skill/SkillRegistry.java` — Skill注册表
- `src/main/java/com/paicli/skill/SkillContextBuffer.java` — Skill上下文缓冲
- `src/main/java/com/paicli/skill/SkillFrontmatterParser.java` — 自实现YAML解析

---

*文档生成时间: 2026-07-21 | 基于 PaiCLI v16.1.0*
