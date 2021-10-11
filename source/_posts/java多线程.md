---
title: 并发编程基础
tags: 
 - 多线程
categories:
 - 并发编程
---

# 并发编程

## 概念对比

### 进程和线程

* **进程**
  * 程序由指令和数据组成，但这些指令要运行，数据要读写，就必须将指令加载至 CPU，数据加载至内存。在指令运行过程中还需要用到磁盘、网络等设备。进程就是用来加载指令、管理内存、管理 IO 的 
  * 当一个程序被运行，从磁盘加载这个程序的代码至内存，这时就开启了一个进程。 
  * 进程就可以视为程序的一个实例。大部分程序可以同时运行多个实例进程（例如记事本、画图、浏览器 等），也有的程序只能启动一个实例进程（例如网易云音乐、360 安全卫士等） 
* **线程** 
  * 一个进程之内可以分为一到多个线程。 
  * 一个线程就是一个指令流，将指令流中的一条条指令以一定的顺序交给 CPU 执行 
  * Java 中，线程作为最小调度单位，进程作为资源分配的最小单位。 在 windows 中进程是不活动的，只是作 为线程的容器
* **二者对比** 
  * 进程基本上相互独立的，而线程存在于进程内，是进程的一个子集
  * 进程拥有共享的资源，如内存空间等，供其内部的线程共享 
  * 进程间通信较为复杂 同一台计算机的进程通信称为 IPC（Inter-process communication） 不同计算机之间的进程通信，需要通过网络，并遵守共同的协议，例如 HTTP 线程通信相对简单，因为它们共享进程内的内存，一个例子是多个线程可以访问同一个共享变量 
  * 线程更轻量，线程上下文切换成本一般上要比进程上下文切换低

### 并行与并发 

单核 cpu 下，线程实际还是串行执行的。操作系统中有一个组件叫做任务调度器，将cpu 的时间片（windows 下时间片最小约为 15 毫秒）分给不同的程序使用，只是由于 cpu 在线程间（时间片很短）的切换非常快，人类感 觉是 同时运行的 。总结为一句话就是： 微观串行，宏观并行 ， 一般会将这种线程轮流使用 CPU 的做法称为并发（concurrent。

多核 cpu下，每个 核（core） 都可以调度运行线程，这时候线程可以是并行的。

引用 Rob Pike 的一段描述： 并发（concurrent）是同一时间应对（dealing with）多件事情的能力 并行（parallel）是同一时间动手做（doing）多件事情的能力

### 同步与异步

以调用方角度来讲：

* 如果 需要等待结果返回，才能继续运行就是同步 
* 不需要等待结果返回，就能继续运行就是异步

多线程可以让方法执行变为异步的（即不要巴巴干等着）比如说读取磁盘文件时，假设读取操作花费了 5 秒钟，如 果没有线程调度机制，这 5 秒 cpu 什么都做不了，其它代码都得暂停...

* 比如在项目中，视频文件需要转换格式等操作比较费时，这时开一个新线程处理视频转换，避免阻塞主线程
* tomcat 的异步 servlet 也是类似的目的，让用户线程处理耗时较长的操作，避免阻塞 tomcat 的工作线程 
* ui 程序中，开线程进行其他操作，避免阻塞 ui 线程

### 线程上下文切换（Thread Context Switch） 

因为以下一些原因导致 

* cpu 不再执行当前的线程，转而执行另一个线程的代码 
* 线程的 cpu 时间片用完 
* 垃圾回收 
* 有更高优先级的线程需要运行 
* 线程自己调用了 sleep、yield、wait、join、park、synchronized、lock 等方法 

当 Context Switch 发生时，需要由操作系统保存当前线程的状态，并恢复另一个线程的状态，Java 中对应的概念就是程序计数器（Program Counter Register），它的作用是记住下一条 jvm 指令的执行地址，是线程私有的。 

状态包括程序计数器、虚拟机栈中每个栈帧的信息，如局部变量、操作数栈、返回地址等 

Context Switch 频繁发生会影响性能



### 多线程小结论

