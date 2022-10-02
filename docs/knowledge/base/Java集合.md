## 接口继承关系和实现

集合类存放于 Java.util 包中，主要有 3 种：set(集）、list(列表包含 Queue）和 map(映射)。

1. Collection：Collection 是集合 List、Set、Queue 的最基本的接口。
2. Iterator：迭代器，可以通过迭代器遍历集合中的数据
3. Map：是映射表的基础接口

## List

Java 的 List 是非常常用的数据类型。List 是有序的 Collection。Java List 一共三个实现类：
分别是 ArrayList、Vector 和 LinkedList。

### ArrayList（数组）

ArrayList 是最常用的 List 实现类，内部是通过数组实现的，它允许对元素进行快速随机访问。数
组的缺点是每个元素之间不能有间隔，当数组大小不满足时需要增加存储能力，就要将已经有数
组的数据复制到新的存储空间中。当从 ArrayList 的中间位置插入或者删除元素时，需要对数组进
行复制、移动、代价比较高。因此，它适合随机查找和遍历，不适合插入和删除。

### Vector（数组实现、线程同步）

Vector 与 ArrayList 一样，也是通过数组实现的，不同的是它支持线程的同步，即某一时刻只有一
个线程能够写 Vector，避免多线程同时写而引起的不一致性，但实现同步需要很高的花费，因此，
访问它比访问 ArrayList 慢。

### LinkList（链表）

LinkedList 是用链表结构存储数据的，很适合数据的动态插入和删除，随机访问和遍历速度比较
慢。另外，他还提供了 List 接口中没有定义的方法，专门用于操作表头和表尾元素，可以当作堆
栈、队列和双向队列使用。

## Set

Set 注重独一无二的性质,该体系集合用于存储无序(存入和取出的顺序不一定相同)元素，值不能重
复。对象的相等性本质是对象 hashCode 值（java 是依据对象的内存地址计算出的此序号）判断
的，如果想要让两个不同的对象视为相等的，就必须覆盖 Object 的 hashCode 方法和 equals 方
法。

### HashSet（Hash 表）

哈希表边存放的是哈希值。HashSet 存储元素的顺序并不是按照存入时的顺序（和 List 显然不
同） 而是按照哈希值来存的所以取数据也是按照哈希值取得。元素的哈希值是通过元素的
hashcode 方法来获取的, HashSet 首先判断两个元素的哈希值，如果哈希值一样，接着会比较
equals 方法 如果 equls 结果为 true ，HashSet 就视为同一个元素。如果 equals 为 false 就不是
同一个元素。

哈希值相同 equals 为 false 的元素是怎么存储呢,就是在同样的哈希值下顺延（可以认为哈希值相
同的元素放在一个哈希桶中）。也就是哈希一样的存一列。如图 1 表示 hashCode 值不相同的情
况；图 2 表示 hashCode 值相同，但 equals 不相同的情况。

## Map

