---
title: Java Concurrency之Concurency Basics
date: 2018-01-28 15:55:07
tags: [java, concurrency]
---

*Concurrency is the ability to do more than one thing at the same time.*

*Concurrency*包含两种情况：

1. 同时运行多个应用。
2. 单个应用的多个部分同时执行。

<!-- more -->

### Concurrency: Under the Hood

我们都知道计算机可以同时运行multiple tasks，但是这是怎么做到的呢？如今的计算机有multiple processors，如果一个计算机只有一个processor的话可以达到concurrency的目的吗？实际上，计算机可以执行的tasks的数量跟processors的数量之间没有直接的关系。

那么，multiple tasks是如何同时运行在单个CPU上的呢？Multiple tasks并不是真正地在相同的时刻同时执行，它与*parallel*的概念不同。

当我们说，“multiple tasks are executing at the same time”的时候，这句话的真实含义是“multiple tasks are making progress during the same period of time”。这些tasks的执行是交错进行的。操作系统在tasks之间频繁切换，这样在我们看来这些tasks好像是同时在执行一样。

因此，*concurrency*与*parallelism*不同。事实上，*parallelism*是不可能在单个processor上实现的。

### Unit of Concurrency

*Concurrency*是一个较为宽泛的概念，它分为不同的层级：

* **Multiprocessing** 多个processors/CPUs同时执行。这里并发的单位是CPU。
* **Multitasking** 多个tasks/processes在单个CPU上同时执行。操作系统通过在这些tasks之间频繁切换来执行他们。这里并发的单位是process。
* **Multithreading** 相同program的不同部分同时执行。这里，我们将同样的program分为不同的部分/线程，并且将他们并发的执行。

### Processes and Threads

#### Process

*A process is a program in execution. It has its own address space, a call stack, and link to any resources such as open files.*

#### Thread

*A thread is a path of execution within a process.* 每个process至少有一个thread, 称为main thread。Main thread可以创建该Process的其他threads。

一个process的所有threads共享这个process的资源，包括memory和open files。但是，每个thread有它自己的call stack。

因为threads共享process的地址空间，因此创建新的threads和threads之间通信会很高效。

### Common Problems associated with Concurrency

Concurrency可以提高CPU的利用率(<font color="red">TODO Why?</font>)，但是带来了如下问题：

* **Thread interference errors (Race Conditions)**: 这种情况在多个线程同时读写共享变量时发生，并且读写操作之间发生了重叠。

  这会造成结果的不可预见性（取决于读写操作的顺序）。

  这种情况可以通过加exclusive lock来解决。这会带来新的问题**deadlock**和**starvation**。

* **Memory consistency errors**: 它指的是不同的线程对于相同的数据产生了inconsistent views。发生的原因是有的线程更新的共享数据，但是没有传播给其他线程，导致这些线程还在用旧数据。

### Reference

* [Java Concurrency / Multithreading Basics](https://www.callicoder.com/java-concurrency-multithreading-basics/)