1. 单核 cpu 下，多线程不能实际提高程序运行效率，只是为了能够在不同的任务之间切换，不同线程轮流使用 cpu ，不至于一个线程总占用 cpu，别的线程没法干活 
2.  多核 cpu 可以并行跑多个线程，但能否提高程序运行效率还是要分情况的： 有些任务，经过精心设计，将任务拆分，并行执行，当然可以提高程序的运行效率。但不是所有计算任务都能拆分（参考后文的【阿姆达尔定律】） 也不是所有任务都需要拆分，任务的目的如果不同，谈拆分和效率没啥意义 
3.  IO 操作不占用 cpu，只是我们一般拷贝文件使用的是【阻塞 IO】，这时相当于线程虽然不用 cpu，但需要一直等待 IO 结束，没能充分利用线程。所以才有后面的【非阻塞 IO】和【异步 IO】优化

## Thead类

### 基本属性

线程ID：不能实时修改

线程名称：线程的名称可以实时修改

线程优先级：不能实时修改，Java线程可以有优先级的设定，高优先级的线程比低优先级的线程有更高的几率得到执行

1. 记住当线程的优先级没有指定时，所有线程都携带普通优先级。
2. 优先级可以用从1到10的范围指定。10表示最高优先级，1表示最低优先级，5是普通优先级。
3. 记住优先级最高的线程在执行时被给予优先。但是不能保证线程在启动时就进入运行状态。
4. 与在线程池中等待运行机会的线程相比，当前正在运行的线程可能总是拥有更高的优先级。
5. 由调度程序决定哪一个线程被执行。
6. t.setPriority()用来设定线程的优先级。
7. 记住在线程开始方法被调用之前，线程的优先级应该被设定。
8. 你可以使用常量，如MIN_PRIORITY,MAX_PRIORITY，NORM_PRIORITY来设定优先级。

守护线程：不能实时修改，默认情况下，Java 进程需要等待所有线程都运行结束，才会结束。有一种特殊的线程叫做守护线程，只要其它非守 护线程运行结束了，即使守护线程的代码没有执行完，也会强制结束。

```java
log.debug("开始运行...");
Thread t1 = new Thread(() -> {
   log.debug("开始运行...");
   sleep(2);
   log.debug("运行结束...");
}, "daemon");
// 设置该线程为守护线程
t1.setDaemon(true);
t1.start();
sleep(1);
log.debug("运行结束...");
```

打印结果：

```sh
08:26:38.123 [main] c.TestDaemon - 开始运行... 
08:26:38.213 [daemon] c.TestDaemon - 开始运行... 
08:26:39.215 [main] c.TestDaemon - 运行结束... 
```

守护线程并没有执行完就被终止了。

> * 垃圾回收器线程就是一种守护线程 
> * Tomcat 中的 Acceptor 和 Poller 线程都是守护线程，所以 Tomcat 接收到 shutdown 命令后，不会等 待它们处理完当前请求

线程组：可以批量管理线程或线程组对象，有效地对线程或线程组对象进行组织。用户创建的所有线程都属于指定线程组，如果没有显示指定属于哪个线程组，那么该线程就属于默认线程组（即main线程组）。默认情况下，子线程和父线程处于同一个线程组。只有在创建线程时才能指定其所在的线程组，线程运行中途不能改变它所属的线程组，也就是说线程一旦指定所在的线程组，就直到该线程结束。

### 线程的状态

java线程和操作系统中的线程是一一对应的。我们先看操作系统中的线程状态：

![image-20211011130747733](java%E5%A4%9A%E7%BA%BF%E7%A8%8B/image-20211011130747733.png)



* 【初始状态】仅是在语言层面创建了线程对象，还未与操作系统线程关联 

* 【可运行状态】（就绪状态）指该线程已经被创建（与操作系统线程关联），可以由 CPU 调度执行 

* 【运行状态】指获取了 CPU 时间片运行中的状态 ，当 CPU 时间片用完，会从【运行状态】转换至【可运行状态】，会导致线程的上下文切换。 

