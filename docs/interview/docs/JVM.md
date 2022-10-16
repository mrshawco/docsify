## 1. 请谈谈你对 volatile 的理解
**volatile 是 JVM 提供的轻量级的同步机制**

1. 保证可见性
2. 禁止指令排序（有序性）
3. **不保证原子性**

### 可见性

#### JMM（Java内存模型）

JMM（Java 内存模型 Java Memory Model，简称 JMM）本身是一种抽象的概念并不真实存在，它描述的是一组规则或规范，通过这组规范定义了程序中各个变量（包括实例字段，静态字段和构成数组对象的元素）的访问方式。

JMM 关于同步的规定：

1. 线程解锁前，必须把共享变量的值刷新回主内存
2. 线程加锁前，必须读取主内存的最新值到自己的工作内存
3. 加锁解锁是同一把锁

由于 JVM 运行程序的实体是线程，而每个线程创建时 JVM 都会为其创建一个工作内存（有些地方称为栈空间），工作内存是每个线程的私有数据区域，而 Java 内存模型中规定所有变量都存储在主内存，主内存是共享内存区域，所有线程都可以访问，**但线程对变量的操作（读取赋值等）必须在工作内存中进行，首先要将变量从主内存拷贝的自己的工作内存空间，然后对变量进行操作，操作完成后再将变量写回主内存**，不能直接操作主内存中的变量，各个线程中的工作内存中存储着主内存中的变量副本拷贝，因此不同的线程间无法访问对方的工作内存，线程间的通信（传值）必须通过主内存来完成，其简要访问过程如下图：

