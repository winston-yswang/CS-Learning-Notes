## 1、容器概述

Java容器，也叫作集合，主要由两大接口派生而来：一个是 `Collection` 接口，主要用于存放单一元素；另一个是 `Map` 接口，主要用于存放键值对。其中 `Collection` 接口，又派生了三个常见的子接口：`List` 、`Set` 和 `Queue`。

Java 容器框架如下图所示：

![java-collection-hierarchy](pics\java-collection-hierarchy.png)

下面粗略介绍一下容器底层数据结构：

首先介绍 `Collection` 接口下的容器：

（1）List

- `ArrayList`：`Object[]` 数组
- `Vector`：`Object[]` 数组
- `LinkedList`：双向链表

（2）Set

- `HashSet`（无序，唯一）：基于`HashMap` 实现的，底层采用 `HashMap` 来保存元素
- `LinkedHashSet`：`LinkedHashSet` 是 `HashSet` 的子类，并且其内部是通过 `LinkedHashMap` 来实现的。
- `TreeSet`(有序，唯一)：红黑树

（3）Queue

- `PriorityQueue`：`Object[]` 数组来实现二叉堆
- `ArrayQueue`：`Object[]` 数组 + 双指针

接下是 `Map` 接口下的容器：

- `HashMap`： JDK1.8 之前 `HashMap` 由数组+链表组成的，数组是 `HashMap` 的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）。JDK1.8 以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间

- `LinkedHashMap`： `LinkedHashMap` 继承自 `HashMap`，所以它的底层仍然是基于拉链式散列结构即由数组和链表或红黑树组成。另外，`LinkedHashMap` 在上面结构的基础上，增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。同时通过对链表进行相应的操作，实现了访问顺序相关逻辑。

- `Hashtable`： 数组+链表组成的，数组是 `Hashtable` 的主体，链表则是主要为了解决哈希冲突而存在的
- `TreeMap`： 红黑树（自平衡的排序二叉树）



## 2、Collection

### 2.1 List 接口

上文讲到 `List` 是 `Collection` 的子接口，那么它的三个实现类之间有什么区别呢？

首先来讲讲 `ArrayList` 和 `Vector` 两者都是 `List` 的实现类，且底层使用 `Object[]` 存储，区别在于 `ArrayList` 查找高效却线程不安全，而 `Vector` 支持线程的同步故是线程安全，当然效率相对较低。

接着比较一下 `ArrayList` 和 `LinkedList` ，主要从以下方面来看：

1.  是否保证线程安全：两者都是线程不同步的，也即是不保证线程安全；
2. 底层数据结构：`ArrayList` 底层适用 `Object[]` 数组，而 `LinkedList` 则是双向链表；
3. 查找、插入和删除是否受元素位置的影响：`ArrayList` 支持随机元素访问，但是`LinkedList` 的首尾插入和删除高效。
4. 内存空间占用：`ArrayList` 的空间浪费主要体现在在 list 列表的结尾会预留一定的容量空间，而 LinkedList 的空间花费则体现在它的每一个元素需要保存索消耗空间。

可见两者的区别主要是取绝于底层数据结构的特性。

#### ArrayList 扩容机制

先从 `ArrayList` 的构造函数谈起，分有参和无参两种情况。有参构造情况下，会直接创建对应大小的数组空间；无参构造时，实际上初始化赋值的是一个空数组。当真正对数组进行添加元素操作时，才真正分配容量。即向数组中添加第一个元素时，数组容量扩为 10。

随着数据量增加，数组当前容量小于所需的容量时，则会调用 `ArrayList` 扩容的核心方法 `grow()` ，它会计算新容量为数组原本容量的1.5倍左右，并且做两重检查：一是新容量是否大于 `minCapacity` （最小需要容量），若还是小于 `minCapacity` ，那么就把 `minCapacity` 当作数组的新容量；二是 `minCapacity` 是否大于 `MAX_ARRAY_SIZE`（可分配的最大容量），若是则新容量则为`Integer.MAX_VALUE`，否则新容量大小则为 `MAX_ARRAY_SIZE`。在确定数组新容量后，则调用 `Arrays.copyof()` 申请一个新的数组，将数据拷贝。

