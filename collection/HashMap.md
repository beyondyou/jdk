#HashMap

## 概述

`HashMap`是基于Map接口实现的Hash表结构，key和value都允许为null。HashMap类粗略的等价于HashTable类，与HashTable不同的是HashMap不是同步的，并且允许为null。

## 继承结构

<img src="/Users/chen/jdk/images/collections/HashMap.png" alt="HashMap" style="zoom:50%;" />

## 字段

```java
public class HashMap{
  	/** 一些默认设置的参数 **/
  	// 默认的初始化容器大熊啊必须是2的幂次方
		static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
  	static final int MAXIMUM_CAPACITY = 1 << 30;
  	// 默认的加载因子
  	static final float DEFAULT_LOAD_FACTOR = 0.75f;
  	// 链表转成红黑树的阈值
  	static final int TREEIFY_THRESHOLD = 8;
  	// 由红黑树转成链表的最小阈值
  	static final int UNTREEIFY_THRESHOLD = 6;
  	// 
  	static final int MIN_TREEIFY_CAPACITY = 64;
  
  	/* ---------------- HashMap的真正字段结构 -------------- */
  	// 由Node数组构成的hash表
  	transient Node<K,V>[] table;
  	transient Set<Map.Entry<K,V>> entrySet;
  	// map中包含key-value映射对的数量
  	transient int size;
  	// 该字段用来让迭代器在HashMap的集合视图上fail-fast
  	transient int modCount;
  	int threshold;
  	final float loadFactor;
}
```



## 构造器



```java
/** 默认构造器 
 * 只将加载因子初始化为默认的加载因子0.75f
 */
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

**get操作**

1.计算给定key的hash值，由于原生hash函数hash冲突的概率比较大，所以HashMap通过扰动函数重新计算了hash值，也就是`(h = key.hashCode()) ^ (h >>> 16)`

2.根据key的hash值和容量-1能够定位到key在hash表数组中的位置

3.总是先判断已经定位到的hash表的第一个节点，比较其hash值和key的值，如果该位置上存储的是链表/红黑树，则需要遍历链表/红黑树，直到找到与key相对应的value值。

```java
// 
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 根据key计算的hash值与上(&)哈希表的容量-1(table.length-1) 来表示元素所在的位置,如果该位置的第一个节点为空
    // 则直接返回null，如果不为空，则需要比较hash值和key的值（有可能需要遍历链表/红黑树）
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 然后比较key的hash值key的真实值，
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 这里表示存在hash冲突的情况下，需要遍历链表/红黑树获取到元素
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
            	// 从红黑树中获取元素
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 遍历链表获取元素
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

#### put操作

往hashmap中put的过程：

1、计算给定键的哈希值
3、先判断hash表是否初始化，没有初始化的话需要调用resize方法对hash表初始化
3、根据键的哈希值定位元素应该保存在map数组中的位置index。
4、判断是否会产生hash冲突
	a.如果定位到数组中元素为空，直接创建一个Node并根据index放入数组中
	b.如果定位到数组中元素不为空，则表明产生了hash冲突，需要将这些hash冲突的元素以链表的方式组织起来，
	当链表中元素数量到达8时，将链表转化成红黑树。
