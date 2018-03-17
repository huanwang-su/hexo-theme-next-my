---
title: java并发容器Map
date: 2018/3/16 08:28:25
category:
- java基础
- 多线程
tag:
- 多线程 
comments: true  
---
# Map #
![](http://images.blogjava.net/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency16part1ConcurrentMap1_10A52/image_2.png)

可用的线程安全版本Map 实现是ConcurrentHashMap/ConcurrentSkipListMap/Hashtable/Properties 四个，但是Hashtable 是过时的类库，因此如果可以的应该尽可能的使用ConcurrentHashMap 和ConcurrentSkipListMap。

## ConcurrentHashMap ##
除了实现Map 接口里面对象的方法外，ConcurrentHashMap 还实现了ConcurrentMap 里面的
四个方法。
### API ###
![](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency16part1ConcurrentMap1_10A52/image_4.png)

- V putIfAbsent(K key,V value)    
  如果不存在key对应的值,则将value以key加入Map,否则返回key对应的旧值
- boolean remove(Object key,Object value)    
  只有目前将键的条目映射到给定值时，才移除该键的条目
- boolean replace(K key,V oldValue,V newValue)	<br>
  只有目前将键的条目映射到给定值时，才替换该键的条目
- V replace(K key,V value)	<br>
  只有当前键存在的时候更新此键对于的值。
### HashMap原理 ###
**哈希算法**:是将任意长度的二进制值映射为固定长度的较小二进制值

- 对象数组table
- size 描述的是Map 中元素的大小
- threshold 描述的是达到指定元素个数后需要扩容
- loadFactor 是扩容因子(loadFactor>0)，也就是计算threshold 的。
- table.length 数组的大小。
>如果存取的元素大小达到了table.length*loadFactor 个，那么就需要扩充容量了。在HashMap 中每次扩容就是将扩大数组的一倍，使数组大小为原来的两倍。

key 是一个无尽的超大集合，而table 是一个较小的有限集合，那么一种方式就是将key 编码后的hashCode 值取模映射到table 上，但是由于与(&)是比取模(%)更高效的操作，因此Java 中采用hash 值与数组大小-1后取与来确定数组索引的。

	static int indexFor(int h, int length) {
		return h & (length-1);
	}
**解决碰撞冲突**

1. 同一个索引的数组元素组成一个链表，查找允许时循环链表找到需要的元素。
2. 尽可能的将元素均匀的分布在数组上。

![](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency17part2ConcurrentMap2_FF15/image_4.png)

**第一点**
table 中每一个元素是一个Map.Entry,这里链表上所有元素的hash 经过清单1 的indexFor 后将得到相同的数组索引；next 是指向下一个元素的索引，同一个链表上的元素就是通过next 串联起来的。<br>
**第二点**

	    static int hash(int h) {
    		// This function ensures that hashCodes that differ only by
    		// constant multiples at each bit position have a bounded
    		// number of collisions (approximately 8 at default load factor).
    		h ^= (h >>> 20) ^ (h >>> 12);
    		return h ^ (h >>> 7) ^ (h >>> 4);
    	}
**第三点**构造数组时数组的长度是2的倍数,loadFactor 的默认值0.75和capacity 的默认值16是经过大量的统计分析得出的
### get操作 ###

	public V get(Object key) {
		if (key == null)
			return getForNullKey();
		int hash = hash(key.hashCode());
		for (Entry<K,V> e = table[indexFor(hash, table.length)];e != null;e = e.next) {
			Object k;
			if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
				//不同的hash 可能映射到同一个table[index]上，而相同的key 却同时映射到相同的hash 上
				return e.value;
		}
		return null;
	}
### put 操作 ###
	public V put(K key, V value) {
		if (key == null)
			return putForNullKey(value);
		int hash = hash(key.hashCode());
		int i = indexFor(hash, table.length);  //查找index
		for (Entry<K,V> e = table[i]; e != null; e = e.next) { //查找列表
			Object k;
			if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
				V oldValue = e.value;
				e.value = value;
				e.recordAccess(this);
				return oldValue;
			}
		}
		modCount++;
		addEntry(hash, key, value, i); //添加新元素
		return null;
	}
	void addEntry(int hash, K key, V value, int bucketIndex) {
		Entry<K,V> e = table[bucketIndex];
		table[bucketIndex] = new Entry<K,V>(hash, key, value, e);//同时添加在首部,next指向e
		if (size++ >= threshold)
			resize(2 * table.length);
	}
