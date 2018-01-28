---
title: Java Concurrency之Thread and Runnable
date: 2018-01-28 17:39:42
tags: [java, concurrency]

---

### Creating and Strating a Thread

Java有两种方式创建线程。

#### 1. By extending Thread class

继承`Thread`类，重载`run()`方法。`run()`方法中是该线程的执行逻辑。当创建好之后，通过`start()`方法来启动它。

<!-- more -->

```Java
public class ThreadExample extends Thread {

    // run() method contains the code that is executed by the thread.
    @Override
    public void run() {
        System.out.println("Inside : " + Thread.currentThread().getName());
    }

    public static void main(String[] args) {
        System.out.println("Inside : " + Thread.currentThread().getName());

        System.out.println("Creating thread...");
        Thread thread = new ThreadExample();

        System.out.println("Starting thread...");
        thread.start();
    }
}
```

```shell
# Output
Inside : main
Creating thread...
Starting thread...
Inside : Thread-0
```

可以通过`Thread(String name)`构造方法来自定义线程名字。

#### 2. By providing a Runnable object 

`Runnable`接口是一个重要的模板，用来在线程中执行某个对象。它定义了单个方法`run()`，其中包含的是意图在线程中执行的代码。

任何一个类，如果它的实例想要单独的线程中执行，那么它需要实现`Runnable`接口。

**`Thread`类也实现了`Runnable`，不过它的`run()`方法是空的。**

如果想要创建一个新的进程，需要：

1. 该类实现`Runnable`接口。
2. 为该类创建一个实例。
3. 将该实例传给`Thread(Runnable target)`构造方法。

```Java
public class RunnableExample implements Runnable {

    public static void main(String[] args) {
        System.out.println("Inside : " + Thread.currentThread().getName());

        System.out.println("Creating Runnable...");
        Runnable runnable = new RunnableExample();

        System.out.println("Creating Thread...");
        Thread thread = new Thread(runnable);

        System.out.println("Starting Thread...");
        thread.start();
    }

    @Override
    public void run() {
        System.out.println("Inside : " + Thread.currentThread().getName());
    }
}
```

```Shell
# Output
Inside : main
Creating Runnable...
Creating Thread...
Starting Thread...
Inside : Thread-0
```

以上的这种写法需要创建一个实现了`Runnable`的类，再将其实例化，我们可以利用Java的*anonymous class*语法创建一个anonymous runnable。

> <font color="red">TODO</font> [anonymous class](https://docs.oracle.com/javase/tutorial/java/javaOO/anonymousclasses.html)



> *Anonymous classes enable you to make your code more concise. They enable you to declare and instantiate a class at the same time.* - From Java doc.

```Java
public class RunnableExampleAnonymousClass {

    public static void main(String[] args) {
        System.out.println("Inside : " + Thread.currentThread().getName());

        System.out.println("Creating Runnable...");
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println("Inside : " + Thread.currentThread().getName());
            }
        };

        System.out.println("Creating Thread...");
        Thread thread = new Thread(runnable);

        System.out.println("Starting Thread...");
        thread.start();
    }
}
```

再加上Java 8的lambda表达式：

```Java
public class RunnableExampleLambdaExpression {

    public static void main(String[] args) {
        System.out.println("Inside : " + Thread.currentThread().getName());

        System.out.println("Creating Runnable...");
        Runnable runnable = () -> {
            System.out.println("Inside : " + Thread.currentThread().getName());
        };

        System.out.println("Creating Thread...");
        Thread thread = new Thread(runnable);

        System.out.println("Starting Thread...");
        thread.start();

    }
```

### Runnable or Thread, Which one to use?

通过扩展`Thread`类的方法是存在局限性的。一旦通过`Thread`将类进行扩展，我们就不能再从其他类中扩展，因为Java不允许*multiple inheritance*。

从design practice角度来看，*inheritance*意味着你想要扩展父类的方法，但是当创建一个线程时，通常只是实现了`run()`方法而没有进行扩展。

所以一般来说，要用`Runnable`来创建线程。

### Pausing execution of a Thread using sleep()

`sleep()`方法可以暂停线程的执行。

```Java
public class ThreadSleepExample {

    public static void main(String[] args) {
        System.out.println("Inside : " + Thread.currentThread().getName());

        String[] messages = {"If I can stop one heart from breaking,",
                "I shall not live in vain.",
                "If I can ease one life the aching,",
                "Or cool one pain,",
                "Or help one fainting robin",
                "Unto his nest again,",
                "I shall not live in vain"};

        Runnable runnable = () -> {
            System.out.println("Inside : " + Thread.currentThread().getName());

            for(String message: messages) {
                System.out.println(message);
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    throw new IllegalStateException(e);
                }
            }
        };

        Thread thread = new Thread(runnable);

        thread.start();
    }
}
```

```Shell
# Output
Inside : main
Inside : Thread-0
If I can stop one heart from breaking,
I shall not live in vain.
If I can ease one life the aching,
Or cool one pain,
Or help one fainting robin
Unto his nest again,
I shall not live in vain
```

当有线程中断了当前线程的执行，*sleep()*方法会抛出`InterruptedException`。该异常是checked exception，必须被处理。

### Waiting for completion of another thread using join()

`join()`方法允许一个线程等待另一个线程的完成。

```Java
public class ThreadJoinExample {

    public static void main(String[] args) {
        // Create Thread 1
        Thread thread1 = new Thread(() -> {
            System.out.println("Entered Thread 1");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                throw new IllegalStateException(e);
            }
            System.out.println("Exiting Thread 1");
        });

        // Create Thread 2
        Thread thread2 = new Thread(() -> {
            System.out.println("Entered Thread 2");
            try {
                Thread.sleep(4000);
            } catch (InterruptedException e) {
                throw new IllegalStateException(e);
            }
            System.out.println("Exiting Thread 2");
        });

        System.out.println("Starting Thread 1");
        thread1.start();

        System.out.println("Waiting for Thread 1 to complete");
        try {
            thread1.join(1000);
        } catch (InterruptedException e) {
            throw new IllegalStateException(e);
        }

        System.out.println("Waited enough! Starting Thread 2 now");
        thread2.start();
    }
}
```

```Shell
Starting Thread 1
Waiting for Thread 1 to complete
Entered Thread 1
Waited enough! Starting Thread 2 now
Entered Thread 2
Exiting Thread 1
Exiting Thread 2
```

需要注意：

* `Thread.join()`等待的真实时间是min(线程终止所需时间，方法参数指明的时间)。
* `join()`可以无参数，会一直等待直到线程死亡。