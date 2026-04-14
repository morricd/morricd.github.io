---
title: Java 创建线程的多种方式
author: morric
date: 2020-03-19 24:05:00 +0800
categories: [开发记录, Java]
tags: [java]
---

Java 中创建线程主要有四种方式：继承 `Thread` 类、实现 `Runnable` 接口、通过 `Callable` + `FutureTask`，以及使用线程池。每种方式各有适用场景，本文逐一介绍其用法与优缺点。

---

## 一、继承 Thread 类

最直接的方式，重写 `run()` 方法定义线程逻辑：

```java
/**
 * @author morric
 */
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println(getName());
        System.out.println("MyThread 继承 Thread 类创建线程");
    }
}

public static void main(String[] args) {
    System.out.println(Thread.currentThread().getName());

    MyThread myThread = new MyThread();
    myThread.start();
}
```

输出结果：

```
main
Thread-0
MyThread 继承 Thread 类创建线程
```

> **优点**
>
> 1. 写法简单，直接重写 `run()` 即可，无需额外的类或接口；
> 2. 可以直接调用 `Thread` 的实例方法，如 `getName()`、`setName()` 等，便于调试和识别线程。

> **缺点**
>
> 1. **单继承限制**：Java 中一个类只能继承一个父类，继承了 `Thread` 就无法再继承其他类，扩展性受限；
> 2. 线程类同时承载了"任务逻辑"和"线程控制"两个职责，不符合单一职责原则；
> 3. 多线程间共享资源较为复杂，通常需要借助静态变量或加锁机制。

---

## 二、实现 Runnable 接口

将任务逻辑与线程本身解耦，是更推荐的方式：

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        // Runnable 中没有 getName()，需通过 Thread.currentThread() 获取
        System.out.println(Thread.currentThread().getName());
        System.out.println("RunnableThread 实现 Runnable 接口创建线程");
    }
}

public static void main(String[] args) {
    System.out.println(Thread.currentThread().getName());

    Thread myThread = new Thread(new MyRunnable());
    myThread.start();
}
```

输出结果：

```
main
Thread-1
RunnableThread 实现 Runnable 接口创建线程
```

> **优点**
>
> 1. **避免单继承限制**：实现接口不影响继承其他类，设计更灵活；
> 2. 将执行逻辑与线程控制分离，更符合面向对象设计思想；
> 3. 同一个 `Runnable` 实例可以被多个线程共享，适合多线程并发处理同一份数据的场景。

> **缺点**
>
> 1. 无法直接调用 `Thread` 的实例方法，需通过 `Thread.currentThread()` 间接获取当前线程；
> 2. `run()` 方法没有返回值，无法直接获取任务执行结果。

与继承 `Thread` 相比，实现 `Runnable` 接口在绝大多数场景下是更好的选择，尤其是需要多个线程共享同一资源时。

---

## 三、通过 Callable 和 FutureTask 创建线程

自 Java 1.5 起，引入了 `Callable` 和 `Future`，支持在任务执行完毕后获取返回值。

```java
public class MyThread {

    public static void main(String[] args) throws Exception {
        CallableThread callableThread = new CallableThread();
        FutureTask<String> futureTask = new FutureTask<>(callableThread);
        new Thread(futureTask).start();

        System.out.println("主线程先做其他重要的事情");

        // 通过 isDone() 判断子线程是否执行完毕，避免无谓地阻塞
        while (!futureTask.isDone()) {
            // 可以在这里做其他事情
        }

        // get() 会阻塞直到子线程执行完毕并返回结果
        System.out.println(futureTask.get());
    }
}

