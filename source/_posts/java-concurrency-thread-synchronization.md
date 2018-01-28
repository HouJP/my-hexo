---
title: Java Concurrency之Thread Synchronization
date: 2018-01-28 23:15:16
tags: [java, concurrency]
---

### Concurrency Issues

当多个线程并发读写共享数据的时候，会造成如下问题：

1. Thread interference errors
2. Memory consistency errors

<!-- more -->

### Thread Interference Errors (Race Conditions)

举例：

```Java
class Counter {
    int count = 0;

    public void increment() {
        count = count + 1;
    }

    public int getCount() {
        return count;
    }
}

```

```Java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class RaceConditionExample {

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(10);

        Counter counter = new Counter();

        for(int i = 0; i < 1000; i++) {
            executorService.submit(() -> counter.increment());
        }

        executorService.shutdown();
        executorService.awaitTermination(60, TimeUnit.SECONDS);
    
        System.out.println("Final count is : " + counter.getCount());
    }
}
```

结果不是1000，而是 992或996或993或...

因为当我们执行increment()方法时，发生了如下操作：

1. 检索当前count的当前值
2. 检索到的值加1
3. 将增加后的值存储到count

不同线程的这些步骤发生了交叉造成了不可预期的结果。

>When multiple threads try to read and write a shared variable concurrently, and these read and write operations overlap in execution, then the final outcome depends on the order in which the reads and writes take place, which is unpredictable. This phenomenon is called [Race condition](https://en.wikipedia.org/wiki/Race_condition).
>
>The section of the code where a shared variable is accessed is called [Critical Section](https://en.wikipedia.org/wiki/Critical_section).

这个错误可以通过对shared variables的synchronizing access来避免。

### Memory Consistency Errors

> Memory inconsistency errors occur when different threads have inconsistent views of the same data. This happens when one thread updates some shared data, but this update is not propagated to other threads, and they end up using the old data.

为什么会发生这种现象：

1. Compiler可能对你的程序进行了优化来提升性能。
2. Processors可能尝试进行了优化，比如一个processor可能从temporary register读取变量（含有变量最近一次读取的值），而不是main memory（含有变量最新的值）。

例如：

```java
public class MemoryConsistencyErrorExample {
    private static boolean sayHello = false;

    public static void main(String[] args) throws InterruptedException {

        Thread thread = new Thread(() -> {
           while(!sayHello) {
           }

           System.out.println("Hello World!");

           while(sayHello) {
           }

           System.out.println("Good Bye!");
        });

        thread.start();

        Thread.sleep(1000);
        System.out.println("Say Hello..");
        sayHello = true;

        Thread.sleep(1000);
        System.out.println("Say Bye..");
        sayHello = false;
    }
}
```

期望的输出结果：

```Shell
# Ideal Output
Say Hello..
Hello World!
Say Bye..
Good Bye!
```

实际输出结果：

```Shell
# Actual Output
Say Hello..
Say Bye..
```

程序甚至没有终止。为什么呢？

第一个线程没有意识到主线程改变了`sayHello`变量的值。

我们可以用`volatile`关键字来避免这个问题。

### Synchronization

Thread interference和memory consistency errors可以在满足以下两个条件的情况下被避免：

1. 一次只有一个线程可以读写共享变量。
2. 如果有线程修改了共享变量，它自动与其他随后读写该共享变量的线程建立了happens-before的关系。这保证了完成的修改操作可以被其他线程看到。

Java有`synchronized`关键字来synchronize access共享变量来避免这些问题。

#### Synchronized Methods

```Java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

class SynchronizedCounter {
    private int count = 0;

    // Synchronized Method 
    public synchronized void increment() {
        count = count + 1;
    }

    public int getCount() {
        return count;
    }
}

public class SynchronizedMethodExample {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(10);

        SynchronizedCounter synchronizedCounter = new SynchronizedCounter();

        for(int i = 0; i < 1000; i++) {
            executorService.submit(() -> synchronizedCounter.increment());
        }

        executorService.shutdown();
        executorService.awaitTermination(60, TimeUnit.SECONDS);

        System.out.println("Final count is : " + synchronizedCounter.getCount());
    }
}
```

Synchronization的概念通常都会与一个对象绑定。在上述例子中，同一个对象的`increment()`方法不能被同时多次调用，但是不同对象的`increment()`方法可以被不同的线程同时调用。

在静态方法中，synchronization与Class对象绑定。

#### Synchronized Blocks

Java使用*intrinsic lock*或者*monitor lock*来管理线程的同步。每个对象都有一个与之关联的*intrinsic lock*。

当线程调用对象的synchronized method时，会自动获取该对象的intrinsic lock，在方法调用结束后释放它。如果方法抛出了异常，也会释放该锁。

对于静态方法，线程会请求与类有关的`Class`对象的intrinsic lock，它与类任意实例的intrinsic lock不同。

`synchronized`关键字也可以作为block statement，与`synchronized method`不同，`synchronized statements`必须指明提供intrinsic lock的对象：

```Java
public void increment() {
    // Synchronized Block - 

    // Acquire Lock
    synchronized (this) { 
        count = count + 1;
    }   
    // Release Lock
}
```

如果一个线程获取了一个对象的intrinsic lock，那么其他进程必须等待锁被释放。然而，拥有该锁的进程可以多次获取该锁而不会有任何问题。

> The idea of allowing a thread to acquire the same lock more than once is called *Reentrant Synchronization*.

### Volatile Keyword

Volatile keyword用来避免多线程程序中的memory consistency errors。它告诉compiler不要对变量做优化，compiler不会优化或者重排序该变量的指令。

而且，变量的值会从main memory来读取，而不是临时寄存器。

```Java
public class VolatileKeywordExample {
    private static volatile boolean sayHello = false;

    public static void main(String[] args) throws InterruptedException {

        Thread thread = new Thread(() -> {
           while(!sayHello) {
           }

           System.out.println("Hello World!");

           while(sayHello) {
           }

           System.out.println("Good Bye!");
        });

        thread.start();

        Thread.sleep(1000);
        System.out.println("Say Hello..");
        sayHello = true;

        Thread.sleep(1000);
        System.out.println("Say Bye..");
        sayHello = false;
    }
}
```

这样会得到想要的输出：

```Shell
# Output
Say Hello..
Hello World!
Say Bye..
Good Bye!
```

### Reference

* [Java Concurrency issues and Synchronization](https://www.callicoder.com/java-concurrency-issues-and-thread-synchronization/)