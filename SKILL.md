---
name: systematic-thinking
description: 陌生项目快速上手解剖术——当你面对一个完全看不懂的复杂项目时，从哪下刀、按什么顺序读、怎么把看到的代码跟自己学过的知识对上号，最终形成系统理解
version: "2.0"
author: paicli
tags: [learning, codebase, onboarding, thinking, methodology, beginner]
---

# 陌生项目快速上手：从菜鸡到大师的代码库解剖术

> 你打开一个实习项目，几百个 Java 文件，不知道从哪开始看。同事讲的术语（Agent、ReAct、DAG、熔断）你一个都不懂。你怀疑自己是不是太菜了。
>
> **不是你的问题。是你缺一套解剖陌生代码库的流程。**
>
> 这套流程只需要你已有的知识：Java 基础、JUC 的 Future/线程池、OS 课的进程/线程、计网的 TCP。用你知道的去推导你不知道的。

---

## 心态准备：三个核心信念

**信念 1：任何复杂的系统，都是简单的东西一层层叠出来的。**

Kubernetes 底层是 Go 的 goroutine。PaiCLI 底层是 JDK 的 `ExecutorService`。你看不懂是因为还没找到那个"简单的东西"在哪一层。

**信念 2：你读代码的目的不是"全懂"，而是"建立索引"。**

就像你去图书馆不是为了读完所有书，而是记住哪类书在几楼。读项目也一样——每个包干什么、出了问题去哪找、想加功能改哪个文件。全懂是结果，索引是手段。

**信念 3：遇到不懂的概念，先问"这是为了解决什么问题"。**

不要查 API 怎么用，先查它要解决什么。ReAct 看不懂？别管代码。先问：它和 Plan-Execute 的区别是什么？什么时候用哪个？问题清楚了，代码自然读得懂。

---

## 第一刀切哪里：从入口建立调用链

不管你面对什么项目，第一刀永远切在 `main()`。

### Step 1：画出启动流程

打开 `Main.java`，不要逐行读。找这几样东西：

```
1. 配置文件从哪加载的？  →  PaiCliConfig.load()
2. 核心对象在哪创建的？  →  new Agent(llmClient, toolRegistry)
3. 循环入口在哪里？      →  while(true) { readLine() → agent.run() }
4. 有哪些外部依赖？      →  JLine(终端) / OkHttp(网络) / SQLite(存储)
```

**只画 10 步以内**。10 步之后你已经不在"启动"阶段了，后面再说。

### Step 2：画一次完整的数据流

从用户输入一句话到输出结果，经过哪些步骤？用最粗的粒度：

```
用户输入 "帮我读 README.md"
  → CLI 解析 (是命令还是对话?)
  → Agent.run()
  → 构建 system prompt (注入记忆、skill、项目上下文)
  → 调 LLM API (发送 messages + tools)
  → LLM 返回 tool_calls [read_file("README.md")]
  → 执行工具 (ToolRegistry → 读文件)
  → 结果回灌到消息历史
  → 再次调 LLM (现在它看到了文件内容)
  → LLM 返回最终回复
  → 流式渲染到终端
```

**只画一次完整路径**。分支、异常、并行后面再看。先建立"主路径"的脑图。

### Step 3：拆包——每个包一句话说清楚职责

打开项目目录，对每个 package 用一句话回答"这个包是为了解决什么问题存在的"：

```
agent/     → "怎么让 LLM 能多轮对话、能自己调工具、能把复杂任务拆开执行"
tool/      → "LLM 说要调工具，谁来真正执行？怎么防止它写坏文件？"
memory/    → "对话太长怎么办？上次说的关键信息这次怎么还记得？"
llm/       → "怎么把 7 家不同的大模型厂商统一成一个接口调用？"
hitl/      → "LLM 想执行 rm -rf，谁来拦住它？"
plan/      → "复杂任务怎么拆成 A→B→C 的顺序执行？"
```

**如果某个包一句话说不清，说明你没理解它的定位。回到 Step 2 看看数据流中它负责哪一段。**

---

## 从已知推导未知：别急着查文档

看到一段代码用了你没见过的技术，不要立刻 Google。先问自己：

### 如果让我用已学的知识来实现，我会怎么做？

**案例**：看到项目里有 7 个 LLM Client（GLM/DeepSeek/Step/Kimi...），你第一反应是"这些类怎么写的"。

停。先回答这个问题：

> "如果让我用 Java 基础实现'同一套接口、不同厂商有不同实现'，我会怎么做？"

你学过的知识：
- 接口 `interface LlmClient { String chat(...) }`
- 7 个类 `class DeepSeekClient implements LlmClient`
- 怎么创建？`new DeepSeekClient()` — 但代码里不能写死
- 怎么按配置动态创建？→ 工厂模式 `LlmClientFactory.create("deepseek")`

