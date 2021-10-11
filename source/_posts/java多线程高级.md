---
title: 并发编程高级
tags: 
 - 多线程
categories:
 - 并发编程
---

# synchronized 原理

## 对象头

以 32 位虚拟机为例：

普通对象：

![image-20211011141122946](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/image-20211011141122946.png)

数组对象：

![image-20211011141136060](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/image-20211011141136060.png)

其中 Mark Word 结构为：

![image-20211011141202463](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/image-20211011141202463.png)

## Monitor 

Monitor 被翻译为监视器或管程。每个 Java 对象都可以关联一个 Monitor 对象，如果使用 synchronized 给对象上锁（重量级）之后，该对象头的 Mark Word 中就被设置指向 Monitor 对象的指针。Monitor 结构如下：

![image-20211011141320076](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/image-20211011141320076.png)

1. 刚开始 Monitor 中 Owner 为 null 
2. 当 Thread-2 执行 synchronized(obj) 就会将 Monitor 的所有者 Owner 置为 Thread-2，Monitor中只能有一 个 Owner 
3. 在 Thread-2 持有锁的过程中，如果 Thread-3，Thread-4，Thread-5 也来执行 synchronized(obj)，就会进入 EntryList BLOCKED 
4. Thread-2 执行完同步代码块的内容，然后唤醒 EntryList 中等待的线程来竞争锁，竞争是非公平的 
5. 图中 WaitSet 中的 Thread-0，Thread-1 是之前获得过锁，但条件不满足进入 WAITING 状态的线程，后面讲 wait-notify 时会分析

> * synchronized 必须是进入同一个对象的 monitor 才有上述的效果 
> * 不加 synchronized 的对象不会关联监视器，不遵从以上规则

## synchronized 字节码

```java
static final Object lock = new Object();
static int counter = 0;
public static void main(String[] args) {
   synchronized (lock) {
   		counter++;
   }
}
```

字节码：

```java
public static void main(java.lang.String[]);
 descriptor: ([Ljava/lang/String;)V
 flags: ACC_PUBLIC, ACC_STATIC
Code:
     stack=2, locals=3, args_size=1
         0: getstatic #2 // <- lock引用 （synchronized开始）
         3: dup
         4: astore_1 // lock引用 -> slot 1
         5: monitorenter // 将 lock对象 MarkWord 置为 Monitor 指针
         6: getstatic #3 // <- i
         9: iconst_1 // 准备常数 1
         10: iadd // +1
         11: putstatic #3 // -> i
         14: aload_1 // <- lock引用
         15: monitorexit // 将 lock对象 MarkWord 重置, 唤醒 EntryList
         16: goto 24
               // 下面是synchronized代码块发生异常执行的字节码，保证有异常也能释放锁
         19: astore_2 // e -> slot 2 
         20: aload_1 // <- lock引用
         21: monitorexit // 将 lock对象 MarkWord 重置, 唤醒 EntryList
         22: aload_2 // <- slot 2 (e)
         23: athrow // throw e
         24: return
 Exception table:
     from to target type
         6 16 19 any
         19 22 19 any
 LineNumberTable:
     line 8: 0
     line 9: 6
     line 10: 14
     line 11: 24
 LocalVariableTable:
   	Start Length Slot Name Signature
 			  0 25 0 args [Ljava/lang/String;
 StackMapTable: number_of_entries = 2
     frame_type = 255 /* full_frame */
         offset_delta = 19
         locals = [ class "[Ljava/lang/String;", class java/lang/Object ]
         stack = [ class java/lang/Throwable ]
     frame_type = 250 /* chop */
     		offset_delta = 4
```

## synchronized锁优化

Java SE1.6为了减少获得锁和释放锁所带来的性能消耗，引入了“偏向锁”和“轻量级锁”，所以在Java SE1.6里锁一共有四种状态，无锁状态，偏向锁状态，轻量级锁状态和重量级锁状态，它会随着竞争情况逐渐升级。锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率，下文会详细分析。

### 轻量级锁

