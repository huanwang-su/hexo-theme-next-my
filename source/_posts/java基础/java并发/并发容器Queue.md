---
title: java并发容器queue
date: 2018/3/16 08:28:25
category:
- java基础
- 多线程
tag:
- 多线程 
comments: true  
---

# Queue #
![](http://i.imgur.com/bKZm2Wd.png)

Queue并不是线程安全的，为了解决这个问题，引入了可阻塞的队列BlockingQueue。对于BlockingQueue 而言所有操作的是线程安全的，并且队列的操作可以被阻塞，直到满足某种条件。

![](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/ead4e8800e0c_FD45/image_4.png)

## ConcurrentLinkedQueue ##
一个基于链接节点的无界线程安全队列。此队列按照FIFO（先进先出）原则对元素进行排序。

![](http://images.blogjava.net/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency20part5ConcurrentLinkedQu_C9AC/image_2.png)

1. 所有结构（head/tail/item/next）都是volatile 类型。
2. 所有结构的操作都带有原子操作
3. 由于队列中任何一个节点（Node）只有下一个节点的引用，所以这个队列是单向的
4. 没有对队列长度进行计数，所以队列的长度是无限的
5. 初始情况下队列头和队列尾都指向一个空节点，但是非null，这是为了方便操作，不需要每次去判断head/tail 是否为空。但是head 却不作为存取元素的节点，tail 在不等于head 情况下保存一个节点元素

### 入队 ###

	public boolean offer(E e) {
		if (e == null) throw new NullPointerException();
		Node<E> n = new Node<E>(e, null);
		for (;;) {
			Node<E> t = tail;
			Node<E> s = t.getNext();
			if (t == tail) {
				if (s == null) {
					if (t.casNext(s, n)) {
						casTail(t, n);
						return true;
					}
				} else {
					casTail(t, s);
				}
			}
		}
	}
CAS 操作来修改.包括几个if();
### 出队 ###

	public E poll() {
		for (;;) {
			Node<E> h = head;
			Node<E> t = tail;
			Node<E> first = h.getNext();
			if (h == head) {
				if (h == t) {
					if (first == null)
						return null;
					else
						casTail(t, first);
				} else if (casHead(h, first)) {
					E item = first.getItem();
					if (item != null) {
						first.setItem(null);
						return item;
					}
	// else skip over deleted item, continue loop,
				}
			}
		}
	}
头结点是为了标识队列起始，也为了减少空指针的比较，所以头结点总是一个item 为null的非null 节点。也就是说head!=null 并且head.item==null 总是成立。所以实际上获取的是head.next，一旦将头结点head 设置为head.next 成功就将新head 的item 设置为null。至于以前就的头结点h，h.item=null 并且h.next 为新的head，但是由于没有对h 的引用，所以最终会被GC回收。
### 遍历队列大小 ###
	public int size() {
		int count = 0;
		for (Node<E> p = first(); p != null; p = p.getNext()) {
			if (p.getItem() != null) {
				// Collections.size() spec says to max out
				if (++count == Integer.MAX_VALUE)
					break;
			}
		}
		return count;
	}
## BlockingQueue ##

![](http://i.imgur.com/zY7oAk6.png)

对于Queue 来说，BlockingQueue 是主要的线程安全版本。这是一个可阻塞的版本，也就是允许添加/删除元素被阻塞，直到成功为止。

BlockingQueue 相对于Queue 而言增加了两个操作：put/take

## LinkedBlockingQueue 原理 ##
![](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency21part5ConcurrentLinkedQu_E370/image_4.png)

引入了两个锁，一个入队列锁，一个出队列锁。当然同时有一个队列不满的Condition和一个队列不空的Condition。一个锁意味着入队列和出队列同时只能有一个在进行，另一个必须等待其释放锁。而从ConcurrentLinkedQueue 的实现原理来看，事实上head 和last (ConcurrentLinkedQueue 中是tail)是分离的，互相独立的，这意味着入队列实际上是不会修改出队列的数据的，同时出队列也不会修改入队列，也就是说这两个操作是互不干扰的。
### 阻塞的入队列 ###

	public void put(E e) throws InterruptedException {
		if (e == null) throw new NullPointerException();
		int c = -1;
		final ReentrantLock putLock = this.putLock;
		final AtomicInteger count = this.count;
		putLock.lockInterruptibly();
		try {
			try {
				while (count.get() == capacity)
					notFull.await();
				} catch (InterruptedException ie) {
					notFull.signal(); // propagate to a non-interrupted thread
					throw ie;
				}
			insert(e);
			c = count.getAndIncrement();
			if (c + 1 < capacity)
				notFull.signal();
			} finally {
				putLock.unlock();
			}
		if (c == 0)
		signalNotEmpty();
	}

- 如果在入队列的时候线程被中断，那么就需要发出一个notFull 的信号，表示下一个入队列的线程能够被唤醒（如果阻塞的话）。
- 入队列成功后如果队列不满需要补一个notFull 的信号。
- 入队列的过程允许被中断，所以总是抛出InterruptedException 异常。
- 如果队列不为空并且可能有一个元素的话就唤醒一个出队列线程。这么做说明之前队列一定为空，因为在加入队列之后队列最多只能为1，那么说明未加入之前是0，那么就可能有被阻塞的出队列线程，所以就唤醒一个出队列线程

### notify丢失通知问题 ###

假设线程A 因为某种条件在条件队列中等待，同时线程B 因为另外一种条件在同一个条件队列中等待，也就是说线程A/B 都被同一个Conditon.await()挂起，但是等待的条件不同。现在假设线程B 的线程被满足，线程C 执行一个notify 操作，此时JVM 从Conditon.await()的多个线程（A/B）中随机挑选一个唤醒，不幸的是唤醒了A。此时A 的条件不满足，于是A 继续挂起。而此时B 仍然在傻傻的等待被唤醒的信号。也就是说本来给B 的通知却被一个无关的线程持有了，真正需要通知的线程B 却没有得到通知，而B 仍然在等待一个已经发生过的通知。

调用notifyall 会唤醒所有线程，然后这N 个线程竞争同一个锁，最多只有一个线程能够得到锁，于是其它线程又回到挂起状态。这意味每一次唤醒操作可能带来大量的上下文切换（如果N 比较大的话），同时有大量的竞争锁的请求。
### 出队列过程 ###
	public E take() throws InterruptedException {
		E x;
		int c = -1;
		final AtomicInteger count = this.count;
		final ReentrantLock takeLock = this.takeLock;
		takeLock.lockInterruptibly();
		try {
			try {
				while (count.get() == 0)
					notEmpty.await();
				} catch (InterruptedException ie) {
					notEmpty.signal(); // propagate to a non-interrupted thread
					throw ie;
				}
				x = extract();
				c = count.getAndDecrement();
				if (c > 1)
					notEmpty.signal();
			} finally {
				takeLock.unlock();
			}
		if (c == capacity)
			signalNotFull();
		return x;
	}
1.  获取出队列的锁takeLock，检测队列大小，如果队列为空，那么就挂起线程，等待队列不为空notEmpty 的唤醒。
1.  将元素从头部移除，同时修改队列头部引用head。
1.  队列大小减1。
1.  释放锁takeLock。
1.  唤醒notFull 线程（如果有挂起的入队列线程），告诉生产者，现在还有空闲的空间。
### 为什么有异常 ###
在锁机制里面也是总遇到，这是因为，Java 里面没有一种直接的方法中断一个挂起的线程，所以通常情况下等于一个处于WAITING 状态的线程，允许设置一个中断位，一旦线程检测到这个中断位就会从WAITING 状态退出，以一个InterruptedException 的异常返回。所以只要是对一个线程挂起操作都会导致InterruptedException 的可能，比如Thread.sleep()、Thread.join()、Object.wait()。尽管LockSupport.park()不会抛出一个InterruptedException 异常，但是它会将当前线程的的interrupted 状态位置上，而对于Lock/Condition 而言，当捕捉到interrupted 状态后就认为线程应该终止任务，所以就抛出了一个InterruptedException 异常。

### 查询队列头元素 ###
	public E peek() {
		if (count.get() == 0)
			return null;
		final ReentrantLock takeLock = this.takeLock;
		takeLock.lock();
		try {
			Node<E> first = head.next;
			if (first == null)
				return null;
			else
				return first.item;
		} finally {
			takeLock.unlock();
		}
	}
这里读取了Node的item 值，但是整个过程却是使用了takeLock 而非putLock。换句话说putLock 对Node.item的操作，peek()线程可能不可见！所以Node.item是**volatile**的
### 队列尾部加入元素 ###
	private void insert(E x) {
		last = last.next = new Node<E>(x);
	}

last=new Node<E>(x)可能发生重排序

- 构建一个Node 对象n；
- 将Node 的n 赋给last
- 初始化n，设置item=x

在第二步peek 线程可能拿到了新的Node n，这时候它读取item，得到了一个null。显然这是不可靠的。
所以对item 采用volatile.

出队了poll/take 和peek 都是使用的takeLock 锁，所以不会导致此问题。

删除操作和遍历操作由于同时获取了takeLock 和putLock，所以也不会导致此问题。

### 批量从队列中移除元素 ###
由于批量操作只需要一次获取锁，所以效率会比每次获取锁要高。但是需要说明的，需要同时获取takeLock/putLock 两把锁，因为当移除完所有元素后这会涉及到尾节点的修改（last 节点仍然指向一个已经移走的节点）。

	public int drainTo(Collection<? super E> c, int maxElements) {
		if (c == null)
			throw new NullPointerException();
		if (c == this)
			throw new IllegalArgumentException();
		fullyLock();
		try {
			int n = 0;
			Node<E> p = head.next;
			while (p != null && n < maxElements) {
				c.add(p.item);
				p.item = null;
				p = p.next;
				++n;
			}
			if (n != 0) {
				head.next = p;
				assert head.item == null;
				if (p == null)
					last = head;
				if (count.getAndAdd(-n) == capacity)
					notFull.signalAll();
			}
			return n;
		} finally {
			fullyUnlock();
		}
	}

## ArrayBlockingQueue 原理 ##
1. 入队列就将尾索引往右移动一个，新元素加入尾索引的位置；
2. 出队列就将头索引往尾索引方向移动一个，同时将旧头索引元素设为null，返回旧头索引的元素。
3. 一旦数组已满，那么就不允许添加新元素（除非扩充容量）
4. 如果尾索引移到了数组的最后（最大索引处），那么就从索引0开始，形成一个“闭合”的数组。
5. 由于头索引和尾索引之间的元素都不能为空（因为为空不知道take 出来的元素为空还是队列为空），所以删除一个头索引和尾索引之间的元素的话，需要移动删除索引前面或者后面的所有元素，以便填充删除索引的位置。
6. 由于是阻塞队列，那么显然需要一个锁，另外由于只是一份数据（一个数组），所以只能有一个锁，也就是同时只能有一个线程操作队列

![](http://images.blogjava.net/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency22part7BlockingQueue2_1216F/image_4.png)

首先有一个数组E[]，用来存储所有的元素。由于ArrayBlockingQueue 最终设置为一个不可扩展大小的Queue，所以这里items 就是初始化就固定大小的数组（final 类型）；另外有两个索引，头索引takeIndex，尾索引putIndex；一个队列的大小count；要支持阻塞就必须需要一个锁lock 和两个条件（非空、满），这三个元素都是不可变更类型的（final）。

### 添加元素 ###
先判断数量,再决定挂起

	public void put(E e) throws InterruptedException {
		if (e == null) throw new NullPointerException();
		final E[] items = this.items;
		final ReentrantLock lock = this.lock;
		lock.lockInterruptibly();
		try {
			try {
				while (count == items.length)
					notFull.await();
			} catch (InterruptedException ie) {
				notFull.signal(); // propagate to non-interrupted thread
				throw ie;
			}
			insert(e);
		} finally {
			lock.unlock();
		}
	}

### 移除元素 ###
先判断数量,再决定挂起

	public E take() throws InterruptedException {
		final ReentrantLock lock = this.lock;
		lock.lockInterruptibly();
		try {
			try {
				while (count == 0)
					notEmpty.await();
			} catch (InterruptedException ie) {
				notEmpty.signal(); // propagate to non-interrupted thread
				throw ie;
			}
			E x = extract();
			return x;
		} finally {
			lock.unlock();
		}
	}

### 数据结构 ###
![](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency22part7BlockingQueue2_1216F/image_10.png)

这个队列是循环的
### 删除任意一个元素 ###
	public boolean remove(Object o) {
		if (o == null) return false;
		final E[] items = this.items;
		final ReentrantLock lock = this.lock;
		lock.lock();
		try {
			int i = takeIndex;
			int k = 0;
			for (;;) {
				if (k++ >= count)
					return false;
				if (o.equals(items[i])) {
					removeAt(i);
					return true;
				}
				i = inc(i);
			}
		} finally {
			lock.unlock();
		}
	}
	void removeAt(int i) {
		final E[] items = this.items;
	// if removing front item, just advance
		if (i == takeIndex) {
			items[takeIndex] = null;
			takeIndex = inc(takeIndex);
		} else {
	// slide over all others up through putIndex.
			for (;;) {
				int nexti = inc(i);
				if (nexti != putIndex) {
					items[i] = items[nexti];
					i = nexti;
				} else {
					items[i] = null;
					putIndex = i;
					break;
				}
			}
		}
		--count;
		notFull.signal();
	}

对于其他的操作，由于都是带着Lock 的操作
## PriorityBlockingQueue ##
PriorityBlockingQueue 是无界的，因此就只有非空的信号，也就是说只有take()才能阻塞， put 是永远不会阻塞（ 除非达到Integer.MAX_VALUE 直到抛出一个OutOfMemoryError 异常）。

只有take()操作的时候才可能因为队列为空而挂起。同时其它需要操作队列变化和大小的只需要使用独占锁ReentrantLock 就可以了，非常方便。需要说明的是PriorityBlockingQueue采用了一个公平的锁

## 直接交换的 BlockingQueue—SynchronousQueue ##每个插入操作必须等待另一个线程的移除操作，同样任
何一个移除操作都等待另一个线程的插入操作。因此此队列内部其实没有任何一个元素，或者说容量是0，严格说并不是一种容器。由于队列没有容量，因此不能调用peek 操作，因为只有移除元素时才有元素。

SynchronousQueue 内部没有容量，但是由于一个插入操作总是对应一个移除操作，反过来同样需要满足。那么一个元素就不会再SynchronousQueue 里面长时间停留，一旦有了插入线程和移除线程，元素很快就从插入线程移交给移除线程。也就是说这更像是一种信道（管道），资源从一个方向快速传递到另一方向。

尽管元素在SynchronousQueue 内部不会“停留”，但是并不意味之SynchronousQueue 内部没有队列。实际上SynchronousQueue 维护者线程队列，也就是插入线程或者移除线程在不同时存在的时候就会有线程队列。既然有队列，同样就有公平性和非公平性特性，公平性保证正在等待的插入线程或者移除线程以FIFO 的顺序传递资源。

显然这是一种快速传递元素的方式，也就是说在这种情况下元素总是以最快的方式从插入着（生产者）传递给移除着（消费者），这在多任务队列中是最快处理任务的方式。

## DelayQueue ##
它描述的是一种延时队列。这个队列的特性是，队列中的元素都要延迟时间（超时时间），只有一个元素达到了延时时间才能出队列，也就是说每次从队列中获取的元素总是最先到达延时的元素。这种队列的场景就是计划任务。比如以前要完成计划任务，很有可能是使用Timer/TimerTask，这是一种循环检测的方式，也就是在循环里面遍历所有元素总是检测元素是否满足条件，一旦满足条件就执行相关任务。显然这中方式浪费了很多的检测工作，因为大多数时间总是在进行无谓的检测。而DelayQueue 却能避免这种无谓的检测

## 单向队列总结 ##
![](http://images.blogjava.net/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency23part8BlockingQueue3_1086D/image_2.png)

如果不需要阻塞队列，优先选择ConcurrentLinkedQueue；如果需要阻塞队列，队列大小固定优先选择ArrayBlockingQueue，队列大小不固定优先选择LinkedBlockingQueue；如果需要对队列进行排序，选择PriorityBlockingQueue；如果需要一个快速交换的队列，选择SynchronousQueue；如果需要对队列中的元素进行延时操作，则选择DelayQueue。

## Deque ##
![](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency24part9Deque_1425C/image_2.png)

![](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency24part9Deque_1425C/image_4.png)
### ArrayDeque ###
![](http://www.blogjava.net/images/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency24part9Deque_1425C/image_6.png)

ArrayDeque 并不是一个固定大小的队列，每次队列满了以后就将队列容量扩大一倍（doubleCapacity()），因此加入一个元素总是能成功，而且也不会抛出一个异常。

### LinkList ###
![](http://images.blogjava.net/blogjava_net/xylz/WindowsLiveWriter/JavaConcurrency24part9Deque_1425C/image_8.png)

在示意图中，LinkedList 总是有一个“傀儡”节点，用来描述队列“头部”，但是并不表示头部元素，它是一个执行null 的空节点。

双向链表的数据结构比较简单，操作起来也比较容易，从事从“傀儡”节点开始，“傀儡”节点的下一个元素就是队列的头部，前一个元素是队列的尾部，换句话说，“傀儡”节点在头部和尾部之间建立了一个通道，是整个队列形成一个循环，这样就可以从任意一个节点的任意一个方向能遍历完整的队列。

## LinkedBlockDeque ##
1. 要想支持阻塞功能，队列的容量一定是固定的，否则无法在入队的时候挂起线程。也就是capacity 是final 类型的。
2. 既然是双向链表，每一个结点就需要前后两个引用，这样才能将所有元素串联起来，支持双向遍历。也即需要prev/next 两个引用。
3. 双向链表需要头尾同时操作，所以需要first/last 两个节点，当然可以参考LinkedList那样采用一个节点的双向来完成，那样实现起来就稍微麻烦点。
4. 既然要支持阻塞功能，就需要锁和条件变量来挂起线程。这里使用一个锁两个条件变量来完成此功能。

实现略