此外，ArrayList 添加大量元素之前，用户也可以主动使用`ensureCapacity` 方法，以减少增量重新分配的次数。

```java
    /**
     * 要分配的最大数组大小
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * ArrayList扩容的核心方法。
     */
    private void grow(int minCapacity) {
        // oldCapacity为旧容量，newCapacity为新容量
        int oldCapacity = elementData.length;
        //将oldCapacity 右移一位，其效果相当于oldCapacity /2，
        //我们知道位运算的速度远远快于整除运算，整句运算式的结果就是将新容量更新为旧容量的1.5倍，
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //然后检查新容量是否大于最小需要容量，若还是小于最小需要容量，那么就把最小需要容量当作数组的新容量，
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
       // 如果新容量大于 MAX_ARRAY_SIZE,进入(执行) `hugeCapacity()` 方法来比较 minCapacity 和 MAX_ARRAY_SIZE，
       //如果minCapacity大于最大容量，则新容量则为`Integer.MAX_VALUE`，否则，新容量大小则为 MAX_ARRAY_SIZE 即为 `Integer.MAX_VALUE - 8`。
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

```



### 2.2 Set 接口

Set 具有无序性和不可重复性。无序性不等于随机性 ，无序性是指存储的数据在底层数组中并非按照数组索引的顺序添加 ，而是根据数据的哈希值决定的。不可重复性是指添加的元素按照 equals()判断时 ，返回 false，需要同时重写 equals()方法和 HashCode()方法。

`HashSet` 是 `Set` 接口的主要实现类 ，`HashSet` 的底层是 `HashMap`，线程不安全的，可以存储 null 值；

`LinkedHashSet` 是 `HashSet` 的子类，能够按照添加的顺序遍历；

`TreeSet` 底层使用红黑树，元素是有序的，排序的方式有自然排序和定制排序。



### 2.3 Queue 接口

`Queue` 是单端队列，只能从一端插入元素，另一端删除元素，实现上一般遵循 **先进先出（FIFO）** 规则。

`Queue` 扩展了 `Collection` 的接口，根据 **因为容量问题而导致操作失败后处理方式的不同** 可以分为两类方法: 一种在操作失败后会抛出异常，另一种则会返回特殊值。

| `Queue` 接口 | 抛出异常  | 返回特殊值 |
| ------------ | --------- | ---------- |
| 插入队尾     | add(E e)  | offer(E e) |
| 删除队首     | remove()  | poll()     |
| 查询队首元素 | element() | peek()     |

`Deque` 是双端队列，在队列的两端均可以插入或删除元素。

`Deque` 扩展了 `Queue` 的接口, 增加了在队首和队尾进行插入和删除的方法，同样根据失败后处理方式的不同分为两类：

| `Deque` 接口 | 抛出异常      | 返回特殊值      |
| ------------ | ------------- | --------------- |
| 插入队首     | addFirst(E e) | offerFirst(E e) |
| 插入队尾     | addLast(E e)  | offerLast(E e)  |
| 删除队首     | removeFirst() | pollFirst()     |
| 删除队尾     | removeLast()  | pollLast()      |
| 查询队首元素 | getFirst()    | peekFirst()     |
| 查询队尾元素 | getLast()     | peekLast()      |

事实上，`Deque` 还提供有 `push()` 和 `pop()` 等其他方法，可用于模拟栈。

#### ArrayDeque 与 LinkedList 的区别

`ArrayDeque` 和 `LinkedList` 都实现了 `Deque` 接口，两者都具有队列的功能，但两者有什么区别呢？

