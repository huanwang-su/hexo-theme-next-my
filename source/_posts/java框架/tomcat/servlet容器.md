---
title: tomcat servlet容器
date: 2018/3/16 08:28:25
category:
- java框架
- tomcat
tag:
- tomcat 
comments: true  
---
![](http://img.blog.csdn.net/20150811090655055?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

# 5 servlet容器

容器有4种，都继承自org.apache.catalina.Container接口

## 5.1 Container 

***以下结合容器接口代码，拆分了接口***

4种容器：

- Engine 表示整个Catalina servlet引擎
- Host 表示包含一个或个Context的虚拟主机
- Context 表示一个web应用，包含多个Wrapper
- Wrapper 表示一个独立的servlet

这4种容器都有各自实现的类，分别是 StandardXXX，在org.apache.catalina.core内

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl0m355fixj30uv0csq6g.jpg)

每一个上层容器都包含多个下层子容器

```java
/** 父子容器关系，接口 */
public void addChild(Container child);
public void removeChild(Container child);
public Container findChild(String name);
public Container[] findChildren();
public Container getParent();
public void setParent(Container container);
```

容器可以包含一些组件，如载入器、记录器、管理器、域和Resource，暂且可以忽略

```java
public Log getLogger();
public void setRealm(Realm realm);
public Realm getRealm();
public ClassLoader getParentClassLoader();
public void setParentClassLoader(ClassLoader parent);
public AccessLog getAccessLog();
public void setCluster(Cluster cluster);
public Cluster getCluster();
```

在部署应用时，可以通过配置server.xml来决定使用哪种容器。这是通过管道（pipeline）和阀值（valve）的实现。

### 5.1.1 Lifecycle接口

Container 继承自 Lifecycle，所以每一个**Container都是一个组件，有组件的周期** ，下面是Lifecycle的接口

> Lifecycle：Catalina components may implement this interface in order to provide a consistent mechanism to start and stop the component.  
>
> 代表了一个组件的周期
>
> ```
>             start()
>   -----------------------------
>   |                           |
>   | init()                    |
>  NEW -»-- INITIALIZING        |
>  | |           |              |     ------------------«-----------------------
>  | |           |auto          |     |                                        |
>  | |          \|/    start() \|/   \|/     auto          auto         stop() |
>  | |      INITIALIZED --»-- STARTING_PREP --»- STARTING --»- STARTED --»---  |
>  | |         |                                                            |  |
>  | |destroy()|                                                            |  |
>  | --»-----«--    ------------------------«--------------------------------  ^
>  |     |          |                                                          |
>  |     |         \|/          auto                 auto              start() |
>  |     |     STOPPING_PREP ----»---- STOPPING ------»----- STOPPED -----»-----
>  |    \|/                               ^                     |  ^
>  |     |               stop()           |                     |  |
>  |     |       --------------------------                     |  |
>  |     |       |                                              |  |
>  |     |       |    destroy()                       destroy() |  |
>  |     |    FAILED ----»------ DESTROYING ---«-----------------  |
>  |     |                        ^     |                          |
>  |     |     destroy()          |     |auto                      |
>  |     --------»-----------------    \|/                         |
>  |                                 DESTROYED                     |
>  |                                                               |
>  |                            stop()                             |
>  ----»-----------------------------»------------------------------
> ```

### 5.1.2 ContainerBase 

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl8px28fc1j307r06oglk.jpg)

LifecycleBase和其接口在第六章描述

LifecycleMBeanBase实现了JmxEnable和Container接口

#### 5.1.2.1 Container接口

主要对父子容器的操作，监听，以及管道，阀等组件

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl8q0p9i8qj30d90olmyh.jpg)



#### 5.1.2.2 JmxEnable接口

用于JMX的管理

``` java
public interface JmxEnabled extends MBeanRegistration {
    String getDomain();
    void setDomain(String domain);
    ObjectName getObjectName();
}
```

#### 5.1.2.3 ContainerBase 实现

ContainerBase 主要处理4个公共组件，cluster，childContainer，pipeLine和Realm，对于childContainer使用线程池操作，线程池在init时创建，意味着每个容器都有个线程池

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl8u0hslbcj30ij0gfwf3.jpg)

``` java
    @Override
    protected void initInternal() throws LifecycleException {
        BlockingQueue<Runnable> startStopQueue = new LinkedBlockingQueue<>();
        startStopExecutor = new ThreadPoolExecutor(
                getStartStopThreadsInternal(),
                getStartStopThreadsInternal(), 10, TimeUnit.SECONDS,
                startStopQueue,
                new StartStopThreadFactory(getName() + "-startStop-"));
        startStopExecutor.allowCoreThreadTimeOut(true);
        super.initInternal();
    }
    @Override
    protected synchronized void startInternal() throws LifecycleException {
        logger = null;
        getLogger();
        Cluster cluster = getClusterInternal();
        if (cluster instanceof Lifecycle) {
            ((Lifecycle) cluster).start();
        }
        Realm realm = getRealmInternal();
        if (realm instanceof Lifecycle) {
            ((Lifecycle) realm).start();
        }
        // Start our child containers, if any
        Container children[] = findChildren();
        List<Future<Void>> results = new ArrayList<>();
        for (int i = 0; i < children.length; i++) {
            results.add(startStopExecutor.submit(new StartChild(children[i])));
        }
        boolean fail = false;
        for (Future<Void> result : results) {
            try {
                result.get();
            } catch (Exception e) {
                log.error(sm.getString("containerBase.threadedStartFailed"), e);
                fail = true;
            }

        }
        if (fail) {
            throw new LifecycleException(
                    sm.getString("containerBase.threadedStartFailed"));
        }

        // Start the Valves in our pipeline (including the basic), if any
        if (pipeline instanceof Lifecycle)
            ((Lifecycle) pipeline).start();


        setState(LifecycleState.STARTING);
        threadStart();// 用thread启动处理backgroundProcessor()方法
    }

	stopInternal类似startInternal
```



## 5.2 管道任务

联想tomcat的过滤器，一次调用是经过了多个过滤器的，过滤器类似valve，pipeline是装载valve的容器，通常是链表。

- pipeline：管道包含该servlet容器将要调用的任务。


- valve：一个阀表示一个具体的任务。

![](http://ww1.sinaimg.cn/large/0063bT3gly1fl0nnawc7vj30h303974a.jpg)

在ContainerBase（四大组件的基类）中包含有一个pipeline和HashMap<String, Container> children 的子容器

```java
package org.apache.catalina;
public interface Pipeline {
    public Valve getBasic();
    public void setBasic(Valve valve);
    public void addValve(Valve valve);
    public Valve[] getValves();
    public void removeValve(Valve valve);
    public Valve getFirst();
    public boolean isAsyncSupported();
    public Container getContainer();
    public void setContainer(Container container);
    public void findNonAsyncValves(Set<String> result);
}

package org.apache.catalina;
public interface Valve {
    public Valve getNext();
    public void setNext(Valve valve);
    public void backgroundProcess();
    public void invoke(Request request, Response response) throws IOException, ServletException;
    public boolean isAsyncSupported();
}
```

在调用了servlet的执行方法时，实际上是执行了pipeline中的valve。注意pipeline中包含一个baseValve，不同于valve[]。

> ### 5.2.1 ValveContext 实现阀的遍历
>
> 在tomcat8.5中被淘汰。。。。。。
>
> 参考深入剖析Tomcat一书，在低版本中，通过ValveContext实现遍历，ContainerBase和pipeline都有public void invoke(Request request, Response response)的方法签名，ValveContext是Pipeline的内部类，也有该签名，通过层层调用，最后由ValveContext遍历。同时会发现ContainerBase和pipeline也删除了该签名方法
>
> **淘汰原因**：只想让Container和Pipeline只做容器，而不处理，将处理抽象出来

### 5.2.2 Contained接口

```java
package org.apache.catalina;

public interface Contained {
    Container getContainer();
    void setContainer(Container container);
}
```

ValveBase和PipelineBase均实现该接口

## 5.3 Wrapper容器

Wrapper内部处理流程

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl67jf3fhqj30m80enjso.jpg)

