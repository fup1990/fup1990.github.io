---
title: HashMap
date: 2018-08-16 18:03:06
categories:
- 源码
tags:
- 源码
- 数据结构
- 集合
---

# 概要

HashMap 是一个关联数组、哈希表，它是线程不安全的，允许key为null,value为null。遍历时无序。  其底层数据结构是数组称之为哈希桶，每个桶里面放的是链表，链表中的每个节点，就是哈希表中的每个元素。  在JDK8中，当链表长度达到8，会转化成红黑树，以提升它的查询、插入效率，它实现了Map<K,V>, Cloneable, Serializable接口。

# 数据结构

![img](HashMap/hashmap.png)

# 构造方法

```java
//最大容量 2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;
//默认的加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//哈希桶，存放链表。 长度是2的N次方，或者初始化时为0.
transient Node<K,V>[] table;

//加载因子，用于计算哈希表元素数量的阈值。  threshold = 哈希桶.length * loadFactor;
final float loadFactor;
//哈希表内元素数量的阈值，当哈希表内元素数量超过阈值时，会发生扩容resize()。
int threshold;

public HashMap() {
	//默认构造函数，赋值加载因子为默认的0.75f
	this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
public HashMap(int initialCapacity) {
	//指定初始化容量的构造函数
	this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
//同时指定初始化容量 以及 加载因子， 用的很少，一般不会修改loadFactor
public HashMap(int initialCapacity, float loadFactor) {
	//边界处理
	if (initialCapacity < 0)
		throw new IllegalArgumentException("Illegal initial capacity: " +
										   initialCapacity);
	//初始容量最大不能超过2的30次方
	if (initialCapacity > MAXIMUM_CAPACITY)
		initialCapacity = MAXIMUM_CAPACITY;
	//显然加载因子不能为负数
	if (loadFactor <= 0 || Float.isNaN(loadFactor))
		throw new IllegalArgumentException("Illegal load factor: " +
										   loadFactor);
	this.loadFactor = loadFactor;
	//设置阈值为  》=初始化容量的 2的n次方的值
	this.threshold = tableSizeFor(initialCapacity);
}
//新建一个哈希表，同时将另一个map m 里的所有元素加入表中
public HashMap(Map<? extends K, ? extends V> m) {
	this.loadFactor = DEFAULT_LOAD_FACTOR;
	putMapEntries(m, false);
}
//根据期望容量cap，返回2的n次方形式的 哈希桶的实际容量 length。 返回值一般会>=cap 
static final int tableSizeFor(int cap) {
//经过下面的 或 和位移 运算， n最终各位都是1。
	int n = cap - 1;
	n |= n >>> 1;
	n |= n >>> 2;
	n |= n >>> 4;
	n |= n >>> 8;
	n |= n >>> 16;
	//判断n是否越界，返回 2的n次方作为 table（哈希桶）的阈值
	return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
//将另一个Map的所有元素加入表中，参数evict初始化时为false，其他情况为true
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
	//拿到m的元素数量
	int s = m.size();
	//如果数量大于0
	if (s > 0) {
		//如果当前表是空的
		if (table == null) { // pre-size
			//根据m的元素数量和当前表的加载因子，计算出阈值
			float ft = ((float)s / loadFactor) + 1.0F;
			//修正阈值的边界 不能超过MAXIMUM_CAPACITY
			int t = ((ft < (float)MAXIMUM_CAPACITY) ?
					 (int)ft : MAXIMUM_CAPACITY);
			//如果新的阈值大于当前阈值
			if (t > threshold)
				//返回一个 》=新的阈值的 满足2的n次方的阈值
				threshold = tableSizeFor(t);
		}
		//如果当前元素表不是空的，但是 m的元素数量大于阈值，说明一定要扩容。
		else if (s > threshold)
			resize();
		//遍历 m 依次将元素加入当前表中。
		for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
			K key = e.getKey();
			V value = e.getValue();
			putVal(hash(key), key, value, false, evict);
		}
	}
}
```

# Node<K,V>

 Node是HashMap的一个内部类，实现了Map.Entry接口，本质是就是一个映射(键值对)。

