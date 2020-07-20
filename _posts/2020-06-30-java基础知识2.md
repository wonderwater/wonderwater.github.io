---
layout: post
title:  "HashMap和ConcurrentHashMap源码分析"
date: 2020-06-30 21:00
tags:
    - java
---

基于java8，分析HashMap的实现和原理，进而分析ConcurrentHashMap的实现和原理。在读源码的过程中，对不解的地方，回溯jsr的commit记录，尽可能知根知底

## hashmap in java 8

### resize - initialize

```java
newCap = DEFAULT_INITIAL_CAPACITY; // 16
threshold = DEFAULT_INITIAL_CAPACITY * DEFAULT_INITIAL_CAPACITY; // (16*75%)
Node<K,V>[] table = (Node<K,V>[])new Node[newCap];
```

初始化的代码，HashMap主要存储结构：Node数组，Node用于表示key-value
数组的长度一定是2^n次方，有以下几个优点
1. 这对通过后续的hash值，求数组索引有效率的提升（否则一般需要用耗时的求余操作）
2. 配合resize技巧

数组的每一项Node，一般是链表结构；但在一定条件下会转换成红黑树结构，那么此时的Node对象实际是TreeNode：
1. table.length至少大于MIN_TREEIFY_CAPACITY（64）
2. 如果长度达到TREEIFY_THRESHOLD（8）

有转成红黑树，也有转会链表的情况：链表长度小于UNTREEIFY_THRESHOLD（6）


### hash

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
一个新的Node对象，要加入到table中，首先要确定加入数组的哪一项，然后在确定插入该项的链表或树

### put

1. 通过key的哈希结果值hash，table下标i=`(table.length - 1) & hash`
2. 如果table[i] == null, 直接生成Node放入
3. 如果table[i]是TreeNode实例，红黑树插入操作
4. 否则就是链表结构，遍历链表，插入或更新Node，并检查是否需要转换为红黑树
5. 最后，如果当前存储的Node数超过阈值threshold，则resize

#### 红黑树插入操作[code][1]
table[i]是rb-tree的root，rb-tree是平衡二叉树结构，进行搜索时，通过和当前节点比较，判断向左或向右遍历
1. 通过hash值比较
2. key的类实现了Comparable，进行比较
3. 通过System.identityHashCode，计算出key的值，进行比较



#### 转换为红黑树

1. 先将链表转成双向链表
2. 初始化红黑树，然后将每个节点取出，加入红黑树，平衡红黑树
3. 最后将红黑树的root节点，调整到双向链表头部（简单的链表操作，把root移除，然后插入头部），写回table[i]

这里的红黑树实现和《算法导论》是一致的，再平衡的大概过程：
首先插入节点x是红色的，位于叶子节点，这时不影响黑高度，但是xp也可能是红色的，就违反了规则，我们讨论xp==xpp.left的情况，另一个情况类似。
这时候有主要的三种情况需要处理：
1. x的叔节点是红色的，也就是xpp的节点都是红的，那么把xpp的子节点置黑，xpp置红，这时候，我们的焦点就是xpp，把xpp当做x，继续判断
2. x的叔节点是黑色的，并且x==xp.right，那么left_rotate(xp)，然后我们关注的焦点就是xp，把xp当做x，继续判断
3. x的叔节点是黑色的，并且x==xp.left，那么right_rotate(xpp)，xpp置红，然后我们关注的焦点就是xp，把xp当做x，继续判断

以上三种情况，最后x都是红的，那么当xp为黑时，或者x就是root（需要置黑），就可以结束了



## ConcurrentHashMap in java 8

### 初始化
和HashMap相似，构造函数不做数据结构的初始化，延迟数据结构的初始化，一般在put中

