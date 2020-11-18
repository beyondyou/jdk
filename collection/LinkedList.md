#LinkedList

## 概述

LinkedList是由一个双向链表组成的集合列表，能够存储任何元素（包括null），从LinkedList实现来看，在插入和删除元素的时候能够发挥双向链表的优化，插入/删除一个元素很快。查找元素的时候是顺序查找效率相比ArrayList来说要低一点。

### 继承结构

<img src="/Users/chen/jdk/images/collections/LinkedList.png" alt="LinkedList" style="zoom:50%;" />

### 构造器

```java
//无参构造器
public LinkedList() {
}
// 传入一个集合列表作为构造器参数
public LinkedList(Collection<? extends E> c) {
  			// 初始化默认构造器
        this();
  			// 将参数所有的元素都添加到LinkedList中
        addAll(c);
}
```

### 全局变量

```java
public class LinkedList<E> {
    // 链表结合大小
		transient int size = 0;
  	// 头节点
  	transient Node<E> first;
    // 尾节点
    transient Node<E> last;
}

// Node是一个双向链表的结构，作为LinkedList私有的静态内部类
private static class Node<E> {
    // 集合元素的值
    E item;
    // 链表的后继节点
    Node<E> next;
    // 链表的前驱节点
    Node<E> prev;
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```



###集合操作

1. add添加元素

```java
/**
 * a.将给定的元素append到链表的结尾
 */
public boolean add(E e) {
    linkLast(e);
    return true;
}
void linkLast(E e) {
	  // 将链表中最后一个节点赋值给一个临时变量
    final Node<E> l = last;
    // 根据元素初始化一个节点，这里会初始化一个前继节点和为null的后置节点
    final Node<E> newNode = new Node<>(l, e, null);
    // 将last的指针指向新加入的元素节点
    last = newNode;
    if (l == null)
    	// 表示链表加入第一个元素
        first = newNode;
    else
    	// 将之前的结尾节点的后继指针执行新的节点
        l.next = newNode;
    // 更新集合大小
    size++;
    modCount++;
}
/**
 * b.在指定的位置插入元素
 */
public void add(int index, E element) {
	// 校验index是否有效
    checkPositionIndex(index);

    if (index == size)
    	// 直接将元素加入到链表的结尾
        linkLast(element);
    else
    	// 将元素插入到链表
        linkBefore(element, node(index));
}
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    // 初始化元素的前驱节点和后继节点
    final Node<E> newNode = new Node<>(pred, e, succ);
    // 将succ节点的前驱指针指向新节点
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
    	// 将succ的后继指针指向当前节点
        pred.next = newNode;
    size++;
    modCount++;
}
```

**linkedBefore图解**

<img src="/Users/chen/jdk/images/collections/linkedBefore的过程.png" alt="linkedBefore的过程" style="zoom:50%;" />



2. remove移除元素

```java
/**
 * a.移除给定元素第一次出现的元素，移除成功返回ture，移除失败返回false
 */
public boolean remove(Object o) {
    // 判断移除的元素是否为null,移除null和非null的元素都流程都一样，主要是为了防止为null的时候在使用equals会报空指针
    if (o == null) {
    	// 从头结点(第一个节点)开始往下遍历，知道找到一地个为null的元素，然后对元素进行移除
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
              	// unlink x节点
                unlink(x);
                return true;
            }
        }
    } else {
    	// 从头结点(第一个节点)开始往下遍历，知道找到第一个和传入的参数相等的元素，然后对元素进行移除
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}

/**
 * 移除链表中的元素
 */
E unlink(Node<E> x) {
    // assert x != null;
    // 链表中元素的值
    final E element = x.item;
    // 链表的后继节点
    final Node<E> next = x.next;
    // 链表的前驱节点
    final Node<E> prev = x.prev;
    
    if (prev == null) {
    	// 如果前驱节点为null，直接将first指向当前元素的后继节点
        first = next;
    } else {
    	// 将前驱节点的next指针指向当前节点的后继节点
        prev.next = next;
        x.prev = null;
    }
  
    if (next == null) {
    	// 如果后继节点为null直接将last指向元素的前驱节点
        last = prev;
    } else {
    	// 将后继节点的prev指针指向当前节点的前驱节点
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}

/**
 * b. 移除给定位置的元素
 */
public E remove(int index) {
	// 校验index的合法性
    checkElementIndex(index);
    return unlink(node(index));
}
```

3. get获取元素

```java
// 获取给定位置的元素
public E get(int index) {
  	// 校验index是否是有效的 主要是和size做比较，并且index不能小于0
    checkElementIndex(index);
    return node(index).item;
}

Node<E> node(int index) {
    // assert isElementIndex(index);
    // 这里主要是将链表化为两个部分，如果index小于链表大小的一半，就从first往后找，否则的话就从last往前找
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

4. set

```java
/**
 * 替换元素，将index节点的元素的值替换成新的值
 */
public E set(int index, E element) {
	// 校验index是否有效
    checkElementIndex(index);
    // 定位到index的节点元素
    Node<E> x = node(index);
    E oldVal = x.item;
    // 将节点的值更改为新的值，将老的值返回
    x.item = element;
    return oldVal;
}
```

5. indexOf操作

```java
// 从链表的头部开始遍历
public int indexOf(Object o) {
    int index = 0;
    if (o == null) {
      	// 遍历链表，当第一次出现null，返回index
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        // 遍历链表，当第一次出现和元素相等时，返回index
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}
// 从链表的尾部开始像前遍历
public int lastIndexOf(Object o) {
    int index = size;
    if (o == null) {
    	// 从后往前遍历，碰到第一给为null的元素，返回index
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (x.item == null)
                return index;
        }
    } else {
    	// 从后往前遍历，碰到第一个和元素相等时，返回index
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (o.equals(x.item))
                return index;
        }
    }
    return -1;
}
```

