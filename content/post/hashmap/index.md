---
title: HashMap
description: 
slug: hashmap
date: '2026-03-08T16:58:37+08:00'
categories:
    - 编程
tags:
    - notion
---

# 一致性hash算法

一致性哈希算法是分布式系统中用于数据分片和负载均衡的重要算法。它主要解决了传统哈希算法在节点动态增减时需要大量数据迁移的问题。

## 核心概念

一致性哈希将整个哈希空间组织成一个虚拟的环形结构，通常使用0到2^32-1的范围。算法的工作原理如下：

**节点映射**：将服务器节点通过哈希函数映射到环上的某个位置。例如，可以对服务器的IP地址进行哈希运算。

**数据映射**：将数据的键通过相同的哈希函数映射到环上，然后顺时针查找第一个遇到的服务器节点，该节点就负责存储这个数据。

## 主要优势

相比传统的取模哈希（hash(key) % N），一致性哈希的优势在于：

当有节点加入或离开时，只需要迁移相邻节点之间的部分数据，而不是重新分布所有数据。这大大减少了数据迁移的开销。

算法具有良好的平衡性和单调性，即使在节点数量变化时也能保持相对稳定的数据分布。

## 虚拟节点优化

为了解决节点分布不均匀的问题，通常会引入虚拟节点的概念。每个物理节点在环上对应多个虚拟节点，这样可以：

- 提高数据分布的均匀性
- 减少单个节点故障的影响范围
- 更好地适应异构环境
## 实际应用

一致性哈希广泛应用于：

- 分布式缓存系统（如Redis Cluster）
- 分布式存储系统（如Cassandra、DynamoDB）
- 负载均衡器
- CDN系统
这个算法是构建可扩展分布式系统的基础组件之一，特别适合需要频繁扩缩容的云环境。

# 构造函数

HashMap只允许有一个key为null,但是允许多个value为null,HashMap线程不安全

`树化条件（链表转红黑树）:`

1. **链表长度达到8个节点**：当某个桶中的链表长度达到`TREEIFY_THRESHOLD = 8`时
1. **数组长度达到64**：同时HashMap的数组长度必须达到`MIN_TREEIFY_CAPACITY = 64`
**注意**：如果链表长度达到8但数组长度小于64，HashMap会选择扩容而不是树化。