```java
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
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
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

通过cas操作`U.compareAndSwapInt`，设置成功，要再检查table是否已经初始化完成，才可以做进一步操作
cas失败，则出让CPU资源等待

### Node的类型
根据ConcurrentHashMap的注释，Node为一下几类：

1. TreeNode，用于平衡树中，而不是链表中。
2. TreeBins，保存TreeNodes的root。
3. ForwardingNodes用于resizing期间，放置于table[i]
4. ReservationNodes用于在computeIfAbsent和相关方法中建立值时，作占位符。

ForwardingNodes、TreeBin和ReservationNodes的hash值分别是-1，-2，-3。
他们并不是作为key-value保存的，而是用作调节控制的，它们的hash是负值（即最高位是1）

hash函数：
```java
    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS; // HASH_BITS == 0x7fffffff
    }
```

hash为负是做调整的Node，所以计算hash时，最高位置0

### put

```java
    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
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
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

1. table为空，则初始化
2. `(f=tabAt(tab, i = (n - 1) & hash)))==null`，尝试cas方式将当前的key-value的Node放入
    tabAt用的是Unsafe.getObjectVolatile，那为什么不用`table[i]==null`？因为虽然table是volatile，
    可以保证对table，其本身的读写可见性，但是其中元素需要可见性保证时，需要用Unsafe的方法。
    JUC库也提供类似的，比如AtomicReferenceArray，实现方式也是用的Unsafe.getObjectVolatile
3. 如果f.hash==MOVE，MOVE==-1，这就是控制节点了，MOVE代表f是ForwardingNodes类，则加入扩容
4. synchronized(f)，再进行操作，这里用的关键字进行同步，代码注释解释因为不想再引入锁字段，额外占用空间，所以直接用的节点对象进行同步
    这里一进入，需要double check，检查当前节点是否还是头部节点：tabAt(tab, i) == f
    如果fh>0，则是一个普通的Node，简单的链表操作，成功后需要判断是否变成红黑树
    如果f是TreeBin类，则说明已经变为红黑树，进行红黑树插入操作
5. 插入成功后，计数+1，并检查是否需要扩容，或者尝试加入正在进行的扩容

从上面的代码可以观察一下，函数的出口在哪？

我们看到`for (Node<K,V>[] tab = table;;)`，这是没有条件的，是死循环，要依靠`for`循环中的body，跳出循环，
所以要观察一下这个for循环的直接break或者return语句在哪，在1.table对应槽位（也就是链表头结点）的赋值，2.链表或红黑树的插入的后续处理，这两处
还发现这样的break执行后，紧接着执行`addCount(1L, binCount)`，所以它执行的条件就是break的条件，分别是1. 链表头结点的赋值的情况、2. 老值oldVal==null的情况。

addCount用于统计map中的节点数相当于HashMap中的size+1，
第二个参数名check，<0则不检查扩容，<=1检查扩容（且在+1操作没有出现竞争的情况下），
该putVal方法，binCount变量初始值0，始终不会小于0，所以属于<=1的情况。

另外，那么在扩容时，即f.hash==MOVED，是不做插入的，
当节点是链表头结点或红黑树的TreeBin才可能做插入。
这也因此减小了实现的复杂度。

### get

get方式几乎是无锁的，也就是说，即使链表正在插入、正在扩容，也是不影响的。要做并发控制的是红黑树的遍历。

代码非常简单：

