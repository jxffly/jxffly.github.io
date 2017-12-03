---
layout: post
title:  "ConcurrentHashMap的解读"
date: 2017-11-08 22:21:49
tags: java 并发
---
### ConCurrentHashMap和HashMap对比：
1.hashMap：线程不安全，在高并发下，进行resize的时候会产生，节点的循环引用，导致死循环，hashMap完全废掉
2.HashTable：容器使用synchronized来保证线程安全，但在线程竞争激烈的情况下HashTable的效率非常低下。因为当一个线程访问HashTable的同步方法时，其他线程访问HashTable的同步方法时，可能会进入阻塞或轮询状态。如线程1使用put进行添加元素，线程2不但不能使用put方法添加元素，并且也不能使用get方法来获取元素，所以竞争越激烈效率越低

## java1.7和1.8的对比

#### 锁分段技术(jdk1.7)

**<u>HashTable</u>**  容器在竞争激烈的并发环境下表现出效率低下的原因是:所有访问HashTable的线程都必须竞争同一把锁，那假如容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效的提高并发访问效率，这就是<u>**ConcurrentHashMap**</u>所使用的锁分段技术，首先将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。

##### **<u>实现的机理</u>（分段锁机制，特别适合读多写少的场景）**

ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。Segment是一种可重入锁ReentrantLock，在ConcurrentHashMap里扮演锁的角色，HashEntry则用于存储键值对数据。一个ConcurrentHashMap里包含一个Segment数组，Segment的结构和HashMap类似，是一种数组和链表结构， 一个Segment里包含一个HashEntry数组，每个HashEntry是一个链表结构的元素， 每个Segment守护者一个HashEntry数组里的元素,当对HashEntry数组的数据进行修改时，必须首先获得它对应的Segment锁。

![segment](/image/seg.png)

#### java1.8的ConcurrentHashMap

- **改进一**：取消segments字段，直接采用transient volatile HashEntry<K,V>[] table保存数据，采用table数组元素作为锁，从而实现了对每一行数据进行加锁，进一步减少并发冲突的概率。



- **改进二**：(类似hashMap)将原先table数组＋单向链表的数据结构，变更为table数组＋单向链表＋红黑树的结构。对于hash表来说，最核心的能力在于将key hash之后能均匀的分布在数组中。如果hash之后散列的很均匀，那么table数组中的每个队列长度主要为0或者1。但实际情况并非总是如此理想，虽然ConcurrentHashMap类默认的加载因子为0.75，但是在数据量过大或者运气不佳的情况下，还是会存在一些队列长度过长的情况，如果还是采用单向列表方式，那么查询某个节点的时间复杂度为O(n)；因此，对于个数超过8(默认值)的列表，jdk1.8中采用了红黑树的结构，那么查询的时间复杂度可以降低到O(logN)，可以改进性能。



## java1.8的concurrentHashMap的解读

#### 1.<u>初始化的参数</u>

```java
/** * races. Updated via CAS. * 记录容器的容量大小，通过CAS更新 */

private transient volatile long baseCount;

 /** * 这个sizeCtl是volatile的，那么他是线程可见的，一个思考:它是所有修改都在CAS中进行，但是sizeCtl为什么不设计成LongAdder(jdk8出现的)类型呢？ * 或者设计成AtomicLong(在高并发的情况下比LongAdder低效)，这样就能减少自己操作CAS了。 * * 来看下注释，当sizeCtl小于0说明有多个线程正则等待扩容结果，参考transfer函数 * * sizeCtl等于0是默认值，大于0是扩容的阀值 */

private transient volatile int sizeCtl;

 /** * 自旋锁 （锁定通过 CAS） 在调整大小和/或创建 CounterCells 时使用。 在CounterCell类更新value中会使用，功能类似显示锁和内置锁，性能更好 * 在Striped64类也有应用 */

private transient volatile int cellsBusy;

```