### 5.3.1 Wrapper接口

Wrapper实现类主要负责管理基础Servlet类的servlet的生命周期，是最低级的容器。

Wrapper的父容器只能是Context，对子容器操作是会抛异常

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl5agchpivj30jr0oumya.jpg)

> 为啥是最低的，参看 StandardXXX.java
>
> ![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl0poim6h2j30dd0770su.jpg)

#### 5.3.1.1 获取servlet：

``` java
public void load() throws ServletException;
public Servlet allocate() throws ServletException;
```

- allocate()：分配一个已准备好调用其service()方法的Servlet的初始化实例。 
- load()：如果尚未有一个已初始化的实例，则加载并初始化此Servlet（一对一）的实例。可以在服务器启动时预加载部署描述符中标记的Servlet。

### 5.3.2 StandardWrapper

StandardWrapper主要负责载入它所代表的servlet类，并进行实例化，但是它并不调用servlet的service方法，该方法由StandardWrapperValve对象完成（作为基础阀）

当第一次请求某个servlet时，StandardWrapper载入servlet类，由于会动态载入，因此必须知道完全限定名，可以调用setServletClass设置。

#### 5.3.2.1 方法调用序列

方法的调用是通过管道调用的，基础阀最后调用，基础阀会调用子容器的管道

![](http://img.blog.csdn.net/20150811090655055?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### 5.3.2.2 load

如果还没有至少一个初始化实例，则加载并初始化此servlet的一个实例。 

load调用了loadServlet()

``` java
public synchronized Servlet loadServlet() throws ServletException {

	// Nothing to do if we already have an instance or an instance pool
	if (!singleThreadModel && (instance != null))
		return instance;

	PrintStream out = System.out;
	if (swallowOutput) {
		SystemLogHandler.startCapture();
	}

	Servlet servlet;
	try {
		long t1=System.currentTimeMillis();
		// Complain if no servlet class has been specified
		if (servletClass == null) {
			unavailable(null);
			throw new ServletException
				(sm.getString("standardWrapper.notClass", getName()));
		}

		InstanceManager instanceManager = ((StandardContext)getParent()).getInstanceManager();
		try {
			servlet = (Servlet) instanceManager.newInstance(servletClass);
		} catch (ClassCastException e) {
			unavailable(null);
			//...
		} catch (Throwable e) {
			unavailable(null);
			//...	
		}

		if (multipartConfigElement == null) {
			MultipartConfig annotation =
					servlet.getClass().getAnnotation(MultipartConfig.class);
			if (annotation != null) {
				multipartConfigElement =
						new MultipartConfigElement(annotation);
			}
		}

		processServletSecurityAnnotation(servlet.getClass());

		if (servlet instanceof ContainerServlet) {
			((ContainerServlet) servlet).setWrapper(this);
		}

		classLoadTime=(int) (System.currentTimeMillis() -t1);

		if (servlet instanceof SingleThreadModel) {
			if (instancePool == null) {
				instancePool = new Stack<>();
			}
			singleThreadModel = true;
		}

		initServlet(servlet);

		fireContainerEvent("load", this);

		loadTime=System.currentTimeMillis() -t1;
	} finally {
		if (swallowOutput) {
			//log
		}
	}
	return servlet;
}
```

#### 5.3.2.2 allocate

分配一个已经准备好调用service（）方法的Servlet的初始化实例。 如果servlet类没有实现SingleThreadModel，则（仅）已初始化的实例可能会立即返回。 如果servlet类实现了SingleThreadModel，那么Wrapper实现必须确保这个实例在被deallocate（）调用解除分配之前不会被再次分配。

SingleThreadModel防止一个servlet的service同时被多个线程调用，通过synchronized和池化处理。请注意，SingleThreadModel不能解决所有线程安全问题。 例如，即使在使用SingleThreadModel servlet的时候，会话属性和静态变量仍然可以同时被多个线程上的多个请求访问。该接口已被淘汰

``` java
@Override
public Servlet allocate() throws ServletException {

	// If we are currently unloading this servlet, throw an exception
	if (unloading) {
		throw new ServletException(sm.getString("standardWrapper.unloading", getName()));
	}
	boolean newInstance = false;
	// If not SingleThreadedModel, return the same instance every time
	if (!singleThreadModel) {
		// Load and initialize our instance if necessary
		if (instance == null || !instanceInitialized) {
			synchronized (this) {
				if (instance == null) {
					try {
						// Note: We don't know if the Servlet implements
						// SingleThreadModel until we have loaded it.
						instance = loadServlet();
						newInstance = true;
						if (!singleThreadModel) {
							countAllocated.incrementAndGet();
						}
					} catch (ServletException e) {
						//...
					}
				}
				if (!instanceInitialized) {
					initServlet(instance);
				}
			}
		}

		if (singleThreadModel) {
			if (newInstance) {
				// Have to do this outside of the sync above to prevent a
				// possible deadlock
				synchronized (instancePool) {
					instancePool.push(instance);
					nInstances++;
				}
			}
		} else {
			// For new instances, count will have been incremented at the
			// time of creation
			if (!newInstance) {
				countAllocated.incrementAndGet();
			}
			return instance;
		}
	}

	synchronized (instancePool) {
		while (countAllocated.get() >= nInstances) {
			// Allocate a new instance if possible, or else wait
			if (nInstances < maxInstances) {
				try {
					instancePool.push(loadServlet());
					nInstances++;
				} catch (ServletException e) {
					//...
				}
			} else {
				try {
					instancePool.wait();
				} catch (InterruptedException e) {
					// Ignore
				}
			}
		}
		countAllocated.incrementAndGet();
		return instancePool.pop();
	}
}
```

- countAllocated.incrementAndGet() 用来计算分配次数

- 对于非singleThreadModel采用单例模式，但由于第一次加载不知道singleThreadModel，要特殊处理

- 对于singleThreadModel采用对象池，利用了wait，signal

- 对于singleThreadModel的回收

  ``` java
  @Override
  public void deallocate(Servlet servlet) throws ServletException {
  	// If not SingleThreadModel, no action is required
  	if (!singleThreadModel) {
  		countAllocated.decrementAndGet();
  		return;
  	}
  	// Unlock and free this instance
  	synchronized (instancePool) {
  		countAllocated.decrementAndGet();
  		instancePool.push(servlet);
  		instancePool.notify();
  	}
  }	

  ```

  #### 5.3.2.2 ServletConfig与初始化

  一个servlet配置对象，由servlet容器在初始化期间将信息传递给servlet。servlet初始化需要一个ServletConfig

  servlet.init(facade);为了安全传入个外观类，

  > 初始化时示例：init() in GenericServlet
  >
  > ```` java
  > if (getServletConfig().getInitParameter("input") != null)
  >    input = Integer.parseInt(getServletConfig().getInitParameter("input"));
  > ````
  >
  > ​

  ``` java
  public interface ServletConfig {
      public String getServletName();
      public ServletContext getServletContext();
      public String getInitParameter(String name);
      public Enumeration<String> getInitParameterNames();
  }
  ```

  在StandardWrapper中

  ``` 
  // wrapper name同servletName
  public String getServletName() {
  	return (getName());
  }
  @Override
  public ServletContext getServletContext() {

  	if (parent == null)
  		return (null);
  	else if (!(parent instanceof Context))
  		return (null);
  	else
  		return (((Context) parent).getServletContext());
  }
  @Override
  public String getInitParameter(String name) {

  	return (findInitParameter(name));

  }    
  @Override
  public Enumeration<String> getInitParameterNames() {

  	parametersLock.readLock().lock();
  	try {
  		return Collections.enumeration(parameters.keySet());
  	} finally {
  		parametersLock.readLock().unlock();
  	}

  }
  	
  ```

  ​

### 5.4.3 StandardWrapperFacade

只是用来构建Servlet的，简化版StandardWrapper

``` java
public final class StandardWrapperFacade implements ServletConfig {
    public StandardWrapperFacade(StandardWrapper config) {
        super();
        this.config = config;
    }
    private final ServletConfig config;
    private ServletContext context = null;
    @Override
    public String getServletName() {
        return config.getServletName();
    }
    @Override
    public ServletContext getServletContext() {
        if (context == null) {
            context = config.getServletContext();
            if (context instanceof ApplicationContext) {
                context = ((ApplicationContext) context).getFacade();
            }
        }
        return (context);
    }
	@Override
    public String getInitParameter(String name) {
        return config.getInitParameter(name);
    }
	@Override
    public Enumeration<String> getInitParameterNames() {
        return config.getInitParameterNames();
    }
}
```

### 5.4.4 StandardWrapperValve

StandardWrapperValve是StandardWrapper的基础阀，wrapper的最后一个valve，要完成两个操作：

- 执行与该servlet关联的全部过滤器
- 调用servlet的service方法

StandardWrapperValve的实现主要是invoke方法

1. 调用StandardWrapper的allocate获取servlet
2. 调用ApplicationFilterFactory.createFilterChain创建过滤器链
3. 调用过滤器链的doFilter，其中包括servlet的service方法
4. 释放filterChain
5. 调用Wrapper的deallocate
6. 若servlet不会再用，调用Wrapper的unload()

**源码分部解析**

1. 校验父容器的有效性，以及Wrapper分配servlet，处理该过程的异常，异常会response.sendError，发送错误相应给客户端

   ``` java
   public final void invoke(Request request, Response response)
   	throws IOException, ServletException {
   	// Initialize local variables we may need
   	boolean unavailable = false;
   	Throwable throwable = null;
   	// This should be a Request attribute...
   	long t1=System.currentTimeMillis();
   	requestCount.incrementAndGet();
   	StandardWrapper wrapper = (StandardWrapper) getContainer();
   	Servlet servlet = null;
   	Context context = (Context) wrapper.getParent();

   	// Check for the application being marked unavailable
   	if (!context.getState().isAvailable()) {
   		response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
   					   sm.getString("standardContext.isUnavailable"));
   		unavailable = true;
   	}
   	// Check for the servlet being marked unavailable
   	if (!unavailable && wrapper.isUnavailable()) {
   		container.getLogger().info(sm.getString("standardWrapper.isUnavailable",
   				wrapper.getName()));
   		long available = wrapper.getAvailable();
   		if ((available > 0L) && (available < Long.MAX_VALUE)) {
   			response.setDateHeader("Retry-After", available);
   			response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
   					sm.getString("standardWrapper.isUnavailable",
   							wrapper.getName()));
   		} else if (available == Long.MAX_VALUE) {
   			response.sendError(HttpServletResponse.SC_NOT_FOUND,
   					sm.getString("standardWrapper.notFound",
   							wrapper.getName()));
   		}
   		unavailable = true;
   	}

   	// Allocate a servlet instance to process this request
   	try {
   		if (!unavailable) {
   			servlet = wrapper.allocate();
   		}
   	} catch (UnavailableException e) {
   		container.getLogger().error(
   				sm.getString("standardWrapper.allocateException",
   						wrapper.getName()), e);
   		long available = wrapper.getAvailable();
   		if ((available > 0L) && (available < Long.MAX_VALUE)) {
   			response.setDateHeader("Retry-After", available);
   			response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
   					   sm.getString("standardWrapper.isUnavailable",
   									wrapper.getName()));
   		} else if (available == Long.MAX_VALUE) {
   			response.sendError(HttpServletResponse.SC_NOT_FOUND,
   					   sm.getString("standardWrapper.notFound",
   									wrapper.getName()));
   		}
   	} catch (ServletException e) {
   		container.getLogger().error(sm.getString("standardWrapper.allocateException",
   						 wrapper.getName()), StandardWrapper.getRootCause(e));
   		throwable = e;
   		exception(request, response, e);
   	} catch (Throwable e) {
   		ExceptionUtils.handleThrowable(e);
   		container.getLogger().error(sm.getString("standardWrapper.allocateException",
   						 wrapper.getName()), e);
   		throwable = e;
   		exception(request, response, e);
   		servlet = null;
   	}
   ```

   ​

2. 创建过滤器链

   ``` java
   MessageBytes requestPathMB = request.getRequestPathMB();
   DispatcherType dispatcherType = DispatcherType.REQUEST;
   if (request.getDispatcherType()==DispatcherType.ASYNC) dispatcherType = DispatcherType.ASYNC;
   request.setAttribute(Globals.DISPATCHER_TYPE_ATTR,dispatcherType);
   request.setAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR,
                        requestPathMB);
   // Create the filter chain for this request
   ApplicationFilterChain filterChain =
     ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);
   ```

3. dofilter

   在filterChain.doFilter时有很多异常，还分为异步同步处理

   ``` java
   try {
   	if ((servlet != null) && (filterChain != null)) {
   		// Swallow output if needed
   		if (context.getSwallowOutput()) {
   			try {
   				SystemLogHandler.startCapture();
   				if (request.isAsyncDispatching()) {
   					request.getAsyncContextInternal().doInternalDispatch();
   				} else {
   					filterChain.doFilter(request.getRequest(),
   							response.getResponse());
   				}
   			} finally {
   				String log = SystemLogHandler.stopCapture();
   				if (log != null && log.length() > 0) {
   					context.getLogger().info(log);
   				}
   			}
   		} else {
   			if (request.isAsyncDispatching()) {
   				request.getAsyncContextInternal().doInternalDispatch();
   			} else {
   				filterChain.doFilter
   					(request.getRequest(), response.getResponse());
   			}
   		}

   	}
   } catch (ClientAbortException e) {
   	throwable = e;
   	exception(request, response, e);
   } catch (IOException e) {
   	container.getLogger().error(sm.getString(
   			"standardWrapper.serviceException", wrapper.getName(),
   			context.getName()), e);
   	throwable = e;
   	exception(request, response, e);
   } catch (UnavailableException e) {
   	container.getLogger().error(sm.getString(
   			"standardWrapper.serviceException", wrapper.getName(),
   			context.getName()), e);
   	//            throwable = e;
   	//            exception(request, response, e);
   	wrapper.unavailable(e);
   	long available = wrapper.getAvailable();
   	if ((available > 0L) && (available < Long.MAX_VALUE)) {
   		response.setDateHeader("Retry-After", available);
   		response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
   				   sm.getString("standardWrapper.isUnavailable",
   								wrapper.getName()));
   	} else if (available == Long.MAX_VALUE) {
   		response.sendError(HttpServletResponse.SC_NOT_FOUND,
   					sm.getString("standardWrapper.notFound",
   								wrapper.getName()));
   	}
   	// Do not save exception in 'throwable', because we
   	// do not want to do exception(request, response, e) processing
   } catch (ServletException e) {
   	Throwable rootCause = StandardWrapper.getRootCause(e);
   	if (!(rootCause instanceof ClientAbortException)) {
   		container.getLogger().error(sm.getString(
   				"standardWrapper.serviceExceptionRoot",
   				wrapper.getName(), context.getName(), e.getMessage()),
   				rootCause);
   	}
   	throwable = e;
   	exception(request, response, e);
   } catch (Throwable e) {
   	ExceptionUtils.handleThrowable(e);
   	container.getLogger().error(sm.getString(
   			"standardWrapper.serviceException", wrapper.getName(),
   			context.getName()), e);
   	throwable = e;
   	exception(request, response, e);
   }
   ```

4. 释放过滤器

   ``` java
   // Release the filter chain (if any) for this request
   if (filterChain != null) {
   	filterChain.release();
   }
   ```

5. 回收servlet

   参考Wrapper中的实现，

   ``` java
   // Deallocate the allocated servlet instance
   try {
   	if (servlet != null) {
   		wrapper.deallocate(servlet);
   	}
   } catch (Throwable e) {
   	ExceptionUtils.handleThrowable(e);
   	container.getLogger().error(sm.getString("standardWrapper.deallocateException",
   					 wrapper.getName()), e);
   	if (throwable == null) {
   		throwable = e;
   		exception(request, response, e);
   	}
   }
   ```

6. 若servlet不会再用，调用Wrapper的unload()，getAvailable代表下次可用时间，后面的time提供统计信息

   ``` java
   // If this servlet has been marked permanently unavailable,
   // unload it and release this instance
   try {
   	if ((servlet != null) &&
   		(wrapper.getAvailable() == Long.MAX_VALUE)) {
   		wrapper.unload();
   	}
   } catch (Throwable e) {
   	ExceptionUtils.handleThrowable(e);
   	container.getLogger().error(sm.getString("standardWrapper.unloadException",
   					 wrapper.getName()), e);
   	if (throwable == null) {
   		throwable = e;
   		exception(request, response, e);
   	}
   }
   long t2=System.currentTimeMillis();

   long time=t2-t1;
   processingTime += time;
   if( time > maxTime) maxTime=time;
   if( time < minTime) minTime=time;
   ```

### 5.4.4 过滤器 

#### 5.4.4.1 FilterDef

org.apache.tomcat.util.descriptor.web.FilterDef表示一个Filter的定义，as represented in a `<filter>` element in the deployment descriptor

parameters表示该过滤器的初始化参数

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl60btr6fxj30jh0l1aan.jpg)

