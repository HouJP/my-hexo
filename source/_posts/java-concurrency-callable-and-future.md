---
title: Java Concurrency之Callable and Future
date: 2018-01-28 22:34:49
tags: [java, concurrency]

---

### Callable

`Runnable`虽然方便但是不能让tasks的执行拥有返回值。

Java还提供了`Callable`接口来让tasks返回结果。`Callable`和`Runnable`类似，除了可以返回结果以及会抛出一个checked exeption。

<!-- more -->

`Callable`接口拥有单个方法`call()`：

```Java
Callable<String> callable = new Callable<String>() {
    @Override
    public String call() throws Exception {
        // Perform some computation
        Thread.sleep(2000);
        return "Return some result";
    }
};
```

`Callable`不需要在`Thread.sleep()`外使用try/catch代码块，因为Callable可以抛出checked exception

> <font color="red">TODO</font>Java中checked exception指什么？

使用lambda表达式：

```Java
Callable<String> callable = () -> {
    // Perform some computation
    Thread.sleep(2000);
    return "Return some result";
};
```

### Executing Callable tasks using ExecutorService and obtaining the result using Future

Executor service`submit()`方法可以提交task给线程执行，但是并不知道什么时候结果可以被获取。因此，它返回来一种特殊的类型`Future`。这个类型可以用于在结果准备好之后获取结果。

`Future`的概念和其他语言如JavaScript中的`Promise`相似。

> `Future` represents the result of a computation that will be completed at a later point of time in future.

```Java
public class FutureAndCallableExample {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        ExecutorService executorService = Executors.newSingleThreadExecutor();

        Callable<String> callable = () -> {
            // Perform some computation
            System.out.println("Entered Callable");
            Thread.sleep(2000);
            return "Hello from Callable";
        };

        System.out.println("Submitting Callable");
        Future<String> future = executorService.submit(callable);

        // This line executes immediately
        System.out.println("Do something else while callable is getting executed");

        System.out.println("Retrieve the result of the future");
        // Future.get() blocks until the result is available
        String result = future.get();
        System.out.println(result);

        executorService.shutdown();
    }

}
```

```Shell
# Output
Submitting Callable
Do something else while callable is getting executed
Retrieve the result of the future
Entered Callable
Hello from Callable
```

`ExecutorService.submit()`会直接返回`Future`。一旦拿到它，就可以并行执行其他tasks，这时候之前提交的task还在执行。之后可以用`future.get()`来获取结果。

`get()`方法会阻塞，直到任务完成。`Future` API也提供了一个`isDone()`方法来判断task是否完成。

```Java
import java.util.concurrent.*;

public class FutureIsDoneExample {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        ExecutorService executorService = Executors.newSingleThreadExecutor();

        Future<String> future = executorService.submit(() -> {
            Thread.sleep(2000);
            return "Hello from Callable";
        });

        while(!future.isDone()) {
            System.out.println("Task is still not done...");
            Thread.sleep(200);
        }

        System.out.println("Task completed! Retrieving the result");
        String result = future.get();
        System.out.println(result);

        executorService.shutdown();
    }
}
```

```Shell
# Output
Task is still not done...
Task is still not done...
Task is still not done...
Task is still not done...
Task is still not done...
Task is still not done...
Task is still not done...
Task is still not done...
Task is still not done...
Task is still not done...
Task completed! Retrieving the result
Hello from Callable
```

### Cancelling a Future

`Future.cancel()`可以取消future。返回一个boolean表示是否取消成功。

`cancel()`方法接收`mayInterruptIfRunning`参数。`true`表示如果这个task正在执行就会被中断执行，`false`表示如果正在执行会等待执行完毕。

```Java
import java.util.concurrent.*;

public class FutureCancelExample {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        ExecutorService executorService = Executors.newSingleThreadExecutor();

        long startTime = System.nanoTime();
        Future<String> future = executorService.submit(() -> {
            Thread.sleep(2000);
            return "Hello from Callable";
        });

        while(!future.isDone()) {
            System.out.println("Task is still not done...");
            Thread.sleep(200);
            double elapsedTimeInSec = (System.nanoTime() - startTime)/1000000000.0;

            if(elapsedTimeInSec > 1) {
                future.cancel(true);
            }
        }

        System.out.println("Task completed! Retrieving the result");
        String result = future.get();
        System.out.println(result);

        executorService.shutdown();
    }
}
```

```Shell
# Output
Task is still not done...
Task is still not done...
Task is still not done...
Task is still not done...
Task is still not done...
Task completed! Retrieving the result
Exception in thread "main" java.util.concurrent.CancellationException
        at java.util.concurrent.FutureTask.report(FutureTask.java:121)
        at java.util.concurrent.FutureTask.get(FutureTask.java:192)
        at FutureCancelExample.main(FutureCancelExample.java:34)
```

Task被取消后，`future.get()`会抛出`CancellationException`异常。改进：

```Java
if(!future.isCancelled()) {
    System.out.println("Task completed! Retrieving the result");
    String result = future.get();
    System.out.println(result);
} else {
    System.out.println("Task was cancelled");
}
```

### Adding Timeouts

`future.get()`会阻塞线程并等待task完成，如果不设置超时选项，`future.get()`在任务无法完成的情况下会被永远阻塞。

```Java
future.get(1, TimeUnit.SECONDS);
```

如果无法再规定时间内完成，会抛出`TimeoutException`。

### invokeAll

提交多个tasks，并且等待他们全部完成。

可以提交Callables的集合给`invokeAll()`方法，它会返回Futures的列表。任何一个`future.get()`的调用都会被阻塞，除非全部完成。

```Java
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.*;

public class InvokeAllExample {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        ExecutorService executorService = Executors.newFixedThreadPool(5);

        Callable<String> task1 = () -> {
            Thread.sleep(2000);
            return "Result of Task1";
        };

        Callable<String> task2 = () -> {
            Thread.sleep(1000);
            return "Result of Task2";
        };

        Callable<String> task3 = () -> {
            Thread.sleep(5000);
            return "Result of Task3";
        };

        List<Callable<String>> taskList = Arrays.asList(task1, task2, task3);

        List<Future<String>> futures = executorService.invokeAll(taskList);

        for(Future<String> future: futures) {
            // The result is printed only after all the futures are complete. (i.e. after 5 seconds)
            System.out.println(future.get());
        }

        executorService.shutdown();
    }
}
```

```Shell
# Output
Result of Task1
Result of Task2
Result of Task3
```

### invokeAny

批量提交，等待任一完成。

`invokeAny()`方法会接受`Callables`集合，返回执行最快的Callable的结果。它不会返回Future。

```Java
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.*;

public class InvokeAnyExample {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        ExecutorService executorService = Executors.newFixedThreadPool(5);

        Callable<String> task1 = () -> {
            Thread.sleep(2000);
            return "Result of Task1";
        };

        Callable<String> task2 = () -> {
            Thread.sleep(1000);
            return "Result of Task2";
        };

        Callable<String> task3 = () -> {
            Thread.sleep(5000);
            return "Result of Task3";
        };

        // Returns the result of the fastest callable. (task2 in this case)
        String result = executorService.invokeAny(Arrays.asList(task1, task2, task3));

        System.out.println(result);

        executorService.shutdown();
    }
}
```

```Shell
# Output
Result of Task2
```

### Reference

[Java Callable and Future Tutorial](https://www.callicoder.com/java-callable-and-future-tutorial/)