---
title: java 引用类型
date: 2018/3/16 08:28:25
category:
- java基础
- java基础
tag:

comments: true  
---
## 强引用 ##
当我们使用new 这个关键字创建对象时被创建的对象就是强引用，如Object object = new Object() 这个Object()就是一个强引用了，如果一个对象具有强引用。垃圾回收器就不会去回收有强引用的对象。如当jvm内存不足时，具备强引用的对象，虚拟机宁可会报内存空间不足的异常来终止程序，也不会靠垃圾回收器去回收该对象来解决内存。

## 软引用 ##
如果一个对象具备软引用，如果内存空间足够，那么垃圾回收器就不会回收它，如果内存空间不足了，就会回收该对象。当然没有被回收之前，该对象依然可以被程序调用。

## 弱引用 ##
如果一个对象只具有弱引用，只要垃圾回收器在自己的内存空间中线程检测到了，就会立即被回收，对应内存也会被释放掉。相比软引用弱引用的生命周期要比软引用短很多。不过，如果垃圾回收器是一个优先级很低的线程，也不一定会很快就会释放掉软引用的内存。

## 虚引用 ##
如果一个对象只具有虚引用，那么它就和没有任何引用一样，随时会被jvm当作垃圾进行回收。

## 引用和队列的使用 ##
 强引用一般是不会和队列一起使用的，这个过滤掉。
 软引用可以和一个引用队列来联合使用，一般软引用可以用来实现内存敏感的高速缓存，如果软引用的所引用的对象被垃圾回收，java虚拟机就会把引用入到与之关联的引用队列中去。
 弱引用和队列使用在一起，如果弱引用被所引用的对象回收了，java虚拟机就会把这个弱引用加入到关联的队列中去；
 虚引用，在java虚拟机回收虚引用时，会把这个虚引用放到与之关联的引用队列中去。程序可以通过判断引用队列中是否已经引用了虚引用，来了解引用对象是否要被垃圾回收。程序如果发现某个虚引用已经被加入到引用队列，那么可以在所引用的对象内存前，采取一些逻辑处理。 

## jdk中的引用实现类 ##
代表软引用的类：java.lang.ref.SoftReference

代表弱引用的类：java.lang.ref.WeakReference

代表虚引用的类：java.lang.ref.PhantomReference

他们同时继承了：java.lang.ref.Reference

引用队列：java.lang.ref.ReferenceQueue,这个引用队列是可以三种引用类型联合使用的，以便跟踪java虚拟机回收所引用对象的活动。

## 引用实现 ##
### 强引用 ###
	Object object = new Object() 
### 软引用 ###
	/**
	* 实现一个软引用
	*/
	public static  void testSoftReference(){
		//创建一个强引用Test
		String str = new String("Test");
		//创建一个引用队列
		ReferenceQueue<string> rq = new ReferenceQueue<string>();
		//实现一个软引用，将强引用类型str和是实例化的rq放到软引用实现里面
		SoftReference<string> srf = new SoftReference<string>(str,rq);
		//通过软引用get方法获取强引用中创建的内存空间Test值
		System.out.println(srf.get());
		//程序执行下gc现在jvm的内存空间还有很多所以gc不会回收str的对象
		System.gc();
		//所以这里执行get还是会打印Test的
		System.out.println(srf.get());
	}
![](http://www.2cto.com/uploadfile/Collfiles/20140316/20140316100002206.jpg)

### 弱引用 ###
	/**
	* 实现一个弱引用
	*/
	public static  void testWeakReference(){
		String str = new String("Test");
		ReferenceQueue<string> rq = new ReferenceQueue<string>();
		//实现一个弱引用，将强引用类型str和是实例化的rq放到弱引用实现里面
		WeakReference<string> wrf = new WeakReference<string>(str,rq);
		//给str给值为null。 然后再通过WeakReference弱引用的get()方法获得Test对象的引用
		str = null;
		//程序多执行gc，提高gc执行线程频率来回收对象。
		System.gc();
		//假如它还没有被垃圾回收，那么接下来在第执行wrf.get()方法会返回Test对象的引用，并且使得这个对象被str1强引用。
		//再接下来在行执行rq.poll()方法会返回null，因为此时引用队列中没有任何引用。
		String str1 = wrf.get(); 
		System.out.println(str1);
		//ReferenceQueue的poll()方法用于返回队列中的引用，如果没有则返回null。
		Reference<string> ref= (Reference<string>) rq.poll();
		System.out.println("弱引用已消失"+ref);
		if(ref!=null){
			System.out.println(ref.get());
		}
	}
