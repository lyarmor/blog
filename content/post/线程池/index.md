---
title: 线程池
description: 
slug: 线程池
date: '2025-09-28T22:14:46+08:00'
image: cover.jpg
categories:
    - 编程
tags:
    - 线程池
    - java
---

# 核心参数

在Java中，线程池最核心的实现是 `java.util.concurrent.ThreadPoolExecutor` 类。理解了它的构造函数，基本就理解了其工作原理。

它的构造函数有7个重要的参数，这也是面试时经常被问到的：

1. `int corePoolSize`：**核心线程数**。就像银行的正式柜员，即使没有客户，他们也一直待命，不会被解雇。
2. `int maximumPoolSize`：**最大线程数**。银行能容纳的最多柜员数（正式+临时）。当任务队列满了之后，如果当前线程数小于最大线程数，线程池会创建新的线程来处理任务。
3. `long keepAliveTime`：**线程空闲时间**。临时柜员（非核心线程）在没有任务时，还能存活多久。超过这个时间，就会被解雇。
4. `TimeUnit unit`：`keepAliveTime` 的**时间单位**（如秒、毫秒等）。
5. `BlockingQueue<Runnable> workQueue`：**任务队列**。等候区。当所有核心线程都在忙时，新来的任务会先被存放到这个队列中等待。
6. `ThreadFactory threadFactory`：**线程工厂**。用来创建新线程的，可以自定义线程的名称、是否为守护线程等。
7. `RejectedExecutionHandler handler`：**拒绝策略**。当线程数达到最大，并且任务队列也满了之后，用来处理新任务的策略。

**线程池的工作流程可以归纳为以下几步：**

1. 当一个新任务提交给线程池时，线程池首先判断 **核心线程数（corePoolSize）** 是否已满。如果未满，则创建一个新的核心线程来执行任务。
2. 如果核心线程数已满，线程池会判断 **任务队列（workQueue）** 是否已满。如果未满，则将新任务放入任务队列中等待。
3. 如果任务队列也已满，线程池会判断 **最大线程数（maximumPoolSize）** 是否已满。如果未满，则创建一个新的非核心线程来执行任务。
4. 如果最大线程数也已满，那么就会触发 **拒绝策略（RejectedExecutionHandler）**。

| 参数名 (Parameter)             | Java类型 (Type)            | 解释 (Explanation)                                                                                             | 一句话总结                        |
| ------------------------------ | -------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| **`corePoolSize`**             | `int`                      | **核心线程数**。线程池中长期保持的线程数量，即使它们是空闲的。这是线程池的“常备军”。                           | **线程池里有多少个正式工。**      |
| **`maximumPoolSize`**          | `int`                      | **最大线程数**。线程池能容纳的同时执行的线程总数的上限。这是线程池的“总兵力”（正式工 + 临时工）。              | **线程池最多能有多少个员工。**    |
| **`keepAliveTime`**            | `long`                     | **空闲线程存活时间**。当线程数大于 `corePoolSize` 时，多余的空闲线程（临时工）在被终止前等待新任务的最长时间。 | **临时工没活干时能待多久。**      |
| **`unit`**                     | `TimeUnit`                 | **时间单位**。为 `keepAliveTime` 参数指定单位，例如 `TimeUnit.SECONDS` (秒)、`TimeUnit.MILLISECONDS` (毫秒)。  | **计算 `keepAliveTime` 的单位。** |
| **`workQueue`**                | `BlockingQueue<Runnable>`  | **任务队列**（阻塞队列）。当核心线程都在忙时，用于存储等待执行的任务。这是线程池的“等候区”或“仓库”。           | **没处理完的任务放哪里。**        |
| **`threadFactory`**            | `ThreadFactory`            | **线程工厂**。用于创建新线程的工厂。我们可以通过它来自定义线程的名称、优先级等信息。                           | **怎么造出一个新的线程。**        |
| **`rejectedExecutionHandler`** | `RejectedExecutionHandler` | **拒绝策略**。当线程池和任务队列都满了，无法处理新任务时所采取的策略。                                         | **忙不过来时，新任务怎么办。**    |

# 核心方法:

## execute()

`ctl`变量通过原子整型将线程池的运行状态（runState）和工作线程数量（workerCount）打包在一个 int 里。高3位用于表示线程池状态，低 29位用于表示工作线程数量。所有对线程池状态和线程数量的变更都通过原子操作完成，保证并发安全和一致性。这样可以高效地同时管理线程池的生命周期和线程数量，避免竞争条件。

```java
public void execute(Runnable command) {
    // 1. 前置检查：任务不能为空
    if (command == null) {
        throw new NullPointerException();
    }

    // 2. 获取 ctl 的当前值，包含了【运行状态】和【工作线程数】
    int c = ctl.get();

    // =================  第一层决策：尝试创建核心线程  =================
    // workerCountOf(c) 是一个位运算，从 ctl 中解析出当前工作线程数
    if (workerCountOf(c) < corePoolSize) {
        // 条件成立：当前线程数 <核心线程数
        // 尝试创建一个新的工作线程（核心线程）来执行这个新任务
        // addWorker() 是一个非常关键的内部方法，负责原子性地增加线程数并启动线程
        // 如果 addWorker 成功，任务就直接交给新线程了，execute方法结束。
        if (addWorker(command, true)) { // true表示这是核心线程
            return;
        }
        // 如果 addWorker 失败（可能因为其他线程同时操作，或者线程池状态突然变了），
        // 重新获取一下ctl的值，准备进入下一层决策。
        c = ctl.get();
    }

    // =================  第二层决策：尝试将任务入队  =================
    // isRunning(c) 检查线程池是否处于RUNNING状态
    // workQueue.offer(command) 尝试把任务加入阻塞队列，不阻塞，立即返回true/false
    if (isRunning(c) && workQueue.offer(command)) {
        // 条件成立：线程池在运行，并且任务成功加入队列
        
        // 【关键的双重检查 Double-Check】
        // 重新检查一遍线程池状态，因为在任务入队的瞬间，可能发生了shutdown()
        int recheck = ctl.get();
        
        // 如果线程池已经不是RUNNING状态了，就需要把刚才入队的任务移除
        if (!isRunning(recheck) && remove(command)) {
            // 如果移除成功，就执行拒绝策略
            reject(command);
        } 
        // 如果线程池还在运行，或者已经关闭但队列非空，
        // 检查一下当前工作线程数是否为0
        else if (workerCountOf(recheck) == 0) {
            // 如果为0，说明可能所有核心线程都因为异常死掉了。
            // 此时必须启动一个“非核心线程”来处理队列里的任务，否则任务会永远堆积。
            // addWorker(null, false) 的 null 表示这个新线程不是为了执行特定任务，
            // 而是为了去队列里拉活儿。false表示按maximumPoolSize的限制来创建。
            addWorker(null, false);
        }
        
        // 如果双重检查都通过，那么任务就安稳地躺在队列里了，execute方法结束。
        return;
    }

    // =================  第三层决策：尝试创建非核心线程  =================
    // 如果代码能走到这里，说明：
    // 1. 核心线程数已满
    // 2. 任务队列也满了
    // 现在做最后的尝试：创建“临时工”（非核心线程）
    // addWorker(command, false) 的 false 表示这次创建线程要遵守 maximumPoolSize 的限制。
    else if (!addWorker(command, false)) {
        // 如果 addWorker 返回 false，说明连非核心线程也创建失败了。
        // 唯一的原因就是：当前线程数已经达到了 maximumPoolSize。
        // 此时，线程池彻底饱和，只能执行拒绝策略。
        reject(command);
    }
}
```