- `ArrayDeque` 是基于可变长的数组和双指针来实现，而 `LinkedList` 则通过链表来实现。
- `ArrayDeque` 不支持存储 `NULL` 数据，但 `LinkedList` 支持。
- `ArrayDeque` 插入时可能存在扩容过程, 不过均摊后的插入操作依然为 O(1)。虽然 `LinkedList` 不需要扩容，但是每次插入数据时均需要申请新的堆空间，均摊性能相比更慢。

从性能的角度上，选用 `ArrayDeque` 来实现队列要比 `LinkedList` 更好。此外，`ArrayDeque` 也可以用于实现栈。

#### PriorityQueue

`PriorityQueue` 是在 JDK1.5 中被引入的, 其与 `Queue` 的区别在于元素出队顺序是与优先级相关的，即总是优先级最高的元素先出队。

这里列举其相关的一些要点：

- `PriorityQueue` 利用了二叉堆的数据结构来实现的，底层使用可变长的数组来存储数据
- `PriorityQueue` 通过堆元素的上浮和下沉，实现了在 O(logn) 的时间复杂度内插入元素和删除堆顶元素。
- `PriorityQueue` 是非线程安全的，且不支持存储 `NULL` 和 `non-comparable` 的对象。
- `PriorityQueue` 默认是小顶堆，但可以接收一个 `Comparator` 作为构造参数，从而来自定义元素优先级的先后。

`PriorityQueue` 在面试中可能更多的会出现在手撕算法的时候，典型例题包括堆排序、求第K大的数、带权图的遍历等，所以需要会熟练使用才行。



## 3、Map

### 3.1 HashMap

首先了解一下 **`HashMap` 和 `Hashtable` 的区别**：

- 线程是否安全： `HashMap` 是非线程安全的，`Hashtable` 是线程安全的,因为 `Hashtable` 内部的方法基本都经过`synchronized` 修饰。--> 顺带推出前者效率会比后者更高
- 初始容量大小和扩容机制不同
- 底层数据不同

再来了解一下 **`HashMap` 和 `TreeMap` 的区别**：

`TreeMap` 和`HashMap` 都继承自`AbstractMap` ，但是需要注意的是`TreeMap`它还实现了`NavigableMap`接口和`SortedMap` 接口。

![TreeMap继承结构](pics\TreeMap继承结构.png)

实现 `NavigableMap` 接口让 `TreeMap` 有了对集合内元素的搜索的能力。

实现`SortedMap`接口让 `TreeMap` 有了对集合中的元素根据键排序的能力。默认是按 key 的升序排序，不过我们也可以指定排序的比较器。

**综上，相比于`HashMap`来说 `TreeMap` 主要多了对集合中的元素根据键排序的能力以及对集合内元素的搜索的能力。**

### 3.2 HashMap 检查重复

当你把对象加入`HashSet`时，`HashSet` 会先计算对象的`hashcode`值来判断对象加入的位置，同时也会与其他加入的对象的 `hashcode` 值作比较，如果没有相符的 `hashcode`，`HashSet` 会假设对象没有重复出现。但是如果发现有相同 `hashcode` 值的对象，这时会调用`equals()`方法来检查 `hashcode` 相等的对象是否真的相同。如果两者相同，`HashSet` 就不会让加入操作成功。

### 3.3 HashMap 的底层实现

JDK1.8 之前 `HashMap` 底层是 **数组和链表** 结合在一起使用也就是 **链表散列**。**HashMap 通过 key 的 hashCode 经过扰动函数处理过后得到 hash 值，然后通过 (n - 1) & hash 判断当前元素存放的位置（这里的 n 指的是数组的长度），如果当前位置存在元素的话，就判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。**

**所谓扰动函数指的就是 HashMap 的 hash 方法。使用 hash 方法也就是扰动函数是为了防止一些实现比较差的 hashCode() 方法 换句话说使用扰动函数之后可以减少碰撞。**

所谓 **“拉链法”** 就是：将链表和数组相结合。也就是说创建一个链表数组，数组中每一格就是一个链表。若遇到哈希冲突，则将冲突的值加到链表中即可。

![jdk1.8之前的内部结构-HashMap](pics\jdk1.8之前的内部结构-HashMap.png)

 **JDK1.8 之后**在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间。

