---

title: jvm
date: 2021-09-22 19:01:40
tags:
 - jvm
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



我们再来看示例2，代码中调用了 `System.gc()` 函数，该函数会告诉jvm进行full gc(只是告诉，jvm决定最终是否执行)，但是生产上，我们通常开启`-XX:+DisableExplicitGC` 参数来禁用这种显示GC。

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