#### 5.4.4.2 FilterConfig

过滤器配置对象，由servlet容器在初始化期间将信息传递给过滤器。实现了Def和Context的关联

ApplicationFilterConfig是其直接实现类

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl60jixmr2j30kk0eoq3b.jpg)

- 构造函数由Context和FilterDef组成
- instanceManager管理器，创建和销毁Filter实例
- getFilter用instanceManager创建，再初始化
- 初始化用filter.init(this);

#### 5.4.4.3 Filter

过滤器是一个对资源请求（servlet或静态内容）或资源响应执行过滤任务的对象。过滤器在doFilter方法中执行过滤。 每个过滤器都可以通过一个FilterConfig对象获取其初始化参数，例如，可以使用对ServletContext的引用来加载过滤任务所需的资源。

过滤器是在Web应用程序的部署描述符中配置的

举例：

- Authentication Filters 
- Logging and Auditing Filters 
- Image conversion Filters 
- Data compression Filters 
- Encryption Filters 
- Tokenizing Filters 
- Filters that trigger resource access events 
- XSL/T filters 
- Mime-type chain Filter 

``` java
public interface Filter {
    public void init(FilterConfig filterConfig) throws ServletException;
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException;
    public void destroy();
}

```

doFilter方法的典型实现将遵循以下模式：

