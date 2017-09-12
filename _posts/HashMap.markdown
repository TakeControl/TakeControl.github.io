## 理解Map

Map的基本思想是维护*键-值*关联，因此你可以使用键来查找值，我们可以先自己简单实现一个Map:

```java
package top.shixiaoyu.map;
/**
 * 极其简单的map实现，不考虑增容、性能，仅作演示
 * Created by ShiYu on 2017/9/4.
 */
public class SimpleMap<K,V>  {
   private Object[][] pairs;
   private int index;
   
   public SimpleMap(int length){
       pairs=new Object[length][2];
   }

    /**
     * 存储键值对
     * @param key 键
     * @param value 值
     */
   public void put(K key,V value){
       if(index>=pairs.length){
           throw new ArrayIndexOutOfBoundsException();
       }
       //示例代码，不考虑键重复问题
       pairs[index++]=new Object[]{key,value};
   }

    /**
     * 根据key值获取value
     * @param key 键值
     * @return value
     */
   public V get(K key){
       for(int i=0;i<index;i++){
           if(key.equals(pairs[i][0])){//示例代码，不考虑空指针异常
               return (V)pairs[i][1];
           }
       }
       return null;
   }
}

```

上述代码是说明性的，极其缺乏效率，并且尺寸固定，极不灵活，java工具包中提供的各种map实现则没有这些问题，今天我们先来看一下HashMap的具体实现。

## HashMap实现原理

HashMap的实现原理很简单，HashMap中维护一个数组，当调用put方法时，先根据key进行hash散列，散列出对应的数组下标，然后将value值存入对应的数组中，但为了解决hash键值冲突问题，数组中不直接保存值，而是保存值的list。这样，当调用get方法时，首先将key散列得到数组下标，然后根据下标得到数组中对应的list,然后再便利list,根据key从list中取出对应的值。

HashMap的实现在JDK1.8做了一些改变，相对于更早的版本，1.8中的HashMap在处理键值冲突时，设置了一个阀门，默认为8，当冲突值大于阀门时，则将list改变为红黑树，从而避免当list过大时，线性查询带来的性能问题。

以上只是对HashMap实现的简单概述，HashMap的实现还有很多细节，比如当数组容量不够时，会进行再散列等。

下面介绍关键的代码：

```java
/**内部静态类，存储value的Node
 static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }


	 //默认初始容量：表示在创建时所拥有的桶位数，即数组的初始容量，可通过构造函数手动指定初始容量
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    //负载因子：尺寸／容量，尺寸是指数组中当前存储等项数，当数组的负载情况达到负载因子的水平时，HashMap会自动增加数组长度，并将现有对象重新飞配（又称为再散列）。
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //key哈希冲突时，采用红黑树而不采用list的阀门值，当冲突数大于阀门值后，将list转化为红黑树
    static final int TREEIFY_THRESHOLD = 8;
	//数组，在第一次被使用时初始化，当必要时（负载超过负载因子）将进行再散列（resize)
    transient Node<K,V>[] table;
    //map中包含的key-value数量
     transient int size;
     //map被修改的次数，用来实现迭代时同步异常的fail-fast机制，不明白的可以翻看jdk文档
     transient int modCount;
     

```
下面是put方法实现的流程图
![put路程图](http://www.shixiaoyu.top/dist/images/hashmap_put.png)

下面是put方法的实现

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * 实现map的put方法
     *
     * @param hash  key的哈希值
     * @param key  key
     * @param value 被put的值
     * @param onlyIfAbsent 若是true，不改变已经存在的值
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //1 判断table是否为空，是否需要扩容操作
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //2 给p赋值，为一个节点的位置，若为空，创建一个新节点，否则追加到list或红黑树
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))//比较哈希值是否相同，key是否相同,key值是否相同
                e = p;//key已存在，直接覆盖
            else if (p instanceof TreeNode)//若p是treeNode对象
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);//在红黑树中插入键值对
            else {//否则遍历链表，准备插入
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                    	//	创建要插入到节点
                        p.next = newNode(hash, key, value, null);
                        
                        //若链表长度大于treeify阀门，进行链表=》红黑树的转换
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果存在key值，直接覆盖value
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //判断是否超过最大容量，需要扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

下面是get方法实现

```java
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
     * 实现get方法
     *
     * @param hash  key的哈希值
     * @param key the key key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {//table不为空，根据哈希值取node不为空
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))//检测第一个code是否刚好是查找的值
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)//如果是红黑树，则从红黑树中获取值
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {//遍历node列表获取值
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```


以上就是HashMap中put和get的实现，通过源码解析我们能够发现HashMap是线程不安全的，原因在于当多个线程调用put方法时，如果key的哈希值冲突，几个线程会同时得到现在的头节点，然后线程A写入在头节点之后，线程B也写入新的头节点，那么B就会覆盖A的写入操作，造成A的写入操作丢失。此外当键值对总数超过阀门时，会调用resize操作，这个操作会新生成一个新的容量数组，然后对原数组所有键值对重新计算并写入新数组，当多个线程同时检测到超过阀门限制时，会同时调用resize操作，各自生成新的数组并rehash后赋值给table,结果只有最后一个线程生成的新数组赋值给table,其它线程均丢失。
