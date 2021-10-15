# java进程内存组成

![img](java%E8%BF%9B%E7%A8%8B%E5%86%85%E5%AD%98%E5%8D%A0%E7%94%A8/aHR0cHM6Ly9waWMxLnpoaW1nLmNvbS84MC92Mi0yODlmZTE5NDA0M2NiOTQ3MjE2OGE1YTczZmQ1ZmYwY18xNDQwdy5qcGc)

* Java堆最明显的部分。这是Java对象所在的位置。堆占用了-Xmx 内存。
* 垃圾收集器GC结构和算法需要额外的内存来进行堆管理。这些结构是Mark Bitmap，Mark Stack（用于遍历对象图），Remembered Sets（用于记录区域间引用）等。其中一些是直接可调的，例如-XX:MarkStackSizeMax，其他一些依赖于堆布局，例如，较大的是G1区域（-XX:G1HeapRegionSize），较小的是记忆集。GC内存开销因GC算法而异。-XX:+UseSerialGC并且-XX:+UseShenandoahGC开销最小。G1或CMS可能很容易使用总堆大小的10％左右。
* 代码缓存包含动态生成的代码：JIT编译的方法，解释器和运行时存根。它的大小受限于-XX:ReservedCodeCacheSize（默认为240M）。关闭-XX:-TieredCompilation以减少编译代码的数量，从而减少代码缓存的使用。
* 编译器JIT编译器本身也需要内存来完成它的工作。这可以通过关闭分层编译或减少编译器线程数来再次减少：-XX:CICompilerCount。
* 类加载类元数据（方法字节码，符号，常量池，注释等）存储在称为Metaspace的堆外区域中。加载的类越多 - 使用的元空间越多。总使用量可以受限-XX:MaxMetaspaceSize（默认为无限制）和 -XX:CompressedClassSpaceSize（默认为1G）。
* 符号表JVM的两个主要哈希表：Symbol表包含名称，签名，标识符等，String表包含interned字符串。如果本机内存跟踪指示String表占用大量内存，则可能意味着应用程序过度调用String.intern。
* 线程，堆栈大小由-Xss。每个线程的默认值是1M，但幸运的是事情并没有那么糟糕。操作系统懒惰地分配内存页面，即在第一次使用时，因此实际内存使用量将低得多（通常每个线程堆栈80-200 KB）。
* 直接缓存：应用程序可以通过调用显式请求堆外内存ByteBuffer.allocateDirect。默认的堆外限制等于-Xmx，但可以覆盖它-XX:MaxDirectMemorySize。除了直接的ByteBuffers，还可以有MappedByteBuffers映射到进程虚拟内存的文件。NMT（native memory trace）不跟踪它们，但MappedByteBuffers也可以占用物理内存。而且没有一种简单的方法来限制它们可以承受多少。您可以通过查看进程内存映射来查看实际使用情况：`pmap -x <pid>`
* 本地库包：加载的JNI代码System.loadLibrary可以根据需要分配尽可能多的堆外内存，而无需JVM端的控制。这也涉及标准的Java类库。特别是，未封闭的Java资源可能成为本机内存泄漏的来源。典型的例子是ZipInputStream或DirectoryStream。
* **分配器问题**，进程通常直接从OS（通过mmap系统调用）或使用malloc- 标准libc分配器请求本机内存。反过来，malloc请求OS使用大块内存mmap，然后根据自己的分配算法管理这些块。问题是 - 该算法可能导致碎片和[过多的虚拟内存使用](https://www.ibm.com/developerworks/community/blogs/kevgrig/entry/linux_glibc_2_10_rhel_6_malloc_may_show_excessive_virtual_memory_usage?lang=en)。[jemalloc](http://jemalloc.net/)，替代分配器，通常看起来比常规libc更智能malloc，因此切换到jemalloc可能导致更小的空闲。

那么一个Java进程最大占用的物理内存为：

`Max Memory = eden + survivor + old + String Constant Pool + Code cache + compressed class space + Metaspace + Thread stack(*thread num) + Direct + Mapped + JVM + Native Memory`



堆和非堆有以下几个概念：

**init**：表示JVM在启动时从操作系统申请内存管理的初始内存大小(以字节为单位)。JVM可能从操作系统请求额外的内存，也可以随着时间的推移向操作系统释放内存（经实际测试，这个内存并没有过主动释放）。这个init的值可能不会定义。

**used**：表示当前使用的内存量(以字节为单位)

**committed**：表示保证可供 Jvm使用的内存大小(以字节为单位)。 已提交内存的大小可能随时间而变化(增加或减少)。 JVM也可能向系统释放内存，导致已提交的内存可能小于 init，但是committed永远会大于等于used。

**max**：表示可用于内存管理的最大内存(以字节为单位)。



JDK8 提供的 NMT（ [Native Memory Tracking](https://link.jianshu.com/?t=https%3A%2F%2Fdocs.oracle.com%2Fjavase%2F8%2Fdocs%2Ftechnotes%2Fguides%2Ftroubleshoot%2Ftooldescr007.html)） 用于追踪 JVM 内存使用，也就是统计 malloc / mmap 的调用情况。启用方式：`-XX:NativeMemoryTracking=detail`

我们找了一台自己的机器测试，JVM 启动参数如下：

```sh
nohup $JAVA_HOME/bin/java -server \
-XX:+UseG1GC \
-XX:ConcGCThreads=1 \
-XX:ParallelGCThreads=4 \
-Xmx2g \
-Xms2g \
-XX:+ExplicitGCInvokesConcurrent \
-XX:MaxDirectMemorySize=256m \
-XX:MaxMetaspaceSize=64m \
-XX:NativeMemoryTracking=detail \
```

启动之后通过 jcmd 查看详细信息，`jcmd pid VM.native_memory` ：

```sh
Total: reserved=3590817KB, committed=2295813KB
-                 Java Heap (reserved=2097152KB, committed=2097152KB)
                            (mmap: reserved=2097152KB, committed=2097152KB)

-                     Class (reserved=1061169KB, committed=13489KB)
                            (classes #2295)
                            (malloc=305KB #2057)
                            (mmap: reserved=1060864KB, committed=13184KB)

-                    Thread (reserved=42328KB, committed=42328KB)
                            (thread #42)
                            (stack: reserved=42148KB, committed=42148KB)
                            (malloc=132KB #212)
                            (arena=48KB #82)

-                      Code (reserved=250310KB, committed=7082KB)
                            (malloc=710KB #1820)
                            (mmap: reserved=249600KB, committed=6372KB)

-                        GC (reserved=125871KB, committed=125871KB)
                            (malloc=15279KB #12893)
                            (mmap: reserved=110592KB, committed=110592KB)

-                  Compiler (reserved=146KB, committed=146KB)
                            (malloc=15KB #77)
                            (arena=131KB #3)

-                  Internal (reserved=3896KB, committed=3896KB)
                            (malloc=3864KB #6264)
                            (mmap: reserved=32KB, committed=32KB)

-                    Symbol (reserved=3662KB, committed=3662KB)
                            (malloc=2471KB #15155)
                            (arena=1191KB #1)

-    Native Memory Tracking (reserved=767KB, committed=767KB)
                            (malloc=130KB #2060)
                            (tracking overhead=636KB)

-               Arena Chunk (reserved=1420KB, committed=1420KB)
                            (malloc=1420KB)

-                   Unknown (reserved=4096KB, committed=0KB)
                            (mmap: reserved=4096KB, committed=0KB)

```

Java Heap 的分配通过 mmap，而不是 malloc，也就说这块内存不受 glibc 管理。

**reserved**：reserved memory 是指JVM 通过mmaped PROT_NONE 申请的虚拟地址空间，在页表中已经存在了记录（entries），保证了其他进程不会被占用。在堆内存下，就是xmx值，jvm申请的最大保留内存。

**committed**：committed memory 是JVM向操做系统实际分配的内存（malloc/mmap）,mmaped PROT_READ | PROT_WRITE，相当于程序实际申请的可用内存。在堆内存下，就是xms值，最小堆内存，heap committed memory。

> **committed申请的内存并不是说直接占用的物理内存，由于操作系统的内存管理是惰性的，对于已申请的内存虽然会分配地址空间，但并不会直接占用物理内存，真正使用的时候才会映射到实际的物理内存。所以committed > reserved也是很可能的**



当Java程序启动后，会根据Xmx为堆预申请（malloc之类的方法）Xms大小的虚拟内存，但是由于操作系统的内存管理是惰性的，有一个内存延迟分配的概念。malloc虽然会分配内存地址空间，但是并没有映射到实际的物理内存，只有当对该地址空间赋值时，才会真正的占用物理内存，才会影响RES的大小。

经过GC之后，虽然堆内存已经被回收了，堆占用很低，但GC的回收只是针对Jvm申请的这块内存区域，并不会调用操作系统释放内存。所以该进程的内存并不会释放，这时就会出现进程内存远远大于【堆+非堆】的情况。

* **RES（Resident Set Size）是常驻内存的意思，进程实际使用的物理内存**
* **VIRT(virtual memory usage )**: 虚拟内存。





linux为了解决多线程下内存分配竞争而引起的性能问题，增强了动态内存分配行为，使用了一种叫做arena的memory pool,在64位系统下面缺省配置是一个arena大小为64M，一个进程可以最多有cpu cores * 8个arena。假设机器是8核的，那么最多可以有8 * 8 = 64个arena，也就是会使用64 * 64 = 4096M内存。

我们可以通过设置系统环境变量来改变arena的数量： `export MALLOC_ARENA_MAX=8`（一般建议配置程序cpu核数）