先查找,有则替换,无则添加在首部
### HashMap 扩容 ###
	void resize(int newCapacity) {
		Entry[] oldTable = table;
		int oldCapacity = oldTable.length;
		if (oldCapacity == MAXIMUM_CAPACITY) {
			threshold = Integer.MAX_VALUE;
			return;
		}
		Entry[] newTable = new Entry[newCapacity];
		transfer(newTable);
		table = newTable;
		threshold = (int)(newCapacity * loadFactor);
	}
	//重新分布Map中的元素
	void transfer(Entry[] newTable) {
		Entry[] src = table;
		int newCapacity = newTable.length;
		for (int j = 0; j < src.length; j++) {
			Entry<K,V> e = src[j];
			if (e != null) {
				src[j] = null;
				do {
					Entry<K,V> next = e.next;
					int i = indexFor(e.hash, newCapacity);
					e.next = newTable[i];
					newTable[i] = e;
					e = next;
				} while (e != null);
			}
		}
	}
扩充过程会导致元素数据的所有元素进行重新hash 计算，这个过程也叫rehash。显然这是一个非常耗时的过程，否则扩容都会导致
所有元素重新计算hash。因此尽可能的选择合适的初始化大小是有效提高HashMap 效率的关键。太大了会导致过多的浪费空间， 太小了就可能会导致繁重的rehash 过程。在这个过程中loadFactor 也可以考虑。

### ... ###