1. 检查请求
2. 包装请求对象,筛选内容或标题

  3. 自定义包装响应对象

  4. a）使用FilterChain对象chain.doFilter调用链中的下一个实体，

     b）或者不将请求/响应对传递给过滤器链中的下一个实体以阻止请求处理

  5. 在调用过滤器链中的下一个实体之后，在响应上直接设置header。

#### 5.4.4.4 FilterChain

``` java
public interface FilterChain {
	public void doFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException;
}

```

ApplicationFilterChain是其实现子类

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl65phrspxj30ks0dw3yx.jpg)

- doFilter最后会调用servlet的servlet方法
- lastServicedRequest和lastServicedResponse是线程本地变量，在一次请求中的线程是不会变的。
- FilterChain保存的链是ApplicationFilterConfig对象

``` java
@Override
public void doFilter(ServletRequest request, ServletResponse response)
  throws IOException, ServletException {

  if( Globals.IS_SECURITY_ENABLED ) {
    final ServletRequest req = request;
    final ServletResponse res = response;
    try {
      java.security.AccessController.doPrivileged(
        new java.security.PrivilegedExceptionAction<Void>() {
          @Override
          public Void run()
            throws ServletException, IOException {
            internalDoFilter(req,res);
            return null;
          }
        }
      );
    } catch( PrivilegedActionException pe) {
      //...
    }
  } else {
    internalDoFilter(request,response);
  }
}

private void internalDoFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException {

  // Call the next filter if there is one
  if (pos < n) {
    ApplicationFilterConfig filterConfig = filters[pos++];
    try {
      Filter filter = filterConfig.getFilter();

      if (request.isAsyncSupported() && "false".equalsIgnoreCase(
        filterConfig.getFilterDef().getAsyncSupported())) {
        request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR, Boolean.FALSE);
      }
      if( Globals.IS_SECURITY_ENABLED ) {
        final ServletRequest req = request;
        final ServletResponse res = response;
        Principal principal =
          ((HttpServletRequest) req).getUserPrincipal();
        Object[] args = new Object[]{req, res, this};
        SecurityUtil.doAsPrivilege ("doFilter", filter, classType, args, principal);
      } else {
        filter.doFilter(request, response, this);
      }
    } catch (IOException | ServletException | RuntimeException e) {
      //...
    }
    return;
  }

  // We fell off the end of the chain -- call the servlet instance
  try {
    if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
      lastServicedRequest.set(request);
      lastServicedResponse.set(response);
    }

    if (request.isAsyncSupported() && !servletSupportsAsync) {
      request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR,
                           Boolean.FALSE);
    }
    // Use potentially wrapped request from this point
    if ((request instanceof HttpServletRequest) &&
        (response instanceof HttpServletResponse) &&
        Globals.IS_SECURITY_ENABLED ) {
      final ServletRequest req = request;
      final ServletResponse res = response;
      Principal principal =
        ((HttpServletRequest) req).getUserPrincipal();
      Object[] args = new Object[]{req, res};
      SecurityUtil.doAsPrivilege("service",
                                 servlet,
                                 classTypeUsedInService,
                                 args,
                                 principal);
    } else {
      servlet.service(request, response);
    }
  } catch (IOException | ServletException | RuntimeException e) {
    //...
  } finally {
    if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
      lastServicedRequest.set(null);
      lastServicedResponse.set(null);
    }
  }
}

```