class CallableThread implements Callable<String> {
    @Override
    public String call() throws Exception {
        System.out.println(Thread.currentThread().getName());
        System.out.println("通过 Callable 和 FutureTask 创建线程");
        return "Hello";
    }
}
```

输出结果：

```
主线程先做其他重要的事情
Thread-0
通过 Callable 和 FutureTask 创建线程
Hello
```

### Callable 与 FutureTask 说明

> `Callable` 位于 `java.util.concurrent` 包下，与 `Runnable` 类似，但它的 `call()` 方法**有返回值**，且可以抛出受检异常。泛型参数 `V` 表示返回值类型。
{: .prompt-tip }

> `FutureTask` 同时实现了 `Runnable` 和 `Future` 接口，可以直接提交给 `Thread` 执行，也可以提交到线程池。它封装了异步计算的结果，通过 `get()` 方法获取返回值。
{: .prompt-tip }

> 注意：`futureTask.get()` 会**阻塞主线程**，直到子线程执行完毕才继续往下走。可以通过 `isDone()` 提前判断任务是否完成，或使用带超时参数的 `get(long timeout, TimeUnit unit)` 避免无限等待。
{: .prompt-tip }

`FutureTask` 还有一个重要特性：即使其 `run()` 方法被多次调用，任务也只会执行一次。此外，可以通过 `cancel()` 方法取消尚未开始的任务。

**使用步骤总结：**

1. 创建实现 `Callable` 接口的类，重写 `call()` 方法；
2. 用 `FutureTask` 包装 `Callable` 对象；
3. 将 `FutureTask` 作为任务提交给 `Thread` 或线程池执行；
4. 通过 `futureTask.get()` 获取异步执行结果。

> **优点**
>
> 1. 支持返回值，适合需要获取异步任务结果的场景；
> 2. 支持异常传递，`call()` 中抛出的异常可以在 `get()` 时捕获；
> 3. 与线程池配合使用，任务管理更灵活。

> **缺点**
>
> 1. 代码相对繁琐，需要额外引入 `FutureTask` 进行包装；
> 2. `get()` 存在阻塞风险，使用时需要注意超时处理。

---

## 四、通过线程池创建线程

线程池通过复用已有线程来执行任务，避免了频繁创建和销毁线程带来的性能开销，是生产环境中最常用的方式。

```java
public class MyThread {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        System.out.println(Thread.currentThread().getName());
        System.out.println("通过线程池创建线程");

        ExecutorService executorService = new ThreadPoolExecutor(
            1,                              // corePoolSize：核心线程数
            1,                              // maximumPoolSize：最大线程数
            60L,                            // keepAliveTime：空闲线程存活时间
            TimeUnit.SECONDS,               // 时间单位
            new ArrayBlockingQueue<>(10)    // 工作队列，最多缓冲 10 个任务
        );

        executorService.execute(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName());
            }
        });

        executorService.shutdown(); // 任务提交完毕后关闭线程池
    }
}
```

输出结果：

```
main
通过线程池创建线程
pool-1-thread-1
```

> 线程池的核心思想是"池化复用"：预先创建一批线程，任务到来时从池中取出可用线程执行，执行完毕后线程归还到池中，而非销毁。当所有线程都在忙碌时，新任务会进入阻塞队列（`BlockingQueue`）等待调度。

`ThreadPoolExecutor` 是线程池的核心实现，关键参数说明：

| 参数              | 说明                                                           |
| ----------------- | -------------------------------------------------------------- |
| `corePoolSize`    | 核心线程数，线程池始终保持的最小线程数                         |
| `maximumPoolSize` | 最大线程数，队列满时可临时扩展至此数量                         |
| `keepAliveTime`   | 非核心线程的空闲存活时间，超时后回收                           |
| `workQueue`       | 任务等待队列，常用 `ArrayBlockingQueue`、`LinkedBlockingQueue` |

> **优点**
>
> 1. 减少线程创建和销毁的性能开销，响应速度更快；
> 2. 可以控制最大并发数，防止资源耗尽；
> 3. 提供任务队列、拒绝策略等机制，具备更完善的流量控制能力。

> **缺点**
>
> 1. 配置不当（如队列无界、线程数过大）容易引发 OOM 或资源耗尽；
> 2. 相比直接创建线程，使用成本略高，需要理解各参数含义。

---

## 总结对比

| 方式                      | 有返回值 | 可抛受检异常 | 推荐程度 | 适用场景                 |
| ------------------------- | :------: | :----------: | :------: | ------------------------ |
| 继承 `Thread`             |    ❌     |      ❌       |   一般   | 简单场景，快速验证       |
| 实现 `Runnable`           |    ❌     |      ❌       |   推荐   | 通用场景，资源共享       |
| `Callable` + `FutureTask` |    ✅     |      ✅       |   推荐   | 需要获取异步结果         |
| 线程池                    |    ✅     |      ✅       | 强烈推荐 | 生产环境，高并发任务管理 |

实际开发中，**优先使用线程池**，它在性能、可控性和可维护性上都远优于手动创建线程。需要获取任务返回值时，配合 `Callable` + `Future` 使用即可。