* 【阻塞状态】 如果调用了阻塞 API，如 BIO 读写文件，这时该线程实际不会用到 CPU，会导致线程上下文切换，进入 【阻塞状态】。 等 BIO 操作完毕，会由操作系统唤醒阻塞的线程，转换至【可运行状态】 。与【可运行状态】的区别是，对【阻塞状态】的线程来说只要它们一直不唤醒，调度器就一直不会考虑 调度它们 

* 【终止状态】表示线程已经执行完毕，生命周期已经结束，不会再转换为其它状态

这是从 Java API 层面来描述的 根据 Thread.State 枚举，分为六种状态：

![image-20211011131151590](java%E5%A4%9A%E7%BA%BF%E7%A8%8B/image-20211011131151590.png)

* NEW 线程刚被创建，但是还没有调用 start() 方法 

* RUNNABLE 当调用了 start() 方法之后，注意，Java API 层面的 RUNNABLE 状态涵盖了 操作系统 层面的 【可运行状态】、【运行状态】和【阻塞状态】（由于 BIO 导致的线程阻塞，在 Java 里无法区分，仍然认为 是可运行） BLOCKED ， WAITING ， TIMED_WAITING 都是 Java API 层面对【阻塞状态】的细分，后面会在状态转换一节 详述 
* TERMINATED 当线程代码运行结束







### 创建线程的方式

* 继承Thread类

  ```java
  // 构造方法的参数是给线程指定名字，推荐
  Thread t1 = new Thread("t1") {
     @Override
     // run 方法内实现了要执行的任务
     public void run() {
     log.debug("hello");
     }
  };
  t1.start();
  ```

* 实现Runable接口

  ```java
  Runnable runnable = new Runnable() {
     public void run(){
     // 要执行的任务
     }
  };
  // 创建线程对象
  Thread t = new Thread( runnable );
  // 启动线程
  t.start(); 
  ```

* FutureTask

  ```java
  // 创建任务对象
  FutureTask<Integer> task3 = new FutureTask<>(() -> {
     log.debug("hello");
     return 100;
  });
  // 参数1 是任务对象; 参数2 是线程名字，推荐
  new Thread(task3, "t3").start();
  // 主线程阻塞，同步等待 task 执行完毕的结果
  Integer result = task3.get();
  log.debug("结果是:{}", result);
  ```

**继承Thread和实现Runable接口对比**

* 继承Thread需要重写run方法，此方法是线程被调度时，需要执行的方法
* 实现Runable接口，当线程被调度时，依然时执行run方法，此方法的内容是调用Runable的run方法。
* 从本质上将，两者并没有区别，一个是将线程执行的任务直接写在了run方法中，另一个是通过Runable接口中转了一次。对应设计模式中的？？？

### 查看某个进程的所有线程

* ps -fe :查看所有进程 
* ps -fT -p  pid: 查看某个进程（PID）的所有线程
* kill pid : 杀死进程 
* top :  按大写 H 切换是否显示线程 
* top -H -p  pid : 查看某个进程（PID）的所有线程
* jps: 命令查看所有 Java 进程 
* jstack pid: 查看某个 Java 进程（PID）的所有线程状态

开启远程监控：

```sh
java -Djava.rmi.server.hostname=`ip地址` -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=`连接端口` -Dcom.sun.management.jmxremote.ssl=是否安全连接 -Dcom.sun.management.jmxremote.authenticate=是否认证  MainClass 
```

### start和run

**start**: native方法

 启动一个新线程，在新的线程 运行 run 方法中的代码。start 方法只是让线程进入就绪，里面代码不一定立刻 运行（CPU 的时间片还没分给它）。每个线程对象的 start方法只能调用一次，如果调用了多次会出现`IllegalThreadStateException`。线程一旦完成执行就可能不会重新启动。

**run**：线程被调度时，调用改方法

如果该线程是使用单独的 Runnable 运行对象构造的，则调用该 Runnable 对象的 run 方法； 否则，此方法不执行任何操作并返回。Thread 的子类应该覆盖这个方法。

### sleep和yield

**sleep**: native方法

