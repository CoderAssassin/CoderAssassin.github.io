---
layout:     post
title:      "Fail-Fast和Fail-Safe详解"
subtitle:   " Fail-Fast 和 Fail-Safe "
date:       2018-08-31 21:47:00
author:     "Aliyang"
header-img: "img/post-bg-java-FailFast.jpg"
tags:
    - Java
---
## 什么是fail-fast和fail-safe？
* fail-fast(快速失败)：快速失败和**java.util**包下的容器类有关，比如ArrayList，HashSet等。在平时使用的时候，我们会获取对应的容器的Iterator对象，然后通过Iterator对象来遍历所有的元素。在多线程环境中，假设线程A利用Iterator对ArrayList的对象进行遍历的时候，此时若线程B对ArrayList对象的结构进行改变(添加或者删除元素)，这个时候Iterator在遍历的时候就会出错，抛出**ConcurrentModificationException**异常。
	* 具体的原理是：在这些容器中，都有**modCount和expectedModCount**两个变量，这两个变量用来判断并发情况下原容器对象是否发生了结构上的改变。在Iterator容器里的每个方法开始的时候，都会判断expectedModCound是否等于modCount，当不相等的时候就会抛出ConcurrentModificationException异常，终止程序。
	* 对于快速失败，可以看我的另一篇博客[Iterator详解](https://coderassassin.github.io/2018/08/31/java-Iterator/)来具体了解其原理
* fail-safe(安全失败)：安全失败和**java.util.concurrent**包下的并发容器有关，这种机制并不像快速失败一样，在原容器对象发生结构上的修改时会终止进程并抛出异常。根本原因在于：**安全失败是在原容器的副本上进行迭代的，因此，原容器发生结构上的改变的时候是不会影响到迭代的，当迭代完成之后会替换原来的容器**。
	* 缺陷：在迭代的时候可能当前的容器副本不是最新的；并且因为是创建原容器对象的副本，所以会增加一点时间和空间的开销。

## CopyOnWriteArrayList的迭代
CopyOnWrite(写时复制)在java.util.concurrent包中有两种实现：CopyOnWriteArrayList和CopyOnWriteArraySet。这里我们以CopyOnWriteArrayList为例来讲解。
下面看下CopyOnWriteArrayList里的迭代器：
``` java
	//Iterator和ListIterator获取的都是一样的CowIterator对象
	public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0);
    }
    public ListIterator<E> listIterator() {
        return new COWIterator<E>(getArray(), 0);
    }
    public ListIterator<E> listIterator(int index) {
        Object[] elements = getArray();
        int len = elements.length;
        if (index < 0 || index > len)
            throw new IndexOutOfBoundsException("Index: "+index);

        return new COWIterator<E>(elements, index);
    }
```
上面三个方法是我们从CopyOnWriteArrayList获取对象的方法，其实都一样，返回的都是**COWIterator**的实例，所以关键是COWIterator类，这是CopyOnWriteArrayList的静态内部类：
``` java
	static final class COWIterator<E> implements ListIterator<E> {
    	private final Object[] snapshot;//当前时刻并发容器的快照
        private int cursor;
        
        private COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            snapshot = elements;
        }
        
        public boolean hasNext() {
            return cursor < snapshot.length;
        }

        public boolean hasPrevious() {
            return cursor > 0;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            if (! hasPrevious())
                throw new NoSuchElementException();
            return (E) snapshot[--cursor];
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor-1;
        }

		//注意：删除/修改和添加操作都会抛出异常，不允许这样的操作
        public void remove() {
            throw new UnsupportedOperationException();
        }

        public void set(E e) {
            throw new UnsupportedOperationException();
        }

        public void add(E e) {
            throw new UnsupportedOperationException();
        }

        @Override
        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            Object[] elements = snapshot;
            final int size = elements.length;
            for (int i = cursor; i < size; i++) {
                @SuppressWarnings("unchecked") E e = (E) elements[i];
                action.accept(e);
            }
            cursor = size;
        }
    }
```
COWIterator类也不是很复杂，可以说COWIterator是Iterator和ListIterator的方法结合，但是有两个地方不一样：

* COWIterator是复制一份现有元素数组的快照到snapshot
* COWIterator若执行remove()、set()和add()操作都会报异常

注意：**COWIterator里的快照snapshot只是简单地创建一个新的对象指向现有的元素数组elements，其实并算不上真正的复制，只是对原数组的引用，也就是说，对原数组的修改操作，snapshot是能看出来的。**
那么，CopyOnWrite的“写时复制”体现在哪里？我们看一下具体的对CopyOnWriteArrayList的修改操作是怎样的，以add为例：
``` java
	public boolean add(E e) {
        final ReentrantLock lock = this.lock;//加可重入锁
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);//关键点，数组复制
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
看一下上边的代码就知道什么叫“写时复制”了。因为是并发修改，所以首先加可重入锁，注意在fail-fast里是不会加锁的，所以会出现问题。关键在于**Arrays.copyOf(elements,len+1)**，这句话是复制一份比原数组长度多1的快照数组，**注意：这个方法还是浅复制！！！**
如果在调用Iterator的时候对原容器对象进行添加/删除/修改(修改整个元素对象)的话，那么迭代器是发现不了的，这就体现了fail-safe的特点。**注意：如果是修改的某个元素的引用对象所指向的对象的相关属性的话，在Iterator里边还是能发现的，因为是浅拷贝。**
**因为在迭代的过程中，原容器对象里的元素是可以发生改变的，所以CopyOnWriteArrayList是弱一致性的。**

## ConcurrentHashMap的迭代
ConcurrentHashMap里边的迭代就复杂了些，这里根据key、value和Entry分别建立了不同的迭代类，我们以key为例看一下是如何实现的(jdk 1.8版本代码)：
``` java
	//集合视图类
	abstract static class CollectionView<K,V,E>
        implements Collection<E>, java.io.Serializable {
        	final ConcurrentHashMap<K,V> map;
            CollectionView(ConcurrentHashMap<K,V> map)  { this.map = map; }
            ....
            public abstract Iterator<E> iterator();
        }
```
上边是一个抽象的静态内部类，就是对ConcurrentHashMap里边元素审查的一个类，有好几种实现类，关键在于里边的抽象方法iterator()是我们最熟悉的，以其中一个实现类KeySetView为例看一下具体的实现：
``` java
	public static class KeySetView<K,V> extends CollectionView<K,V,K>
        implements Set<K>, java.io.Serializable {
        	....
            public Iterator<K> iterator() {
                Node<K,V>[] t;
                ConcurrentHashMap<K,V> m = map;
                int f = (t = m.table) == null ? 0 : t.length;
                return new KeyIterator<K,V>(t, f, 0, f, m);
        	}
        }
```
具体是按key、value和Entry定义迭代器的，看一下KeyIterator类：
``` java
	//基本的迭代器类
	static class BaseIterator<K,V> extends Traverser<K,V> {
        final ConcurrentHashMap<K,V> map;
        Node<K,V> lastReturned;//最新的返回节点
        BaseIterator(Node<K,V>[] tab, int size, int index, int limit,
                    ConcurrentHashMap<K,V> map) {
            super(tab, size, index, limit);
            this.map = map;
            advance();//获取第一个有效节点
        }

        public final boolean hasNext() { return next != null; }
        public final boolean hasMoreElements() { return next != null; }
		//定义删除节点方法，逻辑和Iterator的next()方法很类似
        public final void remove() {
            Node<K,V> p;
            if ((p = lastReturned) == null)
                throw new IllegalStateException();
            lastReturned = null;
            map.replaceNode(p.key, null, null);
        }
    }
	//对key的迭代类
	static final class KeyIterator<K,V> extends BaseIterator<K,V>
        implements Iterator<K>, Enumeration<K> {
        KeyIterator(Node<K,V>[] tab, int index, int size, int limit,
                    ConcurrentHashMap<K,V> map) {
            super(tab, index, size, limit, map);
        }

        public final K next() {
            Node<K,V> p;
            if ((p = next) == null)
                throw new NoSuchElementException();
            K k = p.key;
            lastReturned = p;
            advance();
            return k;
        }

        public final K nextElement() { return next(); }
    }
```
上述代码，也是有个迭代器基类BaseIterator，以及根据key、value和Entry的分别的迭代器子类。BaseIterator继承自Traverser，advance()也是这个类里边的方法，接下来我们继续看这个类里边的实现：
``` java
	static class Traverser<K,V> {
    	Node<K,V>[] tab;        // 当前数组列表
        Node<K,V> next;         // 下一个实体节点
        TableStack<K,V> stack, spare; // to save/restore on ForwardingNodes
        int index;              // 下一个bin
        int baseIndex;          // 初始表的当前索引
        int baseLimit;          // 初始表的索引边界
        final int baseSize;     // 初始表大小
        
        Traverser(Node<K,V>[] tab, int size, int index, int limit) {
            this.tab = tab;
            this.baseSize = size;
            this.baseIndex = this.index = index;
            this.baseLimit = limit;
            this.next = null;
        }
        ....
        //拿到下一个合理的节点
        final Node<K,V> advance() {
            Node<K,V> e;
            if ((e = next) != null)
                e = e.next;
            for (;;) {//遍历
                Node<K,V>[] t; int i, n;  // must use locals in checks
                if (e != null)
                    return next = e;
                if (baseIndex >= baseLimit || (t = tab) == null ||
                    (n = t.length) <= (i = index) || i < 0)
                    return next = null;
                if ((e = tabAt(t, i)) != null && e.hash < 0) {
                    if (e instanceof ForwardingNode) {//是ForwardingNode，跳到下一个数组元素
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        e = null;
                        pushState(t, i, n);
                        continue;
                    }
                    else if (e instanceof TreeBin)//是红黑树，拿到树的第一个节点
                        e = ((TreeBin<K,V>)e).first;
                    else
                        e = null;
                }
                if (stack != null)
                    recoverState(n);
                else if ((index = i + baseSize) >= n)
                    index = ++baseIndex; // visit upper slots if present
            }
        }
    }
```
上面代码是ConcurrentHashMap里边对迭代器的大致实现，这里传过去的表还是原来的引用，接下来，以put()方法为例，看下在迭代的时候，是如何对容器对象进行修改的：
``` java
	public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();//初始化表
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {//直接插入
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)//碰到ForwardingNode，进行元素转移
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {//链表
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {//遍历链表
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
                        else if (f instanceof TreeBin) {//遍历红黑树
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
                    if (binCount >= TREEIFY_THRESHOLD)//当前链表节点个数大于等于8
                        treeifyBin(tab, i);//判断是扩容还是转红黑树
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
看了下ConcurrentHashMap里边的put()操作，发现和CopyOnWriteArrayList很不一样，这里并没有复制一份原来数组的快照，而是直接在原数组上操作，迭代的时候也是在原数组上操作，因此，**ConcurrentHashMap是弱一致性的，中间数组结构的变化都会引起迭代的变化。**
