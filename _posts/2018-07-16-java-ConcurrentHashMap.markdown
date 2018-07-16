---
layout:     post
title:      "ConcurrentHashMap详解"
subtitle:   " java并发"
date:       2018-07-16 17:00:00
author:     "Aliyang"
header-img: "img/post-bg-2015.jpg"
tags:
    - 生活
---
# ConcurrentHashMap理解

[TOC]

这几天看了一些关于ConcurrentHashMap的文章，加深了理解，写篇简短的总结记录一下，免得下一次又得再找别人的文章。
ConcurrentHashMap在JDK 1.6、1.7和1.8有差别，尤其是JDK 1.8版本变化较大，JDK 1.6、1.7版本都是采用的分段锁(Segment (继承ReentrantLock))来实现对部分数据加锁，而在1.8中，进行了重新设计，加入了Node、TreeNode和TreeBin等数据结构。
## 预知识：HashMap
基本数据结构使用Entry，创建一个Entry数组，每个位置处是一个链表。
``` java
static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;
        }
        ```
当出现哈希碰撞的时候，采用链地址法解决冲突。
1.7版本使用的是链表，1.8版本使用的是红黑树，当节点个数大于8个的时候，转换为红黑树。
> 允许一个key为null，value为null的Entry，放到位置0处。

