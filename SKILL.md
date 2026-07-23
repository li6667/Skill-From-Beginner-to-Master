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

---

## 面试八股对照表：项目中每个知识点对应的高频考题

> 面试官问的基础知识（"八股"），本质是在考你**这个东西到底解决了什么问题**。
> 下面把项目中涉及的技术点，全部映射到标准面试题 + 核心得分点。

### 一、操作系统（OS）

#### 1.1 用户态与内核态 → 为什么 interrupt() 对 socket read 无效？

**项目场景**：Plan-Execute 并发任务卡死时，靠 OkHttp readTimeout 兜底而非 Thread.interrupt()。

**高频面试题**：

> **Q: 用户态和内核态的区别？为什么要有这种区分？**

**得分点**：
- CPU 指令集分为特权指令和非特权指令，内核态可以执行所有指令（访问硬件、管理内存），用户态只能执行非特权指令
- 用户程序运行在用户态（Ring 3），OS 内核运行在内核态（Ring 0），切换需要**系统调用**（syscall），涉及 CPU 上下文切换，开销较大
- `socket.read()` → 调用 glibc 的 `read()` → 触发 `syscall` → 进入内核态的 `recv()` → 在内核的 Socket 缓冲区上阻塞等待数据
- Java 的 `Thread.interrupt()` 只设置 JVM 层面的标志位，**无法穿透到内核态去中断 `recv()`**
- 所以必须在进入系统调用前，用 `setsockopt(SO_RCVTIMEO)` 设好超时 → 超时后**内核主动关闭 Socket** → `recv()` 返回错误 → Java 层抛 `SocketException`

> **Q: 什么是系统调用？举个例子？**

**得分点**：
- 用户程序访问系统资源（文件、网络、进程）的唯一合法入口
- 常见系统调用：`open()`/`read()`/`write()`（文件IO）、`socket()`/`connect()`/`recv()`（网络IO）、`fork()`/`exec()`/`waitpid()`（进程）、`malloc()` 底层调 `brk()`/`mmap()`
- 举例：你写了 `new FileInputStream("a.txt").read()`，实际上经过了 N 层调用，最终触发 `syscall read`

#### 1.2 进程 vs 线程 → 为什么 Process.destroyForcibly() 必不可少？

**项目场景**：LLM 调 `execute_command` 起了一个 Shell 子进程，超时后需要杀。

**高频面试题**：

> **Q: 进程和线程的区别？**

| 维度 | 进程 | 线程 |
|------|------|------|
| 资源 | 独立的地址空间、文件描述符、信号处理 | 共享进程的地址空间和文件描述符 |
| 调度 | 进程是**资源分配**的基本单位 | 线程是**CPU 调度**的基本单位 |
| 通信 | IPC（管道、消息队列、共享内存、Socket） | 共享内存（需要注意同步） |
| 创建开销 | 大（复制页表、创建 PCB） | 小（只需分配栈 + TCB） |
| 崩溃影响 | 一个进程崩溃不影响其他 | 一个线程崩溃可能导致整个进程挂 |
| 在 Java 中 | JVM 实例 = 一个进程 | `new Thread()` = 一个线程 |
| 杀死方式 | `kill -9 PID`（SIGKILL） | `Thread.interrupt()`（协作式，不一定停） |

**关键得分点**：
- 子进程是 OS 级资源，有独立的 PID，不归 JVM 管
- Thread 的 interrupt 对子进程完全无效（它们甚至不在同一个地址空间）
- 必须用 `Process.destroyForcibly()` → Unix 下是 `kill -9`，Windows 下是 `TerminateProcess`
- `destroy()` vs `destroyForcibly()`：前者发 SIGTERM（可被进程捕获和忽略），后者发 SIGKILL（不可捕获，必杀）

> **Q: 上下文切换的开销具体是什么？**

- 保存/恢复寄存器、PC、栈指针
- TLB（快表）刷新 → 内存访问变慢
- CPU Cache 污染 → 命中率下降
- 线程切换：同进程内约 1-5μs；进程切换：还要切换页表，约 5-10μs

