---
title: spring AOP
date: 2018/3/16 08:28:25
category:
- java框架
- spring
tag:
- spring 
comments: true  
---
### 概念

- 切面（Aspect）：一个关注点的模块化。在Spring AOP中，切面可以使用基于模式）或者基于Aspect注解方式来实现。通俗点说就是我们加入的切面类（比如log类
- 连接点（Joinpoint）：在程序执行过程中某个特定的点，比如某方法调用的时候或者处理异常的时候。在Spring AOP中，一个连接点总是表示一个方法的执行。
- 通知（Advice）：在切面的某个特定的连接点上执行的动作。其中包括了“around”、“before”和“after”等不同类型的通知
- 切入点（Pointcut）：匹配连接点的断言。通知和一个切入点表达式关联，并在满足这个切入点的连接点上运行（例如，当执行某个特定名称的方法时）。切入点表达式如何和连接点匹配是AOP的核心：Spring缺省使用AspectJ切入点语法。
- 引入（Introduction）：用来给一个类型声明额外的方法或属性（也被称为连接类型声明（inter-type declaration））。Spring允许引入新的接口（以及一个对应的实现）到任何被代理的对象。例如，你可以使用引入来使一个bean实现IsModified接口，以便简化缓存机制。
- 目标对象（Target Object）： 被一个或者多个切面所通知的对象。也被称做被通知（advised）对象。 
- AOP代理（AOP Proxy）：AOP框架创建的对象，用来实现切面契约（例如通知方法执行等等）。在Spring中，AOP代理可以是JDK动态代理或者CGLIB代理。
- 织入（Weaving）：把切面连接到其它的应用程序类型或者对象上，并创建一个被通知的对象。这些可以在编译时（例如使用AspectJ编译器），类加载时和运行时完成。Spring和其他纯Java AOP框架一样，在运行时完成织入。

### 通知类型：

- 前置通知（Before advice）：在某连接点之前执行的通知，但这个通知不能阻止连接点之前的执行流程（除非它抛出一个异常）。
- 后置通知（After returning advice）：在某连接点正常完成后执行的通知：例如，一个方法没有抛出任何异常，正常返回。
- 异常通知（After throwing advice）：在方法抛出异常退出时执行的通知。
- 最终通知（After (finally) advice）：当某连接点退出的时候执行的通知（不论是正常返回还是异常退出）。
- 环绕通知（Around Advice）：包围一个连接点的通知，如方法调用。这是最强大的一种通知类型。环绕通知可以在方法调用前后完成自定义的行为。它也会选择是否继续执行连接点或直接返回它自己的返回值或抛出异常来结束执行。

### 基于注解的实现

spring.xml

``` xml
<aop:aspectj-autoproxy/>  
	<context:component-scan base-package="com.xxx.webmvc.templete.aop" />
```

例子：

``` java
@Component // 加入到IoC容器
@Aspect // 指定当前类为切面类
public class ExampleAop {

    // 指定切入点表达式，拦截那些方法，即为那些类生成代理对象
    // @Pointcut("execution(* com.bie.aop.UserDao.save(..))") ..代表所有参数
    // @Pointcut("execution(* com.bie.aop.UserDao.*())") 指定所有的方法
    // @Pointcut("execution(* com.bie.aop.UserDao.save())") 指定save方法

    @Pointcut("execution(* com.xxx.webmvc.templete.service.UserService.add(..))")
    public void pointCut() {
    }

    @Before(value="pointCut() && args(user)")
    public void begin(User user) {
        System.out.println("before "+user);
    }

    @After(value="pointCut()")
    public void close(JoinPoint point) {
        point.getArgs();
        System.out.println("after");
    }
    
    @AfterThrowing(value="pointCut()", throwing = "t")
    public void AfterThrowing(Throwable t) {
        System.out.println("method AfterThrowing "+ t);
    } 
    
    @AfterReturning(value="pointCut()", returning = "obj")
    public void AfterReturning(Object obj) {
        System.out.println("method AfterReturning "+obj);
    } 
    
    @Around(value = "execution(* com.xxx.webmvc.templete.service.UserService.query(..)) && args(user)")
    public Object Around(ProceedingJoinPoint pjp, User user) {
        
        System.out.println("method Around start");
        Object object = null;
        try {
             object = pjp.proceed();
        } catch (Throwable e) {
            System.out.println("method exception");
        }
        System.out.println("method Around end and return "+object);
        return object;
    } 
}
```

错误：

1.  error at ::0 formal unbound in pointcut 

   原因：参数绑定错误，原`"execution(* com.xxx.webmvc.templete.service.UserService.query(..))"`改为`"execution(* com.xxx.webmvc.templete.service.UserService.query(..)) && args(user)"`即可

2. 不生效

   注意`<aop:aspectj-autoproxy/>  `

