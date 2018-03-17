---
title: spring整体框架
date: 2018/3/16 08:28:25
category:
- java框架
- spring
tag:
- spring 
comments: true  
---

# 1 spring的框架介绍

spring框架是一个分层架构，它包含一系列的功能要素，并被分为大约20个模块，如下图所示

![img](http://images2017.cnblogs.com/blog/1143792/201708/1143792-20170829102431155-6949964.png)

这些模块被总结为以下几个部分：

- Core Container

  Core Container(核心容器)包含有Core、Beans、Context和Expression Language模块 
  Core和Beans模块是框架的基础部分，提供IoC(转控制)和依赖注入特性。这里的基础概念是BeanFactory，它提供对Factory模式的经典实现来消除对程序性单例模式的需要，并真正地允许你从程序逻辑中分离出依赖关系和配置

  - Core模块主要包含Spring框架基本的核心工具类
  - Beans模块是所有应用都要用到的，它包含访问配置文件、创建和管理bean以及进行Inversion of Control/Dependency Injection(Ioc/DI)操作相关的所有类
  - Context模块构建于Core和Beans模块基础之上，提供了一种类似于JNDI注册器的框架式的对象访问方法。Context模块继承了Beans的特性，为Spring核心提供了大量扩展，添加了对国际化(如资源绑定)、事件传播、资源加载和对Context的透明创建的支持。ApplicationContext接口是Context模块的关键
  - Expression Language模块提供了一个强大的表达式语言用于在运行时查询和操纵对象，该语言支持设置/获取属性的值，属性的分配，方法的调用，访问数组上下文、容器和索引器、逻辑和算术运算符、命名变量以及从Spring的IoC容器中根据名称检索对象

- Data Access/Integration

  - JDBC模块提供了一个JDBC抽象层，它可以消除冗长的JDBC编码和解析数据库厂商特有的错误代码，这个模块包含了Spring对JDBC数据访问进行封装的所有类
  - ORM模块为流行的对象-关系映射API，如JPA、JDO、Hibernate、iBatis等，提供了一个交互层，利用ORM封装包，可以混合使用所有Spring提供的特性进行O/R映射，如前边提到的简单声明性事务管理

- Web

  Web上下文模块建立在应用程序上下文模块之上，为基于Web的应用程序提供了上下文，所以Spring框架支持与Jakarta Struts的集成。Web模块还简化了处理多部分请求以及将请求参数绑定到域对象的工作。Web层包含了Web、Web-Servlet、Web-Struts和Web、Porlet模块

  - Web模块：提供了基础的面向Web的集成特性，例如，多文件上传、使用Servlet listeners初始化IoC容器以及一个面向Web的应用上下文，它还包含了Spring远程支持中Web的相关部分
  - Web-Servlet模块web.servlet.jar：该模块包含Spring的model-view-controller(MVC)实现，Spring的MVC框架使得模型范围内的代码和web forms之间能够清楚地分离开来，并与Spring框架的其他特性基础在一起
  - Web-Struts模块：该模块提供了对Struts的支持，使得类在Spring应用中能够与一个典型的Struts Web层集成在一起
  - Web-Porlet模块：提供了用于Portlet环境和Web-Servlet模块的MVC的实现

- AOP

  AOP模块提供了一个符合AOP联盟标准的面向切面编程的实现，它让你可以定义例如方法拦截器和切点，从而将逻辑代码分开，降低它们之间的耦合性，利用source-level的元数据功能，还可以将各种行为信息合并到你的代码中

  Spring AOP模块为基于Spring的应用程序中的对象提供了事务管理服务，通过使用Spring AOP，不用依赖EJB组件，就可以将声明性事务管理集成到应用程序中

- Test

  Test模块支持使用Junit和TestNG对Spring组件进行测试

#  2 容器的基本实现

## 2.1 容器示例

``` java
package superfan.springLearn.bean;

import java.util.List;

public class Person {
	
	private String name;
	private int age;
	private String sex;
	private List<Person> family;
	
	public Person(String name, int age) {
		super();
		this.name = name;
		this.age = age;
	}
	
	public void init() {
		System.out.println("init");
	}
	
	public void destory() {
		System.out.println("destory");
	}
    //getter and setter
}

```

XML配置

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"     	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation=" http://www.springframework.org/schema/beans         
						http://www.springframework.org/schema/beans/spring-beans-3.0.xsd  
						http://www.springframework.org/schema/context            
						http://www.springframework.org/schema/context/spring-context-3.0.xsd">

	<bean id="me"  class="superfan.springLearn.bean.Person"  scope="singleton"
		depends-on="date" lazy-init="false"  init-method="init" destroy-method="destory">
		<constructor-arg name="name" value="wang" />
		<constructor-arg name="age" type="int" value="11"/>
		<property name="family">
			<list>
				<ref bean="father" />
				<ref bean="mother" />
			</list>
		</property>
	</bean>
	<bean id="date" class="java.util.Date" />
	<bean id="father" class="superfan.springLearn.bean.Person">
		<constructor-arg name="name" value="ff" />
		<constructor-arg name="age" type="int" value="11" />
	</bean>

	
	<bean id="mother" class="superfan.springLearn.bean.Person">
		<constructor-arg name="name" value="mm" />
		<constructor-arg name="age" type="int" value="11" />
	</bean>
	
</beans>
```



测试文件

``` java
public class BeanLoadTest {
	
	@SuppressWarnings({ "deprecation", "unused","resource" })
	@Test
	public void beanLoad() {
		BeanFactory beanFactory = new ClassPathXmlApplicationContext("beans.xml"); 
		System.out.println(beanFactory.getBean("me"));
		
	}
}
```



## 2.2 Spring的结构组成

### beans包的层级结构

beans包中的各个源码包的功能如下

- src/main/java 用于展现Spring的主要逻辑
- src/main/resources 用于存放系统的配置文件
- src/test/java 用于对主要逻辑进行单元测试
- src/test/resources 用于存放测试用的配置文件

## 2.3主要接口和类介绍

### 2.3.1 主要的类

由于spring功能强大，类和接口数量很多，这里只总结一小部分，是对整个框架的一种简单理解

- Context 应用程序上下文，与main交互
- BeanFactory Bean的工厂，存储实例的容器，内部是map实现，但注意保存的不是真正的对象，而是一个BeanDefine，及Bean的定义

  - DefaultListableBeanFactory
- BeanDefinition Bean的定义，可以理解为将XML文件的内容转换为内存对象，类似class文件转换的Class<T>对象

  - AbstractBeanDefinition 内部属性为scope，lazyInit，initMethodName这些等，及XML的属性
- BeanDefinitionReader 从文件中获取Bean对象，参考ClassLoader，参考tomcat的ClassLoader，支持多种形式

  - AbstractBeanDefinitionReader BeanDefinitionReader接口的抽象实现
  - XmlBeanDefinitionReader 读取XML文件的实现
- DocumentLoader 由于是dom解析XML，定义的读取类

  - DefaultDocumentLoader 接口实现，依赖dom包，包括对XML验证的步骤
- DefaultBeanDefinitionDocumentReader 该类不实现BeanDefinitionReader接口，从Document（XML）节点进行获取BeanDefinition
- BeanDefinitionParser BeanDefinition的解析器
  - BeanDefinitionParserDelegate DefaultBeanDefinitionDocumentReader的成员，解析任务委托给他，包括各种标签
- Resource 对资源文件的封装，负责文件的读取，提供文件（可能来自网络）的基本信息。继承自InputStreamSource，提供getInputStream功能

  - ClassPathResource 获取classpath路径下的文件
- ResourceLoader 加载资源类，与Resource 对应

### 2.3.2 注册大致流程
1. main函数调用Context获取BeanFactory对象
2. Context通过ResourceLoader将xml路径转化为Resource对象
3. Context创建BeanFactory并初始化
4. Context通过Reader来读取Resource并且注册到BeanFactory中
   1. Reader通过DocumentLoader 获取xml文件的dom结构
   2. DocumentReader通过dom结构获取BeanDefinition
      1. DocumentReader将dom结构交给BeanDefinitionParser进行解析各种标签等等，获取BeanDefinition
   3. 注册id，别名到BeanFactory
5. 完成

## 2.4 核心类介绍

### 2.4.1 DefaultListableBeanFactory

![img](http://images2017.cnblogs.com/blog/1143792/201708/1143792-20170829152136718-840167036.png)

各个类和接口的作用：

- AliasRegistry：定义对alias的简单增删改等操作
- SimpleAliasRegistry：主要使用map作为alias的缓存，并对接口AliasRegistry进行实现
- SingletonBeanRegistry：定义对单例的注册及获取
- BeanFactory：定义获取bean及bean的各种属性
- DefaultSingletonBeanRegistry：对接口SingletonBeanRegistry各函数的实现
- HierarchicalBeanFactory：继承BeanFactory，也就是在BeanFactory定义的功能的基础上增加了对parentFactory的支持
- BeanDefinitionRegistry：定义对BeanDefinition的各种增删改操作
- FactoryBeanRegistrySupport：在DefaultSingletonBeanRegistry基础上增加了对FactoryBean的特殊处理功能
- ConfigurableBeanFactory：提供配置Factory的各种方法
- ListableBeanFactory：根据各种条件获取bean的配置清单
- AbstractBeanFactory：综合FactoryBeanRegistrySupport和ConfigurationBeanFactory的功能
- AutowireCapableBeanFactory：提供创建bean、自动注入、初始化以及应用bean的后处理器
- AbstractAutowireCapableBeanFactory：综合AbstractBeanFactory并对接口AutowireCapableBeanFactory进行实现
- ConfigurableListableBeanFactory：BeanFactory配置清单，指定忽略类型及接口等
- DefaultListableBeanFactory：综合上面所有功能，主要是对Bean注册后的处理

### 2.4.2 XmlBeanDefinitionReader 

XML配置文件的读取是Spring中重要的功能，因为Spring的大部分功能都是以配置作为切入点的。

各个类的功能：

![img](http://images2017.cnblogs.com/blog/1143792/201708/1143792-20170829163649968-564205900.png)

- ResourceLoader：定义资源加载器，主要应用于根据给定的资源文件地址返回对应的Resource
- BeanDefinitionReader：主要定义资源文件读取并转换为BeanDefinition的各个功能
- EnvironmentCapable：定义获取Environment方法
- DocumentLoader：定义从资源文件加载到转换为Document的功能
- AbstractBeanDefinitionReader：对EnvironmentCapable、BeanDefinitionReader类定义的功能进行实现
- BeanDefinitionDocumentReader：定义读取Document并注册BeanDefinition功能
- BeanDefinitionParserDelegate：定义解析Element的各种方法

整个XML配置文件读取的大致流程，在XmlBeanDefinitionReader中主要包含以下几步处理

1. 通过继承自AbstractBeanDefinitionReader中的方法，来使用ResourceLoader将资源文件路径转换为对应的Resource文件
2. 通过DocumentLoader对Resource文件进行转换，将Resource文件转换为Document文件
3. 通过实现接口BeanDefinitionDocumentReader的DefaultBeanDefinitionDocumentReader类对Document进行解析，并使用BeanDefinitionParserDelegate对Element进行解析

### 2.4.3 配置文件封装

Spring的配置文件读取是通过ClassPathResource进行封装的，Spring对其内部使用到的资源实现了自己的抽象结构：Resource接口来封装底层资源

``` java
public interface InputStreamSource {
    InputStream getInputStream() throws IOException;
}
public interface Resource extends InputStreamSource {
    boolean exists();

    boolean isReadable();

    boolean isOpen();

    URL getURL() throws IOException;

    URI getURI() throws IOException;

    File getFile() throws IOException;

    long lastModified() throws IOException;

    Resource createRelative(String var1) throws IOException;

    String getFilename();

    String getDescription();
}

```

InputStreamSource封装任何能返回InputStream的类，比如File、Classpath下的资源和Byte Array等， 它只有一个方法定义：getInputStream()，该方法返回一个新的InputStream对象

Resource接口抽象了所有Spring内部使用到的底层资源：File、URL、Classpath等。首先，它定义了3个判断当前资源状态的方法：存在性(exists)、可读性(isReadable)、是否处于打开状态(isOpen)。另外，Resource接口还提供了不同资源到URL、URI、File类型的转换，以及获取lastModified属性、文件名(不带路径信息的文件名，getFilename())的方法，为了便于操作，Resource还提供了基于当前资源创建一个相对资源的方法：createRelative()，还提供了getDescription()方法用于在错误处理中的打印信息

对不同来源的资源文件都有相应的Resource实现：文件(FileSystemResource)、Classpath资源(ClassPathResource)、URL资源(UrlResource)、InputStream资源(InputStreamResource)、Byte数组(ByteArrayResource)等，相关类图如下所示

![img](http://images2017.cnblogs.com/blog/1143792/201708/1143792-20170830151426905-1910649735.png)

## 2.5 bean注册源码追踪

程序入口，以Context入口，常用

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnb70egj30jc043t8w.jpg)

ClassPathXmlApplication 通过setConfigLocation将配置文件路径保存到本地，再通过调用refresh进行初始化。因为该类支持刷新，看它的继承结构。refresh等同于第一次加载

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnb73qlj30k902dweh.jpg)
![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnb7vn4j30j505ijri.jpg)
![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnb8pmzj30hg08774t.jpg)

在 521行看到了关键的beanFactory，及DefaultListableBeanFactory

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnb86guj30ic04n0sy.jpg)

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnb8ho7j30j2082aal.jpg)

这里的130行简单的创建了个工厂，里面啥都没有，133行才是装载BeanFinition

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbnh0jj30kb08dgmc.jpg)

这里我们看到了XMLReader，至于为什么是XMLReader，看类名是用XML读取，这里注意组合关系，BeanFactory不负责获取BeanDefinition，是由Context中让Reader去获取的，再填入BeanFactory

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnb8by4j30lh05iaae.jpg)

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnb8gmej30ja04o3ym.jpg)

Reader从循环各个Locations里获取beans

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbo04jj30nh09mjs6.jpg)

222行resourceLoader加载resource

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbaq1xj30m904iwep.jpg)
![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbamrvj30k402mt8r.jpg)
![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbsjknj30kf0ee75g.jpg)
![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbcrezj30ld03ddfz.jpg)

Xml用dom解析，获取dom树

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbc470j30ms03gglr.jpg)
![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbdb1bj30kg065dg7.jpg)

解析为dom，包括了校验xml文件是否符合规范的步骤

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbcztmj30mr03kmxf.jpg)
![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbcifaj30m50403yp.jpg)

通过DocumentReader来解析

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbzwogj30l00ezta6.jpg)

148行委托给BeanDefinitionParserDelegate来解析

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbrfm9j30ms0b5dgh.jpg)

这里的175和178行分别对应解析spring标签和用户自定义标签

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fko1p0rjjfj30kl07cq3d.jpg)

解析4个子标签import，alias，bean，。。。

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbsh2bj30n108wt9e.jpg)

305行获取BeanDefinition的封装类bdHolder

310行将获取到的BeanDefinition的封装类bdHolder向BeanFactory里注册，包括id，alias等

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbyqdhj30n20f5jss.jpg)
![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbx3m1j30jz0deaav.jpg)

437行前获取id和alias

437获取BeanDefinition

最后封装为BeanDefinitionHolder，并返回

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbzmxsj30nh0f9jsn.jpg)

解析其他属性，如class，构造函数等

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnchjoqj30o60g0409.jpg)

解析其他属性，如layz-init，init等等属性

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnblswkj30iq07fjrv.jpg)

310行向BeanFactory注册

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnc4uiwj30pz08bq3v.jpg)
![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnc1g19j30mf0f40u5.jpg)![](http://ww1.sinaimg.cn/large/0063bT3ggy1fknxnbkcvij30l604omxb.jpg)

注册BeanName和Alias

##  2.5 bean获取源码追踪

