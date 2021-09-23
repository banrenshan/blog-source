---

title: jvm
date: 2021-09-22 19:01:40
tags:
 - jvm
categories:
 - java
---









## StringTable

### 示例一：内存溢出

```java
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        int i = 0;
        while (true) {
            list.add(String.valueOf(i).intern());
            i++;
        }
    }
```

异常信息如下：

```sh
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
```

这个错误是由于JVM花费太长时间执行GC且只能回收很少的堆内存时抛出的。根据Oracle官方文档，默认情况下，如果Java进程花费98%以上的时间执行GC，并且每次只有不到2%的堆被回收，则JVM抛出此错误。换句话说，这意味着我们的应用程序几乎耗尽了所有可用内存，垃圾收集器花了太长时间试图清理它，并多次失败。

我们可以增加`-XX:-UseGCOverheadLimit`选项来关闭`GC Overhead limit exceeded`，此时抛出异常：

```sh
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

上面的例子说明，字符串常量池是存储在堆中的，jdk1.6之前，是存储在永久代,我们指定永久代最大大小：`-XX:MaxPermSize=10m`,此时抛出异常：

```shell
Exception in thread "main" java.lang.OutOfMemoryError:PermGen space
```









## 直接内存

* 常见于NIO操作，用于数据缓冲区，参考示例1
* 分配回收成本比较高，但读写性能高，参考示例1
* 不受JVM内存回收管理，参考示例2

### 示例1:基本使用

```java
package cn.zhao.jvm;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class DirectMemoryTest1 {

    public static final String PATH = "F:\\共享目录\\[电影天堂www.dytt89.com]白蛇2：青蛇劫起-2021_HD国语中字.mp4";
    public static final String OUTFILE = "D:\\aa.mp4";
    public static final int _1MB = 1024 * 1024;

    public static void main(String[] args) {
        long start1 = System.currentTimeMillis();
        testIO();
        long end1 = System.currentTimeMillis();
        System.err.println("io:" + (end1 - start1)); //10730

        long start2 = System.currentTimeMillis();
        testNiIO();

        long end2 = System.currentTimeMillis();
        System.err.println("nio:" + (end2 - start2)); //10054
    }

    public static void testIO() {
        try (FileInputStream fileInputStream = new FileInputStream(PATH);
             FileOutputStream fileOutStream = new FileOutputStream(OUTFILE)) {
            byte[] buffer = new byte[_1MB];
            while (fileInputStream.read(buffer) != -1) {
                fileOutStream.write(buffer, 0, buffer.length);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void testNiIO() {
        try (FileChannel inChannel = new FileInputStream(PATH).getChannel();
             FileChannel outChannel = new FileOutputStream("D:\\aa.mp4").getChannel()) {
            ByteBuffer buffer = ByteBuffer.allocateDirect(_1MB);

            while (inChannel.read(buffer) != -1) {
                buffer.flip();
                outChannel.write(buffer);
                buffer.clear();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```



运行多次，ByteBuffer的方式始终比传统方式快几百毫秒。下面是传统IO和ByteBuffer方式的原理图：



![image-20210922201357845](jvm/image-20210922201357845.png)



![image-20210922201452700](jvm/image-20210922201452700.png)



由此可见，直接内存下，省去了系统内存和java堆内存之间的相互copy。

### 示例2:垃圾回收

```java
    public static void main(String[] args) throws IOException {
        ByteBuffer buffer = ByteBuffer.allocateDirect(_1GB);
        System.err.println("分配内存完毕"); // 通过win10任务管理器，看到分配内存1GB
        System.in.read();
        buffer = null;
        System.gc();
        System.err.println("释放内存完毕");  // 通过win10任务管理器，看到释放内存1GB
        System.in.read();
    }
```



不是说，直接内存不归垃圾收集器管理么？这究竟是怎么一回事，我们先来看一段代码：

```java
public class DirectMemoryTest3 {

    static final int _1GB = 1024 * 1024 * 1024;

    public static void main(String[] args) throws IOException {
        Unsafe unsafe = getUnsafe();
        long l = unsafe.allocateMemory(_1GB);
        unsafe.setMemory(l, _1GB, (byte) 0);
        System.in.read();

        unsafe.freeMemory(l);
        System.in.read();

    }

    public static Unsafe getUnsafe() {
        try {
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            Unsafe unsafe = (Unsafe) field.get(null);
            return unsafe;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

Unsafe可以操作直接内存，控制内存的分配和释放，基于此，我们来看ByteBuffer的源码：

```java
    DirectByteBuffer(int cap) {                 
        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);

        long base = 0;
        try {
            base = unsafe.allocateMemory(size); //分配内存
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0); //分配内存
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap)); //释放内存
        att = null;
    }
```

> 细心的读者，可能注意到了，在DirectByteBuffer实例创建的时候，分配内存之前调用了`Bits.reserveMemory`方法，如果分配失败调用了`Bits.unreserveMemory`，同时在Deallocator释放完直接内存的时候，也调用了`Bits.unreserveMemory`方法。这两个方法，主要是记录jdk已经使用的直接内存的数量，当分配直接内存时，需要进行增加，当释放时，需要减少，源码如下：
>
> ```java
> static void reserveMemory(long size, int cap) {
>     //如果直接有足够多的直接内存可以用，直接增加直接内存引用的计数
>     synchronized (Bits.class) {
>         if (!memoryLimitSet && VM.isBooted()) {
>             maxMemory = VM.maxDirectMemory();
>             memoryLimitSet = true;
>         }
>         // -XX:MaxDirectMemorySize limits the total capacity rather than the
>         // actual memory usage, which will differ when buffers are page
>         // aligned.
>         if (cap <= maxMemory - totalCapacity) {//维护已经使用的直接内存的数量
>             reservedMemory += size;
>             totalCapacity += cap;
>             count++;
>             return;
>         }
>     }
>    //如果没有有足够多的直接内存可以用，先进行垃圾回收
>     System.gc();
>     try {
>         Thread.sleep(100);//休眠100秒，等待垃圾回收完成
>     } catch (InterruptedException x) {
>         // Restore interrupt status
>         Thread.currentThread().interrupt();
>     }
>     synchronized (Bits.class) {//休眠100毫秒后，增加直接内存引用的计数
>         if (totalCapacity + cap > maxMemory)
>             throw new OutOfMemoryError("Direct buffer memory");
>         reservedMemory += size;
>         totalCapacity += cap;
>         count++;
>     } 
> }
> //释放内存时，减少引用直接内存的计数
> static synchronized void unreserveMemory(long size, int cap) {
>     if (reservedMemory > 0) {
>         reservedMemory -= size;
>         totalCapacity -= cap;
>         count--;
>         assert (reservedMemory > -1);
>     }
> }
> ```
>
> 通过上面代码的分析，我们事实上可以认为`Bits`类是直接内存的分配担保，当有足够的直接内存可以用时，增加直接内存应用计数，否则，调用System.gc，进行垃圾回收，需要注意的是，`System.gc`只会回收堆内存中的对象，但是我们前面已经讲过，DirectByteBuffer对象被回收时，那么其引用的直接内存也会被回收，试想现在刚好有其他的DirectByteBuffer可以被回收，那么其被回收的直接内存就可以用于本次DirectByteBuffer直接的内存的分配



DirectByteBuffer本身是一个Java对象，其是位于堆内存中的，JDK的GC机制可以自动帮我们回收，但是其申请的直接内存，不在GC范围之内，无法自动回收。好在JDK提供了一种机制，可以为堆内存对象注册一个钩子函数(其实就是实现Runnable接口的子类)，当堆内存对象被GC回收的时候，会回调run方法，我们可以在这个方法中执行释放DirectByteBuffer引用的直接内存，即在run方法中调用Unsafe 的`freeMemory` 方法。注册是通过sun.misc.Cleaner类来实现的

Cleaner类继承PhantomReference（虚引用），垃圾回收器在回收ByteBuffer的引用时，触发虚引用的clean方法，该方法会调用Deallocator的run方法，执行清理直接内存的操作。

Deallocator实现了Runnable接口，如下面代码

```java
    private static class Deallocator
        implements Runnable
    {

        private static Unsafe unsafe = Unsafe.getUnsafe();

        private long address;
        private long size;
        private int capacity;

        private Deallocator(long address, long size, int capacity) {
            assert (address != 0);
            this.address = address;
            this.size = size;
            this.capacity = capacity;
        }

        public void run() {
            if (address == 0) {
                // Paranoia
                return;
            }
            unsafe.freeMemory(address); //释放内存
            address = 0;
            Bits.unreserveMemory(size, capacity);
        }

    }

```



我们再来看示例2，代码中调用了 `System.gc()` 函数，该函数会告诉jvm进行full gc(只是告诉，jvm决定最终是否执行)，但是生产上，我们通常开启`-XX:+DisableExplicitGC` 参数来禁用这种显示GC。但是这样就会出现直接内存长时间得不到释放的问题，导致系统内存告急的情况，道理很明显，因为直接内存的释放与获取比堆内存更加耗时，每次创建DirectByteBuffer实例分配直接内存的时候，都调用System.gc，可以让已经使用完的DirectByteBuffer得到及时的回收。

虽然System.gc只是建议JVM去垃圾回收，可能JVM并不会立即回收，但是频繁的建议，JVM总不会视而不见。

不过，这并不是绝对的，因为System.gc导致的是FullGC，可能会暂停用户线程，也就是JVM不能继续响应用户的请求，对于一些要求延时比较短的应用，是不希望JVM频繁的进行FullGC的。
所以笔者的建议是：禁用System.gc，调大最大可以使用的直接内存:

```shell
-XX:+DisableExplicitGC -XX:MaxDirectMemorySize=256M
```

### 示例3:直接内存溢出

```
public class DirectMemoryTest4 {
    static final int _1GB = 1024 * 1024 * 1024;

    public static void main(String[] args) {
        List list = new ArrayList();
        while (true) {
            ByteBuffer buffer = ByteBuffer.allocateDirect(_1GB);
            list.add(buffer);
        }
    }
}
```

异常信息如下：

```shell
Exception in thread "main" java.lang.OutOfMemoryError: Direct buffer memory
```

