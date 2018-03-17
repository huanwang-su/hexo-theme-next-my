---
title: java并发容器List/Set 
date: 2018/3/16 08:28:25
category:
- java基础
- 多线程
tag:
- 多线程 
comments: true  
---
对于List或者Set 而言，增、删操作其实都是针对整个容器，因此每次操作都不可避免的需要锁定整个容器空间，性能肯定会大打折扣。要实现一个线程安全的List/Set，只需要在修改操作的时候进行同步即可， 比如使用java.util.Collections.synchronizedList(List<T>) 或者java.util.Collections.synchronizedSet(Set<T>) 。当然也可以使用Lock 来实现线程安全的List/Set。

通常情况下我们的高并发都发生在“多读少写”的情况，因此如果能够实现一种更优秀的算法这对生产环境还是很有好处的。ReadWriteLock 当然是一种实现。CopyOnWriteArrayList/CopyOnWriteArraySet 确实另外一种思路。

CopyOnWriteArrayList/CopyOnWriteArraySet 的基本思想是一旦对容器有修改，那么就“复制”一份新的集合，在新的集合上修改，然后将新集合复制给旧的引用。当然了这部分少不了要加锁。显然对于CopyOnWriteArrayList/CopyOnWriteArraySet 来说最大的好处就是“读”操作不需要锁了。

## 读实现 ##
1. List 仍然是基于数组的实现，因为只有数组是最快的。
2. 为了保证无锁的读操作能够看到写操作的变化，因此数组array 是volatile 类型的。
3. get/indexOf/iterator 等操作都是无锁的，同时也可以看到所操作的都是某一时刻array
的镜像（这得益于数组是不可变化的）
4. add/set/remove/clear 等元素变化的都是需要加锁的，这里使用的是ReentrantLock。


	    /** The array, accessed only via getArray/setArray. */
    	private volatile transient Object[] array;
    	public E get(int index) {
    		return (E)(getArray()[index]);
    	}
    	private static int indexOf(Object o, Object[] elements,	int index, int fence) {
    		if (o == null) {
    			for (int i = index; i < fence; i++)
    				if (elements[i] == null)
    					return i;
    		} else {
    			for (int i = index; i < fence; i++)
    				if (o.equals(elements[i]))
    					return i;
    		}
    		return -1;
    	}
    	public Iterator<E> iterator() {
    		return new COWIterator<E>(getArray(), 0);
    	}

## 写实现 ##

	public void clear() {
		final ReentrantLock lock = this.lock;
		lock.lock();
    	try {
    		setArray(new Object[0]);
    	} finally {
    		lock.unlock();
    	}
    }
	
	public E set(int index, E element) {
		final ReentrantLock lock = this.lock;
		lock.lock();
		try {
			Object[] elements = getArray();
			Object oldValue = elements[index];
			if (oldValue != element) {
				int len = elements.length;
				Object[] newElements = Arrays.copyOf(elements, len);
				newElements[index] = element;
				setArray(newElements);
			} else {
	// Not quite a no-op; ensures volatile write semantics
				setArray(elements);
			}
			return (E)oldValue;
		} finally {
			lock.unlock();
		}
	}
	final void setArray(Object[] a) {
		array = a;
	}

对于set 操作，如果元素有变化，修改后setArray(newElements);将新数组赋值还好理解。那
么如果一个元素没有变化，也就是上述代码的else 部分，为什么还需要进行一个无谓的setArray 操作？

## 复制 ##
	private final CopyOnWriteArrayList<E> al;
	/**
	* Creates an empty set.
	*/
	public CopyOnWriteArraySet() {
		al = new CopyOnWriteArrayList<E>();
	}
	public boolean add(E e) {
		return al.addIfAbsent(e);
	}
对于CopyOnWriteArraySet ,只是持有一个CopyOnWriteArrayList，仅仅在add/addAll 的时候检测元素是否存在，如果存在就不加入集合中。