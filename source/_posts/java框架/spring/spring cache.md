---
title: spring cache
date: 2018/3/16 08:28:25
category:
- java框架
- spring
tag:
- spring 
comments: true  
---
## spring cache的使用

### 缓存某些方法的执行结果

设置好缓存配置之后我们就可以使用 @Cacheable 注解来缓存方法执行的结果了
spring cache的使用是非常简单的,只需要在方法上标注 @Cacheable 注解就可以

```
@Cacheable("getName")  
public String getName(String id) {  
    return xxx;  
}  
  
@Cacheable("getUser")  
public User searchCity(String userId){  
    return this.query(userId);     
}  

```

整个@Cacheable的流程是:先从缓存中读取，如果没有再调用方法获取数据，然后把数据添加到缓存中.

### 缓存数据一致性保证

在我们配置了查询用户名的方法缓存之后,假如这时候我们改名了,我想让我自己查询的时候能看到我自己的改变。这时候我们该怎么做呢？

#### @CacheEvict移除缓存

我们这时候可以在修改方法上加上@CacheEvict注解,来更新查询方法的结果缓存

```
@CacheEvict(value = "getName", key = "#user.id") //移除指定key的数据  
public User updateUser(User user) {  
    users.update(user);  
    return user;  
}  

```

这时候再调用新的查询接口,我们就不会走缓存里的方法,会重新去数据库中查询.
因为调用修改用户的方法的时候, @CacheEvict会根据相应的key和value作为条件从缓存中移除相应的数据.

#### @CachePut 既要保证方法被调用，又希望结果被缓存

据前面的例子，我们知道，如果使用了 @Cacheable 注释，则当重复使用相同参数调用方法的时候，方法本身不会被调用执行，即方法本身被略过了，取而代之的是方法的结果直接从缓存中找到并返回了.

现实中并不总是如此，有些情况下我们希望方法一定会被调用，因为其除了返回一个结果，还做了其他事情，例如记录日志，调用接口等，这个时候，我们可以用 @CachePut 注释，这个注释可以确保方法被执行，同时方法的返回值也被记录到缓存中.

```
@Cacheable(value="userCache")
 public User getUserById(String id) { 
   // 方法内部实现不考虑缓存逻辑，直接实现业务
   return getUserFromDB(id); 
 } 

 // 更新 userCache 缓存
 @CachePut(value="userCache",key="#user.id")
 public User updateUser(User user) { 
   return updateUserToDB(user); 
 } 
 
 private User updateUserToDB(User user) { 
   // do some query
   return user; 
 }

```

如上面的代码所示，我们首先用 getUserById 方法查询一个人的信息，这个时候会查询数据库一次，但是也记录到缓存中了。然后我们修改了用户信息，调用了 updateUser 方法，这个时候会执行数据库的更新操作且记录到缓存，我们再次修改信息并调用 updateUser 方法，然后通过 getUserById 方法查询，这个时候，由于缓存中已经有数据，所以不会查询数据库，而是直接返回最新的数据

#### 三种方式的对比

@Cacheable、@CachePut、@CacheEvict 注释介绍

- @Cacheable 主要针对方法配置，能够根据方法的请求参数对其结果进行缓存
- @CachePut 主要针对方法配置，能够根据方法的请求参数对其结果进行缓存，和 @Cacheable 不同的是，它每次都会触发真实方法的调用
- @CachEvict 主要针对方法配置，能够根据一定的条件对缓存进行清空

### 基本原理

一句话介绍就是Spring AOP的动态代理技术。 如果读者对Spring AOP不熟悉的话，可以去看看官方文档

### 注意和限制

#### 基于 proxy 的 spring aop 带来的内部调用问题

上面介绍过 spring cache 的原理，即它是基于动态生成的 proxy 代理机制来对方法的调用进行切面，这里关键点是对象的引用问题.

如果对象的方法是内部调用（即 this 引用）而不是外部引用，则会导致 proxy 失效，那么我们的切面就失效，也就是说上面定义的各种注释包括 @Cacheable、@CachePut 和 @CacheEvict 都会失效，我们来演示一下。

```
public Account getAccountByName2(String accountName) { 
   return this.getAccountByName(accountName); 
 } 

 @Cacheable(value="accountCache")// 使用了一个缓存名叫 accountCache 
 public Account getAccountByName(String accountName) { 
   // 方法内部实现不考虑缓存逻辑，直接实现业务
   return getFromDB(accountName); 
 }
 

```

上面我们定义了一个新的方法 getAccountByName2，其自身调用了 getAccountByName 方法，这个时候，发生的是内部调用（this），所以没有走 proxy，导致 spring cache 失效

要避免这个问题，就是要避免对缓存方法的内部调用，或者避免使用基于 proxy 的 AOP 模式，可以使用基于 aspectJ 的 AOP 模式来解决这个问题。

#### @CacheEvict 的可靠性问题

我们看到,`@CacheEvict` 注释有一个属性 `beforeInvocation`,缺省为 false,即缺省情况下，都是在实际的方法执行完成后，才对缓存进行清空操作。期间如果执行方法出现异常，则会导致缓存清空不被执行。我们演示一下