#### 5.4.5.5 过滤器链的创建

利用ApplicationFilterFactory创建

filter配置实例：url-pattern是必选的

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl6731h9kcj30m906ft8s.jpg)

FilterConfig对象保存在context的filterConfigs中，通过匹配DispatcherType和servlet-name和url匹配，在33和49行。FilterMap和xml的属性基本一样

``` java
public static ApplicationFilterChain createFilterChain(ServletRequest request,
		Wrapper wrapper, Servlet servlet) {
	if (servlet == null)
		return null;
	// Create and initialize a filter chain object
	ApplicationFilterChain filterChain = null;
	//略去处理
	filterChain = new ApplicationFilterChain();

	filterChain.setServlet(servlet);
	filterChain.setServletSupportsAsync(wrapper.isAsyncSupported());

	// Acquire the filter mappings for this Context
	StandardContext context = (StandardContext) wrapper.getParent();
	FilterMap filterMaps[] = context.findFilterMaps();

	// If there are no filter mappings, we are done
	if ((filterMaps == null) || (filterMaps.length == 0))
		return (filterChain);

	// Acquire the information we will need to match filter mappings
	DispatcherType dispatcher =
			(DispatcherType) request.getAttribute(Globals.DISPATCHER_TYPE_ATTR);

	String requestPath = null;
	Object attribute = request.getAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR);
	if (attribute != null){
		requestPath = attribute.toString();
	}

	String servletName = wrapper.getName();

	// Add the relevant path-mapped filters to this filter chain
	for (int i = 0; i < filterMaps.length; i++) {
		if (!matchDispatcher(filterMaps[i] ,dispatcher)) {
			continue;
		}
		if (!matchFiltersURL(filterMaps[i], requestPath))
			continue;
		ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
			context.findFilterConfig(filterMaps[i].getFilterName());
		if (filterConfig == null) {
			// FIXME - log configuration problem
			continue;
		}
		filterChain.addFilter(filterConfig);
	}

	// Add filters that match on servlet name second
	for (int i = 0; i < filterMaps.length; i++) {
		if (!matchDispatcher(filterMaps[i] ,dispatcher)) {
			continue;
		}
		if (!matchFiltersServlet(filterMaps[i], servletName))
			continue;
		ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
			context.findFilterConfig(filterMaps[i].getFilterName());
		if (filterConfig == null) {
			// FIXME - log configuration problem
			continue;
		}
		filterChain.addFilter(filterConfig);
	}

	// Return the completed filter chain
	return filterChain;
}
```

### 思考

1. 既然SingleThreadModel无法保证线程安全，那如何处理？

   编写无状态的代码

2. servlet和filter关系

   filter过滤servlet，但两者都是独立的，创建FilterChain都是动态的，用完销毁，但filter和servlet对象都缓存在FilterConfig和Wrapper中，创建FilterChain的效率是很高的

## 5.4 Context容器

Context容器代表一个应用程序的

### 5.4.1 Context接口

Context实例代表一个具体的web应用程序，其中包含一个或者多个Wrapper实例。但是Context还支持其他组件，如Sessiong管理器，ClassLoader等

主要定义了一些子组件，监听器，状态变量的增删改查等

- 注意这里的监听器是String类型，实际上是class name的

- servlet名称映射

  ```java
      private HashMap<String, ApplicationFilterConfig> filterConfigs =
              new HashMap<>(); //名称-Conf映射，filter缓存在ApplicationFilterConfig中
      public void addFilterDef(FilterDef filterDef);//<String, FilterDef>，Def与xml配置对应
      public void addFilterMap(FilterMap filterMap);//数组实现，一个FilterMap保存了一个filterName和一系列对应的servletName的关系。创建filter链的时候通过filterMap内部的url或servletName匹配，匹配成功再通过filterName和filterConfigs获取ApplicationFilterConfig，ApplicationFilterConfig中持有filter对象

      public void addServletMappingDecoded(String pattern, String name);//url pattern与servletName的映射
      public void addServletMappingDecoded(String pattern, String name, boolean jspWildcard);
  ```

  ​