**现在去看项目代码，你会发现它就是按你的思路写的。** 你只是不知道它叫"工厂模式"——但你已经会了。

### 看到不认识的 API，先猜它的作用

```java
// 你在项目中看到这行代码
List<Future<TaskExecutionResult>> futures = executor.invokeAll(tasks, 90, TimeUnit.SECONDS);
```

不要查 `invokeAll` 的文档。先猜：

> "executeTaskBatch 要并行跑 4 个 Task。如果逐个 future.get()，每个都要等，总时间是 4 倍。invokeAll 应该是一次提交所有任务，90 秒统一计时，超时后自动 cancel 没完成的。它比逐个 get 强在哪？统一计时起点。"

**用自己的话把 API 的作用说一遍，再看文档验证。** 这样你记住的不是"invokeAll 有三个参数"，而是"invokeAll 解决的问题是批量超时控制"。

---

## 解剖四步法：深入任何一个模块

当你需要深入理解一个具体模块时（比如面试被问到"你们的并发熔断怎么实现的"），用这四步：

### Step 1: 定义问题——拆成子问题

```
主问题：4 个并发 Task 中有一个卡死了，怎么处理？

拆解：
  Q1: 怎么知道它卡死了？（感知）
  Q2: 知道后怎么终止它？（切断）
  Q3: 怎么保证其他 3 个不受影响？（隔离）
```

### Step 2: 盘点武器——只用你学过的类

```
从 Java 标准库里找：
  Future.get(timeout)       → 能感知，不能终止
  Future.cancel(true)       → 能发中断，但 socket.read() 不响应
  invokeAll(tasks, timeout) → 批量 + 自动 cancel，统一计时
  shutdownNow()             → 全局终止，但不保证立即停止
  volatile                  → 多线程可见性
  daemon thread             → JVM 退出自动清理

从 OS/网络知识找：
  Socket readTimeout        → socket.read() 的硬超时（interrupt 无效时的兜底）
  Process.destroyForcibly() → 杀子进程（OS 资源不受 JVM 管）
```

### Step 3: 方案映射——武器对上子问题

| 子问题 | 武器 | 在哪层生效 | 为什么必须是这一层 |
|--------|------|-----------|-------------------|
| HTTP hanging 怎么感知？ | OkHttp readTimeout 300s | 网络层 | `socket.read()` 在内核态阻塞，Java `interrupt()` 管不了 |
| 批量工具怎么超时？ | invokeAll(tasks, 90s) | JUC 层 | 统一计时起点，超时自动 cancel，已完成的正常返回 |
| 命令子进程超时？ | Process.waitFor(60s) + destroyForcibly() | OS 层 | 子进程是独立 OS 资源，不是 Java 线程 |
| LLM 死循环？ | 滑动窗口 + 停滞计数 3 次 | 业务层 | "正常执行但行为是死循环"，所有超时都不会触发 |
| 上面全失效？ | Future.get(150ms) 轮询 + 用户 ESC | 交互层 | 承认自动保护有边界 |

### Step 4: 分析取舍——为什么不用替代方案

每个技术选型问一句"为什么不用 XXX 替代"：

| 采用了 | 被放弃的 | 为什么放弃 |
|--------|---------|-----------|
| invokeAll | CompletableFuture.allOf() | 并发场景简单，invokeAll 刚好够；CF 的链式编排在这里是过度设计 |
| daemon 线程 | user 线程 + 手动 shutdown | 代价是 finally 块可能不执行，换来 JVM 退出保证（僵尸进程更不可接受） |
| executeTaskBatch 不设超时 | 给 future.get() 加 timeout | 信任内层 5 重保护；如果 LLM/工具都超时了，外层还有 5 轮硬上限 |
| 装饰器 extends | 代理 wraps | 装饰器零侵入但耦合继承链；这里继承链短，简洁优先 |

---

## 串成知识网：从 OS 到设计模式

理解完一个模块后，把涉及的知识点按**依赖关系**串起来——从最底层到最上层：

```
你的知识树（以"并发超时控制"为例）：

OS 层（大二《操作系统》）
  └─ Socket 阻塞在 recv() 系统调用，在内核态，Java 中断标志检查不到
  └─ 子进程是独立的 OS 资源，有独立的 PID，不受 JVM 管控
      │
      ▼ （问题：OS 层的阻塞，Java 管不了 → 那谁来管？）
      
网络层（大三《计算机网络》）
  └─ TCP 连接建立（三次握手）和 数据传输 是两个阶段
  └─ Socket.setSoTimeout() 是 OS 提供的选项，超时后 OS 主动关连接
  └─ OkHttp 的 connectTimeout / readTimeout / callTimeout 分别保护三个阶段
      │
      ▼ （问题：OkHttp 超时后线程会抛异常解锁 → 异常怎么被上层捕获？）

JUC 线程基础（大三《Java 并发编程》）
  └─ Thread.interrupt() 是协作式的：设置标志位，不是强制杀线程
  └─ Object.wait() / Thread.sleep() / BlockingQueue.put() 响应中断
  └─ socket.read() / synchronized 不响应中断 ← 这是网络层必须独立设超时的原因
      │
      ▼ （问题：线程可以管理了 → 但任务怎么提交？怎么等结果？）

JUC 任务抽象
  └─ Callable：一个"可以返回结果、可以抛异常"的任务
  └─ Future：任务的"收据"——提交后拿到 Future，之后凭它取结果
  └─ ExecutorService：线程池，复用线程 + 排队任务
  └─ 关键认知：提交任务 ≠ 正在执行 ≠ 已经完成
      │
      ▼ （问题：任务提交了 → 怎么设超时？超时后怎么取消？）

JUC 超时与中断
  └─ Future.get(timeout)：感知层。超时抛 TimeoutException，但工作线程还在跑！
  └─ invokeAll(tasks, timeout)：批量提交 + 统一计时 + 超时自动 cancel
  └─ future.cancel(true)：切断层。发 Thread.interrupt()，工作线程需配合
  └─ shutdownNow()：全局终止。最后手段，不保证优雅
      │
      ▼ （问题：Java 线程都能控制了 → 但是 subprocess 是 OS 的，怎么办？）

进程管理
  └─ Process.waitFor(timeout)：等子进程，等价于 OS 的 waitpid()
  └─ Process.destroyForcibly()：杀子进程，Unix=SIGKILL, Win=TerminateProcess
  └─ 子进程的 stdin/stdout/stderr 是管道，不读会死锁
      │
      ▼ （问题：所有武器齐了 → 怎么组合成一个优雅的设计？）

设计模式
  └─ 分层防御：不是"某一层更强"，而是"每层覆盖上一层的死角"
  └─ 装饰器：HitlToolRegistry extends ToolRegistry，不改原代码加审批
  └─ volatile：保证多线程写值对读线程立即可见
  └─ 不可变对象(record)：天然线程安全，不需要任何同步
```

**这样串起来，你就不只是"用过 Future"——你知道了 Future 为什么存在、它在整个知识体系中的位置、它上面和下面各是什么。**

---

## 实习面试怎么用这套方法

### 被问到"介绍一下你做的项目"

❌ 菜鸡回答：我们项目用了 Spring Boot + MyBatis + Redis...

✅ 用这套方法：我们项目是一个 AI 编程助手，核心要解决三个问题：怎么让 LLM 调用工具（ReAct 循环）、怎么把复杂任务拆开执行（Plan-Execute + DAG）、怎么在并发执行时保证一个任务卡死不影响其他任务（5 层递进防御）。以第三点为例，我们最底层用 OkHttp 的 readTimeout 从 Socket 层面兜底……

**区别在哪**：你说的是设计决策和为什么，不是技术名词堆砌。

### 被问到"你遇到了什么难点"

❌ 菜鸡回答：并发不太好调...

✅ 用这套方法：最大的收获是理解了"分层防御"不是设计模式书上背的，而是从问题推导出来的必然。比如我们发现 Thread.interrupt() 对 socket.read() 无效，追到 OS 内核才发现 recv() 不检查 Java 标志位。这才理解了为什么 OkHttp 必须自己设 readTimeout。这让我把 OS、计网、JUC 三门课的知识串起来了。

**区别在哪**：你说的是知识之间的联系，证明你真的理解了，不是背的。

---

## 实战 Checklist

面对任何一个陌生项目，按这个顺序走：

- [ ] **心态**：告诉自己"我不需要全懂，先建索引"
- [ ] **第一刀**：找到 main()，画启动的 10 步流程
- [ ] **数据流**：选一个最简单的用户输入，画从头到尾的完整路径
- [ ] **拆包**：每个 package 用一句话说清它解决了什么问题
- [ ] **已知推导**：遇到不认识的 API/模式，先用自己学过的类猜怎么实现，再看代码验证
- [ ] **解剖四步**：选一个面试可能被问的模块，走完"问题→武器→映射→取舍"
- [ ] **串知识网**：从 OS/计网 → JUC → 设计模式，画出依赖链
- [ ] **准备好讲法**：一个项目介绍（说设计决策）+ 一个难点故事（说知识之间的联系）