![jdk1.8之后的内部结构-HashMap](pics\jdk1.8之后的内部结构-HashMap.png) 

部分源码：

```java
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;
    final int hash;
    ……
}

public V put(K key, V value) {
        //其允许存放null的key和null的value，当其key为null时，调用putForNullKey方法，放入到table[0]的这个位置
        if (key == null)
            return putForNullKey(value);
        //通过调用hash方法对key进行哈希，得到哈希之后的数值。该方法实现可以通过看源码，其目的是为了尽可能的让键值对可以分不到不同的桶中
        int hash = hash(key);
        //根据上一步骤中求出的hash得到在数组中是索引i
        int i = indexFor(hash, table.length);
        //如果i处的Entry不为null，则通过其next指针不断遍历e元素的下一个元素。
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
}

public V get(Object key) {
    if (key == null)
        return getForNullKey();
    Entry<K,V> entry = getEntry(key);
    return null == entry ? null : entry.getValue();
}
```

### 3.4 HashMap 的扩容

当 HashMap 中的元素越来越多的时候，hash 冲突的几率也就越来越高，因为数组的长度是固定的。所以为了提高查询的效率，就要对 HashMap 的数组进行扩容，而在 HashMap 数组扩容之后，最消耗性能的点就出现了：原数组中的数据必须重新计算其在新数组中的位置，并放进去，这就是 resize（rehash）。

那么 HashMap 什么时候进行扩容呢？**当 HashMap 中的元素个数超过数组大小 loadFactor时，就会进行数组扩容，loadFactor的默认值为 0.75**，这是一个折中的取值。也就是说，默认情况下，数组大小为 16，那么当 HashMap 中元素个数超过 16 * 0.75=12 的时候，就把数组的大小扩展为 2*16=32，即扩大一倍，然后重新计算每个元素在数组中的位置，而这是一个非常消耗性能的操作，所以如果我们已经预知 HashMap 中元素的个数，那么预设元素的个数能够有效的提高 HashMap 的性能。

