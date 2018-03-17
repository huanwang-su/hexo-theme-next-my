---
title: java并发简介
date: 2018/3/16 08:28:25
category:
- java基础
- 多线程
tag:
- 多线程 
comments: true  
---
## 并发的多面性 ##
并发通常提高运行在**单处理器**上的程序的性能,如果没有任务阻塞,单处理器上的并发就没有意义
>阻塞阻塞调用是指调用结果返回之前，当前线程会被挂起（线程进入非可执行状态，在这个状态下，CPU不会给线程分配时间片，即线程暂停运行）。函数只有在得到结果之后才会返回。
>
>1. 同步，就是调用一个功能，该功能没有结束前，一直等结果。
2.  异步，就是调用一个功能，不需要知道该功能结果，该功能有结果后通知（回调通知）
3.  阻塞，就是调用（函数），（函数）没有接收完数据或者没有得到结果之前，不会返回。
4.  非阻塞，就是调用（函数），（函数）立即返回，通过select通知调用者。
### 定义任务 ###
实现Runnable接口并编写run方法
>Thread.yield() :是对线程调度器(java线程一部分,可以将cpu从一个线程转移给另一个线程)的一种建议,表明我已经执行完生命周期中最重要的一部分.
### Thread ###
将Runnable对象提交给Thread来驱动,通过start()启动任务[run()],不过start()是立即返回的
### Executor ###
管理Thread,简化并发编程

	ExecutorService exec=Executor.newCachedThreadPool(); //基于缓存,创建所需数量相同的线程
	ExecutorService exec=Executor.newFixedThreadPool();  //线程的数量有限
	ExecutorService exec=Executor.newSingleThreadExecutor();  //线程的数量为一
	exec.execute(new Runnable(){...});
	exec.shutdown()  //禁止提交新任务,但是旧任务会执行完
    
所有的线程池都会尽可能的复用<br>
在线程数量有限且用完,新任务会进行排队
### Callable ###
完成任务有返回值
	
	exec.submit(new Callable(){...});
submit()返回Future对象,IsDown()查询是否完成,直接调用get()会发生阻塞
### 休眠 ###
sleep(),使任务终止执行给定的时间<br>
sleep()会抛出InterruptedException异常,在run()中被捕获,因为异常不能跨线程传递,必须本地处理
### 优先级 ###
调度器倾向于让优先级高的线程先执行,优先级低的也可以得到执行,只是频率低
	getThread();
    setThread();
	Thread.MAX_PRIORITY;Thread.NORM_PRIORITY;Thread.MAX_PRIORITY;
### 让步 ###
	yield()
让当前线程回到可执行状态，以便让具有相同优先级的线程进入执行状态，但不是绝对的。因为虚拟机可能会让该线程重新进入执行状态。
### 后台线程 ###
	setDaemon(true);
必须在线程启动前调用setDaemon(true)才能设置为后台线程
当最后一个非后台线程终止时,后台线程会突然终止
### 加入线程 ###
如果某个线程在另一个线程t上调用t.join();此线程将被挂起,直到t执行完才恢复.可以加上时间参数

## 共享受限资源 ##
### synchronized  ###
所有的对象都自动含有单一的锁,当在对象上调用其任意synchronized方法的时候,此对象会被加锁,此时该对象的其他synchronized方法只有等到前一个方法调用完毕并释放锁后才能调用.
### java.util.concurrent.Lock ###
比synchronized复杂,但是给了你更细粒度的控制力.你可以尝试获取锁,不成功可以去做其他事,而不至于阻塞.
### 原子性与易变性 ###
原子操作是不能被线程调度机制中断的操作.

原子性指事务的一个完整操作。操作成功则提交，失败则回滚，即一件事要做就做完整，要么就什么都不做 

volatile保证应用中的可视性,如果这个域发生了写操作,那么所有的读操作都可以看到这个修改,volatile域会立即写到主存里,而写操作也发生在主存里
### 原子类 ###
AtomicInteger/AtomicLong/AtomicReference
...参考另一篇
### 临界区 ###
有时你只希望多个线程同时访问方法内部的部分代码而不是访问整个方法,使用synchronized建立,synchronized用来指定某个对象,此对象的锁来做同步控制
		
	synchronized(syncObject){
		//this code can be accessed by only one task at a time
	}
使用Lock对象建立临界区
这种方法可以显著提高访问性能
### 线程本地存储 ###
根除对变量的控制.<br>
线程本地存储是一种自动化机制,可以为使用相同变量的每个不同的线程创建不同的存储
java.lang.ThreadLocal

>ThreadLocal对象通常当作静态域存储,只能通过get和set方法访问该对象的内容.get返回与线程相关联的副本,set将数值插入其线程存储对象中,并返回原有对象
## 终结线程 ##
1.  使用退出标志，使线程正常退出，也就是当run方法完成后线程终止。 
2.  使用stop方法强行终止线程（这个方法不推荐使用，因为stop和suspend、resume一样，也可能发生不可预料的结果，不释放锁）。 
3.  使用interrupt方法中断线程。 


### 在阻塞时终结 ###
**线程状态**

- 新建:线程创建时,只会短暂的处于该状态,它已经分配了必备的资源,并进行了初始化,此线程已经有资格获取cpu
- 就绪:只要调度器把时间片分给线程,线程就可以运行
- 阻塞:线程能够运行,但是有个条件阻止它运行
- 死亡:不再可调度

一个任务进入阻塞状态,可能因为:

- 调用sleep(millisecinds);
- 调用wait()
- 等待某个输入输出完成
- 调用同步控制方法,等待锁
>stop()方法被废弃,因为其不释放锁

### 中断 ###
如果一个线程已经被阻塞,修改中断位会抛出InterruptedException异常,而interrupted()离开run()但不抛出异常

interrupt方法是唯一能将中断状态设置为true的方法

Java中断机制是一种协作机制，也就是说通过中断并不能直接终止另一个线程，而需要被中断的线程自己处理中断。
### 检查中断 ###
	interrupted() //检查中断并返回状态,并清除中断位
	interrupt()   //中断,并设为true

## 线程之间的协作 ##
### wait()/notifyAll() ###
wait()会在等待外部世界产生变化的时候将任务挂起,只有在notify()或者notifyAll()发生时,这个任务才会被唤醒并检查变化

调用sleep()和yield()时锁并没有释放,当调用wait()时,线程挂起,锁被释放.

wait() <==> 挂起该线程+释放锁+唤醒其他线程

在实际中可能会有多个任务处于wait()中,所以notifyAll()更安全

>notifyAll()只会唤醒等待该锁的所有wait()线程

> 何时可以用notify:
> 
> 1. 所有任务必须等待同一个条件
> 1. 当条件发生变化时,只有一个任务起作用
  
### 使用显示的Lock和Condition ###
参考其他