## ConcurrentHashMap原理 ##
默认情况下ConcurrentHashMap 是用了16个类似HashMap 的结构，其中每一个HashMap 拥有一个独占锁。也就是说最终的效果就是通过某种Hash 算法，将任何一个元素均匀的映射到某个HashMap 的Map.Entry 上面，而对某个一个元素的操作就集中在其分布的HashMap上，与其它HashMap 无关。这样就支持最多16个并发的写操作。
![](http://7xsn1m.com1.z0.glb.clouddn.com/ConcurrentMap%E7%B1%BB%E5%9B%BE.png)
ConcurrentHashMap 将整个对象列表分为segmentMask+1个片段（Segment）。其中每一个片段是一个类似于HashMap 的结构，它有一个HashEntry 的数组，数组的每一项又是一个链表，通过HashEntry 的next 引用串联起来。

对于一次Map 的查找来说，首先就需要定位到Segment，然后从过Segment 定位到HashEntry 链表，最后才是通过遍历链表得到需要的元素。

要解决并发问题，加锁是必不可免的。Segment 除了有一个volatile 类型的元素大小count 外， Segment 还是集成自ReentrantLock 的。如果是读操作不加锁，写操作加锁，对于竞争资源来说就需要定义为volatile 类型的。所以volatile 能够近似保证正确性的情况下最大程度的降低加锁带来的影响，同时还与写操作的锁不产生冲突。

同时为了防止在遍历HashEntry 的时候被破坏，那么对于HashEntry 的数据结构来说，除了value 之外其他属性就应该是常量， 否则不可避免的会得到ConcurrentModificationException。这就是为什么HashEntry 数据结构中key,hash,next 是常量的原因(final 类型）。

### 定位Segment ###
	private static int hash(int h) {
		h += (h << 15) ^ 0xffffcd7d;
		h ^= (h >>> 10);
		h += (h << 3);
		h ^= (h >>> 6);
		h += (h << 2) + (h << 14);
		return h ^ (h >>> 16);
	}
	final Segment<K,V> segmentFor(int hash) {
		return segments[(hash >>> segmentShift) & segmentMask];
	}
显然在不能够对Segment 扩容的情况下， segments 的大小就应该是固定的。所以在
ConcurrentHashMap 中segments/segmentMask/segmentShift 都是常量，一旦初始化后就不能被再次修改，其中segmentShift 是查找Segment 的一个常量偏移量。

有了Segment 以后再定位HashEntry 就和HashMap 中定位HashEntry 一样了，先将hash 值与Segment 中HashEntry 的大小减1进行与操作定位到HashEntry 链表，然后遍历链表就可以完成相应的操作了。

### get操作 ###
	V get(Object key, int hash) {
		if (count != 0) { // read-volatile
			HashEntry<K,V> e = getFirst(hash);
			while (e != null) {
				if (e.hash == hash && key.equals(e.key)) {
					V v = e.value;
					if (v != null)
						return v;
					//如果为空还需要加锁再读取一次,尽管ConcurrentHashMap 不允许将value 为null 的值加入，但现在仍然能
					//够读到一个为空的value 就意味着此值对当前线程还不可见（这是因为HashEntry 还没有完全构造完成就赋值导致的，
					return readValueUnderLock(e); // recheck
				}
				e = e.next;
			}
		}
		return null;
	}
	HashEntry<K,V> getFirst(int hash) {
		HashEntry<K,V>[] tab = table;
		return tab[hash & (tab.length - 1)];
	}
	V readValueUnderLock(HashEntry<K,V> e) {
		lock();
		try {
			return e.value;
		} finally {
			unlock();
		}
	}
描述在Segment中如何定位元素。首先判断Segment 的大小count>0，Segment 的大小描述的是HashEntry 不为空(key 不为空)的个数。如果Segment 中存在元素那么就通过
getFirst 定位到指定的HashEntry 链表的头节点上，然后遍历此节点，一旦找到key 对应的元素后就返回其对应的值。
### put操作 ###
修改一个竞争资源肯定是要加锁的

	V put(K key, int hash, V value, boolean onlyIfAbsent) {
		lock();
		try {
			int c = count;
			if (c++ > threshold) // ensure capacity
				rehash();
			HashEntry<K,V>[] tab = table;
			int index = hash & (tab.length - 1);
			HashEntry<K,V> first = tab[index];
			HashEntry<K,V> e = first;
			while (e != null && (e.hash != hash || !key.equals(e.key))) //找到e
				e = e.next;
			V oldValue;
			if (e != null) {
				oldValue = e.value;
				if (!onlyIfAbsent)
					e.value = value;
			}
			else {
				oldValue = null;
				++modCount;
				tab[index] = new HashEntry<K,V>(key, hash, first, value);
				count = c; // write-volatile
			}
			return oldValue;
		} finally {
			unlock();
		}
	}
### remove ###
同put 一样，remove 也需要加锁，这是因为对table 可能会有变更。由于HashEntry 的next 节点是final 类型的，所以一旦删除链表中间一个元素，就需要将删除之前或者之后的元素重新加入新的链表。而Segment 采用的是将删除元素之前的元素一个个重新加入删除之后的元素之前（也就是链表头结点）来完成新链表的构造。

	V remove(Object key, int hash, Object value) {
		lock();
		try {
			int c = count - 1;
			HashEntry<K,V>[] tab = table;
			int index = hash & (tab.length - 1);
			HashEntry<K,V> first = tab[index];
			HashEntry<K,V> e = first;
			while (e != null && (e.hash != hash || !key.equals(e.key)))
				e = e.next;
			V oldValue = null;
			if (e != null) {
				V v = e.value;
				if (value == null || value.equals(v)) {
					oldValue = v;
					//将删除元素之前的元素一个个重新加入删除之后的元素之前（也就是链表头结点）来完成新链表的构造。
					//新构建了列表,复杂,因为是final
					++modCount;
					HashEntry<K,V> newFirst = e.next;
					for (HashEntry<K,V> p = first; p != e; p = p.next)
						newFirst = new HashEntry<K,V>(p.key, p.hash,newFirst, p.value);
					tab[index] = newFirst;
					count = c; // write-volatile
				}
			}
			return oldValue;
		} finally {
			unlock();
		}
	}
例如原来列表是 1->2->3->4->5 删除3后为 2->1->4->5
### size操作 ###
size 操作涉及到统计所有Segment 的大小，这样就会遍历所有的Segment，如果每次加锁就会导致整个Map 都被锁住了，任何需要锁的操作都将无法进行。这里用到了一个比较巧妙的方案解决此问题。

在Segment 中有一个变量modCount，用来记录Segment 结构变更的次数，结构变更包括增加元素和删除元素，每增加一个元素操作就+1，每进行一次删除操作+1，每进行一次清空操作(clear)就+1。也就是说每次涉及到元素个数变更的操作modCount 都会+1，而且一直是增大的，不会减小。遍历两次ConcurrentHashMap 中的segments，每次遍历是记录每一个Segment 的modCount，比较两次遍历的modCount 值的和是否相同，如果相同就返回在遍历过程中获取的Segment的count 的和，也就是所有元素的个数。如果不相同就重复再做一次。重复一次还不相同就将所有Segment 锁住，一个一个的获取其大小(count)，最后将这些count 加起来得到总的大小。当然了最后需要将锁一一释放

	public int size() {
		final Segment<K,V>[] segments = this.segments;
		long sum = 0;
		long check = 0;
		int[] mc = new int[segments.length];
		//用不加锁方法检测RETRIES_BEFORE_LOCK次
		for (int k = 0; k < RETRIES_BEFORE_LOCK; ++k) {
			check = 0;
			sum = 0;
			int mcsum = 0;
			for (int i = 0; i < segments.length; ++i) {
				//顺序不可换。所以修改modCount 总是在修改count之前，也就是说如果读取到了一个count 
				//的值，那么在count 变化之前的modCount 也就能够读取到
				sum += segments[i].count;
				mcsum += mc[i] = segments[i].modCount;
			}
			if (mcsum != 0) {
				for (int i = 0; i < segments.length; ++i) {
					check += segments[i].count;
					if (mc[i] != segments[i].modCount) {
						check = -1; // force retry
						break;
					}
				}
			}
			if (check == sum)
				break;
		}
		//加锁检查
		if (check != sum) { // Resort to locking all segments
			sum = 0;
			for (int i = 0; i < segments.length; ++i)
				segments[i].lock();
			for (int i = 0; i < segments.length; ++i)
				sum += segments[i].count;
			for (int i = 0; i < segments.length; ++i)
				segments[i].unlock();
		}
		//返回结果
		if (sum > Integer.MAX_VALUE)
			return Integer.MAX_VALUE;
		else
			return (int)sum;
	}









