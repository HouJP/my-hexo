---
title: Java Concurrency之Executor Service & Thread Pool
date: 2018-01-28 19:01:57
tags: [java, concurrency]

---

### Executor Framework

大型的多线程应用可能有成百上千个应用，因此需要将线程的创建和应用其他部分的管理进行区分。

**Executors, 是一个帮助你创建和管理线程的框架**：

1. 线程创建：提供了创建线程的多种方法，具体的说就是一个线程池，应用可以用它来并发的执行tasks。
2. 线程管理：它管理线程池中线程的生命周期。你不需要关心线程池中的线程是否active/busy/dead/…。
3. 任务创建和执行：Executors框架提供了方法来提交tasks到线程池运行，并且我们可以决定tasks什么时候被执行。

<!-- more -->

Java Concurrency API定义了如下三种executor interfaces:

* **Executor** 一个简单的接口，包含一个`execute()`方法用来启动`Runnable`对象指定的task。
* **ExecutorService** 是`Executor`的子接口，添加了管理tasks生命周期的方法。它提供了`submit()`方法，该方法的重载版本可以接受`Runnable`和`Callable`对象。Callable对象和Runnable类似，区别在于Callable对象指定的task可以返回一个值。
* **ScheduledExecutorService** `ExecutorService`的子接口。添加了方法类调度tasks的执行。

除了上述三种接口，API还提供了`Executors`类，其中包含了用来创建不同类型executor services的工厂方法。

```Java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecutorsExample {
    public static void main(String[] args) {
        System.out.println("Inside : " + Thread.currentThread().getName());

        System.out.println("Creating Executor Service...");
        ExecutorService executorService = Executors.newSingleThreadExecutor();

        System.out.println("Creating a Runnable...");
        Runnable runnable = () -> {
            System.out.println("Inside : " + Thread.currentThread().getName());
        };

        System.out.println("Submit the task specified by the runnable to the executor service.");
        executorService.submit(runnable);
    }
}
```

```Shell
# Output
Inside : main
Creating Executor Service...
Creating a Runnable...
Submit the task specified by the runnable to the executor service.
Shutting down the executor
Inside : pool-1-thread-1
```

如果task被提交时，thread目前正在执行另一个task，那么新的task就会进入等待队列直到线程空闲。

上述程序不会自动退出，因为executor service会一直监听是否有新的task被提交，除非我们显示关闭它。

ExecutorService提供了两种方法关闭executor:

* **shutdown()** 停止接收新的task，等待之前的tasks执行完毕，然后关闭executor。
* **shutdownNow()** 终止正在执行的task，立即关闭executor。

```java
System.out.println("Shutting down the executor");
executorService.shutdown();
```

创建线程池，并发执行tasks:

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ExecutorsExample {
    public static void main(String[] args) {
        System.out.println("Inside : " + Thread.currentThread().getName());

        System.out.println("Creating Executor Service with a thread pool of Size 2");
        ExecutorService executorService = Executors.newFixedThreadPool(2);

        Runnable task1 = () -> {
            System.out.println("Executing Task1 inside : " + Thread.currentThread().getName());
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException ex) {
                throw new IllegalStateException(ex);
            }
        };

        Runnable task2 = () -> {
            System.out.println("Executing Task2 inside : " + Thread.currentThread().getName());
            try {
                TimeUnit.SECONDS.sleep(4);
            } catch (InterruptedException ex) {
                throw new IllegalStateException(ex);
            }
        };

        Runnable task3 = () -> {
            System.out.println("Executing Task3 inside : " + Thread.currentThread().getName());
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException ex) {
                throw new IllegalStateException(ex);
            }
        };


        System.out.println("Submitting the tasks for execution...");
        executorService.submit(task1);
        executorService.submit(task2);
        executorService.submit(task3);

        executorService.shutdown();
    }
}
```

```java
# Output
Inside : main
Creating Executor Service with a thread pool of Size 2
Submitting the tasks for execution...
Executing Task1 inside : pool-1-thread-1
Shutting down the ExecutorService...
Executing Task2 inside : pool-1-thread-2
Executing Task3 inside : pool-1-thread-1
Completed Task3 inside : pool-1-thread-1
```

Fixed thread pool在多线程编程中很常见。在fixed thread pool中，executor service会保证池中有指定数量的线程。运行。如果一个线程挂了，会立即被新的线程替代。

如果提交的tasks数量超过的可用的线程数量，那么新的tasks会在队列中等待直到轮到它们执行。

### Thread Pool

大多数executor的实现采用的是*thread pools*来执行tasks。在一个线程池中有多个worker threads，这些与`Runnable`/`Callable` tasks不同并且被executor所管理。

创建线程时代价很大的操作，因此应该被避免。Worker threads的存在减少了创建线程的开销，因为executor service只会创建一次线程池，并通过复用线程来执行tasks。

Tasks会通过中间队列**Blocking Queue**被提交到线程池。如果tasks数量大于active threads数量，就被会塞入blocking queue来等待有线程被释放。如果blocking queue满了，新的tasks会被拒绝。

<div align=center>

![executor-service-thread-pool-blocking-queue-example](/img/java-concurrency-executor-service-thread-pool/executor-service-thread-pool-blocking-queue-example.jpg)

</div>

### Schedule Executors

ScheduleExecutorService可以周期性的执行tasks或指定delay的时间。

```java
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class ScheduledExecutorsExample {
    public static void main(String[] args) {
        ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(1);
        Runnable task = () -> {
          System.out.println("Executing Task At " + System.nanoTime());
        };

        System.out.println("Submitting task at " + System.nanoTime() + " to be executed after 5 seconds.");
        scheduledExecutorService.schedule(task, 5, TimeUnit.SECONDS);
        
        scheduledExecutorService.shutdown();
    }
}
```

周期执行：

```Java
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class ScheduledExecutorsPeriodicExample {
    public static void main(String[] args) {
        ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(1);

        Runnable task = () -> {
          System.out.println("Executing Task At " + System.nanoTime());
        };
        
        System.out.println("scheduling task to be executed every 2 seconds with an initial delay of 0 seconds");
        scheduledExecutorService.scheduleAtFixedRate(task, 0,2, TimeUnit.SECONDS);
    }
}
```



### Reference

* [Java ExecutorService and Thread Pools Tutorial](https://www.callicoder.com/java-executor-service-and-thread-pool-tutorial/)