sizeCtl 是控制标识符，不同的值表示不同的意义。
* 负数代表正在进行初始化或扩容操作 ,其中-1代表正在初始化 ,-N 表示有N-1个线程正在进行扩容操作
* 正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小，类似于扩容阈值。它的值始终是当前ConcurrentHashMap容量的0.75倍，这与loadfactor是对应的。实际容量>=sizeCtl，则扩容。

#### 2.初始化方法（类似加强版的hashmap）

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
//不使用锁来同步，使用cas不断的尝试，如果已经初始化则吃线程进行挂起操作
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                //下一次扩容的阈值为上一次size的0.75
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

sizeCtl默认为0，如果ConcurrentHashMap实例化时有传参数，sizeCtl会是一个2的幂次方的值。所以执行第一次put操作的线程会执行Unsafe.compareAndSwapInt方法修改sizeCtl为-1，有且只有一个线程能够修改成功，其它线程通过Thread.yield()让出CPU时间片等待table初始化完成

#### 3.put方法

```java
1. final V putVal(K key, V value, boolean onlyIfAbsent) {  
2.     if (key == null || value == null) throw new NullPointerException();  
3.     int hash = spread(key.hashCode());//自带的hash算法对key的hash值重新hash  
4.     int binCount = 0;  
5. //这边加了一个循环，就是不断的尝试，因为在table的初始化和casTabAt用到了compareAndSwapInt、compareAndSwapObject //因为如果其他线程正在修改tab，那么尝试就会失败，所以这边要加一个for循环，不断的尝试
6.     for (Node<K,V>[] tab = table;;) {  
7.         Node<K,V> f; int n, i, fh;  
8.         // 如果table为空，初始化；否则，根据hash值计算得到数组索引i，如果tab[i]为空，直接新建节点Node即可。注：tab[i]实质为链表或者红黑树的首节点。  
9.         if (tab == null || (n = tab.length) == 0)  
10.             tab = initTable();  
11.         else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {  
12.             if (casTabAt(tab, i, null,  
13.                          new Node<K,V>(hash, key, value, null)))  
14.                 break;                   // no lock when adding to empty bin  
15.         }  
16.         // 如果tab[i]不为空并且hash值为MOVED，说明该链表正在进行transfer操作，返回扩容完成后的table,一起加入扩容  
17.         else if ((fh = f.hash) == MOVED)  
18.             tab = helpTransfer(tab, f);  
19.         else {  
20.             V oldVal = null;  
21.             // 针对首个节点进行加锁操作，而不是segment，进一步减少线程冲突  
22.             synchronized (f) {  
23.                 if (tabAt(tab, i) == f) {  // double check
24.                     if (fh >= 0) {  
25.                         binCount = 1;  
26.                         for (Node<K,V> e = f;; ++binCount) {  
27.                             K ek;  
28.                             // 如果在链表中找到值为key的节点e，直接设置e.val = value即可。  
29.                             if (e.hash == hash &&  
30.                                 ((ek = e.key) == key ||  
31.                                  (ek != null && key.equals(ek)))) {  
32.                                 oldVal = e.val;  
33.                                 if (!onlyIfAbsent)  
34.                                     e.val = value;  
35.                                 break;  
36.                             }  
37.                             // 如果没有找到值为key的节点，直接新建Node并加入链表即可。  
38.                             Node<K,V> pred = e;  
39.                             if ((e = e.next) == null) {  
40.                                 pred.next = new Node<K,V>(hash, key,  
41.                                                           value, null);  
42.                                 break;  
43.                             }  
44.                         }  
45.                     }  
46.                     // 如果首节点为TreeBin类型，说明为红黑树结构，执行putTreeVal操作。  
47.                     else if (f instanceof TreeBin) {  
48.                         Node<K,V> p;  
49.                         binCount = 2;  
50.                         if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,  
51.                                                        value)) != null) {  
52.                             oldVal = p.val;  
53.                             if (!onlyIfAbsent)  
54.                                 p.val = value;  
55.                         }  
56.                     }  
57.                 }  
58.             }  
59.             if (binCount != 0) {  
60.                 // 如果节点数>＝8，那么转换链表结构为红黑树结构。  
61.                 if (binCount >= TREEIFY_THRESHOLD)  
62.                     treeifyBin(tab, i);  
63.                 if (oldVal != null)  
64.                     return oldVal;  
65.                 break;  
66.             }  
67.         }  
68.     }  
69.     // 计数增加1，有可能触发transfer操作(扩容)。  
70.     addCount(1L, binCount);  
71.     return null;  
72. }  
```

