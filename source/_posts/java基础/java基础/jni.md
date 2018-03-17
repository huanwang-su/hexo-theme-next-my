---
title: java jni
date: 2018/3/16 08:28:25
category:
- java基础
- java基础
tag:
- nio 
comments: true  
---

#JNI调用#
JNI是Java Native Interface的缩写，它提供了若干的API实现了Java和其他语言的通信（主要是C&C++）。使用java与本地已编译的代码交互，通常会丧失平台可移植性。但是，有些情况下这样做是可以接受的，甚至是必须的。例如，使用一些旧的库，与硬件、操作系统进行交互，或者为了提高程序的性能。

![](http://i.imgur.com/PBIlev4.png)

##副作用##
一旦使用JNI，JAVA程序就丧失了JAVA平台的两个优点：

1. 程序不再跨平台。要想跨平台，必须在不同的系统环境下重新编译本地语言部分。
2. 程序不再是绝对安全的，本地代码的不当使用可能导致整个程序崩溃。一个通用规则是，你应该让本地方法集中在少数几个类当中。这样就降低了JAVA和C之间的耦合性

## 流程 ##
![](http://images.cnblogs.com/cnblogs_com/hibraincol/201105/201105140840274898.png)

>用javac时 javac -d . HelloWorld.java
>在eclipse中用javah时加包名
>cl -I%java_home%/include -I%java_home%/include/win32 -I -LD simple_HelloWorld.cpp -Fehellodll.dll