## Jdk 1.6、1.7的设计
### 总体设计
![concurrentHashMap1](https://github.com/CoderAssassin/markdownImg/blob/master/concurrentHashmap.png?raw=true "concurrentHashMap1.6/1.7")
采用分段锁的设计，同一个分段内的数据存在竞争，不同分段内的数据不存在竞争，并没有对整个Map数组进行加锁。ConcurrentHashMap存储有多个分段锁，每个分段锁内部有一个数组，数组的每个元素是HashEntry，从数组的元素的next指针找下去形成一条链表。
``` java
static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
        }
        ```
> key和value不能为null，若为null说明当前线程没有处理完而被其他线程看到。

### 重要参数
* ConcurrentLevel(并发度)
并发度就是锁的个数，一经指定不可改变。ConcurrentHashMap的<font color=red>默认并发度为16</font>，如果创建的时候用户自定义并发度，那么创建后的并发度为<font color=red>大于等于用户并发度的最小2次幂值</font>(比如自定义17最后是32)。之所以取2的幂值是因为可以直接进行位运算来扩容。
并发度太小，竞争大；并发度太大，CPU cache命中率下降。
* initialCapacity
决定每个Segment数组的大小，<font color=red>默认为16</font>。
* loadfactor
负载因子，<font color=red>默认为0.75</font>，和initialCapacity一起决定了数组的大小。
* segmentShift
分段锁数组的偏移，用于定位参与hash运算的位数，sshift是分段锁数组大小ssize从1往左移动的位数，32-sshift=segmentShift。因为分段锁数组大小最大16位(65535)，所以segmentShift最大也是65535。
* segmentMask
哈希掩码，为ssize-1，通过(hash>>>segmentShift)&segmentMask来定位在某个分段锁自身的数组里的下标。

### rehash
扩容都是针对某个Segment的HashEntry进行扩容，当加入HashEntry超出数组阈值threshold会进行扩容。假设扩容前某个HashEntry在其所在的Segment的HashEntry数组的索引为i，那么扩容后的新的数组的索引为i(个人理解：扩容是前面的Segment进行了扩容，该Segment没有进行扩容)或者i+capacity(扩容两倍)，大部分HashEntry的index可以保持不变，找到第一个index不变的HashEntry，和前面的节点重排。
``` java
private void rehash(HashEntry<K,V> node) {
           HashEntry<K,V>[] oldTable = table;
           int oldCapacity = oldTable.length;
           int newCapacity = oldCapacity << 1;
           threshold = (int)(newCapacity * loadFactor);
           HashEntry<K,V>[] newTable =
               (HashEntry<K,V>[]) new HashEntry[newCapacity];
           int sizeMask = newCapacity - 1;
           for (int i = 0; i < oldCapacity ; i++) {
               HashEntry<K,V> e = oldTable[i];
               if (e != null) {
                   HashEntry<K,V> next = e.next;
                   int idx = e.hash & sizeMask;
                   if (next == null)   //  链表只有1个实体的情况
                       newTable[idx] = e;
                   else { // 链表有多个实体的情况
                       HashEntry<K,V> lastRun = e;
                       int lastIdx = idx;
                       for (HashEntry<K,V> last = next;
                            last != null;
                            last = last.next) {//遍历到链表尾
                           int k = last.hash & sizeMask;
                           if (k != lastIdx) {
                               lastIdx = k;
                               lastRun = last;
                           }
                       }
                       newTable[lastIdx] = lastRun;
                       for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {//以链表尾节点作为终止标记，将前面所有实体重新插入新的数组
                           V v = p.value;
                           int h = p.hash;
                           int k = h & sizeMask;
                           HashEntry<K,V> n = newTable[k];
                           newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                       }
                   }
               }
           }
           //将新的实体插入新数组对应的链表头
           int nodeIndex = node.hash & sizeMask; // add the new node
           node.setNext(newTable[nodeIndex]);
           newTable[nodeIndex] = node;
           table = newTable;
       }
       ```
### 创建Segment数组
采用**延迟初始化机制**。先初始化数组第一个Segment，put的时候检查Segment是否为null，是的话调用ensureSegment()创建。

### put/putIfAbsent/putAll
获得锁，遍历链表，key相同那么更新值，否则新建HashEntry插入头部，若节点总数超过threshold，那么rehash()扩容，再写入数组。
> 1.7版本优化
申请锁时，先tryLock()，在该过程中对其hashCode对应的HashEntry的链表遍历，如果没找到key相同实体那么重新创建一个，当tryLock()一定次数后用lock()申请锁，这样做目的是使链表被CPU cache缓存。并发时put(),rehash(),remove()等导致链表头结点变化，那么需要重新扫描链表

### remove
类似put等，1.7也进行同样优化。

### get/containKey
通过getObjectVolatile()原子读获得对应链表进行遍历查找。因为是并发的，所以可能得到的结果是过时的数据，所以ConcurrentHashMap具有弱一致性。

### size/containsValue
原理：不加锁循环，遍历所有Segment，获得modcount和，若两次计算得到的一样大小，说明没有其他线程更改ConcurrentHashMap，返回值；否则对所有Segment加锁计算。

## JDK 1.8的设计
![concurrentHashMap1.8](https://github.com/CoderAssassin/markdownImg/blob/master/concurrentHashMap1.png?raw=true "concurrentHashMap1.8")

去掉Segment，采用Nodpe，TreeNode，TreeBin等数据结构，底层采用数组/链表/红黑树的设计。

#### 重要参数
* sizeCtl(控制标识符)
	* -1 
正在初始化。
	* -N
有N-1个线程在扩容。
	* 正数或0
还没有初始化，值表示下一次扩容大小。默认0.75.

#### 新的数据结构
* Node
类似于HashEntry，位于第一层次。增加了find()辅助get()。
* TreeNode
当需要将链表转换成红黑树的时候，先将Node包装成TreeNode，带有next指针。
* TreeBin
转化成红黑树时，对多个TreeNode进行封装，所以ConcurrentHashMap数组中存放的也就是TreeBin了，也是对应的红黑树的根。带有读写锁。
* ForwardingNode
连接两个数组节点，节点或者是Node或者是TreeBin，它的属性都为null，hash为-1。

#### 初始化initTable
查看sizeCtl，只能单线程进行。初始化时，sizeCtl设为-1，然后初始化，完成后设为0.75*n。

#### 扩容transfer
##### 第一步，构建nextTable
**单线程**，容量扩为原来的两倍。总体思想：遍历-复制。
* 若数组的当前位置为空，放一个forwardNode节点
* 若当前位置是Node节点，遍历，构造一个反序序列，尾节点放在新数组的i+n(n为原数组大小)位置，原来Node节点放到i位置。
* 若当前位置是TreeBin节点，也反序，然后放到i+n，原来放到i位置。
* 遍历后，设置sizeCtl为0.75*新容量。

##### 第二步，复制
![concurrentHashMap_Transfer](https://github.com/CoderAssassin/markdownImg/blob/master/concurrentHashMap3.jpg?raw=true)

**多线程**。遍历原数组，若遍历到forwardNode节点，那么跳过继续往下遍历。否则遍历对应的Node链表或者树。每处理完一个节点设置对应Node或者TreeNode/TreeBin的值为forwardNode，另一个线程看到forwardNode直接跳过往后继续。

#### put
总体思想：计算hash，遍历数组，找到对应索引位置，若位置为null，直接插入，否则遍历对应节点。若链表那么插入尾部，否则按照树的方式插入。分如下两种情况：
* 检测是否在扩容，因为transfer第一步设置了forwardNode节点，所以别的线程看到这个节点的时候，一起扩容，也就是进入多线程复制。
* 若插入位置不是forwardNode。若为空，那么直接插入；若不为空，那么加锁，然后插入。当插入后节点个数大于**8**，那么转为红黑树。

#### get
对于链表和树的情况分别查询找。

#### size
类似1.6/1.7，也是弱一致性，但是使用了一些变量和方法来提高一致性。

## Reference
1. [http://www.importnew.com/22007.html](http://www.importnew.com/22007.html)
2. [http://ifeve.com/concurrenthashmap/](http://ifeve.com/concurrenthashmap/)