![](https://s2.loli.net/2022/10/14/Z5KEurI6G48dfw1.png)

#### 代码验证

```java
import java.util.concurrent.TimeUnit;

/**
 * 假设是主物理内存
 */
class MyData {
    //volatile int number = 0;
    int number = 0;

    public void addTo60() {
        this.number = 60;
    }
}

/**
 * 验证volatile的可见性
 * 1. 假设int number = 0， number变量之前没有添加volatile关键字修饰
 */
public class VolatileDemo {
    public static void main(String args[]) {
        // 资源类
        MyData myData = new MyData();

        // AAA线程 实现了Runnable接口的，lambda表达式
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t come in");

            // 线程睡眠3秒，假设在进行运算
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // 修改number的值
            myData.addTo60();

            // 输出修改后的值
            System.out.println(Thread.currentThread().getName() + "\t update number value:" + myData.number);
        }, "AAA").start();

        // main线程就一直在这里等待循环，直到number的值不等于零
        while (myData.number == 0) {
        }

        // 按道理这个值是不可能打印出来的，因为主线程运行的时候，number的值为0，所以一直在循环
        // 如果能输出这句话，说明AAA线程在睡眠3秒后，更新的number的值，重新写入到主内存，并被main线程感知到了
        System.out.println(Thread.currentThread().getName() + "\t mission is over");
    }
}
```

如果不加 volatile 关键字，则主线程会进入死循环，加 volatile 则主线程能够退出，说明加了 volatile 关键字变量，当有一个线程修改了值，会马上被另一个线程感知到，当前值作废，从新从主内存中获取值。对其他线程可见，这就叫可见性。

### 原子性

原子性指不可分割，完整性，也即某个线程正在做某个具体业务时，中间不可以**被加塞或者被分割**。需要整体完整要么**同时成功**，要么**同时失败**。

#### 代码验证

```java
class MyData2 {
    /**
     * volatile 修饰的关键字，是为了增加 主线程和线程之间的可见性，只要有一个线程修改了内存中的值，其它线程也能马上感知
     */
    volatile int number = 0;

    public void addPlusPlus() {
        number ++;
    }
}

public class VolatileAtomicityDemo {
	public static void main(String[] args) {
        MyData2 myData = new MyData2();

        // 创建10个线程，线程里面进行1000次循环
        for (int i = 0; i < 20; i++) {
            new Thread(() -> {
                // 里面
                for (int j = 0; j < 1000; j++) {
                    myData.addPlusPlus();
                }
            }, String.valueOf(i)).start();
        }

        // 需要等待上面20个线程都计算完成后，在用main线程取得最终的结果值
        // 这里判断线程数是否大于2，为什么是2？因为默认是有两个线程的，一个main线程，一个gc线程
        while(Thread.activeCount() > 2) {
            // yield表示不执行
            Thread.yield();
        }

        // 查看最终的值
        // 假设volatile保证原子性，那么输出的值应该为：  20 * 1000 = 20000
        System.out.println(Thread.currentThread().getName() + "\t finally number value: " + myData.number);
	}
}
```

输出结果小于20000。

#### 理论解释

number++ 在多线程下是非线程安全的。

我们可以将代码编译成字节码，可看出 number++ 被编译成3条指令。

![img](https://s2.loli.net/2022/10/14/TtlwiReAVJaOcjg.png)

n++ 拆分为三步骤：

1. 线程读取 n
2. temp = n + 1
3. n = temp

假如当 n = 5 的时候 A，B 两个线程同时读入了 n 的值， 然后 A 线程执行了 temp = n + 1 的操作， 要注意，此时的 n 的值还没有变化，然后 B 线程也执行了 temp = n + 1 的操作，注意，此时 A，B 两个线程保存的 n 的值都是5，temp 的值都是6， 然后 A 线程执行了 n = temp(6) 的操作，此时 n 的值会立即刷新到主存并通知其他线程保存的 n 值失效， 此时 B 线程需要重新读取 n 的值那么此时 B 线程保存的 n 就是6，同时 B 线程保存的 **temp 还仍然是6**， 然后 B 线程执行 n = temp(6)，所以导致了计算结果比预期少了1。

#### 解决方法

1. 可加 synchronized 解决，但它是重量级同步机制，性能上有所顾虑。
2. 如何不加 synchronized 解决 number++ 在多线程下是非线程安全的问题？使用 **AtomicInteger**。

```java
import java.util.concurrent.atomic.AtomicInteger;

class MyData2 {
    /**
     * volatile 修饰的关键字，是为了增加 主线程和线程之间的可见性，只要有一个线程修改了内存中的值，其它线程也能马上感知
     */
	volatile int number = 0;
	AtomicInteger number2 = new AtomicInteger();

    public void addPlusPlus() {
        number ++;
    }
    
    public void addPlusPlus2() {
    	number2.getAndIncrement();
    }
}

public class VolatileAtomicityDemo {

	public static void main(String[] args) {
        MyData2 myData = new MyData2();

        // 创建10个线程，线程里面进行1000次循环
        for (int i = 0; i < 20; i++) {
            new Thread(() -> {
                // 里面
                for (int j = 0; j < 1000; j++) {
                    myData.addPlusPlus();
                    myData.addPlusPlus2();
                }
            }, String.valueOf(i)).start();
        }

        // 需要等待上面20个线程都计算完成后，在用main线程取得最终的结果值
        // 这里判断线程数是否大于2，为什么是2？因为默认是有两个线程的，一个main线程，一个gc线程
        while(Thread.activeCount() > 2) {
            // yield表示不执行
            Thread.yield();
        }

        // 查看最终的值
        // 假设volatile保证原子性，那么输出的值应该为：  20 * 1000 = 20000
        System.out.println(Thread.currentThread().getName() + "\t finally number value: " + myData.number);
        System.out.println(Thread.currentThread().getName() + "\t finally number2 value: " + myData.number2);
	}
}
```

输出结果：

```sh
main	 finally number value: 18766
main	 finally number2 value: 20000
```

### 有序性

计算机在执行程序时，为了提高性能，编译器个处理器常常会对指令做重排，一般分为以下 3 种

1. 编译器优化的重排
2. 指令并行的重排
3. 内存系统的重排

**单线程环境**里面确保程序最终执行的结果和代码执行的结果一致

处理器在进行重排序时必须考虑指令之间的**数据依赖性**

多线程环境中线程交替执行，由于编译器优化重排的存在，两个线程中使用的变量能否保证用的变量能否一致性是无法确定的，结果无法预测

**重排案例1**

```java
public void mySort{
	int x = 11;//语句1
    int y = 12;//语句2
    × = × + 5;//语句3
    y = x * x;//语句4
}
```

可重排序列：

- 1234
- 2134
- 1324

**重排案例2**

int a, b, x, y = 0；

| 线程1        | 线程2  |
| ------------ | ------ |
| x = a;       | y = b; |
| b = 1;       | a = 2; |
| x = 0; y = 0 |        |


如果编译器对这段程序代码执行重排优化后，可能出现下列情况：

| 线程1        | 线程2  |
| ------------ | ------ |
| b = 1;       | a = 2; |
| x = a;       | y = b; |
| x = 2; y = 1 |        |

这也就说明在多线程环境下，由于编译器优化重排的存在，两个线程中使用的变量能否保证一致性是无法确定的。

#### 示例代码

```java
public class ReSortSeqDemo{
	int a = 0;
	boolean flag = false;
    
	public void method01(){
		a = 1;//语句1
		flag = true;//语句2
	}
    
    public void method02(){
        if(flag){
            a = a + 5; //语句3
        }
        System.out.println("retValue: " + a);//可能是6或1或5或0
    }  
}
```

多线程环境中线程交替执行`method01()`和`method02()`，由于编译器优化重排的存在，两个线程中使用的变量能否保证一致性是无法确定的，结果无法预测。

#### 禁止指令排序

volatile 实现禁止指令重排序的优化，从而避免了多线程环境下程序出现乱序的现象

先了解一个概念，内存屏障（Memory Barrier）又称内存栅栏，是一个 CPU 指令，他的作用有两个：

- 保证特定操作的执行顺序
- 保证某些变量的内存可见性（利用该特性实现 volatile 的内存可见性）

由于编译器和处理器都能执行指令重排优化。如果在指令间插入一条 Memory Barrier 则会告诉编译器和 CPU，不管什么指令都不能和这条 Memory Barrier 指令重排序，也就是说**通过插入内存屏障禁止在内存屏障前后的指令执行重排序优化**。内存屏障另外一个作用是强制刷出各种 CPU 的缓存数据，因此任何 CPU 上的线程都能读取到这些数据的最新版本。

对 volatile 变量进行写操作时，会在写操作后加入一条 store 屏障指令，将工作内存中的共享变量值刷新回到主内存。

[![img](https://s2.loli.net/2022/10/14/R2XvhjKfnxL3Q59.png)

对 Volatile 变量进行读操作时，会在读操作前加入一条 load 屏障指令，从主内存中读取共享变量。

[![img](https://s2.loli.net/2022/10/14/hsFc3GQPIbfgETx.png)

**线程安全性保证**

- 工作内存与主内存同步延迟现象导致**可见性**问题

  可以使用 synchronzied 或 volatile 关键字解决，它们可以使用一个线程修改后的变量立即对其他线程可见

- 对于指令重排导致**可见性**问题和**有序性**问题

  可以利用 volatile 关键字解决，因为 volatile 的另一个作用就是禁止指令重排序优化

#### **使用场景**

多线程环境下可能存在的安全问题，发现构造器里的内容会多次输出

```java
@NotThreadSafe
public class Singleton01 {
    private static Singleton01 instance = null;
    private Singleton01() {
        System.out.println(Thread.currentThread().getName() + "  construction...");
    }
    public static Singleton01 getInstance() {
        if (instance == null) {
            instance = new Singleton01();
        }
        return instance;
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i++) {
            executorService.execute(()-> Singleton01.getInstance());
        }
        executorService.shutdown();
    }
}
```

双重检查单例

```java
public class Singleton02 {
    private static volatile Singleton02 instance = null;
    private Singleton02() {
        System.out.println(Thread.currentThread().getName() + "  construction...");
    }
    public static Singleton02 getInstance() {
        if (instance == null) {
            synchronized (Singleton01.class) {
                if (instance == null) {
                    instance = new Singleton02();
                }
            }
        }
        return instance;
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i++) {
            executorService.execute(()-> Singleton02.getInstance());
        }
        executorService.shutdown();
    }
}
```

如果没有加 volatile 就不一定是线程安全的，原因是指令重排序的存在，加入 volatile 可以禁止指令重排。原因是在于某一个线程执行到第一次检测，读取到的 instance 不为 null 时，**instance 的引用对象可能还没有完成初始化。**`instance = new Singleton() `可以分为以下三步完成。

```
memory = allocate();  // 1.分配对象空间
instance(memory);     // 2.初始化对象
instance = memory;    // 3.设置instance指向刚分配的内存地址，此时instance != null
```

步骤2和步骤3不存在数据依赖关系，而且无论重排前还是重排后程序的执行结果在单线程中并没有改变，因此这种重排优化是允许的。

```
memory = allocate();  // 1.分配对象空间
instance = memory;    // 3.设置instance指向刚分配的内存地址，此时instance != null，但对象还没有初始化完成
instance(memory);     // 2.初始化对象
```

但是指令重排只会保证串行语义的执行的一致性(单线程)，但并不会关心多线程间的语义一致性。

所以当一条线程访问 instance 不为 null 时，由于 instance 实例未必已初始化完成，也就造成了线程安全问题。

## 2. CAS 你知道吗？CAS 底层原理？谈谈对 UnSafe 的理解？

Compare And Set

```java
public class CASDemo {
    public static void main(String[] args) {
        AtomicInteger atomicInteger = new AtomicInteger(666);
        // 获取真实值，并替换为相应的值
        boolean b = atomicInteger.compareAndSet(666, 2019);
        System.out.println(b); // true
        boolean b1 = atomicInteger.compareAndSet(666, 2020);
        System.out.println(b1); // false
        atomicInteger.getAndIncrement();
    }
}
```

getAndIncrement() 方法

```java
/**
* Atomically increments by one the current value.
*
* @return the previous value
*/
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
```

引出一个问题：UnSafe 类是什么？我们先看看 AtomicInteger 就使用了 Unsafe 类。

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            // 获取下面 value 的地址偏移量
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
    // ...
}
```

**Unsafe 类：**

1. Unsafe 是 CAS 的核心类，由于 Java 方法无法直接访问底层系统，而需要通过本地（native）方法来访问， Unsafe 类相当一个后门，基于该类可以直接操作特定内存的数据。Unsafe 类存在于 sun.misc 包中，其内部方法操作可以像 C 指针一样直接操作内存，因为 Java 中 CAS 操作执行依赖于 Unsafe 类。
2. 变量 vauleOffset，表示该变量值在内存中的偏移量，因为 Unsafe 就是根据内存偏移量来获取数据的。
3. 变量 value 用 volatile 修饰，保证了多线程之间的内存可见性。

**CAS 是什么？**

1. CAS 的全称 Compare-And-Swap，它是一条 CPU 并发原语。

2. 它的功能是判断内存某一个位置的值是否为预期，如果是则更改这个值，这个过程就是原子的。

3. CAS 并发原体现在 JAVA 语言中就是 sun.misc.Unsafe 类中的各个方法。调用 UnSafe 类中的 CAS 方法，JVM 会帮我们实现出 CAS 汇编指令。这是一种完全依赖硬件的功能，通过它实现了原子操作。由于 CAS 是一种系统原语，原语属于操作系统用语范畴，是由若干条指令组成，用于完成某一个功能的过程，并且原语的执行必须是连续的，在执行的过程中不允许被中断，也就是说 CAS 是一条原子指令，不会造成所谓的数据不一致的问题。

4. 分析一下 getAndAddInt 这个方法


  ```java
  public final int getAndAddInt(Object var1, long var2, int var4) {
      int var5;
      do {
          var5 = this.getIntVolatile(var1, var2);
      } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
      return var5;
  }
  ```

- var1 AtomicInteger 对象本身。
- var2 该对象值得引用地址。
- var4 需要变动的数量。
- var5 是用过 var1，var2 找出的主内存中真实的值。
- 用该对象当前的值与 var5 比较：

    - 如果相同，更新 var5 + var4 并且返回 true,

    - 如果不同，继续取值然后再比较，直到更新完成。

假设线程 A 和线程 B 两个线程同时执行 getAndAddInt 操作（分别跑在不同 CPU 上) ：

1. Atomiclnteger 里面的 value 原始值为3，即主内存中 Atomiclnteger 的 value 为3，根据 JMM 模型，线程 A 和线程 B 各自持有一份值为3的 value 的副本分别到各自的工作内存。
2. 线程 A 通过 getIntVolatile(var1, var2) 拿到 value 值3，这时线程 A 被挂起。
3. 线程 B 也通过 getintVolatile(var1, var2) 方法获取到 value 值3，此时刚好线程 B 没有被挂起并执行 compareAndSwapInt 方法比较内存值也为3，成功修改内存值为4，线程 B 打完收工，一切 OK。
4. 这时线程 A 恢复，执行 compareAndSwapInt 方法比较，发现自己手里的值数字3和主内存的值数字4不一致，说明该值己经被其它线程抢先一步修改过了，那 A 线程本次修改失败，只能重新读取重新来一遍了。
5. 线程 A 重新获取 value 值，因为变量 value 被 volatile 修饰，所以其它线程对它的修改，线程 A 总是能够看到，线程 A 继续执行 compareAndSwapInt 进行比较替换，直到成功。

**CAS 的缺点？**

- 循环时间长开销很大

  如果 CAS 失败，会一直尝试，如果 CAS 长时间一直不成功，可能会给 CPU 带来很大的开销（比如线程数很多，每次比较都是失败，就会一直循环），所以希望是线程数比较小的场景。

- 只能保证一个共享变量的原子操作

  对于多个共享变量操作时，循环 CAS 就无法保证操作的原子性。

- 引出 ABA 问题

## 3. CAS 的 ABA 问题谈一谈？原子更新引用知道吗？

### ABA 问题的产生

CAS 算法实现一个重要前提需要取出内存中某时刻的数据并在当下时刻比较并替换，那么在这个时间差类会导致数据的变化。

比如说一个线程 one 从内存位置 V 中取出 A ，这时候另一个线程 two 也从内存中取出 A，并且线程 two 进行了一些操作将值变成了 B,然后线程 two 又将 V 位置的数据变成 A，这时候线程 one 进行 CAS 操作发现内存中仍然是 A，然后线程 one 操作成功。

尽管线程 one 的 CAS 操作成功，但是不代表这个过程就是没有问题的。

### 原子引用

```java
import java.util.concurrent.atomic.AtomicReference;

class User{
	
	String userName;
	
	int age;
	
    public User(String userName, int age) {
		this.userName = userName;
		this.age = age;
	}

	@Override
	public String toString() {
		return String.format("User [userName=%s, age=%s]", userName, age);
	}
    
}

public class AtomicReferenceDemo {
    public static void main(String[] args){
        User z3 = new User( "z3",22);
        User li4 = new User("li4" ,25);
		AtomicReference<User> atomicReference = new AtomicReference<>();
        atomicReference.set(z3);
		System.out.println(atomicReference.compareAndSet(z3, li4)+"\t"+atomicReference.get().toString());
        System.out.println(atomicReference.compareAndSet(z3, li4)+"\t"+atomicReference.get().toString());
    }
}
```

输出结果：

```sh
true	User [userName=li4, age=25]
false	User [userName=li4, age=25]
```

### AtomicStampedReference 时间戳原子引用

原子引用 + 新增一种机制，那就是修改版本号（类似时间戳），它用来解决 ABA 问题。

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABADemo {
	/**
	 * 普通的原子引用包装类
	 */
	static AtomicReference<Integer> atomicReference = new AtomicReference<>(100);

	// 传递两个值，一个是初始值，一个是初始版本号
	static AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<>(100, 1);

	public static void main(String[] args) {
		System.out.println("============以下是ABA问题的产生==========");

		new Thread(() -> {
			// 把100 改成 101 然后在改成100，也就是ABA
			atomicReference.compareAndSet(100, 101);
			atomicReference.compareAndSet(101, 100);
		}, "t1").start();

		new Thread(() -> {
			try {
				// 睡眠一秒，保证t1线程，完成了ABA操作
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			// 把100 改成 101 然后在改成100，也就是ABA
			System.out.println(atomicReference.compareAndSet(100, 2019) + "\t" + atomicReference.get());
		}, "t2").start();

		try {
			TimeUnit.SECONDS.sleep(2);
		} catch (Exception e) {
			e.printStackTrace();
		}
		
		System.out.println("============以下是ABA问题的解决==========");

		new Thread(() -> {
			// 获取版本号
			int stamp = atomicStampedReference.getStamp();
			System.out.println(Thread.currentThread().getName() + "\t 第一次版本号" + stamp);

			// 暂停t3一秒钟
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}

			// 传入4个值，期望值，更新值，期望版本号，更新版本号
			atomicStampedReference.compareAndSet(100, 101, atomicStampedReference.getStamp(),
					atomicStampedReference.getStamp() + 1);

			System.out.println(Thread.currentThread().getName() + "\t 第二次版本号" + atomicStampedReference.getStamp());

			atomicStampedReference.compareAndSet(101, 100, atomicStampedReference.getStamp(),
					atomicStampedReference.getStamp() + 1);

			System.out.println(Thread.currentThread().getName() + "\t 第三次版本号" + atomicStampedReference.getStamp());
		}, "t3").start();

		new Thread(() -> {
			// 获取版本号
			int stamp = atomicStampedReference.getStamp();
			System.out.println(Thread.currentThread().getName() + "\t 第一次版本号" + stamp);

			// 暂停t4 3秒钟，保证t3线程也进行一次ABA问题
			try {
				TimeUnit.SECONDS.sleep(3);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}

			boolean result = atomicStampedReference.compareAndSet(100, 2019, stamp, stamp + 1);

			System.out.println(Thread.currentThread().getName() + "\t 修改成功否：" + result + "\t 当前最新实际版本号："
					+ atomicStampedReference.getStamp());

			System.out.println(Thread.currentThread().getName() + "\t 当前实际最新值" + atomicStampedReference.getReference());
		}, "t4").start();
	}
}
```

输出结果：

```sh
============以下是ABA问题的产生==========
true	2019
============以下是ABA问题的解决==========
t3	 第一次版本号1
t4	 第一次版本号1
t3	 第二次版本号2
t3	 第三次版本号3
t4	 修改成功否：false	 当前最新实际版本号：3
t4	 当前实际最新值100
```

## 4. 我们知道 ArrayList 是线程不安全，请编写一个不安全的案例并给出解决方案？

```java
public class ContainerDemo {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        Random random = new Random();
        for (int i = 0; i < 100; i++) {
            new Thread(() -> {
                list.add(random.nextInt(10));
                System.out.println(list);
            }).start();
        }
    }
}
```

发现报`java.util.ConcurrentModificationException`

导致原因

- 并发修改导致的异常

解决方案

- `new Vector();`
- `Collections.synchronizedList(new ArrayList<>());`
- `new CopyOnWriteArrayList<>();`

优化建议

- 在读多写少的时候推荐使用 CopeOnWriteArrayList 这个类

## 5. java 中锁你知道哪些？请手写一个自旋锁？

### **1、公平和非公平锁**

是什么

- 公平锁：是指多个线程按照申请的顺序来获取值。
- 非公平锁：是值多个线程获取值的顺序并不是按照申请锁的顺序，有可能后申请的线程比先申请的线程优先获取锁，在高并发的情况下，可能会造成**优先级反转**或者**饥饿现象**。

两者区别

- 公平锁：公平锁就是很公平，在并发环境中，每个线程在获取锁时会先查看此锁维护的等待队列，如果为空，或者当前线程是等待队列的第一个，就占有锁，否则就会加入到等待队列中，以后会按照 FIFO 的规则从队列中取到自己。
- 非公平锁：非公平锁比较粗鲁，上来就直接尝试占有锁，如果尝试失败，就再采用类似公平锁那种方式。

### **2、可重入锁和不可重入锁**

是什么

- 可重入锁：指的是同一线程外层函数获得锁之后，内层递归函数仍然能获取该锁的代码，在同一个线程在外层方法获取锁的时候，在进入内层方法会自动获取锁。也即是说，**线程可以进入任何一个它已经拥有的锁所同步着的代码块**。

  ReentrantLock/synchronized 就是一个典型的可重入锁。

  可重入锁最大的作用是避免死锁。

- 不可重入锁： 所谓不可重入锁，即若当前线程执行某个方法已经获取了该锁，那么在方法中尝试再次获取锁时，就会获取不到被阻塞。

代码实现

Synchronized 可入锁演示程序

```java
class Phone {
    public synchronized void sendSMS() throws Exception{
        System.out.println(Thread.currentThread().getName() + "\t invoked sendSMS()");

        // 在同步方法中，调用另外一个同步方法
        sendEmail();
    }

    public synchronized void sendEmail() throws Exception{
        System.out.println(Thread.currentThread().getId() + "\t invoked sendEmail()");
    }
}

public class SynchronizedReentrantLockDemo {
	public static void main(String[] args) {
        Phone phone = new Phone();

        // 两个线程操作资源列
        new Thread(() -> {
            try {
                phone.sendSMS();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }, "t1").start();

        new Thread(() -> {
            try {
                phone.sendSMS();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }, "t2").start();
	}
}
```

输出结果：

```sh
t1	 invoked sendSMS()
11	 invoked sendEmail()
t2	 invoked sendSMS()
12	 invoked sendEmail()
```

ReentrantLock可重入锁演示程序

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class Phone2 implements Runnable{
    Lock lock = new ReentrantLock();

    /**
     * set进去的时候，就加锁，调用set方法的时候，能否访问另外一个加锁的set方法
     */
    public void getLock() {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t get Lock");
            setLock();
        } finally {
            lock.unlock();
        }
    }

    public void setLock() {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t set Lock");
        } finally {
            lock.unlock();
        }
    }

    @Override
    public void run() {
        getLock();
    }
}

public class ReentrantLockDemo {
    public static void main(String[] args) {
        Phone2 phone = new Phone2();

        /**
         * 因为Phone实现了Runnable接口
         */
        Thread t3 = new Thread(phone, "t3");
        Thread t4 = new Thread(phone, "t4");
        t3.start();
        t4.start();
    }
}
```

输出结果：

```sh
t3	 get Lock
t3	 set Lock
t4	 get Lock	
t4	 set Lock
```

### **3、自旋锁**

是指定尝试获取锁的线程不会立即堵塞，而是**采用循环的方式去尝试获取锁**，这样的好处是减少线程上线文切换的消耗，缺点就是循环会消耗 CPU。

**手动实现自旋锁**

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

public class SpinLockDemo {
    // 现在的泛型装的是Thread，原子引用线程
    AtomicReference<Thread>  atomicReference = new AtomicReference<>();

    public void myLock() {
        // 获取当前进来的线程
        Thread thread = Thread.currentThread();
        System.out.println(Thread.currentThread().getName() + "\t come in ");

        // 开始自旋，期望值是null，更新值是当前线程，如果是null，则更新为当前线程，否者自旋
        while(!atomicReference.compareAndSet(null, thread)) {
			//摸鱼
        }
    }

    public void myUnLock() {
        // 获取当前进来的线程
        Thread thread = Thread.currentThread();

        // 自己用完了后，把atomicReference变成null
        atomicReference.compareAndSet(thread, null);

        System.out.println(Thread.currentThread().getName() + "\t invoked myUnlock()");
    }
    
	public static void main(String[] args) {
        SpinLockDemo spinLockDemo = new SpinLockDemo();

        // 启动t1线程，开始操作
        new Thread(() -> {
            // 开始占有锁
            spinLockDemo.myLock();

            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            // 开始释放锁
            spinLockDemo.myUnLock();
        }, "t1").start();


        // 让main线程暂停1秒，使得t1线程，先执行
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // 1秒后，启动t2线程，开始占用这个锁
        new Thread(() -> {
            // 开始占有锁
            spinLockDemo.myLock();
            // 开始释放锁
            spinLockDemo.myUnLock();
        }, "t2").start();
	}
}

```

输出：

```sh
t1	 come in 
t2	 come in 
t1	 invoked myUnlock()
t2	 invoked myUnlock()
```

### **4、独占锁（写锁）/共享锁（读锁）**

是什么

- 独占锁：指该锁一次只能被一个线程持有。
- 共享锁：该锁可以被多个线程持有。

对于 ReentrantLock 和 synchronized 都是独占锁；对与 ReentrantReadWriteLock 其读锁是共享锁而写锁是独占锁。读锁的共享可保证并发读是非常高效的，读写、写读和写写的过程是互斥的。

ReentrantReadWriteLock 读写锁例子

```java
package com.lun.concurrency;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantReadWriteLock;

class MyCache2 {
    private volatile Map<String, Object> map = new HashMap<>();

    private ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

    public void put(String key, Object value) {
        // 创建一个写锁
        rwLock.writeLock().lock();

        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在写入：" + key);

            try {
                // 模拟网络拥堵，延迟0.3秒
                TimeUnit.MILLISECONDS.sleep(300);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            map.put(key, value);

            System.out.println(Thread.currentThread().getName() + "\t 写入完成");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 写锁 释放
            rwLock.writeLock().unlock();
        }
    }

    public void get(String key) {
        // 读锁
        rwLock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在读取:");

            try {
                // 模拟网络拥堵，延迟0.3秒
                TimeUnit.MILLISECONDS.sleep(300);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            Object value = map.get(key);

            System.out.println(Thread.currentThread().getName() + "\t 读取完成：" + value);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 读锁释放
            rwLock.readLock().unlock();
        }
    }

    public void clean() {
        map.clear();
    }
}

public class ReadWriteWithLockDemo {
    public static void main(String[] args) {
        MyCache2 myCache = new MyCache2();

        // 线程操作资源类，5个线程写
        for (int i = 1; i <= 5; i++) {
            // lambda表达式内部必须是final
            final int tempInt = i;
            new Thread(() -> {
                myCache.put(tempInt + "", tempInt +  "");
            }, String.valueOf(i)).start();
        }

        // 线程操作资源类， 5个线程读
        for (int i = 1; i <= 5; i++) {
            // lambda表达式内部必须是final
            final int tempInt = i;
            new Thread(() -> {
                myCache.get(tempInt + "");
            }, String.valueOf(i)).start();
        }
    }
}
```

输出结果：

```sh
1	 正在写入：1
1	 写入完成
2	 正在写入：2
2	 写入完成
3	 正在写入：3
3	 写入完成
5	 正在写入：5
5	 写入完成
4	 正在写入：4
4	 写入完成
2	 正在读取:
3	 正在读取:
1	 正在读取:
5	 正在读取:
4	 正在读取:
3	 读取完成：3
2	 读取完成：2
1	 读取完成：1
5	 读取完成：5
4	 读取完成：4
```

能保证**读写**、**写读**和**写写**的过程是互斥的时候是独享的，**读读**的时候是共享的。

## 6. CountDownLatch、CyclicBarrier 和Semaphore 使用过吗？

### **1、CountDownLatch**

让一线程阻塞直到另一些线程完成一系列操作才被唤醒。CountDownLatch 主要有两个方法，当一个或多个线程调用 await() 时，调用线程会被阻塞。其它线程调用 countDown() 会将计数器减1(调用 countDown 方法的线程不会阻塞)，当计数器的值变为零时，因调用 await 方法被阻塞的线程会被唤醒，继续执行。

假设一个自习室里有7个人，其中有一个是班长，班长的主要职责就是在其它6个同学走了后，关灯，锁教室门，然后走人，因此班长是需要最后一个走的，那么有什么方法能够控制班长这个线程是最后一个执行，而其它线程是随机执行的。

```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {

        // 计数器
        CountDownLatch countDownLatch = new CountDownLatch(6);

        for (int i = 0; i <= 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "\t 上完自习，离开教室");
                countDownLatch.countDown();
            }, String.valueOf(i)).start();
        }

        countDownLatch.await();

        System.out.println(Thread.currentThread().getName() + "\t 班长最后关门");
    }
}
```

输出结果：

```sh
0	 上完自习，离开教室
6	 上完自习，离开教室
4	 上完自习，离开教室
5	 上完自习，离开教室
3	 上完自习，离开教室
1	 上完自习，离开教室
2	 上完自习，离开教室
main	 班长最后关门
```

### **2、CyclicBarrier**

CyclicBarrier 的字面意思就是可循环（Cyclic）使用的屏障（Barrier）。它要求做的事情是，让**一组线程到达一个屏障**（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活，线程进入屏障通过 CyclicBarrier的await 方法。

程序演示集齐7个龙珠，召唤神龙

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class SummonTheDragonDemo {
    public static void main(String[] args) {
        /**
         * 定义一个循环屏障，参数1：需要累加的值，参数2 需要执行的方法
         */
        CyclicBarrier cyclicBarrier = new CyclicBarrier(7, () -> {
            System.out.println("召唤神龙");
        });

        for (int i = 1; i <= 7; i++) {
            final Integer tempInt = i;
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "\t 收集到 第" + tempInt + "颗龙珠");

                try {
                    // 先到的被阻塞，等全部线程完成后，才能执行方法
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }, String.valueOf(i)).start();
        }
    }
}
```

输出结果：

```sh
2	 收集到 第2颗龙珠
6	 收集到 第6颗龙珠
1	 收集到 第1颗龙珠
7	 收集到 第7颗龙珠
5	 收集到 第5颗龙珠
4	 收集到 第4颗龙珠
3	 收集到 第3颗龙珠
召唤神龙
```

### **3、Semaphore**

信号量主要用于两个目的，一个是用于多个共享资源的互斥使用，另一个用于并发线程数的控制。

正常的锁（concurrency.locks 或 synchronized 锁）在任何时刻都**只允许一个任务访问一项资源**，而 Semaphore 允许 **n 个任务**同时访问这个资源。

模拟一个抢车位的场景，假设一共有6个车，3个停车位

```java
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class SemaphoreDemo {
    public static void main(String[] args) {

        /**
         * 初始化一个信号量为3，默认是false 非公平锁， 模拟3个停车位
         */
        Semaphore semaphore = new Semaphore(3, false);

        // 模拟6部车
        for (int i = 0; i < 6; i++) {
            new Thread(() -> {
                try {
                    // 代表一辆车，已经占用了该车位
                    semaphore.acquire(); // 抢占

                    System.out.println(Thread.currentThread().getName() + "\t 抢到车位");

                    // 每个车停3秒
                    try {
                        TimeUnit.SECONDS.sleep(3);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    System.out.println(Thread.currentThread().getName() + "\t 离开车位");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    // 释放停车位
                    semaphore.release();
                }
            }, String.valueOf(i)).start();
        }
    }
}
```

输出结果：

```sh
1	 抢到车位
2	 抢到车位
0	 抢到车位
0	 离开车位
2	 离开车位
1	 离开车位
5	 抢到车位
4	 抢到车位
3	 抢到车位
5	 离开车位
4	 离开车位
3	 离开车位
```

## 7. 堵塞队列你知道吗？

**1、阻塞队列有哪些**

- ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按 FIFO（先进先出）对元素进行排序。
- LinkedBlokcingQueue：是一个基于链表结构的阻塞队列，此队列按 FIFO（先进先出）对元素进行排序，吞吐量通常要高于 ArrayBlockingQueue。
- SynchronousQueue：是一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于 LinkedBlokcingQueue。

**2、什么是阻塞队列**

![](https://s2.loli.net/2022/10/14/vh5mRKaIn18Z2MW.jpg)

- 阻塞队列，顾名思义，首先它是一个队列，而一个阻塞队列在数据结构中所起的作用大致如图所示：

- 当阻塞队列是空时，从队列中获取元素的操作将会被阻塞。

- 当阻塞队列是满时，往队列里添加元素的操作将会被阻塞。

- 核心方法

  | 方法\行为 | 抛异常    | 特定的值          | 阻塞   | 超时                        |
  | --------- | --------- | ----------------- | ------ | --------------------------- |
  | 插入方法  | add(o)    | offer(o)          | put(o) | offer(o, timeout, timeunit) |
  | 移除方法  |           | poll()、remove(o) | take() | poll(timeout, timeunit)     |
  | 检查方法  | element() | peek()            |        |                             |

- 行为解释：

  - 抛异常：如果操作不能马上进行，则抛出异常。
  - 特定的值：如果操作不能马上进行，将会返回一个特殊的值，一般是 true 或者 false。
  - 阻塞：如果操作不能马上进行，操作会被阻塞。
  - 超时：如果操作不能马上进行，操作会被阻塞指定的时间，如果指定时间没执行，则返回一个特殊值，一般是 true 或者 false。

- 插入方法：

  - add(E e)：添加成功返回 true，失败抛 IllegalStateException 异常。
  - offer(E e)：成功返回 true，如果此队列已满，则返回 false。
  - put(E e)：将元素插入此队列的尾部，如果该队列已满，则一直阻塞。

- 删除方法：

  - remove(Object o)：移除指定元素,成功返回true，失败返回 false。
  - poll()：获取并移除此队列的头元素，若队列为空，则返回 null。
  - take()：获取并移除此队列头元素，若没有元素则一直阻塞。

- 检查方法：

  - element()：获取但不移除此队列的头元素，没有元素则抛异常。
  - peek()：获取但不移除此队列的头；若队列为空，则返回 null。

**3、SynchronousQueue**

SynchronousQueue，实际上它不是一个真正的队列，因为它不会为队列中元素维护存储空间。与其他队列不同的是，它维护一组线程，这些线程在等待着把元素加入或移出队列。

```java
public class SynchronousQueueDemo {
    public static void main(String[] args) {
        SynchronousQueue<Integer> synchronousQueue = new SynchronousQueue<>();
        new Thread(() -> {
            try {
                synchronousQueue.put(1);
                Thread.sleep(3000);
                synchronousQueue.put(2);
                Thread.sleep(3000);
                synchronousQueue.put(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        new Thread(() -> {
            try {
                Integer val = synchronousQueue.take();
                System.out.println(val);
                Integer val2 = synchronousQueue.take();
                System.out.println(val2);
                Integer val3 = synchronousQueue.take();
                System.out.println(val3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

**4、使用场景**

- 生产者消费者模式
- 线程池
- 消息中间件