```java
public interface Context extends Container, ContextBind {
    // ----------------------------------------------------- Manifest Constants
    /**
     * 事件常量
     */
    public static final String ADD_WELCOME_FILE_EVENT = "addWelcomeFile";
    public static final String REMOVE_WELCOME_FILE_EVENT = "removeWelcomeFile";
    public static final String  CLEAR_WELCOME_FILES_EVENT = "clearWelcomeFiles";
    public static final String CHANGE_SESSION_ID_EVENT = "changeSessionId";
    // ------------------------------------------------------------- Properties

    /**
     * 是否允许解析Mutipart
     */
    public boolean getAllowCasualMultipartParsing();
    public void setAllowCasualMultipartParsing(boolean allowCasualMultipartParsing);
    /**
     * app监听器
     */
    public Object[] getApplicationEventListeners();
    public void setApplicationEventListeners(Object listeners[]);

    /**
     * Lifecycle监听器
     */
    public Object[] getApplicationLifecycleListeners();
    public void setApplicationLifecycleListeners(Object listeners[]);

    /**
     * 获取编码
     */
    public String getCharset(Locale locale);

    /**
     * URL of the XML descriptor
     */
    public URL getConfigFile();
    public void setConfigFile(URL configFile);

    /**
     * 是否配置过
     */
    public boolean getConfigured();
    public void setConfigured(boolean configured);

    /**
     * 是否用cookies
     */
    public boolean getCookies();
    public void setCookies(boolean cookies);

    /**
     * cookies name
     */
    public String getSessionCookieName();
    public void setSessionCookieName(String sessionCookieName);

    public boolean getUseHttpOnly();
    public void setUseHttpOnly(boolean useHttpOnly);

    /**
     * Cookie域，可用于跨域访问
     */
    public String getSessionCookieDomain();
    public void setSessionCookieDomain(String sessionCookieDomain);

    /**
     * path表示cookie所在的目录
     */
    public String getSessionCookiePath();
    public void setSessionCookiePath(String sessionCookiePath);

    public boolean getCrossContext();
	public void setCrossContext(boolean crossContext);

    /**
     * 发布名
     */
    public String getAltDDName();
    public void setAltDDName(String altDDName) ;

  
    /**
     * 展示名
     */
    public String getDisplayName();
    public void setDisplayName(String displayName);


    /**
     * 是否是分布式的
     */
    public boolean getDistributable();
    public void setDistributable(boolean distributable);

    /**
     * root目录
     */
    public String getDocBase();
    public void setDocBase(String docBase);


    /**
     * URL encoded context path 
     */
    public String getEncodedPath();

    /**
     * 登陆配置 ，如loginPage，errorPage等
     */
    public LoginConfig getLoginConfig();
    public void setLoginConfig(LoginConfig config);

    /**
     * context path for this web application.
     */
    public String getPath();
    public void setPath(String path);

	/**
	* reload
	*/
    public boolean getReloadable();
    public void setReloadable(boolean reloadable);


    /**
     * override
     */
    public boolean getOverride();
    public void setOverride(boolean override);


    /**
     * privileged
     */
    public boolean getPrivileged();
    public void setPrivileged(boolean privileged);


    /**
     * the Servlet context for which this Context is a facade.
     */
    public ServletContext getServletContext();


    /**
     * the default session timeout (in minutes)
     */
    public int getSessionTimeout();
    public void setSessionTimeout(int timeout);

    public String getWrapperClass();
    public void setWrapperClass(String wrapperClass);

    public Authenticator getAuthenticator();

    public InstanceManager getInstanceManager();
    public void setInstanceManager(InstanceManager instanceManager);

    // --------------------------------------------------------- Public Methods
    public void addApplicationListener(String listener);
    public void addApplicationParameter(ApplicationParameter parameter);
    public void addErrorPage(ErrorPage errorPage);

    public void addFilterDef(FilterDef filterDef);

    public void addFilterMap(FilterMap filterMap);

    public void addFilterMapBefore(FilterMap filterMap);

    public void addLocaleEncodingMappingParameter(String locale, String encoding);

    public void addMimeMapping(String extension, String mimeType);

    public void addParameter(String name, String value);

    public void addRoleMapping(String role, String link);

    public void addSecurityRole(String role);

    @Deprecated
    public void addServletMapping(String pattern, String name);
    @Deprecated
    public void addServletMapping(String pattern, String name, boolean jspWildcard);
    public void addServletMappingDecoded(String pattern, String name);
    public void addServletMappingDecoded(String pattern, String name, boolean jspWildcard);

    public void addWelcomeFile(String name);


    /**
     * Add the classname of a LifecycleListener
     */
    public void addWrapperLifecycle(String listener);

    /**
     * Add the classname of a ContainerListener
     */
    public void addWrapperListener(String listener);

    public Wrapper createWrapper();
    public String[] findApplicationListeners();
    public ApplicationParameter[] findApplicationParameters();
    public ErrorPage findErrorPage(int errorCode);
    public ErrorPage findErrorPage(String exceptionType);
    public ErrorPage[] findErrorPages();

    public FilterDef findFilterDef(String filterName);
    public FilterDef[] findFilterDefs();
    public FilterMap[] findFilterMaps();

    public String findParameter(String name);
    public String[] findParameters();
	
    /**
     * @return the servlet name mapped by the specified pattern (if any);
     */
    public String findServletMapping(String pattern);

    /**
     * @return the patterns of all defined servlet mappings for this
     * Context. 
     */
    public String[] findServletMappings();


    /**
     * @return the context-relative URI of the error page for the specified
     * HTTP status code, if any; otherwise return <code>null</code>.
     */
    public String findStatusPage(int status);

    /**
     * @return the set of HTTP status codes for which error pages have
     * been specified.  If none are specified, a zero-length array
     * is returned.
     */
    public int[] findStatusPages();

    /**
     * @return the associated ThreadBindingListener.
     */
    public ThreadBindingListener getThreadBindingListener();
    public void setThreadBindingListener(ThreadBindingListener threadBindingListener);


    /**
     * @return the set of watched resources for this Context
     */
    public String[] findWatchedResources();

    public boolean findWelcomeFile(String name);
    public String[] findWelcomeFiles();
    public String[] findWrapperLifecycles();
    public String[] findWrapperListeners();
	
	/**
     * Notify all {@link javax.servlet.ServletRequestListener}s that a request
     * has started.
     */
    public boolean fireRequestInitEvent(ServletRequest request);
    /**
     * Notify all {@link javax.servlet.ServletRequestListener}s that a request
     * has ended.
     */
    public boolean fireRequestDestroyEvent(ServletRequest request);

    public void reload();
	
    public void removeApplicationListener(String listener);
    public void removeApplicationParameter(String name);
    public void removeErrorPage(ErrorPage errorPage);
    public void removeFilterDef(FilterDef filterDef);
    public void removeFilterMap(FilterMap filterMap);
    public void removeMimeMapping(String extension);
    public void removeParameter(String name);
    public void removeServletMapping(String pattern);
    public void removeWatchedResource(String name);
    public void removeWelcomeFile(String name);
    public void removeWrapperLifecycle(String listener);
    public void removeWrapperListener(String listener);


    /**
     * @return the real path for a given virtual path
     */
    public String getRealPath(String path);

    /**
     * Is this Context paused whilst it is reloaded?
     */
    public boolean getPaused();

    /**
     * @return the base name to use for WARs, directories or context.xml files
     * for this context.
     */
    public String getBaseName();

    public void setWebappVersion(String webappVersion);
    public String getWebappVersion();

    public void setFireRequestListenersOnForwards(boolean enable);
    public boolean getFireRequestListenersOnForwards();

    public void setSendRedirectBody(boolean enable);
    public boolean getSendRedirectBody();

    public Loader getLoader();
    public void setLoader(Loader loader);

    public WebResourceRoot getResources();
    public void setResources(WebResourceRoot resources);

    public Manager getManager();
    public void setManager(Manager manager);

    public void setCookieProcessor(CookieProcessor cookieProcessor);
    public CookieProcessor getCookieProcessor();

    public void setRequestCharacterEncoding(String encoding);
    public String getRequestCharacterEncoding();

    public void setResponseCharacterEncoding(String encoding);
    public String getResponseCharacterEncoding();
	//......
}

```



### 5.4.2 StandardContext

StandardContext是org.apache.catalina.Context的标准实现，继承自ContainerBase基容器，具备容器的功能，包含Wrapper容器，它的角色是管理在其内部的Wrapper，从Host那里接收请求信息，加载实例化Servlet，然后选择一个合适的Wrapper处理，同一个Context内部中Servlet数据共享，不同Context之间数据隔离。在Context层面上，Filter的作用就出来了，Filter是用来过滤Servlet的组件，Context负责在每个Servlet上面调用Filter过滤，Context与开发Servlet息息相关，跟前面的几个组件相比，Context能够提供更多的信息给Servlet。一个webapp有一个Context，Context负责管理在其内部的组件：Servlet，Cookie，Session等等。

#### 5.4.2.1 主要成员



#### 5.4.2.2 构造函数