1. 调用 sleep 会让当前线程从 Running 进入 Timed Waiting 状态（阻塞） ,该线程不会失去任何监视器的所有权。
2.  其它线程可以使用 interrupt 方法打断正在睡眠的线程，这时 sleep 方法会抛出 InterruptedException ，抛出此异常时会清除中断状态。
3.  睡眠结束后的线程未必会立刻得到执行 
4. 建议用 TimeUnit 的 sleep 代替 Thread 的 sleep 来获得更好的可读性

**yield**: native方法

* 向调度程序提示当前线程愿意放弃其当前对处理器的使用（从runable状态到就绪状态）。 调度程序可能会忽略此请求。
* Yield 是一种启发式尝试，旨在改善线程之间的相对进展，否则会过度使用 CPU。 它的使用应与详细的分析和基准测试相结合，以确保它确实具有预期的效果。
* 很少适合使用这种方法。 它对于调试或测试目的可能很有用，它可能有助于重现由于竞争条件引起的错误。 在设计并发控制结构（例如 java.util.concurrent.locks 包中的结构）时，它也可能很有用。

### join

用于线程间同步，等待这个线程死亡。如果任何线程中断了当前线程。 抛出此异常时清除当前线程的中断状态。我们通过代码来看：

```java
    public static void main(String[] args) {
        Thread thread = new Thread(() -> System.err.println("等待我运行完成"));
        thread.start();
        try {
            // main线程等待 thread运行完成
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.err.println("结束");
    }
```

调用者线程等待join线程执行完成。

源码分析：

```java
    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) { // isAlive是native
                wait(0); // wait是native
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

1. 判断被等待线程是否执行完成
2. 执行完成则继续运行，否则等待,进入阻塞状态，等待被唤醒。

### interrupt 

native方法，中断此线程。

* 如果此线程在调用 Object 类的 wait()、wait(long) 或 wait(long, int) 方法或 join()、join(long)、join(long, int) 方法、sleep(long) 或 sleep(long, int) 等方法时被阻塞，那么它的中断状态将被清除，并且它会收到一个 InterruptedException。参考例1
* 如果此线程在 InterruptibleChannel 上的 I/O 操作中被阻塞，则通道将关闭，线程的中断状态将被设置，线程将收到 java.nio.channels.ClosedByInterruptException。
* 如果该线程在 java.nio.channels.Selector 中被阻塞，则该线程的中断状态将被设置，并且它将立即从选择操作中返回，可能具有非零值，就像调用了选择器的唤醒方法一样。
* 如果前面的条件都不成立，则将设置此线程的中断状态。 参考例2
* 打断 park 线程, 不会清空打断状态，参考例3

考例1：

```java
    public static void test1() {

        Thread thread = new Thread(() -> {
            System.err.println("线程运行中。。。");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                System.err.println("线程被打断了"); 
                e.printStackTrace();
                // 中断状态将被清除
                System.err.println("线程的状态：" + Thread.currentThread().isInterrupted());
            }
        });
        thread.start();
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        thread.interrupt();
    }
```

输出：

```sh
线程运行中。。。
线程被打断了
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at cn.zhao.concurrency.InterruptTest.lambda$test1$0(InterruptTest.java:16)
	at java.lang.Thread.run(Thread.java:745)
线程是否被打断：false
```

考例2:

```java
    public static void test2() {
        Thread t1 = new Thread(() -> {
            while (true) {
                // 中断状态没有被清除
                if (Thread.currentThread().isInterrupted()) {
                    System.err.println("线程被打断了");
                    break;
                }
            }
        });

        t1.start();
        t1.interrupt();
    }
```

例3：

```java
    public static void test3() {
        Thread t1 = new Thread(() -> {
            LockSupport.park();
            boolean interrupted = Thread.currentThread().isInterrupted();
            System.err.println("线程是否被打断：" + interrupted);

            LockSupport.park();
            System.err.println("打断标记已经是 true, 则 park 会失效");

            boolean interrupted1 = Thread.interrupted();
            System.err.println("清除了打断状态");

            LockSupport.park();

            System.err.println("因为park生效了，不能执行这行代码");
        });
        t1.start();
        t1.interrupt();
    }
