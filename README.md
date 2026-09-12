# Systematic Thinking

## 陌生项目快速上手 · 源码理解 · Agent 架构复盘 · 面试表达

> 不从术语开始背，从问题开始推导。  
> 不把类名当答案，把调用链走通才算理解。  
> 不追求“看起来高级”，只解释每一层为什么存在。

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Version](https://img.shields.io/badge/version-3.0-green.svg)
![Focus](https://img.shields.io/badge/focus-codebase%20%7C%20architecture%20%7C%20interview-orange.svg)

## 这是什么？

`systematic-thinking` 是一个面向源码学习和技术面试的 Codex Skill。

它解决的是一种常见的学习断层：

```text
会背 Future 的定义
        ↓
却不知道为什么 MCP 需要 CompletableFuture

知道线程池能并行
        ↓
却说不清线程、Worker、Future 和对象池的关系

知道 ReAct、DAG、Tool Calling
        ↓
却无法从用户输入追到真实代码调用链
```

这个 Skill 用问题驱动的方式把它们串起来：

```text
最简单方案 → 暴露问题 → 引入一层机制 → 利用特性 → 解决问题 → 分析代价与边界
```

## 什么时候使用？

- 接手陌生项目或复杂源码，不知道从哪里开始；
- 能背 Future、线程池、TCP、Agent 等定义，却不会落到代码；
- 想知道两个组件为什么同时存在，或某个中间层是不是多余；
- 需要理解 Agent、MCP、RAG、DAG、Memory、并发和协议的真实关系；
- 准备项目面试，想把源码事实整理成可防守的架构回答。

## 核心方法：五问法

### 1. 最简单方案是什么？

先假设只使用已经掌握的基础能力：一个方法、一个线程、一个 Map、一次 HTTP 调用，能不能完成？

### 2. 它在哪里失败？

从并发、跨进程、超时、扩展性、一致性和上下文膨胀等真实场景找失败点。

### 3. 加入了哪一层？

可能是协议、队列、Future、线程池、锁、缓存、状态机、DAG、对象池或路由器。

### 4. 它利用了什么特性？

例如：`CompletableFuture` 允许另一个线程主动完成结果；`requestId` 让跨进程响应可以回到正确请求；`ExecutorService` 负责复用线程并执行任务。

### 5. 最终解决了什么，又没解决什么？

明确边界：`ConcurrentHashMap` 保护 Map 结构，不代表取出的 Tool 的外部副作用天然线程安全；`Future.get(timeout)` 释放调用方等待，也不代表工作线程已经终止。

## 推荐学习路径

```text
main / CLI 入口
    ↓
一次最简单的数据流
    ↓
每个包一句话职责
    ↓
核心对象的创建方、调用方和状态
    ↓
并发、协议、存储和失败分支
    ↓
替代方案与工程取舍
    ↓
60～120 秒面试表达
```

### 一次代码 Agent 调用链

```text
用户输入
→ CLI 解析
→ Agent 组装 system / user / tools
→ LLM 返回 Tool Call
→ ToolRegistry 找到工具
→ Java Tool 或 MCP Server 执行
→ Tool Result 回灌消息历史
→ 再次调用 LLM
→ 无 Tool Call 时返回最终结果
```

### 一次 MCP Future 调用链

```text
直接让每个请求线程读取 stdout
→ 多线程竞争共享输入流，响应可能错配
→ 单一 stdout reader
→ requestId 解复用
→ pending Map 找到对应 Future
→ CompletableFuture.complete 唤醒请求线程
```

这条链路中：

```text
requestId：跨进程携带的请求身份
ConcurrentHashMap：ID 到等待对象的并发映射
CompletableFuture：JVM 内可由另一个线程主动完成的结果容器
```

## Agent 系统如何分层？

```text
LLM              产生文本或结构化 Tool Call
Agent            维护消息历史并运行 ReAct
Orchestrator     推进任务、依赖、重试和汇总
ToolRegistry     找到本地 Tool 或 MCP Bridge
MCP Client       处理 tools/list、tools/call 等业务语义
JSON-RPC Client  处理 ID、Response、Notification 和 Future
Transport        通过 stdio 或 HTTP 运输 JSON
MCP Server       执行外部工具
```

最容易混淆的两个 ID：

```text
LLM Tool Call ID       用于 Agent 回灌 tool message
JSON-RPC Request ID    用于 MCP Client 关联 Server Response
```

## 面试表达模板

```text
我当时要解决的是：<具体工程问题>。
最简单的实现是：<基础方案>，但在 <场景> 下会出现 <失败点>。
所以我增加了：<机制>，利用它的 <关键特性>，把 <请求/状态/结果> 关联起来。
在代码里，<组件A> 负责创建或调度，<组件B> 负责执行，<组件C> 负责接收和回传结果。
这个方案解决了 <结果>，但边界是 <未覆盖部分>；进一步可以补 <演进方向>。
```

## 诚实边界

这个 Skill 强调“可防守的答案”：

- 没有自动 Router，就不要说系统会自动判断任务复杂度；
- 有线程池，不要说所有工具副作用都线程安全；
- 有 MCP 核心流程，不要说完整覆盖 OAuth、Sampling、Recovery；
- 有缓存统计，不要说每次请求都必然命中；
- 没有真实指标，就不要编造百分比。

## 文件说明

| 文件 | 用途 |
|---|---|
| [`SKILL.md`](SKILL.md) | Codex Skill 入口、触发条件和执行规则 |
| [`README.md`](README.md) | GitHub 首页和完整学习方法 |
| [`architecture-guide.md`](architecture-guide.md) | PaiCLI 架构深入示例 |

## 使用

将本仓库放入 Codex 的 skills 目录，或在支持 Skill 的环境中加载：

```text
load_skill("systematic-thinking")
```

示例请求：

```text
用“最简单方案→问题→引入机制”的方式，解释这个项目的 MCP 请求为什么需要 CompletableFuture。
```

## License

MIT