```java
public StandardContext() {
  super();
  pipeline.setBasic(new StandardContextValve());
  broadcaster = new NotificationBroadcasterSupport();
  // Set defaults
  if (!Globals.STRICT_SERVLET_COMPLIANCE) {
    // Strict servlet compliance requires all extension mapped servlets
    // to be checked against welcome files
    resourceOnlyServlets.add("jsp");
  }
}
```

#### 5.4.2.3 startInternal

startInternal方法主要完成以下工作：

1. 广播启动事件 broadcaster.sendNotification(notification);
2. 设置configured为false
3. 配置资源  setResources(new StandardRoot(this));
4. 配置classLoader  setLoader(new WebappLoader(getParentClassLoader()));
5. 设置cookieProcessor  cookieProcessor = new Rfc6265CookieProcessor();
6. 初始化字符串映射集合 getCharsetMapper();
7. 启动相关组件
   1. 启动classLoader ((Lifecycle) loader).start()
   2. 启动Realm ((Lifecycle) realm).start();
   3. 启动子容器 child.start();
   4. 启动管道 ((Lifecycle) pipeline).start();
   5. 启动SessionManager ContextManager new StandardManager(); ((Lifecycle) manager).start();
8. 触发start过failed事件

#### 5.4.2.4 stopInternal

stopInternal与startInternal相反

#### 5.4.2.5 destroyInternal

销毁子组件

#### 5.4.2.6 backgroundProcess

调用子组件的backgroundProcess

```java
public void backgroundProcess() {

	if (!getState().isAvailable())
		return;

	Loader loader = getLoader();
	if (loader != null) {
		try {
			loader.backgroundProcess();
		} catch (Exception e) {
			log.warn(sm.getString(
					"standardContext.backgroundProcess.loader", loader), e);
		}
	}
	Manager manager = getManager();
	if (manager != null) {
		try {
			manager.backgroundProcess();
		} catch (Exception e) {
			log.warn(sm.getString(
					"standardContext.backgroundProcess.manager", manager),
					e);
		}
	}
	WebResourceRoot resources = getResources();
	if (resources != null) {
		try {
			resources.backgroundProcess();
		} catch (Exception e) {
			log.warn(sm.getString(
					"standardContext.backgroundProcess.resources",
					resources), e);
		}
	}
	InstanceManager instanceManager = getInstanceManager();
	if (instanceManager instanceof DefaultInstanceManager) {
		try {
			((DefaultInstanceManager)instanceManager).backgroundProcess();
		} catch (Exception e) {
			log.warn(sm.getString(
					"standardContext.backgroundProcess.instanceManager",
					resources), e);
		}
	}
	super.backgroundProcess();
}

```

#### 5.4.2.7 reload

synchronized同步的，start-->stop

```java
public synchronized void reload() {

  // Validate our current component state
  if (!getState().isAvailable())
    throw new IllegalStateException
    (sm.getString("standardContext.notStarted", getName()));

  if(log.isInfoEnabled())
    log.info(sm.getString("standardContext.reloadingStarted",
                          getName()));

  // Stop accepting requests temporarily.
  setPaused(true);

  try {
    stop();
  } catch (LifecycleException e) {
    log.error(
      sm.getString("standardContext.stoppingContext", getName()), e);
  }

  try {
    start();
  } catch (LifecycleException e) {
    log.error(
      sm.getString("standardContext.startingContext", getName()), e);
  }

  setPaused(false);

  if(log.isInfoEnabled())
    log.info(sm.getString("standardContext.reloadingCompleted",
                          getName()));

}
```

### 5.4.3 WebResourceRoot

表示Web应用程序的完整资源集合。 Web应用程序的资源由多个资源集组成，当查找资源时，ResourceSets按以下顺序处理：

1. Pre - 由Web应用程序的context.xml中的<PreResource>元素定义的资源。 资源将按照指定的顺序进行搜索。
2. Main - Web应用程序的主要资源 - 即WAR或包含展开的WAR的目录
3. JAR - 由Servlet规范定义的资源JAR。 JAR将按照它们添加到ResourceRoot的顺序进行搜索。
4. Post - 由Web应用程序的context.xml中的<PostResource>元素定义的资源。 资源将按照指定的顺序进行搜索。

应该注意以下约定：

- 写操作（包括删除）只会应用于main ResourceSet。 如果其他资源集中某个资源的存在使主ResourceSet上的操作成为NO-OP，则写入操作将失败。
- ResourceSet中的文件将隐藏在搜索顺序中靠后的ResourceSet中具有相同名称的目录（以及该目录的所有内容）。
- 只有主ResourceSet可以定义一个META-INF / context.xml文件，因为该文件定义了Pre和Post资源。
- 根据Servlet规范，资源JAR中的任何META-INF或WEB-INF目录都将被忽略。
- Pre -和Post- 可以定义WEB-INF / lib和WEB-INF /classes，以便为Web应用程序提供额外的库和或类。

WebResourceRoot实现了Lifecycle接口，作为管理资源的组件

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl8igxjsxhj30fn0q5abb.jpg)

#### 5.4.3.1 WebResource

代表web资源，目录或文件，参考File类

内部实现封装了File，略

#### 5.4.3.2 StandardRoot

WebResourceRoot实现类

成员变量：

```java
private Context context;
private final List<WebResourceSet> preResources = new ArrayList<>();
private WebResourceSet main;
private final List<WebResourceSet> mainResources = new ArrayList<>();
private final List<WebResourceSet> classResources = new ArrayList<>();
private final List<WebResourceSet> jarResources = new ArrayList<>();
private final List<WebResourceSet> postResources = new ArrayList<>();
private final Cache cache= new Cache(this);

```

Cache负责缓存，用CurrencyHashMap缓存， 内部通过backgroundProcess() 来刷新缓存

```java
public class StandardRoot
	@Override
    public void backgroundProcess() {
        cache.backgroundProcess();
        gc();
    }
}

public class Cache {
  protected void backgroundProcess() {
    TreeSet<CachedResource> orderedResources =
      new TreeSet<>(new EvictionOrder());
    orderedResources.addAll(resourceCache.values());

    Iterator<CachedResource> iter = orderedResources.iterator();

    long targetSize =
      maxSize * (100 - TARGET_FREE_PERCENT_BACKGROUND) / 100;
    long newSize = evict(targetSize, iter);

  } 
  
  private long evict(long targetSize, Iterator<CachedResource> iter) {

    long now = System.currentTimeMillis();
    long newSize = size.get();
    while (newSize > targetSize && iter.hasNext()) {
      CachedResource resource = iter.next();
      // Don't expire anything that has been checked within the TTL
      if (resource.getNextCheck() > now) {
        continue;
      }
      // Remove the entry from the cache
      removeCacheEntry(resource.getWebappPath());
      newSize = size.get();
    }
    return newSize;
  }
  
}

```

gc()方法是调用所有的WebResourceSet的gc()，有的WebResourceSet实现类会在内部做缓存

### 5.4.4 StandardContextValve

StandardContext容器实现的基本Valve。

- 调用是直接调用子类的pipeLine的，也就是说容器就只是容器，不负责任何调用，由pipeLine完成
- `Wrapper wrapper = request.getWrapper();`选择Wrapper由request完成