## put方法:

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,  
               boolean evict) {  
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 第一次put时,tab为空,或者将元素删除至空时,length等于0,这时初始化一个table
    if ((tab = table) == null || (n = tab.length) == 0)
	    // 初始化tab
        n = (tab = resize()).length;
    // 通过哈希散列找到对应需要放置的数组下标,如果没有值,就直接放进去
    if ((p = tab[i = (n - 1) & hash]) == null)  
        tab[i] = newNode(hash, key, value, null);
    // 当前node有值
    else {  
        Node<K,V> e; K k;  
        // 匹配到已有的key
        if (p.hash == hash &&  
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 将临时节点e指向根结点p
            e = p;
        // 根结点不匹配,而且这是个红黑树
        else if (p instanceof TreeNode)
	        // 使用红黑树的put方法插值
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 根结点不匹配,但不是红黑树
        else {  
	        // 遍历node节点,同时用binCount记录节点里元素的个数
            for (int binCount = 0; ; ++binCount) {
	            // next为null则表示遍历到了结尾,将新元素插入到尾部
                if ((e = p.next) == null) {  
                    p.next = newNode(hash, key, value, null);
                    // 插入完之后发现元素数量大于树化阈值
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  
	                    // 进行树化
                        treeifyBin(tab, hash);  
                    break;  
                }  
                // 找到相同的key,且这个key在循环开始已经暂存到了临时节点e中了,跳出循环
                if (e.hash == hash &&  
                    ((k = e.key) == key || (key != null && key.equals(k))))  
                    break;  
                p = e;  
            }
        }
        // 如果临时节点e不为空,则表示是有相同的key的
        if (e != null) { // existing mapping for key  
            V oldValue = e.value;
            // 插入新值,返回旧值
            if (!onlyIfAbsent || oldValue == null)  
                e.value = value;  
            afterNodeAccess(e);  
            return oldValue;  
        }  
    }
    // 临时节点为空,新值插入进了node中,操作数加1
    ++modCount;
    // 当前size大于扩容阈值,执行扩容
    if (++size > threshold)  
        resize();  
    afterNodeInsertion(evict);  
    return null;  
}
```
## resize()

```java
final Node<K,V>[] resize() {  
	Node<K,V>[] oldTab = table;
	// 原数组长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length; 
    // 原扩容阈值 
    int oldThr = threshold;  
    int newCap, newThr = 0;
    // 原本的数组有长度
    if (oldCap > 0) {  
	    // 如果已经达到最大长度,不做扩容,让其碰撞吧
        if (oldCap >= MAXIMUM_CAPACITY) {  
            threshold = Integer.MAX_VALUE;  
            return oldTab;  
        }  
        // 原数组长度翻倍后小于最大长度,且原数组长度小于默认的长度(16)
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&  
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 新的扩容阈值翻倍
            newThr = oldThr << 1; // double threshold  
    } 
    // oldCap = 0
    else if (oldThr > 0) // initial capacity was placed in threshold  
        newCap = oldThr;
    // oldCap = 0,且oldThr = 0
    // 初始化数组
    else {               // zero initial threshold signifies using defaults  
        newCap = DEFAULT_INITIAL_CAPACITY;  
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);  
    }  
    // oldCap = 0
    if (newThr == 0) {  
        float ft = (float)newCap * loadFactor;  
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?  
                  (int)ft : Integer.MAX_VALUE);  
    }  
    threshold = newThr;  
    @SuppressWarnings({"rawtypes","unchecked"})  
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];  
    table = newTab;  
    if (oldTab != null) {
	    // 遍历原数组
        for (int j = 0; j < oldCap; ++j) {  
            Node<K,V> e;  
            if ((e = oldTab[j]) != null) {  
                oldTab[j] = null;
                // 如果当前节点只有一个元素
                if (e.next == null)
	                // 迁移仅有的这一个元素即可
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果当前节点是树,使用红黑树的迁移逻辑
                else if (e instanceof TreeNode)  
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 当前节点是链表
                else { // preserve order
				            // 低位
                    Node<K,V> loHead = null, loTail = null;
                    // 高位 
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    // 遍历循环当前链表
                    do {
                        next = e.next;
                        // 如果当前节点 & 原数组长度等于0,就将当前节点存入低位链表
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 否则存入高位链表
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 低位链表不为空,放置在新表原位置
                    if (loTail != null) {  
                        loTail.next = null;  
                        newTab[j] = loHead;  
                    }
                    // 高位链表不为空,放置在新表原位置加原长度的位置
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

# **hashMap多线程不安全的本质**

## **jdk8时多线程put会导致数据丢失**

```java
// 计算index，并对null做处理 
if ((p = tab[i = (n - 1) & hash]) == null)
	tab[i] = newNode(hash, key, value, null);
```
两个线程同时进行put,第一个线程执行完if判断准备插入新节点,时间片切换到第二个线程进行if判断,并且也进入了插入逻辑,这时插入完第一个数之后切换到第一个线程

第一个线程进行插入逻辑,这时就导致先插入进去的头节点被覆盖掉了

## **同时put和get会导致get到null**

```java
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];  
table = newTab;
```
当普通操作触发扩容时,将新建的newTab赋值给了table,但是这时数据还没真正拷贝过去,这时时间片切换到get操作,会导致get不到数据

## **jdk7多线程扩容导致的数据丢失和缓行死锁**

```java
void transfer(Entry[] newTable, boolean rehash) {  
    int newCapacity = newTable.length;  
    for (Entry<K, V> e : table) {  
        while (null != e) {
        // 取出e的下一个数据
            Entry<K, V> next = e.next;  
            if (rehash) {  
                e.hash = null == e.key ? 0 : hash(e.key);  
            }
            // 计算当前节点的
            int i = indexFor(e.hash, newCapacity);
            // 将当前节点的next指向newTable[i]
            e.next = newTable[i];
            // 头插法插入节点中
            newTable[i] = e;
            // 给下一轮循环的e赋值  
            e = next;  
        }  
    }  
}
```

‣
