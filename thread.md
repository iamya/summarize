# Java 线程与虚拟线程总结

`Thread` 是 Java 的基本并发抽象。无论是传统线程还是虚拟线程，对业务代码来说都表现为 `java.lang.Thread`；差别主要在**底层实现方式**、**调度方式**以及**可承载的并发规模**。

> 说明：
> - 下文中的“传统线程”更准确地说是 **platform thread（平台线程）**。
> - 结论主要基于 JDK 21 / JEP 444。
> - 重点放在：线程基本概念、虚拟线程如何实现、两者关系与使用边界。

---

## 1. Java 线程的基本介绍

### 1.1 什么是线程

线程是程序中的最小调度执行单元。Java 虚拟机允许一个进程中同时存在多个执行流，每个执行流通常对应一个 `Thread` 实例。

Java 线程的几个核心作用：

- **并发执行**：多个任务可以同时推进
- **顺序上下文**：一个线程内的方法调用、异常传播、局部变量都沿同一执行栈展开
- **阻塞与唤醒载体**：`sleep`、`wait`、`park`、I/O 阻塞等行为都以线程为承载对象
- **诊断与观测单位**：栈追踪、调试器、线程转储、JFR 等工具都围绕线程工作

### 1.2 Java 中的两类线程

JDK 21 里，`java.lang.Thread` 支持两种实现：

1. **平台线程（Platform Thread）**
2. **虚拟线程（Virtual Thread）**

它们都实现了 Java 的线程语义：

- 都可以执行 `Runnable`
- 都支持中断
- 都支持 `ThreadLocal`
- 都能参与锁、异常传播、调试、线程转储等机制

也就是说，**虚拟线程不是另一套并发模型，而是 `Thread` 的另一种实现**。

### 1.3 平台线程的本质

平台线程是传统意义上的 Java 线程实现方式。它通常与操作系统内核线程近似 1:1 映射：

- Java 创建一个平台线程
- JVM 向操作系统申请一个内核线程
- 这个 Java 线程在其整个生命周期内基本占着这个 OS 线程

因此平台线程有几个典型特点：

- 资源开销较大
- 数量受 OS 线程数和内存限制
- 适合通用任务，但不是“无限便宜”的资源
- 所以工程上常常配合线程池复用

### 1.4 为什么平台线程会成为瓶颈

在高并发服务中，请求往往并不是一直在计算，而是大量时间消耗在：

- 网络 I/O
- 数据库 I/O
- 文件 I/O
- 锁等待
- 下游服务响应等待

如果每个请求绑定一个平台线程，那么：

- 请求阻塞时，线程也被占住
- 线程占住时，对应 OS 线程也被占住
- OS 线程数量无法无限增长

这就是为什么传统 Java 高并发程序常被迫转向：

- 线程池
- 回调风格
- `CompletableFuture`
- Reactor / Reactive 等异步框架

这些方式能提高吞吐，但也会让代码失去“一个请求对应一条直线执行流”的可读性与可观测性。

---

## 2. 虚拟线程是什么

### 2.1 定义

虚拟线程也是 `java.lang.Thread`，但它**不再与某个固定 OS 线程绑定**。

可以把它理解为：

> 平台线程是 JVM 对 OS 线程的直接封装；
> 虚拟线程是 JVM 自己在用户态实现的一种轻量线程。

因此虚拟线程的关键价值不是“单个线程更快”，而是：

- **创建成本更低**
- **阻塞成本更低**
- **可支持非常大的并发数量**

### 2.2 它解决的核心问题

虚拟线程希望保留 Java 最自然的编程模型：

- 一段任务就是一段顺序代码
- 一个请求可以由一个线程从头跑到尾
- 可以正常写阻塞式 I/O
- 可以正常 `try/catch/finally`
- 可以通过线程栈看清一条请求链路

但同时避免平台线程太贵的问题。

所以虚拟线程本质上是在做一件事：

> 让“thread-per-request”风格重新具备大规模并发能力。

### 2.3 它不是什么

虚拟线程常见误区：