```java
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
```

# 扩容

当HashMap的容量达到threshold域值时，就会触发扩容。扩容前后，哈希桶的长度一定会是2的次方。扩容操作时，会new一个新的Node数组作为哈希桶，然后将原哈希表中的所有数据(Node节点)移动到新的哈希桶中，相当于对原哈希表中所有的数据重新做了一个put操作。所以性能消耗很大，可想而知，在哈希表的容量越大时，性能消耗越明显。

```java
final Node<K,V>[] resize() {
	//oldTab 为当前表的哈希桶
	Node<K,V>[] oldTab = table;
	//当前哈希桶的容量 length
	int oldCap = (oldTab == null) ? 0 : oldTab.length;
	//当前的阈值
	int oldThr = threshold;
	//初始化新的容量和阈值为0
	int newCap, newThr = 0;
	//如果当前容量大于0
	if (oldCap > 0) {
		//如果当前容量已经到达上限
		if (oldCap >= MAXIMUM_CAPACITY) {
			//则设置阈值是2的31次方-1
			threshold = Integer.MAX_VALUE;
			//同时返回当前的哈希桶，不再扩容
			return oldTab;
		}//否则新的容量为旧的容量的两倍。 
		else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
				 oldCap >= DEFAULT_INITIAL_CAPACITY)//如果旧的容量大于等于默认初始容量16
			//那么新的阈值也等于旧的阈值的两倍
			newThr = oldThr << 1; // double threshold
	}//如果当前表是空的，但是有阈值。代表是初始化时指定了容量、阈值的情况
	else if (oldThr > 0) // initial capacity was placed in threshold
		newCap = oldThr;//那么新表的容量就等于旧的阈值
	else {}//如果当前表是空的，而且也没有阈值。代表是初始化时没有任何容量/阈值参数的情况               // zero initial threshold signifies using defaults
		newCap = DEFAULT_INITIAL_CAPACITY;//此时新表的容量为默认的容量 16
		 //新的阈值为默认容量16 * 默认加载因子0.75f = 12		
		newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
	}
	if (newThr == 0) {//如果新的阈值是0，对应的是  当前表是空的，但是有阈值的情况
		float ft = (float)newCap * loadFactor;//根据新表容量 和 加载因子 求出新的阈值
		//进行越界修复
		newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
				  (int)ft : Integer.MAX_VALUE);
	}
	//更新阈值 
	threshold = newThr;
	@SuppressWarnings({"rawtypes","unchecked"})
	//根据新的容量 构建新的哈希桶
		Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
	//更新哈希桶引用
	table = newTab;
	//如果以前的哈希桶中有元素
	//下面开始将当前哈希桶中的所有节点转移到新的哈希桶中
	if (oldTab != null) {
		//遍历老的哈希桶
		for (int j = 0; j < oldCap; ++j) {
			//取出当前的节点 e
			Node<K,V> e;
			//如果当前桶中有元素,则将链表赋值给e
			if ((e = oldTab[j]) != null) {
				//将原哈希桶置空以便GC
				oldTab[j] = null;
				//如果当前链表中就一个元素，（没有发生哈希碰撞）
				if (e.next == null)
					//直接将这个元素放置在新的哈希桶里。
					//注意这里取下标 是用 哈希值 与 桶的长度-1 。 
					//由于桶的长度是2的n次方，这么做其实是等于 一个模运算。但是效率更高
					newTab[e.hash & (newCap - 1)] = e;
					//如果发生过哈希碰撞 ,而且是节点数超过8个，转化成了红黑树
				else if (e instanceof TreeNode)
					((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
				//如果发生过哈希碰撞，节点数小于8个。
				//则要根据链表上每个节点的哈希值，依次放入新哈希桶对应下标位置。
				else { // preserve order
					//因为扩容是容量翻倍，所以原链表上的每个节点。
					//现在可能存放在原来的下标，即low位， 或者扩容后的下标，即high位。 
					//high位=  low位+原哈希桶容量
					//低位链表的头结点、尾节点
					Node<K,V> loHead = null, loTail = null;
					//高位链表的头节点、尾节点
					Node<K,V> hiHead = null, hiTail = null;
					Node<K,V> next;//临时节点 存放e的下一个节点
					do {
						next = e.next;
						//这里又是一个利用位运算 代替常规运算的高效点： 
						//利用哈希值 与 旧的容量，可以得到哈希值去模后，
						//是大于等于oldCap还是小于oldCap，等于0代表小于oldCap，应该存放在低位，
						//否则存放在高位
						if ((e.hash & oldCap) == 0) {
							//给头尾节点指针赋值
							if (loTail == null)
								loHead = e;
							else
								loTail.next = e;
							loTail = e;
						}//高位也是相同的逻辑
						else {
							if (hiTail == null)
								hiHead = e;
							else
								hiTail.next = e;
							hiTail = e;
						}//循环直到链表结束
					} while ((e = next) != null);
					//将低位链表存放在原index处，
					if (loTail != null) {
						loTail.next = null;
						newTab[j] = loHead;
					}
					//将高位链表存放在新index处
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

# putValue

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
			   boolean evict) {
	//tab存放 当前的哈希桶， p用作临时链表节点  
	Node<K,V>[] tab; Node<K,V> p; int n, i;
	//如果当前哈希表是空的，代表是初始化
	if ((tab = table) == null || (n = tab.length) == 0)
		//那么直接去扩容哈希表，并且将扩容后的哈希桶长度赋值给n
		n = (tab = resize()).length;
	//如果当前index的节点是空的，表示没有发生哈希碰撞。 直接构建一个新节点Node，挂载在index处即可。
	//这里再啰嗦一下，index 是利用 哈希值 & 哈希桶的长度-1，替代模运算
	if ((p = tab[i = (n - 1) & hash]) == null)
		tab[i] = newNode(hash, key, value, null);
	else {//否则 发生了哈希冲突。
		Node<K,V> e; K k;
		//如果哈希值相等，key也相等，则是覆盖value操作
		if (p.hash == hash &&
			((k = p.key) == key || (key != null && key.equals(k))))
			e = p;//将当前节点引用赋值给e
		else if (p instanceof TreeNode)//红黑树
			e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
		else {//不是覆盖操作，则插入一个普通链表节点
			//遍历链表
			for (int binCount = 0; ; ++binCount) {
				if ((e = p.next) == null) {//遍历到尾部，追加新节点到尾部
					p.next = newNode(hash, key, value, null);
					//如果追加节点后，链表数量>=8，则转化为红黑树
					if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
						treeifyBin(tab, hash);
					break;
				}
				//如果找到了要覆盖的节点
				if (e.hash == hash &&
					((k = e.key) == key || (key != null && key.equals(k))))
					break;
				p = e;
			}
		}
		//如果e不是null，说明有需要覆盖的节点，
		if (e != null) { // existing mapping for key
			//则覆盖节点值，并返回原oldValue
			V oldValue = e.value;
			if (!onlyIfAbsent || oldValue == null)
				e.value = value;
			//这是一个空实现的函数，用作LinkedHashMap重写使用。
			afterNodeAccess(e);
			return oldValue;
		}
	}
	//如果执行到了这里，说明插入了一个新的节点，所以会修改modCount，以及返回null。
	//修改modCount
	++modCount;
	//更新size，并判断是否需要扩容。
	if (++size > threshold)
		resize();
	//这是一个空实现的函数，用作LinkedHashMap重写使用。
	afterNodeInsertion(evict);
	return null;
}

```

![img](HashMap/put.png)

# 小结

-  扩容是一个特别耗性能的操作，所以当程序员在使用HashMap的时候，估算map的大小，初始化的时候给一个大致的数值，避免map进行频繁的扩容。
-  负载因子是可以修改的，也可以大于1，但是建议不要轻易修改，除非情况非常特殊。
-  HashMap是线程不安全的，不要在并发的环境中同时操作HashMap，建议使用ConcurrentHashMap。
-  红黑树大程度优化了HashMap的性能。