## worker核心方法runWorker()

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread(); // 当前线程
    Runnable task = w.firstTask;        // 获取第一个任务
    w.firstTask = null;                 // 拿出来后就置空，防止重复执行
    w.unlock(); // 允许中断（将AQS state从-1变为0）

    boolean completedAbruptly = true; // 任务是否意外中断
    try {
        // 这就是线程复用的“永动机”循环！
        while (task != null || (task = getTask()) != null) {
            w.lock(); // 开始执行任务，给自己上锁，防止被中断

            // ... 省略了部分中断处理的逻辑 ...
            
            try {
                beforeExecute(wt, task); // 任务执行前的钩子方法
                try {
                    task.run(); // 【真正执行我们提交的 Runnable 任务】
                } catch (RuntimeException x) {
                    // ... 异常处理 ...
                } finally {
                    afterExecute(task, null); // 任务执行后的钩子方法
                }
            } finally {
                task = null; // 任务执行完，清空
                w.completedTasks++; // 绩效+1
                w.unlock(); // 解锁，表示任务执行完毕
            }
        }
        completedAbruptly = false; // 正常退出循环
    } finally {
        // 只有当 getTask() 返回 null 时，才会跳出循环走到这里
        processWorkerExit(w, completedAbruptly); // 处理工人的“离职”
    }
}
```

**`runWorker` 逻辑拆解：**

1. **准备阶段**：拿到第一个任务 `firstTask`，并把自己解锁，表示可以正式开始工作了。
2. **核心循环 `while (task != null || (task = getTask()) != null)`**:
    - **判断条件是关键**：这个 `while` 循环会一直持续，直到 `getTask()` 方法返回 `null`。
    - **第一次循环**：`task` 是 `firstTask`，不为 `null`，直接执行。
    - **后续循环**：`task` 为 `null`，会执行 `(task = getTask())`。`getTask()` 会尝试从阻塞队列 `workQueue` 中**阻塞式地**获取一个新任务。
        - 如果队列有任务，`getTask()` 返回任务，`while` 成立，循环继续。
        - 如果队列为空，`getTask()` 会根据配置（是否允许核心线程超时）一直等待或者超时等待。
        - 只有当线程池关闭或线程超时，`getTask()` 才会返回 `null`，导致循环终止。
3. **任务执行**：在 `try-finally` 块中执行 `task.run()`，并保证了加锁、解锁和统计的原子性。
4. **`lock()` & `unlock()` 方法**

这两个方法是 `Worker` 作为锁的体现。

- `lock()`: 在即将执行任务前调用，将 AQS 状态设置为 1。线程池在检查并中断这个 `Worker` 之前，会先 `tryLock()`，如果锁不住，说明工人正在忙，就不会打断它。
- `unlock()`: 任务执行完毕后调用，将 AQS 状态恢复为 0，表示工人现在空闲了。

### **总结：一个 `Worker` 的生命周期**

1. **诞生**: 当 `execute()` 一个任务且需要新线程时，`new Worker(task)` 被调用，`Worker` 对象诞生，并创建了与之绑定的 `Thread`。
2. **上岗**: `worker.thread.start()`被调用，新线程启动，进入`worker.run()`，然后立即进入`runWorker(this)`方法。
3. **工作**: 在 `runWorker` 的 `while` 循环中：
    - 先执行完自己的 `firstTask`。
    - 然后不知疲倦地通过 `getTask()` 从公共的任务队列 `workQueue` 中取新任务来执行。
    - 只要能取到任务，这个循环就不会退出，这个线程就**一直活着**被复用。
4. **退休**: 当线程池关闭，或者该线程被判定为空闲超时，`getTask()` 会返回 `null`。`while` 循环最终退出，`runWorker` 方法执行到 `finally` 块，调用 `processWorkerExit()` 处理“后事”（从员工名册 `workers` 中移除自己），方法执行完毕。
5. **消亡**: `run()` 方法执行结束，线程生命周期终结，被 JVM 回收。

## submit()

```java
    /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
```

整个流程可以总结为：

1. **包装 (Wrapping)**: `submit` 方法用 `FutureTask` 将你的 `Callable` 任务包装成一个既是 `Runnable` 又是 `Future` 的新对象。
2. **执行 (Executing)**: 它利用 `execute` 方法将这个包装后的 `Runnable` 任务扔给线程池。
3. **返回 (Returning)**: 它立即将这个作为 `Future` 的包装对象返回。
4. **通信 (Communicating)**: 线程池在后台执行包装对象的 `run` 方法（这会调用真正的任务），可以通过 `Future` 接口在主线程中与这个正在执行或已完成的任务进行交互（如 `get()` 结果）。