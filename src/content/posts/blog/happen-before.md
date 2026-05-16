---
title: Happens-Before：并发编程中的“先后关系”
published: 2026-05-16
updated: 2026-05-16
description: 从可见性到有序性：理解 Java 内存模型中的 Happens-Before
tags:
  - Java
  - 八股
category: 后端开发
draft: false
author: yukino
---
在单线程程序中，我们一般很少关心“可见性”这个问题。  
例如：
```
int x = 0;
x = 1;
System.out.println(x);
```
这段代码的结果很显然就是1，这是因为所有操作都在一个线程中，前面写入的自然而然地会被后面读到。
但是如果是在多线程环境下，就没这么简单了。

一个线程写入了变量，另一个线程不一定马上能看到。甚至在没有正确同步的情况下，另一个线程可能一直看到旧值。这也是并发编程里很多诡异 bug 的来源。

这时就需要理解一个非常重要的概念：happens-before。
## Happens-Before 是什么？
happens-before 可以翻译成 先行发生关系。

如果操作 A happens-before 操作 B，那么可以认为：

> [!NOTE]
> 操作 A 的结果，对操作 B 是可见的。

注意，这里的“先行发生”并不只是现实时间上的先后顺序。

它更准确地说，是 Java 内存模型提供的一种保证：
如果 A happens-before B，那么 B 一定能够看到 A 对共享变量造成的影响。

换句话说，happens-before 解决的是两个问题：

可见性：一个线程的写入，另一个线程能不能看到？
有序性：编译器和 CPU 的重排序，会不会破坏我们期望的执行顺序？

### 一个没有 Happens-Before 的例子
```
class Demo {
    static boolean ready = false;
    static int number = 0;
 
    public static void main(String[] args) {
        Thread t = new Thread(() -> {
            while (!ready) {
                // wait
            }
            System.out.println(number);
        });
 
        t.start();
 
        number = 42;
        ready = true;
    }
}
```

我们的直觉上可能认为线程最终会打印 42。

但这段代码其实是有问题的。

因为主线程对 number 和 ready 的写入，与子线程对它们的读取之间，没有建立可靠的 happens-before 关系。

所以子线程可能看不到 ready = true，也可能看到 ready = true，却仍然读到旧的 number 值。

### 用 volatile 建立 Happens-Before

可以把 ready 用 volatile 修饰
```
class Demo {
    static volatile boolean ready = false;
    static int number = 0;
 
    public static void main(String[] args) {
        Thread t = new Thread(() -> {
            while (!ready) {
                // wait
            }
            System.out.println(number);
        });
 
        t.start();
 
        number = 42;
        ready = true;
    }
}
```

这时，ready = true 是一次 volatile 写。  
子线程中的 while (!ready) 是 volatile 读。

Java 内存模型规定：

> [!NOTE]
> 对一个 volatile 变量的写，happens-before 后续对这个变量的读。

因此，当子线程读到 ready == true 时，它不仅能看到 ready 的新值，也能看到在 ready = true 之前发生的普通写入，也就是：
```
number = 42;
```
所以这时打印 42 就有了内存模型层面的保证。

### Happens-Before的八大原则
#### 1. 程序次序规则

在同一个线程中，按照代码顺序，前面的操作 happens-before 后面的操作。
```
int a = 1;
int b = a + 1;
```
#### 2. 监视器锁规则

对一个锁的解锁，happens-before 后续对同一个锁的加锁。
```
synchronized (lock) {
    value = 1;
}
```

线程 A 释放 lock，线程 B 之后获得同一个 lock，那么 B 能看到 A 在同步块里的写入。
#### 3. volatile变量规则

对一个 volatile 变量的写，happens-before 后续对这个变量的读。
```
volatile boolean ready = false;
 
ready = true;     // 写
if (ready) { }    // 读
```

#### 4. 线程启动规则

对线程对象调用 start()，happens-before 该线程中的所有操作。

```
Thread t = new Thread(() -> {
    System.out.println("run");
});
 
t.start();
```

start() 之前的操作，对新线程可见。

#### 5. 线程启动规则

线程中的所有操作，happens-before 其他线程检测到该线程已经终止。

常见方式包括：

```
t.join();
```

也可以是 Thread.isAlive() 返回 false。

#### 6. 线程中断规则

对线程调用 interrupt()，happens-before 被中断线程检测到中断事件。

比如：

```
t.interrupt();
```

之后目标线程通过 isInterrupted()、interrupted() 或阻塞方法抛出 InterruptedException 感知到中断。

7. 对象终结规则
一个对象的构造方法执行完成，happens-before 它的 finalize() 方法开始执行。

不过现在 finalize() 已经过时，不建议在实际代码中依赖它。

8. 传递性规则
如果 A happens-before B，B happens-before C，那么 A happens-before C。

```
A -> B
B -> C
所以 A -> C
```

## Happens-Before 不是时间顺序

是最容易误解的一点。

happens-before 不是说操作 A 在现实时间上一定比操作 B 更早发生。

它说的是：
如果 A happens-before B，那么 Java 内存模型保证 B 能看到 A 的结果。

反过来说，如果没有 happens-before，即使 A 在现实时间上先执行，B 也不一定能看到 A 的结果。

这就是并发编程和单线程编程最大的区别之一。

## 总结

happens-before 是理解 Java 并发的核心概念。

它不是单纯的“谁先执行”，而是内存模型里的“谁对谁可见”。

判断一段多线程代码是否可靠时，可以问自己一个问题：

> [!NOTE]
> 一个线程的写入，和另一个线程的读取之间，有没有建立 happens-before 关系？

如果有，代码才有可靠的可见性和顺序保证。

如果没有，那么程序看起来“多数时候正常”，但在高并发、不同 CPU、不同 JVM 优化下，就可能出现难以复现的问题。

简单来说：

> [!NOTE]
> happens-before 是并发程序里判断内存可见性的规则。
> 写多线程代码时，不要只相信代码顺序，要相信同步关系。