---

### 二、计算机网络

#### 2.1 TCP 三次握手 → connectTimeout 保护什么？

**项目场景**：OkHttp 的 connectTimeout=60s。

**高频面试题**：

> **Q: TCP 三次握手的过程？为什么是三次不是两次？**

```
客户端                          服务端
  │                               │
  │──── SYN, seq=x ──────────────→│  ① 客户端：我要连你
  │                               │
  │←── SYN+ACK, seq=y, ack=x+1 ──│  ② 服务端：我收到了，你准备好了吗
  │                               │
  │──── ACK, ack=y+1 ────────────→│  ③ 客户端：我也准备好了
  │                               │
  │     连接建立（ESTABLISHED）     │
```

**为什么不是两次？** 防止已失效的连接请求到达服务端。如果只有两次握手，失效的 SYN 到达后服务端直接 ESTABLISHED，但客户端已经不要这个连接了，服务端资源被浪费。

**为什么不是四次？** 服务端的 SYN 和 ACK 可以合并在一个报文中（捎带应答），没必要分开发。

> **Q: connectTimeout、readTimeout、callTimeout 的区别？**

```
|←──────────── callTimeout（整个请求的硬上限）───────────────→|
|← connectTimeout →|←────── readTimeout ──────→|
   (三次握手阶段)       (建立连接后等数据到达)
```

- `connectTimeout`：TCP 三次握手阶段，服务器不响应 SYN 或 SYN+ACK 丢包时触发
- `readTimeout`：连接建立后，服务器一直不发数据（hanging）时触发——这是**PaiCLI 防止外部 API 卡死的第一道防线**
- `callTimeout`：整个请求的绝对上限，超时直接关闭连接。即使数据在缓慢传输，到时间也断

**面试要强调**：这三个 timeout 是**阶段覆盖**关系，不是并列关系。connectTimeout 触发时 readTimeout 还没开始计时。

---

### 三、Java 并发（JUC）

#### 3.1 Thread 基础 → daemon 线程

**项目场景**：Plan-Execute 并行执行的工作线程设为 daemon。

**高频面试题**：

> **Q: 守护线程（daemon）和用户线程（user）的区别？**

- JVM 退出条件：**所有非守护线程执行完毕**。只要有活着的用户线程，JVM 就不会退出
- 守护线程在 JVM 退出时**直接被终止**，不执行 finally 块，不释放持有的锁
- `thread.setDaemon(true)` 必须在 `start()` 之前调用
- 典型守护线程：GC 线程、后台定时任务
- GC 线程为什么必须是守护线程？→ 如果 JVM 只剩下 GC 线程（用户线程都结束了），说明没有对象在创建，GC 应该随 JVM 一起退出

> **Q: Thread 的 6 种状态？**

```
NEW → RUNNABLE ←→ BLOCKED（等 synchronized 锁）
              ←→ WAITING（wait/join/park，无限期等）
              ←→ TIMED_WAITING（sleep/wait(timeout)/join(timeout)，有限期等）
  → TERMINATED
```

**面试要强调**：RUNNABLE 不等于"正在跑"，它包含了"就绪态"和"运行态"。Java 不区分这两个，实际调度由 OS 决定。

#### 3.2 线程中断 → 协作式的精髓

**项目场景**：`future.cancel(true)` 发中断信号 + `CancellationContext.isCancelled()` 业务配合。

**高频面试题**：

> **Q: Thread.interrupt() 做了什么？为什么说它是"协作式"的？**

```java
// interrupt() 的三句核心：
// 1. 设置中断标志位为 true
// 2. 如果线程在 sleep/wait/join 上阻塞 → 唤醒并抛 InterruptedException，同时清除标志位
// 3. 如果线程在 socket.read() 上阻塞 → 完全没反应

Thread t = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // 业务代码...
    }
});
t.start();
t.interrupt();  // 只设标志，不强制停止
```

**得分点**：
- `interrupt()` 是协作式的（cooperative），不是抢占式的（preemptive）
- 被中断方必须**主动检查**中断标志（`isInterrupted()` 或 `InterruptedException`）
- `stop()` 是抢占式的，已被废弃：因为它会立即释放所有锁，可能导致共享数据处于不一致状态
- 正确的中断写法：在循环条件中检查中断标志，或捕获 `InterruptedException` 后清理退出

> **Q: isInterrupted() 和 interrupted() 的区别？**

```java
Thread.currentThread().isInterrupted();  // 只读标志，不改变
Thread.interrupted();                     // 读标志 + 清除（static方法，作用于当前线程）
```

**面试加分项**：你见过 `InterruptedException` 被吞掉的 bug 吗？→ 在 `ThreadPoolExecutor` 中，如果 `Future.get()` 超时后你不调用 `future.cancel(true)`，工作线程继续运行但 `InterruptedException` 的状态被吞了，可能导致线程永不退出。

#### 3.3 线程池 → ExecutorService

**项目场景**：`Executors.newFixedThreadPool(min(N, 4))` + `invokeAll`。

**高频面试题**：

> **Q: ThreadPoolExecutor 的 7 个参数？**

```java
new ThreadPoolExecutor(
    corePoolSize,      // 核心线程数（常驻，即使空闲也不回收）
    maximumPoolSize,   // 最大线程数
    keepAliveTime,     // 非核心线程空闲存活时间
    unit,              // 时间单位
    workQueue,         // 任务队列（阻塞队列）
    threadFactory,     // 线程工厂（命名、daemon 设置）
    rejectedHandler    // 拒绝策略
);
```

> **Q: 线程池的工作流程？提交一个任务后发生了什么？**

```
1. 当前线程数 < corePoolSize → 创建新线程执行（即使有空闲核心线程也新建）
2. 当前线程数 = corePoolSize → 任务入 workQueue 排队
3. workQueue 满了 → 创建新线程（直到 maximumPoolSize）
4. 线程数 = maximumPoolSize 且 workQueue 满了 → 执行拒绝策略
```

**面试必答**：核心线程和最大线程的区别——核心线程是常驻的（即使空闲也不回收，除非 `allowCoreThreadTimeOut(true)`），最大线程是峰值扩容的上限。

> **Q: 四种拒绝策略？**

| 策略 | 行为 |
|------|------|
| `AbortPolicy`（默认） | 抛 `RejectedExecutionException` |
| `CallerRunsPolicy` | 由提交任务的线程自己执行（**最常用**——实现背压） |
| `DiscardPolicy` | 静默丢弃新任务 |
| `DiscardOldestPolicy` | 丢弃队列中最老的任务，重新提交新任务 |

> **Q: 为什么阿里的 Java 开发手册不推荐用 Executors？**

- `newFixedThreadPool` 用的 `LinkedBlockingQueue` 是**无界**的 → 任务堆积可能导致 OOM
- `newCachedThreadPool` 允许创建 `Integer.MAX_VALUE` 个线程 → 可能耗尽系统资源
- **正确做法**：显式 `new ThreadPoolExecutor(...)`，指定有界队列（如 `ArrayBlockingQueue(200)`）

**PaiCLI 项目中**：`Math.min(N, 4)` 限制了最大并发，且任务是有限批次提交的，所以用 `Executors.newFixedThreadPool` 是安全的。

#### 3.4 Future → 异步结果的凭证

**项目场景**：`future.get()` 阻塞等待任务结果。

**高频面试题**：

> **Q: Future.get(timeout) 超时后，工作线程还在跑吗？**

**答案：还在跑！** 这是一个高频陷阱题。

```java
Future<String> f = pool.submit(() -> {
    Thread.sleep(100_000);  // 任务需要 100 秒
    return "done";
});
try {
    f.get(5, TimeUnit.SECONDS);  // 5 秒超时
} catch (TimeoutException e) {
    // 此时工作线程还在 sleep！！！
    // 必须再调 f.cancel(true) 才能发中断
}
```

**得分点**：`get(timeout)` 只控制**调用方的等待时间**，不控制**任务的执行时间**。要真正终止任务，必须 `f.cancel(true)`。

> **Q: invokeAll(tasks, timeout) 比逐个 get 好在哪？**

- **统一计时起点**：从第一个任务提交开始算，不是每个任务单独算
- **自动 cancel**：超时后自动 `cancel(true)` 所有未完成的任务
- **已完成的正常返回**：超时时已完成的任务的结果不受影响
- **逐 get 的坑**：第 1 个任务耗了 8 秒（你设 timeout=10s），第 2 个只能等 2 秒了。invokeAll 都从 T0 开始算

> **Q: FutureTask 的底层原理？状态机是怎么流转的？**

```java
// FutureTask 内部状态
NEW → COMPLETING → NORMAL（执行完成）
                → EXCEPTIONAL（抛异常）
                → CANCELLED（被取消）
    → INTERRUPTING → INTERRUPTED

// get() 阻塞原理：不是 wait/notify，是 LockSupport.park()
// finishCompletion() 唤醒：LockSupport.unpark() + 遍历 Treiber Stack（waiters链表）
```

#### 3.5 volatile → 多线程可见性

**项目场景**：Task 的 status/result/error 用 volatile 修饰。

**高频面试题**：

> **Q: volatile 的作用？和 synchronized 的区别？**

| | volatile | synchronized |
|---|---|---|
| 可见性 | ✅ 写立即刷主内存，读强制从主内存读 | ✅ 锁释放时刷主内存 |
| 原子性 | ❌ 不保证（i++ 不是原子操作） | ✅ 锁内代码块原子 |
| 有序性 | ✅ 禁止指令重排序（内存屏障） | ✅ 但违反 happens-before 原则 |
| 性能 | 轻量（无锁） | 有锁竞争开销 |
| 适用场景 | 状态标志、DCL 单例 | 复合操作的原子性 |

> **Q: volatile 的底层实现？什么是内存屏障？**

- Java 代码 `volatile boolean flag = true;` → 字节码 `ACC_VOLATILE` → JIT 编译后插入 **Lock 前缀指令**
- x86 下：`lock addl $0x0, (%rsp)` → StoreLoad 屏障的廉价实现
- 四种内存屏障：LoadLoad、StoreStore、LoadStore、StoreLoad（最重）
- volatile 写：前面插入 StoreStore，后面插入 StoreLoad
- volatile 读：后面插入 LoadLoad + LoadStore

> **Q: DCL（双重检查锁定）单例为什么需要 volatile？**

```java
public class Singleton {
    private static volatile Singleton instance;  // 不加 volatile 可能拿到"半初始化"对象

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();  // 这不是原子操作！
                    // 1. 分配内存
                    // 2. 初始化对象
                    // 3. 引用指向内存
                    // 指令重排序可能 3 在 2 之前 → 其他线程拿到未初始化完毕的对象
                }
            }
        }
        return instance;
    }
}
```

#### 3.6 不可变对象 → record 的线程安全

**项目场景**：`AgentMessage`、`ApprovalRequest`、`PlanReviewDecision` 全用 Java record。

**高频面试题**：

> **Q: 为什么不可变对象天然线程安全？**

- 不可变对象的**状态在构造后就固定了**，不存在"写"操作
- 多线程并发读不需要任何同步机制（没有写就不会有竞态条件）
- Java 中的不可变类：`String`、`Integer`、`BigDecimal`、`LocalDate`

> **Q: 如何设计一个不可变类？**

1. 类声明为 `final`（防止子类覆盖方法改变行为）
2. 所有字段 `private final`
3. 不提供 setter
4. 如果字段是可变对象引用（如 `List`），getter 返回**防御性拷贝**
5. 构造器中做防御性拷贝（防止调用方修改传入的可变对象）

