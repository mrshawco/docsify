## JVM 垃圾回收的时候如何确定垃圾？知道什么是 GC Roots ?

- 什么是垃圾
  - 简单来说就是内存中已经不在被使用到的空间就是垃圾
- 要进行垃圾回收，如何判断一个对象是否可以被回收？
  - **引用计数法**
  - 枚举根节点做**可达性分析**



为了解决引用计数法的**循环引用**问题，Java 使用了可达性算法。

[![img](http://img.cuzz.site/2020/20200910141729.jpg)](http://img.cuzz.site/2020/20200910141729.jpg)

跟踪收集器采用的为集中式的管理方式，全局记录对象之间的引用状态，执行时从一些列GC Roots的对象做为起点，从这些节点向下开始进行搜索所有的引用链，当一个对象到GC Roots 没有任何引用链时，则证明此对象是不可用的。

图中，对象Object6、Object7、Object8虽然互相引用，但他们的GC Roots是不可到达的，所以它们将会被判定为是可回收的对象。

哪些对象可以作为 GC Roots 的对象：

- 虚拟机栈（栈帧中的局部变量区，也叫局部变量表）中引用的对象
- 方法区中的类静态属性引用的对象
- 方法去常量引用的对象
- 本地方法栈中 JNI (Native方法)引用的对象



## 你说你做过 JVM 调优和参数配置，请问如何查看 JVM 系统默认值？

**JVM 的参数类型:**

标配参数

- -version
- -help

X 参数（了解）

- -Xint：解释执行
- -Xcomp：第一次使用就编译成本地代码
- -Xmixed：混合模式

XX 参数

- Boolean 类型：-XX：+ 或者 - 某个属性值（+ 表示开启，- 表示关闭）

  - -XX:+PrintGCDetails：打印 GC 收集细节
  - -XX:-PrintGCDetails：不打印 GC 收集细节
  - -XX:+UseSerialGC：使用了串行收集器
  - -XX:-UseSerialGC：不使用了串行收集器

- KV 设置类型：-XX:key=value

  - -XX:MetaspaceSize=128m
  - -XX:MaxTenuringThreshold=15

- jinfo 举例，如何查看当前运行程序的配置

  ```java
  public class HelloGC {
      public static void main(String[] args) {
          System.out.println("hello GC...");
          try {
              Thread.sleep(Integer.MAX_VALUE);
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
      }
  }
  ```

  我们可以使用 `jps -l` 命令，查出进程 id

  ```sh
  1923 org.jetbrains.jps.cmdline.Launcher
  1988 sun.tools.jps.Jps
  1173 org.jetbrains.kotlin.daemon.KotlinCompileDaemon
  32077 com.intellij.idea.Main
  1933 com.cuzz.jvm.HelloGC
  32382 org.jetbrains.idea.maven.server.RemoteMavenServer
  ```

  在使用 `jinfo -flag PrintGCDetails 1933` 命令查看

  ```sh
  -XX:-PrintGCDetails
  ```

  可以看出默认是不打印 GC 收集细节
  也可是使用`jinfo -flags 1933` 查看所有的参数

- 两个经典参数：-Xms 和 - Xmx（如 -Xms1024m）

  - -Xms 等价于 -XX:InitialHeapSize
  - -Xmx 等价于 -XX:MaxHeapSize

**查看 JVM 默认值**

- 查看初始默认值：-XX:+PrintFlagsInitial

  ```
  cuzz@cuzz-pc:~/Project/demo$ java -XX:+PrintFlagsInitial
  [Global flags]
       intx ActiveProcessorCount                      = -1                                  {product}
      uintx AdaptiveSizeDecrementScaleFactor          = 4                                   {product}
      uintx AdaptiveSizeMajorGCDecayTimeScale         = 10                                  {product}
      uintx AdaptiveSizePausePolicy                   = 0                                   {product}
      uintx AdaptiveSizePolicyCollectionCostMargin    = 50                                  {product}
      uintx AdaptiveSizePolicyInitializingSteps       = 20                                  {product}
      uintx AdaptiveSizePolicyOutputInterval          = 0                                   {product}
      uintx AdaptiveSizePolicyWeight                  = 10                                  {product}
     ...
  ```

- 查看修改更新：-XX:+PrintFlagsFinal

  ```
  bool UsePSAdaptiveSurvivorSizePolicy           = true                                {product}
  bool UseParNewGC                               = false                               {product}
  bool UseParallelGC                            := true                                {product}
  bool UseParallelOldGC                          = true                                {product}
  bool UsePerfData                               = true                                {product}
  bool UsePopCountInstruction                    = true                                {product}
  bool UseRDPCForConstantTableBase               = false                               {C2 product}
  ```

  > = 与 := 的区别是，一个是默认，一个是人物改变或者 jvm 加载时改变的参数

- 打印命令行参数(可以看默认垃圾回收器)：-XX:+PrintCommandLineFlags

  ```
  cuzz@cuzz-pc:~/Project/demo$ java -XX:+PrintCommandLineFlags
  -XX:InitialHeapSize=128789376 -XX:MaxHeapSize=2060630016 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
  ```



## 你平时工作用过的 JVM 常用的基本配置参数有哪些？

- -Xms
  - 初始堆内存大小，默认为物理内存 1/64
  - 等价于 -XX:InitialHeapSize
- -Xmx
  - 最大分配堆内存，默认为物理内存的 1/4
  - 等价于 -XX:MaxHeapSize
- -Xss
  - 设置单个线程栈的大小，一般默认为 512-1024k
  - 等价于 -XX:ThreadStackSize
- -Xmn
  - 设置年轻代的大小
  - **整个JVM内存大小=年轻代大小 + 年老代大小 + 永久代大小**，永久代一般固定大小为64m，所以增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。
- -XX:MetaspaceSize
  - 设置元空间大小（元空间的本质和永久代类似，都是对 JVM 规范中的方法区的实现，不过元空间于永久代之间最大区别在于，**元空间并不在虚拟机中，而是使用本地内存**，因此默认情况下，元空间的大小仅受本地内存限制）
  - 元空间默认比较小，我们可以调大一点
- -XX:+PrintGCDetails
  - 输出详细 GC 收集日志信息
  - 设置 JVM 参数为： -Xms10m -Xmx10m -XX:+PrintGCDetails
- -XX:SurvivorRatio
  - 设置新生代中 eden 和 S0/S1 空间比例
  - 默认 -XX:SurvivorRatio=8，Eden : S0 : S1 = 8 : 1 : 1
- -XX:NewRatio
  - 配置年轻代和老年代在堆结构的占比
  - 默认 -XX:NewRatio=2 新生代占1，老年代占2，年轻代占整个堆的 1/3
- -XX:MaxTenuringThreshold
  - 设置垃圾最大年龄



## 强引用、软引用、弱引用和虚引用分别是什么？

在Java语言中，除了基本数据类型外，其他的都是指向各类对象的对象引用；Java中根据其生命周期的长短，将引用分为4类。

**强引用**

- 我们平常典型编码`Object obj = new Object()`中的 obj 就是强引用，通过关键字new创建的对象所关联的引用就是强引用。
- 当JVM内存空间不足，JVM宁愿抛出 OutOfMemoryError 运行时错误（OOM），使程序异常终止，也不会靠随意回收具有强引用的“存活”对象来解决内存不足的问题。
- 对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应强引用赋值为 null，就是可以被垃圾收集的了，具体回收时机还是要看垃圾收集策略。

**软引用**

- 软引用通过SoftReference类实现， 软引用的生命周期比强引用短一些。
- 只有当 JVM 认为内存不足时，才会去试图回收软引用指向的对象：即 JVM 会确保在抛出 OutOfMemoryError 之前，清理软引用指向的对象。
- 软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。后续，我们可以调用ReferenceQueue的poll()方法来检查是否有它所关心的对象被回收。如果队列为空，将返回一个null，否则该方法返回队列中前面的一个Reference对象。
- 应用场景：软引用通常用来实现内存敏感的缓存。如果还有空闲内存，就可以暂时保留缓存，当内存不足时清理掉，这样就保证了使用缓存的同时，不会耗尽内存。

代码验证，我设置 JVM 参数为 `-Xms10m -Xmx10m -XX:+PrintGCDetails`

```java
public class SoftReferenceDemo {
    public static void main(String[] args) {
        Object obj = new Object();
        SoftReference<Object> softReference = new SoftReference<>(obj);
        obj = null;

        try {
            // 分配 20 M
            byte[] bytes = new byte[20 * 1024 * 1024];
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            System.out.println("软引用：" + softReference.get());
        }

    }
}
```

发现当内存不够的时候就会被回收。

```sh
[GC (Allocation Failure) [PSYoungGen: 1234K->448K(2560K)] 1234K->456K(9728K), 0.0016748 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[GC (Allocation Failure) [PSYoungGen: 448K->384K(2560K)] 456K->392K(9728K), 0.0018398 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) [PSYoungGen: 384K->0K(2560K)] [ParOldGen: 8K->358K(7168K)] 392K->358K(9728K), [Metaspace: 3030K->3030K(1056768K)], 0.0057246 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[GC (Allocation Failure) [PSYoungGen: 0K->0K(2560K)] 358K->358K(9728K), 0.0006038 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) [PSYoungGen: 0K->0K(2560K)] [ParOldGen: 358K->340K(7168K)] 358K->340K(9728K), [Metaspace: 3030K->3030K(1056768K)], 0.0115080 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
软引用：null
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
    at com.cuzz.jvm.SoftReferenceDemo.main(SoftReferenceDemo.java:21)
Heap
 PSYoungGen      total 2560K, used 98K [0x00000000ffd00000, 0x0000000100000000, 0x0000000100000000)
  eden space 2048K, 4% used [0x00000000ffd00000,0x00000000ffd18978,0x00000000fff00000)
  from space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
  to   space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
 ParOldGen       total 7168K, used 340K [0x00000000ff600000, 0x00000000ffd00000, 0x00000000ffd00000)
  object space 7168K, 4% used [0x00000000ff600000,0x00000000ff6552f8,0x00000000ffd00000)
 Metaspace       used 3067K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 336K, capacity 388K, committed 512K, reserved 1048576K
```

**弱引用**

- 弱引用通过 WeakReference 类实现， 弱引用的生命周期比软引用短。
- 在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。由于垃圾回收器是一个优先级很低的线程，因此不一定会很快回收弱引用的对象。
- 弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。
- 应用场景：弱应用同样可用于内存敏感的缓存。

代码验证

```java
public class WeakReferenceDemo {
    public static void main(String[] args) {
        Object obj = new Object();
        WeakReference<Object> weakReference = new WeakReference<>(obj);
        System.out.println(obj);
        System.out.println(weakReference.get());

        obj = null;
        System.gc();
        System.out.println("GC之后....");
        
        System.out.println(obj);
        System.out.println(weakReference.get());
    }
}
```

输出

```sh
java.lang.Object@1540e19d
java.lang.Object@1540e19d
GC之后....
null
null
```

引用队列

```java
public class ReferenceQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        Object obj = new Object();
        ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();
        WeakReference<Object> weakReference = new WeakReference<>(obj, referenceQueue);
        System.out.println(obj);
        System.out.println(weakReference.get());
        System.out.println(weakReference);

        obj = null;
        System.gc();
        Thread.sleep(500);

        System.out.println("GC之后....");
        System.out.println(obj);
        System.out.println(weakReference.get());
        System.out.println(weakReference);
    }
}
```

会把该对象的包装类即`weakReference`放入到`ReferenceQueue`里面，我们可以从queue中获取到相应的对象信息，同时进行额外的处理。比如反向操作，数据清理等。

```sh
java.lang.Object@1540e19d
java.lang.Object@1540e19d
java.lang.ref.WeakReference@677327b6
GC之后....
null
null
java.lang.ref.WeakReference@677327b6
```

**虚引用**

- 虚引用也叫幻象引用，通过PhantomReference类来实现，无法通过虚引用访问对象的任何属性或函数。

- 幻象引用仅仅是提供了一种确保对象被 finalize 以后，做某些事情的机制。

- 如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。

- 虚引用必须和引用队列 （ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。

  ```java
  ReferenceQueue queue = new ReferenceQueue ();
  PhantomReference pr = new PhantomReference (object, queue); 
  ```

- 程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取一些程序行动。

- 应用场景：可用来跟踪对象被垃圾回收器回收的活动，当一个虚引用关联的对象被垃圾收集器回收之前会收到一条系统通知。