- **不是更快的 CPU 计算线程**：CPU 密集场景不会因为线程变成虚拟线程而自动加速
- **不是替代所有并发控制工具**：锁、信号量、限流等问题仍然存在
- **不是异步框架语法糖**：它不是 `Future`/回调链的包装，而是重新让同步写法可扩展

一句话概括：

> 虚拟线程提升的是**吞吐与并发规模**，不是单次执行延迟。

---

## 3. 虚拟线程的实现原理

### 3.1 总体模型：M:N 调度

平台线程接近 **1:1** 模型：一个 Java 线程对应一个 OS 线程。

虚拟线程采用 **M:N** 模型：

- M 个虚拟线程
- 由 JVM 调度到 N 个平台线程上运行
- 再由 OS 调度这 N 个平台线程到 CPU 上执行

因此，真正被操作系统看到并调度的，仍然是平台线程；
而虚拟线程的调度主要由 JVM 负责。

### 3.2 carrier thread（载体线程）

虚拟线程运行时，并不是直接跑在 CPU 上，而是先“挂载”到某个平台线程上。这个承载它执行的平台线程称为 **carrier thread**。

可把关系理解为：

- **虚拟线程**：逻辑上的任务执行流
- **载体线程**：临时借给它运行的真实平台线程

重要特征：

- 虚拟线程生命周期内不固定绑定某一个 carrier
- 同一个虚拟线程前后可以运行在不同的 carrier 上
- `Thread.currentThread()` 返回的始终是虚拟线程本身，而不是 carrier

### 3.3 mount / unmount 机制

JVM 调度虚拟线程时，核心动作是：

1. **mount**：把虚拟线程挂到某个平台线程上执行
2. 执行一段 Java 代码
3. 遇到合适时机时 **unmount**：把虚拟线程从 carrier 上卸下
4. carrier 被释放，可去承载别的虚拟线程

这套机制是虚拟线程能够“一个 OS 线程服务很多任务”的关键。

### 3.4 阻塞时为什么还能扩展

平台线程的问题在于：一旦阻塞，对应 OS 线程就跟着阻塞。

虚拟线程不同。对于 JDK 中大多数可感知的阻塞点（尤其是网络 I/O、`BlockingQueue.take()` 等），JVM 会：

- 挂起当前虚拟线程
- 将它从 carrier 上卸下
- 让 carrier 去执行别的虚拟线程
- 等待 I/O 就绪后，再把这个虚拟线程重新提交调度

所以从业务代码视角看，你写的是阻塞代码；
但从底层执行视角看，**阻塞的只是虚拟线程状态，不一定会长期占住 OS 线程**。

### 3.5 调度器实现

根据 JEP 444，JDK 的虚拟线程调度器本质上是：

- 一个 **ForkJoinPool** 风格的工作窃取调度器
- 以 **FIFO** 模式调度虚拟线程
- 默认并行度通常等于可用 CPU 数

可通过系统属性调整，例如：

- `jdk.virtualThreadScheduler.parallelism`
- `jdk.virtualThreadScheduler.maxPoolSize`

这里要注意：

- 调度器中的工作线程本身是平台线程
- 虚拟线程不是直接跑在某种“神奇 CPU 通道”上
- 它仍然最终依赖少量平台线程承接执行

### 3.6 栈如何保存

平台线程通常拥有较重的本地线程栈。

虚拟线程不同：

- 它的执行栈不是长期固定压在某个 OS 线程栈上
- JVM 会把虚拟线程的栈帧以 **stack chunk** 的形式保存在 Java 堆中
- 这些栈片段可以随着执行增长或收缩

这使得 JVM 可以同时容纳海量虚拟线程，而不需要为每个线程都长期保留一个昂贵的本地栈。

### 3.7 pinning（钉住）问题

虚拟线程并不是任何时候都能顺利 unmount。

JEP 444 指出，以下情况会导致虚拟线程在阻塞时被 **pin** 在 carrier 上：

1. 处于 `synchronized` 代码块或方法内
2. 正在执行 `native` 方法或 foreign function

如果这时又发生长时间阻塞（例如长 I/O），就会出现：