```java
final class StandardContextValve extends ValveBase {

  private static final StringManager sm = 
    				StringManager.getManager(StandardContextValve.class);

  public StandardContextValve() {
    super(true);
  }


     /**
     * 根据指定的请求URI，选择适当的Wrapper处理此请求。 如果找不到匹配的Wrapper，
     * 则返回适当的HTTP错误。
     */
  @Override
  public final void invoke(Request request, Response response)
    throws IOException, ServletException {

    // 禁止直接访问 WEB-INF or META-INF
    MessageBytes requestPathMB = request.getRequestPathMB();
    if ((requestPathMB.startsWithIgnoreCase("/META-INF/", 0))
        || (requestPathMB.equalsIgnoreCase("/META-INF"))
        || (requestPathMB.startsWithIgnoreCase("/WEB-INF/", 0))
        || (requestPathMB.equalsIgnoreCase("/WEB-INF"))) {
      response.sendError(HttpServletResponse.SC_NOT_FOUND);
      return;
    }

    // 选择Wrapper
    Wrapper wrapper = request.getWrapper();
    if (wrapper == null || wrapper.isUnavailable()) {
      response.sendError(HttpServletResponse.SC_NOT_FOUND);
      return;
    }

    // Acknowledge the request
    try {
      response.sendAcknowledgement();
    } catch (IOException ioe) {
      container.getLogger().error(sm.getString(
        "standardContextValve.acknowledgeException"), ioe);
      request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, ioe);
      response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
      return;
    }

    if (request.isAsyncSupported()) {
      request.setAsyncSupported(wrapper.getPipeline().isAsyncSupported());
    }
    wrapper.getPipeline().getFirst().invoke(request, response);
  }
}

```

## 5.5 Host

Host是一个容器，代表Catalina servlet引擎中的虚拟主机。用于：

- 您希望使用拦截器来查看由此特定虚拟主机处理的每个请求。
- 希望使用独立的HTTP连接器运行Catalina，但仍希望支持多个虚拟主机。

一般来说，在部署Catalina连接到Web服务器（如Apache）时，不会使用Host，因为连接器将利用Web服务器的组件来确定应该使用哪个Context（或者甚至是Wrapper）来处理这个请求。

连接到Host的父容器通常是一个Engine，但可能是其他一些实现，或者如果没有必要，可以省略。连接到主机的子容器通常是Context的实现。

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl8om2diiwj30ib0fadgq.jpg)

### 5.5.1 StandardHost

Host接口的标准实现

startInternal只是用来添加个errorValve，默认是org.apache.catalina.valves.ErrorReportValve

```java
@Override
protected synchronized void startInternal() throws LifecycleException {

  // Set error report valve
  String errorValve = getErrorReportValveClass();
  if ((errorValve != null) && (!errorValve.equals(""))) {
    try {
      boolean found = false;
      Valve[] valves = getPipeline().getValves();
      for (Valve valve : valves) {
        if (errorValve.equals(valve.getClass().getName())) {
          found = true;
          break;
        }
      }
      if(!found) {
        Valve valve =
          (Valve) Class.forName(errorValve).getConstructor().newInstance();
        getPipeline().addValve(valve);
      }
    } catch (Throwable t) {
      ExceptionUtils.handleThrowable(t);
      log.error(sm.getString(
        "standardHost.invalidErrorReportValveClass",
        errorValve), t);
    }
  }
  super.startInternal();
}

```

### 5.5.2 StandardHostValve

```java
/**
 * 选择合适子Context处理请求
 */
@Override
public final void invoke(Request request, Response response)
	throws IOException, ServletException {

	// Select the Context to be used for this Request
	Context context = request.getContext();
	if (context == null) {
		response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR,
			 sm.getString("standardHost.noContext"));
		return;
	}

	if (request.isAsyncSupported()) {
		request.setAsyncSupported(context.getPipeline().isAsyncSupported());
	}

	boolean asyncAtStart = request.isAsync();
	boolean asyncDispatching = request.isAsyncDispatching();

	try {
		context.bind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);

		if (!asyncAtStart && !context.fireRequestInitEvent(request.getRequest())) {
			return;
		}
		try {
			if (!asyncAtStart || asyncDispatching) {
				context.getPipeline().getFirst().invoke(request, response);
			} else {

				if (!response.isErrorReportRequired()) {
					throw new IllegalStateException(sm.getString("standardHost.asyncStateError"));
				}
			}
		} catch (Throwable t) {
			ExceptionUtils.handleThrowable(t);
			container.getLogger().error("Exception Processing " + request.getRequestURI(), t);
			// If a new error occurred while trying to report a previous
			// error allow the original error to be reported.
			if (!response.isErrorReportRequired()) {
				request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, t);
				throwable(request, response, t);
			}
		}

		// Now that the request/response pair is back under container
		// control lift the suspension so that the error handling can
		// complete and/or the container can flush any remaining data
		response.setSuspended(false);

		Throwable t = (Throwable) request.getAttribute(RequestDispatcher.ERROR_EXCEPTION);

		// Protect against NPEs if the context was destroyed during a
		// long running request.
		if (!context.getState().isAvailable()) {
			return;
		}

		// Look for (and render if found) an application level error page
		if (response.isErrorReportRequired()) {
			if (t != null) {
				throwable(request, response, t);
			} else {
				status(request, response);
			}
		}

		if (!request.isAsync() && !asyncAtStart) {
			context.fireRequestDestroyEvent(request.getRequest());
		}
	} finally {
		// Access a session (if present) to update last accessed time, based
		// on a strict interpretation of the specification
		if (ACCESS_SESSION) {
			request.getSession(false);
		}

		context.unbind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);
	}
}

```



## 5.6 Engine

引擎是一个代表整个Catalina servlet引擎的容器。

- 您希望使用拦截器来查看整个引擎处理的每个请求。
- 希望使用独立的HTTP连接器运行Catalina，但仍希望支持多个虚拟主机。

一般来说，当部署连接到Web服务器（如Apache）的Catalina时，不会使用引擎，因为连接器将利用Web服务器来确定应该使用哪个上下文（或者甚至是哪个包装器）来处理请求。

连接到Engine的子容器通常是Host的实现（表示一个虚拟主机）或Context（表示一个单独的Servlet上下文），这取决于Engine的实现。

如果使用，Engine始终是Catalina层次结构中的顶级Container。因此，实现的setParent（）方法应抛出IllegalArgumentException。

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl8pkdoezjj30ed04h0sq.jpg)

Service包含了连接器组件，暂时不考虑

### 5.6.1 StandardEngine

Engine的标准实现

### 5.6.2 StandardEngineValve

```java
public final void invoke(Request request, Response response)
  throws IOException, ServletException {

  // Select the Host to be used for this Request
  Host host = request.getHost();
  if (host == null) {
    response.sendError
      (HttpServletResponse.SC_BAD_REQUEST,
       sm.getString("standardEngine.noHost",
                    request.getServerName()));
    return;
  }
  if (request.isAsyncSupported()) {
    request.setAsyncSupported(host.getPipeline().isAsyncSupported());
  }

  // Ask this Host to process this request
  host.getPipeline().getFirst().invoke(request, response);

}
```