```

### stop、suspend和resume

这些方法已过时，容易破坏同步代码块，造成线程死锁。

* stop() 停止线程运行 

* suspend() 挂起（暂停）线程运行 

* resume() 恢复线程运行

### 静态getAllStackTraces

返回所有活动线程的堆栈跟踪图。 

```java
        Map<Thread, StackTraceElement[]> allStackTraces =
                Thread.getAllStackTraces();
        allStackTraces.forEach((k, v) -> {
            System.err.println(k + ":" + Arrays.asList(v));
        });
```

打印结果：

```sh
Thread[Finalizer,8,system]:[java.lang.Object.wait(Native Method), java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143), java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164), java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)]
Thread[main,5,main]:[java.lang.Thread.dumpThreads(Native Method), java.lang.Thread.getAllStackTraces(Thread.java:1607), cn.zhao.concurrency.PrintThead.main(PrintThead.java:9)]
Thread[Monitor Ctrl-Break,5,main]:[java.net.SocketInputStream.socketRead0(Native Method), java.net.SocketInputStream.socketRead(SocketInputStream.java:116), java.net.SocketInputStream.read(SocketInputStream.java:171), java.net.SocketInputStream.read(SocketInputStream.java:141), sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:284), sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:326), sun.nio.cs.StreamDecoder.read(StreamDecoder.java:178), java.io.InputStreamReader.read(InputStreamReader.java:184), java.io.BufferedReader.fill(BufferedReader.java:161), java.io.BufferedReader.readLine(BufferedReader.java:324), java.io.BufferedReader.readLine(BufferedReader.java:389), com.intellij.rt.execution.application.AppMainV2$1.run(AppMainV2.java:48)]
Thread[Attach Listener,5,system]:[]
Thread[Signal Dispatcher,9,system]:[]
Thread[Reference Handler,10,system]:[java.lang.Object.wait(Native Method), java.lang.Object.wait(Object.java:502), java.lang.ref.Reference.tryHandlePending(Reference.java:191), java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)]

```

线程名称+线程优先级+线程组。

### 静态setDefaultUncaughtExceptionHandler和setUncaughtExceptionHandler

捕获线程中抛出的异常

```java
        Thread.setDefaultUncaughtExceptionHandler((t, ex) -> {
            System.err.println("线程" + t.getName() + ",异常信息" + ex.getLocalizedMessage());
        });

        new Thread(() -> {
            System.err.println(1 / 0);
        }).start();
```

打印如下：

```sh
线程Thread-0,异常信息/ by zero
```

## 管程

见名知意，是指管理共享变量以及对共享变量操作的过程。

### 临界区

一个程序运行多个线程本身是没有问题的，问题出在多个线程访问共享资源 ；多个线程读共享资源其实也没有问题 ；在多个线程对共享资源读写操作时发生指令交错，就会出现问题。

 一段代码块内如果存在对共享资源的多线程读写操作，称这段代码块为临界区。 例如，下面代码中的临界区

```java
static int counter = 0;
static void increment() 
// 临界区
{ 
 counter++;
}
static void decrement() 
// 临界区
{ 
 counter--;
}
```

### 竞态条件 Race Condition

 多个线程在临界区内执行，由于代码的执行序列不同而导致结果无法预测，称之为发生了竞态条件

### synchronized

为了避免临界区的竞态条件发生，有多种手段可以达到目的。 

* 阻塞式的解决方案：synchronized，Lock 
* 非阻塞式的解决方案：原子变量 

本次使用阻塞式的解决方案：synchronized，来解决上述问题，即俗称的【对象锁】，它采用互斥的方式让同一时刻至多只有一个线程能持有【对象锁】，其它线程再想获取这个【对象锁】时就会阻塞住。这样就能保证拥有锁的线程可以安全的执行临界区内的代码，不用担心线程上下文切换

> 虽然 java 中互斥和同步都可以采用 synchronized 关键字来完成，但它们还是有区别的： 
>
> * 互斥是保证临界区的竞态条件发生，同一时刻只能有一个线程执行临界区代码。 
> * 同步是由于线程执行的先后、顺序不同。需要一个线程等待其它线程运行到某个点，也就是协调多个线程的顺序。

