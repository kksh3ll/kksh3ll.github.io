---
layout: post
title:  HashMap的工作原理
date:   2016-03-11 20:19:29
categories: [Java]
---
## 官方描述
Hash table based implementation of the Map interface. This

implementation provides all of the optional map operations, and permits

null values and the null key. (The HashMap class is roughly equivalent

to Hashtable, except that it is unsynchronized and permits nulls.)

This class makes no guarantees as to the order of the map; in particular,

it does not guarantee that the order will remain constant over time.

我的英语一般，但这几句话的意思我还是能看的差不多，大致意思就是说：`Hash table`基于实现`Map`接口，实现方法提供了所有的可选`map`操作，并且允许赋null值。`HashMap`类基本等价于`Hashtable`，线程不同步，不按插入的顺序排列，也不保证不随时间的变化。

## 构造函数
`HashMap`提供了4个构造函数：
1. `HashMap(int initialCapacity, float loadFactor)`
2. `HashMap(int initialCapacity)`
3. `HashMap()`
4. `HashMap(Map<? extends K, ? extends V> m)`

初始容量是影响`HashMap`性能的一个重要参数，指定了创建Hash表的容量。

## `get()` 和 `put()`的实现
* `get()`的实现
  1. 容量里的第一个节点
  2. 通过`key.equals(k)`去查找对应的`entry`

{% highlight Java %}
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
}
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        // table不为空才进行以下操作，table为null的话直接返回null
        if ((tab = table) != null && (n = tab.length) > 0
          && (first = tab[(n - 1) & hash]) != null) {
            // 直接命中
            if (first.hash == hash
              && ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                // 在树中命中
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 在链表中命中
                do {
                    if (e.hash == hash
                      && ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
           }
       }
        return null;
    }
{% endhighlight %}

 * `put()`的实现
  1. 对`key`的`hashCode`做hash，再计算`index`
  2. 没碰到的话直接放到`bucket`里
  3. 碰撞了，以链表的形式放入`bucket`后
  4. 如果导致链表过长，把链表转换为红黑树
  5. 如果节点存在就替换`old value`（保证key的唯一性）
  6. 如果`bucket`满了，就要`resize`

{% highlight Java %}
public V put(K key, V value) {
        // 对key的hashCode()做hash()
        return putVal(hash(key), key, value, false, true);
    }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // table为空则创建
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 计算index并做特殊处理
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            // 如果hash和key都相同
            if (p.hash == hash
              && ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 如果该链为树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            // 如果该链为链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 如果链表的长度超过了这个阈值，
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash
                      && ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 如果节点存在的话，就替换新值返回旧值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        // 如果大小超过了 加载因子*当前容量，就进行扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
{% endhighlight %}