**负载因子 loadFactor 衡量的是一个散列表的空间的使用程度，负载因子越大表示散列表的装填程度越高，反之愈小。**结合负载因子的定义公式可知，threshold 就是在此 loadFactor 和 capacity 对应下允许的最大元素数目，超过这个数目就重新 resize，以降低实际的负载因子。默认的的负载因子 0.75 是对空间和时间效率的一个平衡选择。当容量超出此最大容量时， resize 后的 HashMap 容量是容量的两倍。[#参考文章](https://www.html.cn/qa/other/20583.html)

### 3.5 HashMap 的长度为何是 2 的幂次方

为了能让 `HashMap` 存取高效，尽量较少碰撞，也就是要尽量把数据分配均匀。hash值得范围通常远大于数组的容量，所以需要将Hash值再次映射到数组的具体位置。

常见的做法的取模运算，得到的余数则是要存放的位置也就是对应的数组下标。

计算机中直接取模运算效率不如位移运算，源码中做了优化，即`hash & (length - 1)` 。当容量是 $2^{n}$ 时，$2^{n} - 1$ 与 hash值进行按位与运算，相当于取hash值中低n位的值作为对应的数组下标。

### 3.6 HashMap 为何是线程不安全

HashMap的线程安全问题大部分出在put 和 扩容(rehash)的过程中。在JDK1.7中，当并发执行扩容操作时会造成环形链和数据丢失的情况。在JDK1.8中，在并发执行put操作时会发生数据覆盖的情况。[#参考文章](https://blog.csdn.net/swpu_ocean/article/details/88917958)



## 4、ConcurrentHashMap

### 4.1 ConcurrentHashMap 和 Hashtable 的区别

`ConcurrentHashMap` 和 `Hashtable` 的区别主要体现在实现线程安全的方式上不同。

- **底层数据结构**： JDK1.7 的 `ConcurrentHashMap` 底层采用 **分段的数组+链表** 实现，JDK1.8 采用的数据结构跟 `HashMap1.8` 的结构一样，是数组+链表/红黑二叉树。而 `Hashtable` 数组+链表组成的，数组是主体，链表则是主要为了解决哈希冲突而存在的。
- **实现线程安全的方式**：`ConcurrentHashMap` 在JDK1.7时采用分段锁实现并发安全，在JDK1.8时并发控制使用 `synchronized` 和CAS来操作。而 `Hashtable` 是利用 `synchronized` 来实现一把全表锁保证线程安全，效率比较低下。

### 4.2 ConcurrentHashMap 底层实现

#### JDK 1.7

![分段锁](pics\分段锁.jpg)

首先将数据分为一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据时，其他段的数据也能被其他线程访问。

**`ConcurrentHashMap` 是由 `Segment` 数组结构和 `HashEntry` 数组结构组成**。

Segment 实现了 `ReentrantLock`，所以 `Segment` 是一种可重入锁，扮演锁的角色。`HashEntry` 用于存储键值对数据。

```java
static class Segment<K,V> extends ReentrantLock implements Serializable {
    
    // The segments, each of which is a specialized hash table
    final Segment<K,V>[] segments;
    
    // Segment是ConcurrentHashMap的静态内部类
    static final class Segment<K,V> extends ReentrantLock implements Serializable {
    	// ... 省略 ...
    	/**
         * The per-segment table. Elements are accessed via
         * entryAt/setEntryAt providing volatile semantics.
         */
        transient volatile HashEntry<K,V>[] table;
        // ... 省略 ...
    }
    
    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
        // ... 省略 ...
    }
    
}
```

一个 `ConcurrentHashMap` 里包含一个 `Segment` 数组。`Segment` 的结构和 `HashMap` 类似，是一种数组和链表结构，一个 `Segment` 包含一个 `HashEntry` 数组，每个 `HashEntry` 是一个链表结构的元素，每个 `Segment` 守护着一个 `HashEntry` 数组里的元素，当对 `HashEntry` 数组的数据进行修改时，必须首先获得对应的 `Segment` 的锁。

#### JDK 1.8

![java8_concurrenthashmap](pics\java8_concurrenthashmap.png)

`ConcurrentHashMap` 的数据结构是数组+链表/红黑二叉树。Java 8 在链表长度超过一定阈值（8）时将链表（寻址时间复杂度为 O(N)）转换为红黑树（寻址时间复杂度为 O(log(N))）。

它取消了 `Segment` 分段锁，采用 CAS 和 `synchronized` 来保证并发安全。

例如在并发执行put操作，先是计算得到 key 的 hash值，接着进入 hash表自旋。在循环中判断：若表为空则进行数组初始化；若数组中插入位置为null的时候，使用CAS来把这个新的Node写入数组中对应的位置，若不为空则说明该位置存在冲突，则需要对数组该位置的头节点使用 `synchronized` 加锁才能进行进入链表或者红黑树插入操作。由于`synchronized` 只锁定当前链表或红黑二叉树的首节点，这样只要 hash 不冲突，就不会产生并发，效率又提升 N 倍。

而扩容操作则相对复杂：首先使用一个Stride（步长）将数据迁移任务划分成 n 个小任务，每个线程每次只负责一部分数据。在数据迁移的过程中，数组的put操作会被阻塞分去帮忙，直到扩容完成。[#参考文章](https://sihai.blog.csdn.net/article/details/79383766?spm=1001.2101.3001.6650.5&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-5.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-5.no_search_link&utm_relevant_index=9)

部分重要源码：

```java

// 业务中使用的hash表
transient volatile Node<K,V>[] table;

// 扩容时才使用的hash表，扩容完成后赋值给table，并将nextTable重置为null
private transient volatile Node<K,V>[] nextTable;

// put操作
public V put(K key, V value) {
    return putVal(key, value, false);
}
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 略
}

// 核心方法transfer,将每个Node移动或复制到新表中
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    // 略
}
```







