---
title: tomcat Connection
date: 2018/3/16 08:28:25
category:
- java框架
- tomcat
tag:
- tomcat 
comments: true  
---

# 2 连接器

Connect(连接器)负责接收外部连接请求，创建Request、Response对象用于请求的数据交换，并分配线程让Container处理该请求。

Connector可以支持多种协议的请求（如HTTP，AJP等）的请求，对Connector的配置在conf/server.xml中 
默认为：

``` xml
<Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```

即在tomcat中支持两种协议的连接器：HTTP/1.1与AJP/1.3

## 2.1 关键属性和方法

![这里写图片描述](http://img.blog.csdn.net/20160813144157471) 
其中，ProtocalHandler是协议处理器接口，不同的协议各自实现。Service是由一个或多个Connector共享一个Container组成，对外部请求提供服务，service主要为了关联Connector和Container。*关于Service，请移步。* Adapter是基于coyote的servlet容器的入口。

在Connector的代码中：

``` java
/**
 * Coyote Protocol handler class name.
 * Defaults to the Coyote HTTP/1.1 protocolHandler.
 */
protected String protocolHandlerClassName =
        "org.apache.coyote.http11.Http11Protocol";
```

可以看到，**默认采用http1.1协议**

### 2.1.1 构造方法

``` java
public Connector() {
  this(null);
}
public Connector(String protocol) {
  setProtocol(protocol);
  // Instantiate protocol handler 
  try {
    Class<?> clazz = Class.forName(protocolHandlerClassName);
    // 实例化协议处理器
    this.protocolHandler = (ProtocolHandler) clazz.newInstance();
  } catch (Exception e) {
    log.error(sm.getString(
      "coyoteConnector.protocolHandlerInstantiationFailed"), e);
  }
}
```

上述代码中根据协议名称构造Connector，继续进入setProtocol

``` java
public void setProtocol(String protocol) {
  if (AprLifecycleListener.isAprAvailable()) { //这里条件不会成立
    if ("HTTP/1.1".equals(protocol)) {
      setProtocolHandlerClassName
        ("org.apache.coyote.http11.Http11AprProtocol");
    } else if ("AJP/1.3".equals(protocol)) {
      setProtocolHandlerClassName
        ("org.apache.coyote.ajp.AjpAprProtocol");
    } else if (protocol != null) {
      setProtocolHandlerClassName(protocol);
    } else {
      setProtocolHandlerClassName
        ("org.apache.coyote.http11.Http11AprProtocol");
    }
  } else {
    if ("HTTP/1.1".equals(protocol)) {
      setProtocolHandlerClassName
        ("org.apache.coyote.http11.Http11Protocol");
    } else if ("AJP/1.3".equals(protocol)) {
      setProtocolHandlerClassName
        ("org.apache.coyote.ajp.AjpProtocol");
    } else if (protocol != null) {
      setProtocolHandlerClassName(protocol);
    }
  }

}
```

### 2.1.2 工作流程

#### 2.1.2.1 初始化和启动

时序图： 
![这里写图片描述](http://dl2.iteye.com/upload/attachment/0088/1248/2651fe99-5d26-3f62-add9-4513868b117c.jpg)

1. Tomcat初始化时会调用Bootstrap的Load方法，会调用Connector的构造方法
2. 接着调用server.init(),调用链会走到Connector的initInternal()方法

``` java
protected void initInternal() throws LifecycleException {
  super.initInternal();
  adapter = new CoyoteAdapter(this);//connector作为参数传入,设置到CoyoteAdapter的属性中
  protocolHandler.setAdapter(adapter);//adapter设置到protocolHandler协议处理器的属性中
  //……忽略非核心代码

  protocolHandler.init();
  //AbstractHttp11JsseProtocol.init()生成JSSEImplementation实例,接着调用AbstractProtocol.init()方法, 进而进入AbstractEndpoint.init()，再调用JIoEndpoint.bind()**核心方法，下面细说**
}
```

JIoEndpoint.bind()中，会设置Acceptor线程数为1，设置最大连接数(default 200)，设置最大连接数的时候会初始化connectionLimitLatch（用于控制Connector的并发连接数，其值即为最大连接数），并初始化serverSocketFactory创建serverSocket。 Acceptor线程在start的时候细说。

``` java
protected void startInternal() throws LifecycleException {
    //……忽略非核心代码
	protocolHandler.start();//调用AbstractProtocol.start()进而进入AbstractEndpoint.start()在调用JIoEndpoint.startInternal()方法核心，见下文
	
}
```

3. 接着调用server.start(),调用链会走到Connector的startInternal()方法 


4. JIoEndpoint.startInternal()所做的事情：

- 创建工作线程池: 其参数如下：corePoolSize=10，maximumPoolSize=200，keepAliveTime=60s 
- 如果在init阶段未初始化ConnectionLatch，则此时会进行初始化 
- 启动Acceptor线程(启动的个数在init中初始化了，default 1)监听的serverSocket上的请求
- 启动一个异步timeout线程
- 再附一张 JIoEndpoint.startInternal()的时序图 

![这里写图片描述](http://images2015.cnblogs.com/blog/846961/201608/846961-20160808211828965-2127290277.png)

### 2.1.3 请求处理

前文中说到Acceptor线程会监听Socket请求并转发给工作线程池，处理请求的即JIoEndpoint的内部类**SocketProcessor**。 

SocketProcessor根据socket的状态进行第一层处理，另外SSL的握手也是由它负责，在处理时又调用`handler.process(socket, SocketStatus.OPEN_READ);`，Hanlder接口是每个具体的Endpoint的内部接口，一般由对应Protocol的一个Handler内部类实现，比如JIoEndponit的handler对应的就是Http11Protocol的Http11ConnectionHandler。 

查看handler的调用链，会到`adapter.service(request, response);`，CoyoteAdapter完成http request的解析和分派，进而调用`connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);`，这样，Connector就把请求传递到了Container，也就是Engine对应Pipeline的第一个Valve的invoke方法。

![这里写图片描述](http://img.blog.csdn.net/20161007132547453) 

**CoyoteAdapter**对象负责将http request解析成HttpServletRequest对象，之后绑定相应的容器，然后从engine开始逐层调用valve直至该servlet。 
**Http11Protocol** 类完全路径org.apache.coyote.http11.Http11Protocol，这是支持HTTP的BIO实现。Http11Protocol包含了JIoEndpoint对象及Http11ConnectionHandler对象，她维护了一个Http11Processor对象池，Http11Processor对象会调用CoyoteAdapter完成HTTP Request的解析和分派。

Tomcat的Connector组件工作具体序列图： 

![这里写图片描述](http://img.blog.csdn.net/20150327153543942?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYzkyOTgzMzYyM2x2Y2hh/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 2.2 协议处理器

Connector支持多种协议的请求，必然需要进行协议的解析等处理，ProtocolHandler类层次结构为： 

![](http://ww1.sinaimg.cn/large/0063bT3ggy1flagw78khcj30aa07q74b.jpg)

从类图中看到，协议上有三种不同的实现方式：**NIO、APR、NIO2**。 

在Connector的代码中：

``` java
/**
 * Coyote Protocol handler class name.
 * Defaults to the Coyote HTTP/1.1 protocolHandler.
 */
protected String protocolHandlerClassName =
        "org.apache.coyote.http11.Http11Protocol";
```

协议处理器可以看为将endpoint和Processor关联起来，不同的协议对应不同的Processor

![](http://ww1.sinaimg.cn/large/0063bT3gly1flazigdaw9j30z908edgq.jpg)

``` java
public abstract class AbstractProtocol<S> implements ProtocolHandler,
        MBeanRegistration {
	private final AbstractEndpoint<S> endpoint;
	private final Set<Processor> waitingProcessors =
            Collections.newSetFromMap(new ConcurrentHashMap<Processor, Boolean>());
	@Override
    public void init() throws Exception {
        // 忽略其他
     	endpoint.init(); 
    }
    @Override
    public void start() throws Exception {
        // 忽略其他
        endpoint.start();
        // Start async timeout thread
        asyncTimeout = new AsyncTimeout(); //处理超时的异步process对象
        Thread timeoutThread = new Thread(asyncTimeout, getNameInternal() + "-AsyncTimeout");
        timeoutThread.setDaemon(true);
        timeoutThread.start();
    }
        @Override
    public void pause() throws Exception {
		// 忽略其他
        endpoint.pause();
    }
	
	    @Override
    public void resume() throws Exception {
        // 忽略其他
        endpoint.resume();
    }
	@Override
    public void stop() throws Exception {
        // 忽略其他
		if (asyncTimeout != null) {
            asyncTimeout.stop();
        }
		endpoint.stop();
    }
	
	@Override
    public void destroy() {
        // 忽略其他
        endpoint.destroy();
    }
    // ------------------------------------------- Connection handler base class
    /**
     * ConnectionHandler 处理连接请求，提供Processors池化缓存，并在socket连接时与Processor绑定
     *
     */
	protected static class ConnectionHandler<S> implements AbstractEndpoint.Handler<S> {
		private final Map<S,Processor> connections = new ConcurrentHashMap<>(); // 默认S 是 Socket
		private final RecycledProcessors recycledProcessors = new 
          									RecycledProcessors(this);//RecycledProcessors是一个栈缓存
		
      	@Override
        public SocketState process(SocketWrapperBase<S> wrapper, SocketEvent status) {
            S socket = wrapper.getSocket();
            Processor processor = connections.get(socket);
            try {
                if (processor == null) {
                    processor = recycledProcessors.pop();  // 缓存中取出
                }
                if (processor == null) {
                    processor = getProtocol().createProcessor();
                    register(processor);
                }
                // 绑定processor with the connection
                connections.put(socket, processor);
                SocketState state = SocketState.CLOSED;
                do {
                    state = processor.process(wrapper, status);
				   release(processor);
                } while ( state == SocketState.UPGRADING);

                if (state == SocketState...){   
                } else {
                    // Connection closed. OK to recycle the processor. Upgrade
                    // processors are not recycled.
                    connections.remove(socket);
                }
                return state;
            }
            // Make sure socket/processor is removed from the list of current
            // connections
            connections.remove(socket);
            release(processor);
            return SocketState.CLOSED;
        }
	
		private void release(Processor processor) {
            if (processor != null) {
                processor.recycle();
                if (!processor.isUpgrade()) {
                    recycledProcessors.push(processor);
                }
            }
        }
		public void release(SocketWrapperBase<S> socketWrapper) {
            S socket = socketWrapper.getSocket();
            Processor processor = connections.remove(socket);
            release(processor);
        }
		
		public final void pause() {
            for (Processor processor : connections.values()) {
                processor.pause();
            }
        }
	}

	
}
```

它的子类只是实现了不同协议对endpoint的设置，以及不同的Processor对象

### 2.2.1 endpoint

![](http://ww1.sinaimg.cn/large/0063bT3ggy1flb3ae3yocj309604z748.jpg)



``` java
private XXX serverSock = null; //取决于实现时用的ServerSocket类别
protected Acceptor[] acceptors; // 处理接受的请求
protected SynchronizedStack<SocketProcessorBase<S>> processorCache;//处理器
private Executor executor; //执行SocketProcessorBase的run
```



##  2.3 Adapter

如果把整个tomcat内核最高抽象程度模块化，可以看成是由连接器Connector和容器Container组成，连接器负责HTTP请求接收及响应，生成请求对象及响应对象并交由容器处理，而容器则根据请求路径找到相应的servlet进行处理。请求响应对象从连接器传送到容器需要一个桥梁——CoyoteAdapter。

``` java
public interface Adapter {
    public void service(Request req, Response res) throws Exception;
    public boolean prepare(Request req, Response res) throws Exception;
    public boolean asyncDispatch(Request req,Response res, SocketEvent status)
            throws Exception;
    public void log(Request req, Response res, long time);
    public void checkRecycled(Request req, Response res);
    public String getDomain();
}

```

只有一个实现类 CoyoteAdapter

在service中，解析了Request后调用了connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);CoyoteAdapter是将org.apache.coyote.Request适配成为org.apache.catalina.connector.Request 

``` java

public class CoyoteAdapter implements Adapter{
    public CoyoteAdapter(Connector connector) {
        super();
        this.connector = connector;
    }
    @Override
    public void service(org.apache.coyote.Request req, org.apache.coyote.Response res)
            throws Exception {

        Request request = (Request) req.getNote(ADAPTER_NOTES);
        Response response = (Response) res.getNote(ADAPTER_NOTES);

        if (request == null) {
            // Create objects
            request = connector.createRequest();
            request.setCoyoteRequest(req);
            response = connector.createResponse();
            response.setCoyoteResponse(res);

            // Link objects
            request.setResponse(response);
            response.setRequest(request);

            // Set as notes
            req.setNote(ADAPTER_NOTES, request);
            res.setNote(ADAPTER_NOTES, response);

            // Set query string encoding
            req.getParameters().setQueryStringCharset(connector.getURICharset());
        }
        boolean async = false;
        boolean postParseSuccess = false;

        req.getRequestProcessor().setWorkerThreadName(THREAD_NAME.get());

        try {
            // Parse and set Catalina and configuration specific request parameters
			// 在解析HTTP报头之后执行必要的处理，以便将请求/响应对传递到容器管道的起始处理。
            postParseSuccess = postParseRequest(req, request, res, response);
            if (postParseSuccess) {
                //check valves if we support async
                request.setAsyncSupported(
                        connector.getService().getContainer().getPipeline().isAsyncSupported());
                // Calling the container
                connector.getService().getContainer().getPipeline().getFirst().invoke(
                        request, response);
            }
            if (request.isAsync()) {
                //忽略非核心
            } else {
                request.finishRequest();
                response.finishResponse();
            }

        } catch (IOException e) {
            // Ignore
        } finally {
            //忽略非核心
            req.getRequestProcessor().setWorkerThreadName(null);
            // Recycle the wrapper request and response
            if (!async) {
                request.recycle();
                response.recycle();
            }
        }
    }

}
```