```
// 清空 accountCache 缓存
 @CacheEvict(value="accountCache",allEntries=true)
 public void reload() { 
   throw new RuntimeException(); 
 }

```

我们的测试代码如下：

```
accountService.getAccountByName("someone"); 
   accountService.getAccountByName("someone"); 
   try { 
     accountService.reload(); 
   } catch (Exception e) { 
    //...
   } 
   accountService.getAccountByName("someone"); 

```

注意上面的代码，我们在 reload 的时候抛出了运行期异常，这会导致清空缓存失败。上面的测试代码先查询了两次，然后 reload，然后再查询一次，结果应该是只有第一次查询走了数据库，其他两次查询都从缓存，第三次也走缓存因为 reload 失败了。

那么我们如何避免这个问题呢？我们可以用 @CacheEvict 注释提供的 beforeInvocation 属性，将其设置为 true，这样，在方法执行前我们的缓存就被清空了。可以确保缓存被清空。

#### 非 public 方法问题

和内部调用问题类似，非 public 方法如果想实现基于注释的缓存，必须采用基于 AspectJ 的 AOP 机制

#### Dummy CacheManager 的配置和作用

有的时候，我们在代码迁移、调试或者部署的时候，恰好没有 cache 容器，比如 memcache 还不具备条件，h2db 还没有装好等，如果这个时候你想调试代码，岂不是要疯掉？这里有一个办法，在不具备缓存条件的时候，在不改代码的情况下，禁用缓存。

方法就是修改 spring的配置文件，设置一个找不到缓存就不做任何操作的标志位

```
<cache:annotation-driven /> 
 
   <bean id="simpleCacheManager" class="org.springframework.cache.support.SimpleCacheManager"> 
     <property name="caches"> 
       <set> 
         <bean 
           class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean"
           p:name="default" /> 
       </set> 
     </property> 
   </bean> 

   <bean id="cacheManager" class="org.springframework.cache.support.CompositeCacheManager">
     <property name="cacheManagers"> 
       <list> 
         <ref bean="simpleCacheManager" /> 
       </list> 
     </property> 
     <property name="fallbackToNoOpCache" value="true" /> 
   </bean> 

```

注意以前的 cacheManager 变为了 simpleCacheManager，且没有配置 accountCache 实例，后面的 cacheManager 的实例是一个 CompositeCacheManager，他利用了前面的 simpleCacheManager 进行查询，如果查询不到，则根据标志位 fallbackToNoOpCache 来判断是否不做任何缓存操作。

#### 使用 guava cache

```
<bean id="cacheManager" class="org.springframework.cache.guava.GuavaCacheManager">
    <property name="cacheSpecification" value="concurrencyLevel=4,expireAfterAccess=100s,expireAfterWrite=100s" />
    <property name="cacheNames">
        <list>
            <value>dictTableCache</value>
        </list>
    </property>
</bean>

```

#### 代码地址

[https://github.com/rollenholt/spring-cache-example](https://link.jianshu.com?t=https://github.com/rollenholt/spring-cache-example)

### 参考链接

csdn的技术文:[Redis 缓存 + Spring 的集成示例 ](https://link.jianshu.com?t=http://blog.csdn.net/defonds/article/details/48716161)  
Ibm的技术文:[注释驱动的 Spring cache 缓存介绍](https://link.jianshu.com?t=https://www.ibm.com/developerworks/cn/opensource/os-cn-spring-cache/)  
开涛大神的技术博客:[Spring Cache抽象详解](https://link.jianshu.com?t=http://jinnianshilongnian.iteye.com/blog/2001040/)

### Spring @Cacheable 的key生成

​       自定义策略是指我们可以通过Spring的EL表达式来指定我们的key。这里的EL表达式可以使用方法参数及它们对应的属性。使用方法参数时我们可以直接使用“#参数名”或者“#p参数index”。下面是几个使用参数作为key的示例。

``` java
@Cacheable(value="users", key="#id")
public User find(Integer id) {
  return null;
}

@Cacheable(value="users", key="#p0")
public User find(Integer id) {
  return null;
}

@Cacheable(value="users", key="#user.id")
public User find(User user) {
  return null;
}

@Cacheable(value="users", key="#p0.id")
public User find(User user) {
  return null;
} 
```

除了上述使用方法参数作为key之外，Spring还为我们提供了一个root对象可以用来生成key。通过该root对象我们可以获取到以下信息。

| **属性名称**    | **描述**           | **示例**               |
| ----------- | ---------------- | -------------------- |
| methodName  | 当前方法名            | #root.methodName     |
| method      | 当前方法             | #root.method.name    |
| target      | 当前被调用的对象         | #root.target         |
| targetClass | 当前被调用的对象的class   | #root.targetClass    |
| args        | 当前方法参数组成的数组      | #root.args[0]        |
| caches      | 当前被调用的方法使用的Cache | #root.caches[0].name |