```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

总结以下几个步骤：
1. 如果头结点的hash值对得上，直接检查是否头结点，是则返回
2. 再检查是否控制节点（即hash<0），由实现类进行查找
3. 链表遍历，查找元素

再并发条件下：
- 假如链表插入新节点
    联系一下put的相关代码，对链表的插入操作，或者Node的更新操作
    
```java
// 值存在更新Node的值  
e.val = value;
// 不存在，插入Node
pred.next = new Node<K,V>(hash, key, value, null);
```
我们看到直接赋值的操作，而且为了保证get方法能《可见》最新的操作，所以val字段和next字段是volatile声明的

- 假如链表结构转为红黑树
首先看转换的时候，做了哪些事

```java
    /**
     * Replaces all linked nodes in bin at given index unless table is
     * too small, in which case resizes instead.
     */
    private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```
首先table长度不够时，先做扩容。然后才是做转换，转换的时候，构造生成TreeNode类型的链表，
最后通过TreeBin的构造函数，生成红黑树，最后赋值到table[index]。
get方法中，遍历链表的头结点e，一开始就赋值了，所以即使转成红黑树，get遍历时用的仍然是转换前的链表

我们注意到treeifyBin用了头结点，进行了同步，和put方法中是一样的，只有拿到头结点的同步，才能进行更新，并且进如同步后要double check。
为了防止更新的数据丢失：如果不同步头结点的话，那么TreeBin正在构造的期间，put成功的，并不会出现在TreeBin中，最后构造结束回写头结点，那么就会丢失期间的更新

- 假如链表结构转为红黑树，并向树插入了数据
和上一种情况一样的，我们遍历的是链表这个副本，不是红黑树，所以这个成功插入树的数据是访问不到的

- 假如正在进行扩容
正在扩容时，头结点的类是ForwardingNodes，hash为MOVE（-1），它的find方法：

```java
        Node<K,V> find(int h, Object k) {
            // loop to avoid arbitrarily deep recursion on forwarding nodes
            outer: for (Node<K,V>[] tab = nextTable;;) {
                Node<K,V> e; int n;
                if (k == null || tab == null || (n = tab.length) == 0 ||
                    (e = tabAt(tab, (n - 1) & h)) == null)
                    return null;
                for (;;) {
                    int eh; K ek;
                    if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    if (eh < 0) {
                        if (e instanceof ForwardingNode) {
                            tab = ((ForwardingNode<K,V>)e).nextTable;
                            continue outer;
                        }
                        else
                            return e.find(h, k);
                    }
                    if ((e = e.next) == null)
                        return null;
                }
            }
        }
```
其中的nextTable对应扩容后的table。这时候table对应槽位的数据已经搬到了nextTable
（因为搬完后，才会对table回写ForwardingNode对象，这里的先搬后写的操作，可以类比转换的操作，可以想到也要对头结点加锁）。
数据到了nextTable，那就对nextTable进行查找，和get方法是一样的

- 树结构的查找
我们刚才看到treeifyBin转换成树的头结点是TreeBin类，它的hash为TREEBIN（-2）

```java
        /**
         * Returns matching node or null if none. Tries to search
         * using tree comparisons from root, but continues linear
         * search when lock not available.
         */
        final Node<K,V> find(int h, Object k) {
            if (k != null) {
                for (Node<K,V> e = first; e != null; ) {
                    int s; K ek;
                    if (((s = lockState) & (WAITER|WRITER)) != 0) {
                        if (e.hash == h &&
                            ((ek = e.key) == k || (ek != null && k.equals(ek))))
                            return e;
                        e = e.next;
                    }
                    else if (U.compareAndSwapInt(this, LOCKSTATE, s,
                                                 s + READER)) {
                        TreeNode<K,V> r, p;
                        try {
                            p = ((r = root) == null ? null :
                                 r.findTreeNode(h, k, null));
                        } finally {
                            Thread w;
                            if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                                (READER|WAITER) && (w = waiter) != null)
                                LockSupport.unpark(w);
                        }
                        return p;
                    }
                }
            }
            return null;
        }
```
注意到，这里有锁的判断，简单的用lockState的位标记做读写锁。
lockState是int类型, 低两位分别是等待标记、写标记，高位是读标记

比如这个代表有3个线程取得了读锁

```txt
00..11    0     0

reader waiter writer
```

在有写/等待操作的前提下，通过双线链表的方式遍历，
否则尝试设置读标记，成功后进行树的查找，最后，当最后一个读操作退出时，unpark等待的线程

我们结合写锁获取的代码进行分析

```java
        /**
         * Possibly blocks awaiting root lock.
         */
        private final void contendedLock() {
            boolean waiting = false;
            for (int s;;) {
                if (((s = lockState) & ~WAITER) == 0) {
                    if (U.compareAndSwapInt(this, LOCKSTATE, s, WRITER)) {
                        if (waiting)
                            waiter = null;
                        return;
                    }
                }
                else if ((s & WAITER) == 0) {
                    if (U.compareAndSwapInt(this, LOCKSTATE, s, s | WAITER)) {
                        waiting = true;
                        waiter = Thread.currentThread();
                    }
                }
                else if (waiting)
                    LockSupport.park(this);
            }
        }

```
1. 首先没有读/写在进行，那么尝试置位写标志，成功后，并清除自己设置的等待，返回
2. 要是读/写在进行，且无线程等待，尝试置位等待标志，成功后，设置waiting线程设为自己
3. 以上的情况都不满足，且自身已经设置等待，就调park

如果单看contendedLock和find两个方法里对应的LockSupport操作，
你会发现，如果多线程竞争写锁，调用contendedLock，那么一个线程置位写锁，或者置位等待，
其他线程会怎么样？这是个for循环，会一直试。因为只有一个等待标记，所以waiting只会在一个线程中为true，
也就是某个线程会掉park，其他线程全部运行，疯狂的跑for循环。

是的，当然不能这样，我们看设计者的思路

```java
/*
* TreeBins also require an additional locking mechanism. While
* list traversal is always possible by readers even during
* updates, tree traversal is not, mainly because of tree-rotations
* that may change the root node and/or its linkages. TreeBins
* include a simple read-write lock mechanism parasitic on the
* main bin-synchronization strategy: Structural adjustments
* associated with an insertion or removal are already bin-locked
* (and so cannot conflict with other writers) but must wait for
* ongoing readers to finish. Since there can be only one such
* waiter, we use a simple scheme using a single "waiter" field to
* block writers. However, readers need never block. If the root
* lock is held, they proceed along the slow traversal path (via
* next-pointers) until the lock becomes available or the list is
* exhausted, whichever comes first. These cases are not fast, but
* maximize aggregate expected throughput.
*/
```

contendedLock在哪里调用呢？在put/merge/compute/replace/computeIfPresent等操作，
这些操作中首先synchronize(table[i])，然后才调用`lockRoot()`，所以写是单线程的，所以我们刚才的多线程竞争写锁的条件是不成立的。
对比读和写的代码可能会有疑问：
写操作：先置waiting位，再park
读操作：如果waiting，再unpark
读和写操作有两步，就可能有竞态条件，就可能先unpark，再park。乍一看，这是典型的丢失更新的情况，导致写操作死等下去
这其实是没有关系的，我们看unpark代码

```java
    /**
     * Makes available the permit for the given thread, if it
     * was not already available.  If the thread was blocked on
     * {@code park} then it will unblock.  Otherwise, its next call
     * to {@code park} is guaranteed not to block. This operation
     * is not guaranteed to have any effect at all if the given
     * thread has not been started.
     *
     * @param thread the thread to unpark, or {@code null}, in which case
     *        this operation has no effect
     */
    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
```
如果thread因park而blocked，那么unblock，否则，下一次调用park保证不会block，
也就是先unpark一个线程，那么该线程park时，不会发生阻塞

> 注意s是本地变量，不存在并发风险。

### transfer()
什么情况下会进行扩容呢？
我们回顾putVal的代码
```java
    // 最后两行，binCount在上面分析过了，>=0
    addCount(1L, binCount);
    return null;
}
```

`addCount`主要是两大功能，1. 计数+1，2. 做扩容相关的工作
1. 计数+1：

当put操作频繁时，如何减轻+1的压力，对的，和map的原理一样，通过hash的方法分散到不同的计数单元(JUC有专门的类LongAdder)。
下面这段代码是统计元素个数的，两个map中的成员变量：baseCount、counterCells数组，把数组中的值和baseCount全加起来就是。
```java
    final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```
再看计数+1实现的：总的来说，先试着加到baseCount变量，失败了就执行fullAddCount，加到counterCells数组中去
```java
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
```
`ThreadLocalRandom.getProbe()`可以理解为一个线程中的随机数，存在Thread对象里的。看它的初始化代码：
```java
// fullAddCount
if ((h = ThreadLocalRandom.getProbe()) == 0) {
    ThreadLocalRandom.localInit();      // force initialization
    h = ThreadLocalRandom.getProbe();
    wasUncontended = true;
}
```


2. 扩容相关的工作
```java
// s就是上述计数的结果，代表map中的键值对的数量
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) {   // _1
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0) // _3
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) // _4
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2)) // _2
                    transfer(tab, null);
                s = sumCount();
            }
        }
```
当s大于阈值sizeCtl，且tab.length不超过最大值，进行扩容相关处理
代码中的注释，_1代表目前正在扩容，_2尝试发起扩容操作，_3扩容已经结束，跳出循环，_4加入到当前的扩容
一般情况下，sizeCtl就是1.5*table.length，进行扩容时sizeCtl为负数，即最高bit为1，bits结构

```text
高16    低16
1...11 0...2
```

按照目前的配置，一个int为32bits，平均分两半，高16bits，和低16bits

- 高16bits中，最高位用作控制标识，表示当前正在扩容，剩下位数用于表示当前扩容前的table.length
    15bits怎么表示这个int数（32bits）呢？
    因为table.length始终可以表达为2^n，对于的bits的形状就是32bits中只有一个1，所以通过1的前导0的个数，就能还原出这个数
    因此我们存前导0的个数，6个bits都就足够了
- 低16bits中，用于表示有多少线程，正在扩容
    当线程初始化扩容前低16bits，即一个short变量，初始值为2，
    当再有一个线程协助扩容，该值进行+1，
    对应的，一个线程退出扩容时，该值-1，所以线程发现自己是最后一个退出时，该值为2，需要做一些扫尾工作。

resizeStamp(n)的代码：
```java
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```
就是算出sizeCtl中，高16bits的值，并存在低16bits，注意这个n是扩容前的table.length

(sc = sizeCtl)<0，负数，则表示正在扩容中，
语句_3的各种判断：
```java
(sc >>> RESIZE_STAMP_SHIFT) != rs // RESIZE_STAMP_SHIFT==16，即检查当前扩容的是不是同一个table.length
 || sc == rs + 1                  // 很显然这是一个bug
 || sc == rs + MAX_RESIZERS       // 同一个bug
 || (nt = nextTable) == null      // 不在扩容了
 || transferIndex <= 0            // 是否已完成扩容，transferIndex代表扩容的任务分配进度
```
[bug的链接][2]，以上条件满足则无需扩容，否则_4尝试加入扩容，sc+1，即sizeCtl的低16bits，表示正在扩容的线程数+1

发起初始化的代码：`U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2)`，这里是2（1+线程数）

接下来就是是真正扩容的代码，我们先看看一个梗概，部分代码先折叠了：
```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        // table是个数组，扩容的时候，要把数据搬到nextTab，分批搬，每次搬stride个槽
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        // 初始化扩容的槽
        if (nextTab == null) {...}
        int nextn = nextTab.length;
        // 准备好，占位对象，table的一个槽，一搬好，就赋值占位
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        // 当最后一个扩容任务要退出时，要检查是否table都搬好了，这个变量就是用于检查的标记
        boolean finishing = false; // to ensure sweep before committing nextTab
        // i代表要搬的table索引
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            // 计算要处理的i
            while (advance) {...}
            // 程序出口：扩容线程退出，最后一个线程检查的处理
            if (i < 0 || i >= n || i + n >= nextn) {...}
            // 该槽位本来没有任何数据，不用搬，直接放占位对象
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            // 检查该槽位是否搬过了，用于最后一个退出的扩容线程做检查的
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            // 真正开始搬，先把头结点同步住
            else {
                synchronized (f) {...}
            }
        }
    }

```
当前线程是如何取得搬运的任务的：
table数组，扩容时，每个槽都要搬到nextTable，所以要区分哪些槽，还没分配给线程搬，用transferIndex，它是成员变量，
它的含义是小于它的索引位置，线程可以领取这些槽，执行搬运。也就是说大于等于它的槽，都有线程开始搬运，或者已经搬完了。
这里的代码，要进到第三个if，领取[nextBound，nextIndex)区间的任务，并更新transferIndex为nextBound。
从这while出来之后的i，就是接下来要搬运的table的槽的索引

```java

while (advance) {
    int nextIndex, nextBound;
    if (--i >= bound || finishing)
        advance = false;
    else if ((nextIndex = transferIndex) <= 0) {
        i = -1;
        advance = false;
    }
    else if (U.compareAndSwapInt
             (this, TRANSFERINDEX, nextIndex,
              nextBound = (nextIndex > stride ?
                           nextIndex - stride : 0))) {
        bound = nextBound;
        i = nextIndex - 1;
        advance = false;
    }
}
```

从上面的代码看到，transferIndex<=0，代表没有不需要再领取任务，这时候可以退出了，即i<0，就到了函数的出口，
此时finishing=false，先把sizeCtl-1，代表自己要退出来了，`if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)`，
这里检查当前线程不是最后一个退出来的，不是最后一个线程，就直接return退出函数，否则finishing=true，i=n，
最后一个线程要把table从n-1到0遍历，检查是否有漏搬的槽，有漏则马上搬。
```java

if (i < 0 || i >= n || i + n >= nextn) {
    int sc;
    if (finishing) {
        nextTable = null;
        table = nextTab;
        sizeCtl = (n << 1) - (n >>> 1);
        return;
    }
    if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
        if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
            return;
        finishing = advance = true;
        i = n; // recheck before commit
    }
}
```

函数出口的判断逻辑`if (i < 0 || i >= n || i + n >= nextn)`，后两个条件，分析起来应该是dead code：
在同一轮扩容中，i最大为n，进入这段代码前，`--i`，所以i<n，而`nextn=2*n`，所以if中的后两个条件，应该是达不到的。
所以，我翻阅了jsr166中这行代码的commit:

> Cooperative resizing, plus other fixes and improvements
> 98ade950 dl <dl> on 2012/12/14 at 4:34 上

看起来是实现协同扩容时写的。

另外，最后一个扩容线程退出时，重新遍历table做了recheck，这里很奇怪，要知道扩容线程退出时，以领取的槽都已经搬完了，为什么还要做检查？
看代码的commit：

> Avoid unbounded recursion
> 9c413f25 dl <dl> on 2013/7/5 at 2:33 上午

没什么有用的信息，这个commit解决的是ForwardingNode.find的问题。
这个疑问最近有答复了：[Doug Lea的答复][3]

> As I mentioned, rechecking that it is never needed and possibly removing is on the todo 
> list for next update. (Because it only impacts one step of resize, the 
> performance impact is relatively low.)

确实是没有必要的recheck，并且作者计划移除了。

### cas优化锁结构的总结
1. 给table[i]赋值时
2. 扩容，通过sizeCtl控制
3. 统计线程count和basecount
4. 红黑树的访问，LOCKSTATE
5. 初始化table，控制sizeCtl


## 学习和总结

1. 高并发编程：使用本地变量，避免共享变量
2. 高并发，均摊负载：helpTransfer，利用线程分散负载

[1]: java.util.HashMap.TreeNode.putTreeVal
[2]: https://bugs.openjdk.java.net/browse/JDK-8214427
[3]: http://cs.oswego.edu/pipermail/concurrency-interest/2020-July/017181.html