5、检查map中键值对数量是否超过阈值，如果超过阈值，则需要扩容

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
    	// 第一次put的时候需要初始化一个Node数组
        n = (tab = resize()).length;
    // 定位key在table数组中的位置，如果该位置没有元素，直接将创建一个node放入table数组中
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {  // 存在hash冲突，需要以链表/红黑树的方式组织元素，通过拉链法也叫做链地址发解决hash冲突
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)  // 如果数组中的元素已经以红黑树组织元素，则向红黑树中添加元素
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
        	// 遍历链表，将value插入链表的尾部，如果原来的链表大小为7，则将链表转化为红黑树
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果遇到相同的key，直接跳出循环，不做任何处理
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                // 往下继续遍历
                p = e;
            }
        }

        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 如果key已经存在了并且允许覆盖原来的value值，则直接覆盖value，然后直接返回老的value值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
          	// 这里为什么直接返回了？因为这里并没有做新增键值对操作，只是对原来已经存在的key做覆盖操作。
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
  	// 提供给LinkedHashMap实现的钩子方法，在HashMap是一个空实现
    afterNodeInsertion(evict);
    return null;
}
```

resize扩容函数

resize函数主要的功能是对hash表进行扩容，根据hash表是否是第一次存放键值对以及HashMap使用的构造方法是什么来确定阈值threshold和容量capacity，确定以后就可以构造一个新的hash表数组（也就是扩容后的新的数组），数组的容量扩大为原先容量的2倍。然后再将所有的原先数组中的节点移动到新的hash表中。

```java
final Node<K,V>[] resize() {
	// 原来的数组
    Node<K,V>[] oldTab = table;
    // 原数组的容量(长度)
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 原来hash表的阈值
    int oldThr = threshold;
    // 给新的容量和新的阈值设置一个默认值0
    int newCap, newThr = 0;
    // hash表已经将数组初始化，并且hash表中已经有元素了 ①
    if (oldCap > 0) {
    	// 首先判旧容量是否大于最大的容量
        if (oldCap >= MAXIMUM_CAPACITY) {
        	// 如果旧容量超过了最大值，则不能再去扩容，直接将返回
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 2倍扩容后的容量还小于最大容量，并且旧容量大于默认的容量， 也会将阈值变成原来的两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // ② 未指定新的阈值，使用的是有参数的构造函数第一次put会走到这里，
    //    因为有参数的构造函数会初始化阈值，但是还没有初始化hash表数组的容量
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // ③ 
    else {               // zero initial threshold signifies using defaults
    	// 这里也就是没有设置容量和加载因子，使用默认的构造函数构造一个HashMap
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 如果上面的判断中走的是②，则需要设置新的阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];  // 初始化一个扩容后的hash表数组
    table = newTab;
    // 原来的hash表中包含元素，需要将这些元素复制到扩容后的hash表中
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)  // 表示index位置下的元素还没有hash冲突，直接赋值给新扩容后的hash表
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)  // 表示hash表中的节点已经是一颗红黑树了，需要单独处理
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order // 普通的链表
                    Node<K,V> loHead = null, loTail = null;  // 表示高位的链表
                    Node<K,V> hiHead = null, hiTail = null;  // 表示低位的链表
                    Node<K,V> next;
                    // 在do...while循环中不断地将这些节点组装起来成为新的链表
                    do {
                        next = e.next;
                        /** (e.hash & oldCap == 0) 表示该节点在新的hash表的下标位置与
                         *  旧的hash表的下标位置⼀致，都为j，如果等于0，则将节点放入到地位链表中
                         *	否则将节点e放入到高位链表中。这里为什么是判断 e.hash & oldCap 
                         *  看下这个博客 https://segmentfault.com/a/1190000015812438,写的非常清楚
                         */
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                            	// 地位链表中还没有元素，将节点e表示为链表的头部
                                loHead = e;
                            else
                            	// 低位将链表的尾部节点的后继指针指向节点e
                                loTail.next = e;
                            loTail = e;
                        }
                        else {   //  则该节点在新的hash表的下标位置 j + oldCap，
                            if (hiTail == null)
                            	// 高位链表中还没有元素，将节点e表示为链表的头部
                                hiHead = e;
                            else
                            	// 高位将链表的尾部节点的后继指针指向节点e
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 将低位链表存放在数组中j的位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 将高位链表存放在数组中 j + oldCap 的位置
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

remove操作

表示从HashMap中移除给定key映射的value值

```java
public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
}
```

### 总结

hashMap为什么容量必须是2的幂次方

hashMap的扰动函数