如果f为null，说明table中这个位置第一次插入元素，利用Unsafe.compareAndSwapObject方法插入Node节点。
如果CAS成功，说明Node节点已经插入，随后addCount(1L, binCount)方法会检查当前容量是否需要进行扩容。
如果CAS失败，说明有其它线程提前插入了节点，自旋重新尝试在这个位置插入节点。
如果f的hash值为-1，说明当前f是ForwardingNode节点，意味有其它线程正在扩容，则一起进行扩容操作。
其余情况把新的Node节点按链表或红黑树的方式插入到合适的位置，这个过程采用同步内置锁实现并发，代码如下:

```java
synchronized (f) {
 if (tabAt(tab, i) == f) {
     if (fh >= 0) {
         binCount = 1;
         for (Node<K,V> e = f;; ++binCount) {
             K ek;
             if (e.hash == hash &&
                 ((ek = e.key) == key ||
                  (ek != null && key.equals(ek)))) {
                 oldVal = e.val;
                 if (!onlyIfAbsent)
                     e.val = value;
                 break;
             }
             Node<K,V> pred = e;
             if ((e = e.next) == null) {
                 pred.next = new Node<K,V>(hash, key,
                                           value, null);
                 break;
             }
         }
     }
     else if (f instanceof TreeBin) {
         Node<K,V> p;
         binCount = 2;
         if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                        value)) != null) {
             oldVal = p.val;
             if (!onlyIfAbsent)
                 p.val = value;
         }
     }
 }
}
```

##### 总体思路

**<u>putVal</u>**(K key, V value, boolean onlyIfAbsent)方法干的工作如下：

1. 检查key/value是否为空，如果为空，则抛异常，否则进行2
2. 进入for死循环，进行3
3. 检查table是否初始化了，如果没有，则调用initTable()进行初始化然后进行 2，否则进行4
4. 根据key的hash值计算出其应该在table中储存的位置i，取出table[i]的节点用f表示。

> 根据f的不同有如下三种情况：
> 1）如果table[i]==null(即该位置的节点为空，没有发生碰撞)， 则利用CAS操作直接存储在该位置，如果CAS操作成功则退出死循环。
>
> 2）如果table[i]!=null(即该位置已经有其它节点，发生碰撞)，碰撞处理也有两种情况
> ​        2.1）检查table[i]的节点的hash是否等于MOVED，如果等于，则检测到正在扩容，则帮助其扩容
> ​        2.2）说明table[i]的节点的hash值不等于MOVED，如果table[i]为链表节点，则将此节点插入链表中即可
> 3）如果table[i]为树节点，则将此节点插入树中即可。插入成功后，进行 5

5. 如果table[i]的节点是链表节点，则检查table的第i个位置的链表是否需要转化为数，如果需要则调用treeifyBin函数进行转化


#### 4.get数据

* Doug Lea采用Unsafe.getObjectVolatile来获取，也许有人质疑，直接table[index]不可以么，为什么要这么复杂？
  在java内存模型中，我们已经知道每个线程都有一个工作内存，里面存储着table的副本，虽然table是volatile修饰的，但不能保证线程每次都拿到table中的最新元素，Unsafe.getObjectVolatile可以直接获取指定内存的数据，保证了每次拿到数据都是最新的。
* 其余情况把新的Node节点按链表或红黑树的方式插入到合适的位置，这个过程采用同步内置锁实现并发，代码如下:
