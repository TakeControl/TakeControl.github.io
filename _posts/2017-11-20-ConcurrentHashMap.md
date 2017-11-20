---
layout:     post
title:      "ConcurrentHashMap源码解读"

date:       2017-11-20 23:00:00
author:     "ShiYu"
catalog: true
tags:
    - Java
---
## 为什么需要ConcurrentHashMap
众所周知HashMap是线程不安全的，关于为什么HashMap是线程不安全的，可以看我以前的文章，有简单介绍过。HashMap在多线程环境中，不仅会出现数据丢失，严重的时候甚至会出现Infinite Loop，导致cpu负载100%从而导致宕机。为了解决HashMap不能用语多线程环境的问题，Sun的工程师们提供了很多方法，比如HashTable、Collections.synchronizedMap(new HashMap(...))、自定义Lock等方法，但这些方法都摆脱不了一个问题，那就是*效率*，以上方法都是对每个方法加上锁，锁是独占资源对，效率自然就提高不上去，这时候ConcurrentHashMap应运而生，ConcurrentHashMap相对于以上方法，利用分段锁巧妙地在保证同步的同时，效率相比前面的方法也有很大提升。下面我们就来看看ConcurrentHashMap的具体实现。

## 结构图
![ConcurrentHashMap结构图](http://139.224.133.147/dist/images/ConcurrentHashMap.png)

## 源码解读
### 数据结构

```java
 /**
     * ConcurrentHashMap list entry. Note that this is never exported
     * out as a user-visible Map.Entry.
     */
    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;

        HashEntry(int hash, K key, V value, HashEntry<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
     }
     
```
相比于1.6版本，1.7中将next由final改变为了volatile,这样做的原因是JDK7懒创建了Segment,除了第一个Segment是在一开始创建外，其它Segment都是*constructed only
when first needed*,这样做减少了初始化的footprint，是对JDK6实现的优化。

源码中的注释是这样描述1.6版本的实现的：
>The previous version of this class relied
 heavily on "final" fields, which avoided some volatile reads at
 the expense of a large initial footprint.
 
 前一个版本的实现严重依赖“final”字段,为了避免不一致的读取从而牺牲了大量的初始化空间。

```
    /**
     * Segments are specialized versions of hash tables.  This
     * subclasses from ReentrantLock opportunistically, just to
     * simplify some locking and avoid separate construction.
     */
    static final class Segment<K,V> extends ReentrantLock implements Serializable {
       
        private static final long serialVersionUID = 2249069246763182397L;

        /**
         * The maximum number of times to tryLock in a prescan before
         * possibly blocking on acquire in preparation for a locked
         * segment operation. On multiprocessors, using a bounded
         * number of retries maintains cache acquired while locating
         * nodes.
         */
        static final int MAX_SCAN_RETRIES =
            Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

        /**
         * The per-segment table. Elements are accessed via
         * entryAt/setEntryAt providing volatile semantics.
         */
        transient volatile HashEntry<K,V>[] table;

        /**
         * The number of elements. Accessed only either within locks
         * or among other volatile reads that maintain visibility.
         */
        transient int count;

        /**
         * The total number of mutative operations in this segment.
         * Even though this may overflows 32 bits, it provides
         * sufficient accuracy for stability checks in CHM isEmpty()
         * and size() methods.  Accessed only either within locks or
         * among other volatile reads that maintain visibility.
         */
        transient int modCount;

        /**
         * The table is rehashed when its size exceeds this threshold.
         * (The value of this field is always <tt>(int)(capacity *
         * loadFactor)</tt>.)
         */
        transient int threshold;

        /**
         * The load factor for the hash table.  Even though this value
         * is same for all segments, it is replicated to avoid needing
         * links to outer object.
         * @serial
         */
        final float loadFactor;

        Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
            this.loadFactor = lf;
            this.threshold = threshold;
            this.table = tab;
        }     
```
MAX_SCAN_RETRIES用于设置[自旋锁](https://baike.baidu.com/item/%E8%87%AA%E6%97%8B%E9%94%81)最大自旋次数，对于单核处理器，自旋没有任何意义，所以取值1，多核处理器最大自旋次数为64。table是一个HashEntry类型的一维数组，被声明为volatile，count记录元素个数，modCount记录修改次数，isEmpty()和size（）会将每个segment中该字段值相加比较以保证数据同步，threshold用于标记需要rehash的数量，当元素个数超过该值后，需要rehash，loadfactor即负载因子，容量*负载因子等于threshold。

## put

```java
    /**
     * Maps the specified key to the specified value in this table.
     * Neither the key nor the value can be null.
     *
     * <p> The value can be retrieved by calling the <tt>get</tt> method
     * with a key that is equal to the original key.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>
     * @throws NullPointerException if the specified key or value is null
     */
    @SuppressWarnings("unchecked")
    public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }
```

首先检查value是否为null，如果是null则跑出NPE，关于ConcurrentHashMap值为什么不允许为null,可参考stackOverflow中这个页面[参考页面](https://stackoverflow.com/questions/698638/why-does-concurrenthashmap-prevent-null-keys-and-values),然后取key的哈希值，再对哈希值进行二次哈希，再哈希可以显著减少哈希值的冲突，紧接着根据哈希值找到对应的segment，然后调用对应segment的put方法。
下面是segment中的put方法代码：

```JAVA
       final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value);
            V oldValue;
            try {
                HashEntry<K,V>[] tab = table;
                int index = (tab.length - 1) & hash;
                HashEntry<K,V> first = entryAt(tab, index);
                for (HashEntry<K,V> e = first;;) {
                    if (e != null) {
                        K k;
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else {
                        if (node != null)
                            node.setNext(first);
                        else
                            node = new HashEntry<K,V>(hash, key, value, first);
                        int c = count + 1;
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node);
                        else
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return oldValue;
        }

```
首先会进行trylock，如果获取锁成功，那么剩下的步骤和普通HashMap进行put值得流程几乎完全一样，如果trylock失败，则执行scanAndLockForPut方法，该方法采用自旋锁解决多处理器获取锁的问题，以下是scanAndLockForPut方法的代码

```java
        /**
         * Scans for a node containing given key while trying to
         * acquire lock, creating and returning one if not found. Upon
         * return, guarantees that lock is held. UNlike in most
         * methods, calls to method equals are not screened: Since
         * traversal speed doesn't matter, we might as well help warm
         * up the associated code and accesses as well.
         *
         * @return a new node if key not found, else null
         */
        private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
            HashEntry<K,V> first = entryForHash(this, hash);
            HashEntry<K,V> e = first;
            HashEntry<K,V> node = null;
            int retries = -1; // negative while locating node
            while (!tryLock()) {
                HashEntry<K,V> f; // to recheck first below
                if (retries < 0) {
                    if (e == null) {
                        if (node == null) // speculatively create node
                            node = new HashEntry<K,V>(hash, key, value, null);
                        retries = 0;
                    }
                    else if (key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                }
                else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                }
                else if ((retries & 1) == 0 &&
                         (f = entryForHash(this, hash)) != first) {
                    e = first = f; // re-traverse if entry changed
                    retries = -1;
                }
            }
            return node;
        }

```
关于自旋锁原理这里不再赘述，各位可自行拜访度娘。

*PS* 在put方法的代码中，有这么一行：
```JAVA
    int index = (tab.length - 1) & hash;
```
这句代码用来寻找数据在桶中的位置，但是关于为什么这样寻找位置我一开始也没太明白，在Stack Overflow中逛了一圈后恍然大悟，原理是这样的：

```
 Whenever a new Hasmap is created the array size of internal Node[] table is always power of 2 and following method guarantees it -

static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
So lets say you provide initial capacity as 5

cap = 5
n = cap - 1 =  4 = 0 1 0 0
n |= n >>> 1;    0 1 0 0 | 0 0 1 0 = 0 1 1 0 = 6
n |= n >>> 2;    0 0 1 1 | 0 1 1 0 = 0 1 1 1 = 7
n |= n >>> 4;    0 0 0 0 | 0 1 1 1 = 0 1 1 1 = 7
n |= n >>> 8;    0 0 0 0 | 0 1 1 1 = 0 1 1 1 = 7
n |= n >>> 16;   0 0 0 0 | 0 1 1 1 = 0 1 1 1 = 7
return n + 1     7 + 1 = 8 
So table size is 8 = 2^3

Now possible index values you can put your element in map are 0-7 since table size is 8. Now lets look at put method. It looks for bucket index as follows -

Node<K,V> p =  tab[i = (n - 1) & hash];
where n is the array size. So n = 8. It is same as saying

Node<K,V> p =  tab[i = hash % n];
So all we need to see now is how

hash % n == (n - 1) & hash
Lets again take an example. Lets say hash of a value is 10.

hash = 10
hash % n = 10 % 8 = 2
(n - 1) & hash = 7 & 10 = 0 1 1 1 & 1 0 1 0 = 0 0 1 0 = 2
```
转自：[Stack Overflow](https://stackoverflow.com/questions/27230938/why-hashmap-insert-new-node-on-index-n-1-hash)

##get

``` java
    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code key.equals(k)},
     * then this method returns {@code v}; otherwise it returns
     * {@code null}.  (There can be at most one such mapping.)
     *
     * @throws NullPointerException if the specified key is null
     */
    public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }

```

1. 首先对key做两次hash确定到取哪个Segment。
2. 然后用UNSAFE类的getObjectVolatitle方法取得对应的Segment,之所以使用该方法而不是直接通过数组下标取值，是因为从java内存模型可知，每个线程都会有对应值的副本，getObjectVolatile方法直接从内存中取值，可以保证取到永远是最新值而不会是有可能过期的副本。
3. 如果取得的Segment不为null,然后再进行一次&运算得到HashEntry链表的位置，然后从链表头开始遍历整个链表（因为Hash可能会有碰撞，所以用一个链表保存），如果找到对应的key，则返回对应的value值，如果链表遍历完都没有找到对应的key，则说明Map中不包含该key，返回null。

## size

```java
 /**
     * Returns the number of key-value mappings in this map.  If the
     * map contains more than <tt>Integer.MAX_VALUE</tt> elements, returns
     * <tt>Integer.MAX_VALUE</tt>.
     *
     * @return the number of key-value mappings in this map
     */
    public int size() {
        // Try a few times to get accurate count. On failure due to
        // continuous async changes in table, resort to locking.
        final Segment<K,V>[] segments = this.segments;
        int size;
        boolean overflow; // true if size overflows 32 bits
        long sum;         // sum of modCounts
        long last = 0L;   // previous sum
        int retries = -1; // first iteration isn't retry
        try {
            for (;;) {
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    for (int j = 0; j < segments.length; ++j)
                        ensureSegment(j).lock(); // force creation
                }
                sum = 0L;
                size = 0;
                overflow = false;
                for (int j = 0; j < segments.length; ++j) {
                    Segment<K,V> seg = segmentAt(segments, j);
                    if (seg != null) {
                        sum += seg.modCount;
                        int c = seg.count;
                        if (c < 0 || (size += c) < 0)
                            overflow = true;
                    }
                }
                if (sum == last)
                    break;
                last = sum;
            }
        } finally {
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return overflow ? Integer.MAX_VALUE : size;
    }

```
size方法与get和put的区别是，它需要遍历所有的segment，没办法再通过分段加锁思想避开同步的性能问题，sun工程师再这里用了一个巧妙的思想，先给3次机会，不lock所有的Segment，遍历所有Segment，累加各个Segment的大小得到整个Map的大小，如果某相邻的两次计算获取的所有Segment的更新的次数（每个Segment都有一个modCount变量，这个变量在Segment中的Entry被修改时会加一，通过这个值可以得到每个Segment的更新操作的次数）是一样的，说明计算过程中没有更新操作，则直接返回这个值。如果这三次不加锁的计算过程中Map的更新次数有变化，则之后的计算先对所有的Segment加锁，再遍历所有Segment计算Map大小，最后再解锁所有Segment。

## ContainsValue

```java
    /**
     * Returns <tt>true</tt> if this map maps one or more keys to the
     * specified value. Note: This method requires a full internal
     * traversal of the hash table, and so is much slower than
     * method <tt>containsKey</tt>.
     *
     * @param value value whose presence in this map is to be tested
     * @return <tt>true</tt> if this map maps one or more keys to the
     *         specified value
     * @throws NullPointerException if the specified value is null
     */
    public boolean containsValue(Object value) {
        // Same idea as size()
        if (value == null)
            throw new NullPointerException();
        final Segment<K,V>[] segments = this.segments;
        boolean found = false;
        long last = 0;
        int retries = -1;
        try {
            outer: for (;;) {
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    for (int j = 0; j < segments.length; ++j)
                        ensureSegment(j).lock(); // force creation
                }
                long hashSum = 0L;
                int sum = 0;
                for (int j = 0; j < segments.length; ++j) {
                    HashEntry<K,V>[] tab;
                    Segment<K,V> seg = segmentAt(segments, j);
                    if (seg != null && (tab = seg.table) != null) {
                        for (int i = 0 ; i < tab.length; i++) {
                            HashEntry<K,V> e;
                            for (e = entryAt(tab, i); e != null; e = e.next) {
                                V v = e.value;
                                if (v != null && value.equals(v)) {
                                    found = true;
                                    break outer;
                                }
                            }
                        }
                        sum += seg.modCount;
                    }
                }
                if (retries > 0 && sum == last)
                    break;
                last = sum;
            }
        } finally {
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return found;
    }

```

containsValue方法采用了和size一样的思想，不过有一点不同就是，containsValue只有当找不到值时才会采用上面的思想来确认数据的确不存在，如果找到了值，会直接中断循环然后返回true。