- 虚拟线程阻塞
- carrier 也被一起占住
- 对应 OS 线程不能释放给别的虚拟线程

这不会破坏正确性，但会损害扩展性。

因此更准确的说法是：

> 虚拟线程不是“绝不会占住 OS 线程”，而是“在大多数可卸载阻塞点上，不会长期占住 OS 线程”。

### 3.8 哪些阻塞不一定能卸载

除了 pinning 以外，JEP 444 也提到：

- 某些文件系统操作
- 某些底层限制场景
- `Object.wait()` 等部分操作

可能不会像普通网络 I/O 那样优雅地 unmount。

不过 JDK 会在一些场景下通过临时扩容调度器中的平台线程来缓解问题；但对于 pinning，本身并不会自动无限补偿。

---

## 4. Java 线程与虚拟线程的关系

### 4.1 不是替代关系，而是实现层级关系

Java 线程与虚拟线程的关系，可以概括为：

- **Java 线程** 是抽象接口层面的统一概念
- **平台线程 / 虚拟线程** 是这个概念的两种底层实现

也就是说：

| 层级 | 含义 |
|---|---|
| `Thread` | Java 的统一线程抽象 |
| Platform Thread | 基于 OS 线程的传统实现 |
| Virtual Thread | 基于 JVM 用户态调度的轻量实现 |

所以“Java 线程”和“虚拟线程”并不是并列的两个完全无关概念。
更准确地说：

> 虚拟线程是 Java 线程的一种实现，而不是 Java 线程之外的新模型。

### 4.2 它们共享大部分语义

平台线程和虚拟线程在语义上高度一致：

- 都是 `Thread`
- 都有线程身份
- 都能中断
- 都能有栈追踪
- 都能使用 `ThreadLocal`
- 都能配合锁、条件队列、阻塞队列等并发原语

这也是虚拟线程很重要的一点：

> 它尽量不改变开发者的心智模型，而是改变线程“贵不贵”的实现成本。

### 4.3 虚拟线程依赖平台线程存在

虚拟线程虽然不等于平台线程，但它并不是完全脱离平台线程运行的。

恰恰相反：

- 虚拟线程最终仍要挂载到平台线程上执行
- 平台线程仍是 JVM 与 OS 调度对接的桥梁
- OS 看不到虚拟线程，只看得到平台线程

因此两者关系不是“二选一”，而是：

> 虚拟线程建立在平台线程之上，由少量平台线程作为 carrier 来承载大量虚拟线程。

### 4.4 编程模型上更接近“线程回归”

过去因为平台线程昂贵，大家往往用：

- 线程池复用线程
- 任务排队
- 回调 / 异步链

虚拟线程出现后，Java 又能回到更直接的模型：

- 一个任务一个线程
- 大量线程并发阻塞也可以接受
- 同步写法重新成为高吞吐服务的可选主流方案

所以从工程实践上看，虚拟线程与平台线程的关系还可以理解为：

> 虚拟线程在保留“线程式编程”优点的同时，削弱了平台线程的资源约束。

---

## 5. 两者对比总结

### 5.1 核心差异表

| 维度 | 平台线程 | 虚拟线程 |
|---|---|---|
| 本质 | JVM 对 OS 线程的封装 | JVM 在用户态实现的轻量线程 |
| 与 OS 线程关系 | 近似 1:1 | M:N，由 JVM 调度到少量平台线程 |
| 创建成本 | 较高 | 很低 |
| 阻塞成本 | 高，常占住 OS 线程 | 低，多数情况下可卸载 |
| 适合场景 | 通用任务、长期后台线程、少量固定工作线程 | 高并发、I/O 密集、thread-per-request 场景 |
| 是否应池化 | 经常需要 | 通常不应池化 |
| CPU 密集任务收益 | 正常 | 通常无额外收益 |
| 调试体验 | 好 | 同样好，且比异步链更直观 |

### 5.2 对线程池观念的影响

平台线程昂贵，所以传统工程经验是：

- 尽量复用
- 限制线程数
- 用线程池承接任务

虚拟线程下，这个观念要调整：

- **不要为了“复用线程”而池化虚拟线程**
- 每个任务直接创建一个虚拟线程通常更合理
- 如果要限制并发，应该用 `Semaphore` 等并发控制手段，而不是拿线程池充当限流器

### 5.3 对 `ThreadLocal` 使用习惯的影响

在平台线程池时代，一些代码习惯把昂贵对象缓存在线程本地变量里，因为线程会被复用。

但虚拟线程通常是：

- 任务级创建
- 不复用
- 数量可能非常大

因此：

- 上下文类 `ThreadLocal` 仍然可以用
- 但把昂贵对象缓存到 `ThreadLocal` 中，往往不再划算，甚至会放大内存占用

### 5.4 对锁使用的影响

虚拟线程不是说不能配合锁，而是要注意：

- 普通短临界区使用 `synchronized` 没问题
- 如果在 `synchronized` 内做高频、长时间阻塞 I/O，可能发生 pinning
- 这类场景可考虑改为 `ReentrantLock`

所以重点不是“全面替换 `synchronized`”，而是：

> 关注高频且长阻塞的临界区，避免把 carrier 白白钉住。

---

## 6. 示例代码

### 6.1 创建一个平台线程

这是最传统的写法。它会创建一个平台线程，底层通常会对应一个 OS 线程。

```java
public class PlatformThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = Thread.ofPlatform().name("platform-worker").start(() -> {
            System.out.println("current = " + Thread.currentThread());
            System.out.println("isVirtual = " + Thread.currentThread().isVirtual());
        });

        thread.join();
    }
}
```

说明：

- `Thread.ofPlatform()` 明确创建平台线程
- `isVirtual()` 会返回 `false`
- 适合少量固定后台任务，或确实需要传统线程语义的场景

### 6.2 创建一个虚拟线程

虚拟线程的 API 形式和普通线程非常接近，但底层实现已经不同。

```java
public class VirtualThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = Thread.ofVirtual().name("virtual-worker").start(() -> {
            System.out.println("current = " + Thread.currentThread());
            System.out.println("isVirtual = " + Thread.currentThread().isVirtual());
        });

        thread.join();
    }
}
```

说明：

- `Thread.ofVirtual()` 创建的是虚拟线程
- 对业务代码而言，使用方式与普通线程几乎一致
- 差别主要在调度和资源成本，而不是 API 风格

### 6.3 一个任务一个虚拟线程

这是官方推荐的典型方式：每个任务直接交给一个新的虚拟线程，而不是把任务塞进固定大小的平台线程池。

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class VirtualThreadExecutorDemo {
    public static void main(String[] args) throws Exception {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            Future<String> user = executor.submit(() -> queryUser());
            Future<String> order = executor.submit(() -> queryOrder());

            System.out.println(user.get());
            System.out.println(order.get());
        }
    }

    static String queryUser() throws InterruptedException {
        Thread.sleep(1000);
        return "user-result";
    }

    static String queryOrder() throws InterruptedException {
        Thread.sleep(1000);
        return "order-result";
    }
}
```

这个例子想表达的是：

- `newVirtualThreadPerTaskExecutor()` 不是传统意义上的线程池复用模型
- 每次 `submit` 都可以创建一个新的虚拟线程
- 即使任务里有阻塞等待，也不会像平台线程那样昂贵

### 6.4 用虚拟线程保留同步阻塞写法

虚拟线程最有价值的点之一，就是让“同步阻塞代码”重新具备高并发可扩展性。

```java
import java.io.IOException;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class FanOutDemo {
    private static final HttpClient CLIENT = HttpClient.newHttpClient();

    public static void main(String[] args) throws Exception {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            Future<String> google = executor.submit(() -> fetch("https://www.google.com"));
            Future<String> github = executor.submit(() -> fetch("https://www.github.com"));

            System.out.println("google size = " + google.get().length());
            System.out.println("github size = " + github.get().length());
        }
    }

    static String fetch(String url) throws IOException, InterruptedException {
        HttpRequest request = HttpRequest.newBuilder(URI.create(url)).GET().build();
        return CLIENT.send(request, HttpResponse.BodyHandlers.ofString()).body();
    }
}
```

这里虽然 `CLIENT.send(...)` 是阻塞写法，但在虚拟线程模型下，这种代码依然适合高并发 I/O 场景。

### 6.5 不要用线程池大小限制虚拟线程并发

如果你的目的不是“复用线程”，而是“限制某类资源最多同时处理 N 个请求”，更合适的工具通常是 `Semaphore`。

```java
import java.util.concurrent.Semaphore;

public class SemaphoreDemo {
    private static final Semaphore SEMAPHORE = new Semaphore(10);

    public static void callRemoteService() throws InterruptedException {
        SEMAPHORE.acquire();
        try {
            // 模拟远程调用
            Thread.sleep(200);
        } finally {
            SEMAPHORE.release();
        }
    }
}
```

要点：

- 虚拟线程本身不贵，不需要靠线程池“省线程”
- 如果是下游容量有限，就直接限制资源访问并发度
- 这样比“固定线程池 + 任务排队”更符合虚拟线程思路

### 6.6 pinning 风险示例

下面这种写法在虚拟线程里就值得警惕：

```java
public class PinningDemo {
    private final Object lock = new Object();

    public void badCase() throws InterruptedException {
        synchronized (lock) {
            // 如果这里是高频且长时间阻塞操作，可能导致 pinning
            Thread.sleep(3000);
        }
    }
}
```

更稳妥的改写思路：

```java
import java.util.concurrent.locks.ReentrantLock;

public class BetterPinningDemo {
    private final ReentrantLock lock = new ReentrantLock();

    public void betterCase() throws InterruptedException {
        lock.lock();
        try {
            Thread.sleep(3000);
        } finally {
            lock.unlock();
        }
    }
}
```

这里不是说任何 `synchronized` 都要替换，而是说：

- 若临界区很短，问题通常不大
- 若临界区里经常发生长时间阻塞，就要小心 carrier 被钉住

---

## 7. 该如何理解与使用虚拟线程

### 7.1 适合场景

虚拟线程最适合：

- Web 服务每请求一线程
- RPC 调用聚合
- 数据库访问型业务
- 大量网络等待型任务
- 希望保留同步阻塞写法、又要高并发吞吐的服务

### 7.2 不适合场景

以下场景不要对虚拟线程抱有错误期待：

- 纯 CPU 密集计算
- 少量长期驻留工作线程场景
- 强依赖线程池复用本地缓存对象的旧设计
- 大量 native 调用且伴随阻塞的路径

### 7.3 实践原则

可以把虚拟线程的使用原则总结为：

1. **把线程视为任务，而不是稀缺工人**
2. **优先保持同步、阻塞、直线式代码**
3. **不要池化虚拟线程**
4. **用专门并发控制工具限制资源，而不是靠线程池大小限制**
5. **关注 pinning 与高频长阻塞临界区**

---

## 8. 最终总结

Java 线程的统一抽象始终是 `java.lang.Thread`。平台线程和虚拟线程只是这个抽象在不同时代的两种实现。

- **平台线程**：直接、通用，但昂贵，数量受 OS 资源限制
- **虚拟线程**：轻量、可海量创建，由 JVM 调度到少量平台线程上执行

虚拟线程的核心创新不在于改变 Java 并发语义，而在于：

> 用 JVM 级别的调度与栈管理，把“高并发阻塞式编程”重新变成可扩展方案。

因此它们的关系可以一句话概括为：

> **虚拟线程是 Java 线程的一种轻量实现；它依赖平台线程承载执行，但大幅降低了线程在高并发 I/O 场景下的资源成本。**

从工程角度看，虚拟线程让 Java 在很多服务端场景中，可以重新用最自然、最好读、最好调试的线程式写法，获得接近异步框架的吞吐能力。

---

## 参考

- JEP 444: Virtual Threads
- Oracle Java SE 21 文档：Virtual Threads
- Java SE 21 API：`java.lang.Thread`