```java
// 经典的不可变类写法（Java 14 之前）
public final class Person {
    private final String name;
    private final List<String> hobbies;  // 可变字段！

    public Person(String name, List<String> hobbies) {
        this.name = name;
        this.hobbies = new ArrayList<>(hobbies);  // 防御性拷贝
    }

    public List<String> getHobbies() {
        return new ArrayList<>(hobbies);  // 返回拷贝，防止外部修改
    }
}

// Java 14+ record 一行搞定，编译器自动生成以上所有
public record Person(String name, List<String> hobbies) {}
```

---

### 四、设计模式

#### 4.1 装饰器模式 → HitlToolRegistry

**项目场景**：`HitlToolRegistry extends ToolRegistry`，在执行工具前加审批。

**高频面试题**：

> **Q: 装饰器模式和代理模式的区别？**

| | 装饰器（Decorator） | 代理（Proxy） |
|---|---|---|
| 目的 | **增强功能**，一层层加能力 | **控制访问**，屏蔽复杂性 |
| 关系 | 被装饰对象**从外部注入** | 被代理对象通常在代理**内部创建** |
| 层次 | 可以无限嵌套（俄罗斯套娃） | 通常一层 |
| 例子 | `BufferedReader(Reader)` 加的缓冲区 | RPC 代理：调用本地接口，实际发网络请求 |
| 用户感知 | 用户知道自己在装饰 | 用户无感知（透明） |

**Java IO 是典型的装饰器**：`new BufferedReader(new InputStreamReader(new FileInputStream("a.txt")))`

**PaiCLI 是典型的装饰器**：`HitlToolRegistry extends ToolRegistry`——用户传进来一个 ToolRegistry 引用，HitlToolRegistry 在外面包一层审批逻辑。HITL 关了就完全透传，零开销。

> **Q: 继承 vs 组合？装饰器用的是哪个？为什么？**

装饰器用**组合**（持有被装饰对象的引用），但 PaiCLI 用的是**继承**（`extends ToolRegistry`）。

为什么选继承？因为 `ToolRegistry` 是一个**具体类而非接口**，有几十个内置工具的注册逻辑在构造函数中。如果用组合（wraps + 委托），需要把 `ToolRegistry` 的所有 public 方法都手动转发一遍，工作量大且容易遗漏。继承可以直接复用所有非 private 方法，只覆写 `executeToolOutput()` 一个方法。

**权衡**：继承耦合度高（子类依赖父类实现细节），但在这种"只覆写一个方法"的简单场景下，收益大于代价。如果未来 `ToolRegistry` 的 `executeToolOutput` 被标记为 `final`，或者审批逻辑更复杂（需要多个工具注册表），就该切换为组合了。

#### 4.2 模板方法模式 → AbstractOpenAiCompatibleClient

**项目场景**：7 个 LLM Client 的实现基类。

**高频面试题**：

> **Q: 模板方法模式解决什么问题？**

**得分点**：
- 在基类中定义算法骨架（`chat()` 方法的核心流程），**延迟**某些步骤到子类实现
- 好莱坞原则："Don't call us, we'll call you"——子类不用关心整体流程，只需实现被回调的方法
- 子类覆写的方法通常叫"钩子方法"（hook method）

```java
// PaiCLI 的实际骨架
abstract class AbstractOpenAiCompatibleClient implements LlmClient {
    public ChatResponse chat(messages, tools, listener) {
        // 1️⃣ 构建请求体（子类可覆写）
        String body = buildRequestBody(messages, tools);

        // 2️⃣ 发送 HTTP POST
        Response response = httpClient.newCall(request).execute();

        // 3️⃣ 流式解析 SSE（子类可覆写 extractReasoningDelta）
        while(...) {
            String reasoning = extractReasoningDelta(delta);  // 钩子
            String content = delta.get("content");
            ...
        }

        // 4️⃣ 返回结果
        return new ChatResponse(content, toolCalls, tokens);
    }

    // 子类覆写点
    protected abstract String getApiUrl();           // GLM 一个 URL，DeepSeek 另一个
    protected abstract Map<String, String> headers(); // 不同厂商的认证头
    protected String extractReasoningDelta(JsonNode delta) { ... }  // 钩子，有默认实现
}
```

> **Q: 模板方法 vs 策略模式？**

- 模板方法用**继承**（基类定骨架），策略用**组合**（注入不同策略对象）
- 模板方法适合"流程固定、步骤可变"，策略适合"算法可整体替换"
- PaiCLI 场景：请求流程完全一致（构建→发送→解析→返回），只是 URL/headers/reasoning 字段名不同 → 模板方法

#### 4.3 工厂模式 → LlmClientFactory

**高频面试题**：

> **Q: 简单工厂、工厂方法、抽象工厂的区别？**

| | 简单工厂 | 工厂方法 | 抽象工厂 |
|---|---|---|---|
| 工厂数量 | 1 个工厂类 | 每种产品一个工厂 | 每种产品族一个工厂 |
| 扩展方式 | 改 switch 加 case | 新增工厂子类 | 新增工厂实现 |
| 违反开闭原则？ | ✅ 是（改代码） | ❌ 否（加代码） | ❌ 否（加代码） |
| 典型例子 | `LlmClientFactory.create("glm")` | JDBC `Connection.createStatement()` | Spring `BeanFactory` |

**PaiCLI 用的是简单工厂 + 注册表**：`LlmClientFactory.createFromConfig()` 遍历 "glm/deepseek/step/kimi/..." 找第一个配置了 API Key 的。

---

### 五、知识速查卡

> 面试前 10 分钟扫一眼，重点记星号项。

| 知识点 | 一句话核心 | 面试频率 |
|--------|-----------|---------|
| 用户态/内核态 | Java interrupt 管不到内核，必须在 Socket 层设超时 | ⭐⭐⭐ |
| 进程 vs 线程 | 进程独立地址空间，线程共享；子进程是 OS 资源，必须用 kill | ⭐⭐⭐⭐ |
| TCP 三次握手 | 防止失效连接，connectTimeout 保护这一阶段 | ⭐⭐⭐⭐⭐ |
| 三 timeout 区别 | 阶段覆盖：connect → read → call，不是并列 | ⭐⭐⭐ |
| daemon 线程 | JVM 不等它；finally 可能不执行 | ⭐⭐⭐ |
| interrupt 协作式 | 只是发信号，对方必须主动配合检查 | ⭐⭐⭐⭐ |
| `isInterrupted` vs `interrupted` | 前者只读，后者读+清除 | ⭐⭐⭐ |
| 线程池 7 参数 | core/max/keepAlive/unit/queue/factory/handler | ⭐⭐⭐⭐⭐ |
| `get(timeout)` ≠ 终止 | 超时只释放调用方，工作线程继续跑 | ⭐⭐⭐⭐ |
| `invokeAll` 优势 | 统一计时 + 自动 cancel + 已完成不受影响 | ⭐⭐⭐ |
| volatile 三特性 | 可见性 ✅ / 原子性 ❌ / 有序性 ✅ | ⭐⭐⭐⭐⭐ |
| DCL 为什么 volatile | 防止指令重排拿到半初始化对象 | ⭐⭐⭐⭐ |
| 不可变对象 | 没有写就没有竞态，天然线程安全 | ⭐⭐⭐⭐ |
| record vs class | record 自动生成构造器/getter/equals/hashCode/toString | ⭐⭐⭐ |
| 装饰器 vs 代理 | 装饰器增强功能 + 外部注入；代理控制访问 + 内部创建 | ⭐⭐⭐⭐ |
| 模板方法 vs 策略 | 继承定骨架 vs 组合换算法 | ⭐⭐⭐⭐ |
| 简单工厂 vs 工厂方法 | 改 switch（违反开闭） vs 加类（不违反开闭） | ⭐⭐⭐ |