如果一个对象虽然有多线程要加锁，但加锁的时间是错开的（也就是没有竞争），那么可以使用轻量级锁来优化。

**轻量级锁加锁**：线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，进入锁膨胀流程。

1.创建锁记录：

![image-20211011143345903](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/image-20211011143345903.png)

2.下图是成功时，锁对象和当前线程栈中所记录相关的信息。

![image-20211011143213954](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/image-20211011143213954.png)

3.发生锁重入时：会创建新的锁记录，用作重入计数

![image-20211011143640396](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/image-20211011143640396.png)

4. 当退出 synchronized 代码块（解锁时）如果有取值为 null 的锁记录，表示有重入，这时重置锁记录，表示重 入计数减一

   

**轻量级锁解锁**：轻量级解锁时，会使用原子的CAS操作来将Displaced Mark Word替换回到对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。

### 锁膨胀

如果在尝试加轻量级锁的过程中，CAS 操作无法成功，这时一种情况就是有其它线程为此对象加上了轻量级锁（有 竞争），这时需要进行锁膨胀，将轻量级锁变为重量级锁。

1. 当 Thread-1 进行轻量级加锁时，Thread-0 已经对该对象加了轻量级锁

   ![image-20211011145628274](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/image-20211011145628274.png)

2. 这时 Thread-1 加轻量级锁失败，进入锁膨胀流程，即为 Object 对象申请 Monitor 锁，让 Object 指向重量级锁地址 ，然后自己进入 Monitor 的 EntryList BLOCKED：

   ![image-20211011145742945](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/image-20211011145742945.png)

3. 当 Thread-0 退出同步块解锁时，使用 cas 将 Mark Word 的值恢复给对象头，失败。这时会进入重量级解锁 流程，即按照 Monitor 地址找到 Monitor 对象，设置 Owner 为 null，唤醒 EntryList 中 BLOCKED 线程

### 自旋优化

重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步块，释放了锁），这时当前线程就可以避免阻塞。 **自旋重试成功的情况**：

![image-20211011150113282](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/image-20211011150113282.png)

**自旋重试失败的情况**：

![image-20211011150203844](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/image-20211011150203844.png)

* 自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势。
* 在 Java 6 之后自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会 高，就多自旋几次；反之，就少自旋甚至不自旋，总之，比较智能。 
* Java 7 之后不提供控制是否开启自旋功能的参数，JVM默认内置功能。

### 偏向锁

Hotspot的作者经过以往的研究发现大多数情况下锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。只有第一次使用CAS将线程ID设置到对象的Mark Word头,之后发现这个线程ID是自己的就表示没有竞争，不用重新CAS。以后只要不发生竞争,这个对象就归该线程所有。

> * 一个对象创建时： 如果开启了偏向锁（默认开启），那么对象创建后，markword 值为 0x05 即最后 3 位为 101，这时它的 thread、epoch、age 都为 0 
>
> * 偏向锁是默认是延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加 VM 参数 - XX:BiasedLockingStartupDelay=0 来禁用延迟 
> * 如果没有开启偏向锁，那么对象创建后，markword 值为 0x01 即最后 3 位为 001，这时它的 hashcode、 age 都为 0，第一次用到 hashcode 时才会赋值

1. 当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID。
2. 以后该线程在进入和退出同步块时不需要花费CAS操作来加锁和解锁，而只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁
3. 如果测试成功，表示线程已经获得了锁
4. 如果测试失败，则需要再测试下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁），如果没有设置，则使用CAS竞争锁（升级为轻量级锁），如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

**偏向锁的撤销**：偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。

1. 偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着
2. 如果线程不处于活动状态，则将对象头设置成无锁状态
3. 如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。

下图中的线程1演示了偏向锁初始化的流程，线程2演示了偏向锁撤销的流程。

![偏向锁的撤销](java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E9%AB%98%E7%BA%A7/%E5%81%8F%E5%90%91%E9%94%81%E7%9A%84%E6%92%A4%E9%94%80.png)

