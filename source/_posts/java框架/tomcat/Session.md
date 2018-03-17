---
title: tomcat session
date: 2018/3/16 08:28:25
category:
- java框架
- tomcat
tag:
- tomcat 
comments: true  
---
# 9 Session管理

Catalina通过Session管理器来管理建立的Session对象，该组件由org.apache.catalina.Manager接口提供

Session管理器与一个Context容器关联，且必须与一个Context容器关联。Session管理器负责创建、更新、销毁Session对象

默认情况下，Session对象放在内存中，但是，在Tomcat中，也可以做持久化。

## 9.1 Session对象

在servlet编程方面中，Session对象由javax.servlet.http.HttpSession接口表示。Catalina的Session接口的实现是StandardSession类，但是为了安全起见，这里使用了一个外观类StandardSessionFacade

![](http://ww1.sinaimg.cn/large/0063bT3gly1fl3vvj7627j30ks06x0ud.jpg)

### 9.1.1 Session接口

session保存了客户端用户请求和Context之间的状态信息

```java
public interface Session {
    public static final String SESSION_CREATED_EVENT = "createSession";
    public static final String SESSION_DESTROYED_EVENT = "destroySession";
    public static final String SESSION_ACTIVATED_EVENT = "activateSession";
    public static final String SESSION_PASSIVATED_EVENT = "passivateSession";
	/** 认证类型 */
    public String getAuthType();
    public void setAuthType(String authType);
  	/** 创建时间 */
    public long getCreationTime();
    public long getCreationTimeInternal();
    public void setCreationTime(long time);
  	/** id */
    public String getId();
    public String getIdInternal();
  	/** 设置id并通知监听器新的session创建了 */
    public void setId(String id);
  	/** 设置id并，可选通知监听器新的session创建了 */
    public void setId(String id, boolean notify);
    public String getIdInternal();
  	/** 获取最新的请求访问时间（绑定在该Session），请求完成时间 */
    public long getThisAccessedTime();
    public long getThisAccessedTimeInternal();
  	/** 获取最新的请求访问时间（绑定在该Session），请求开始时间 */
    public long getLastAccessedTime();
    public long getLastAccessedTimeInternal();
  	/** 获取空闲时间 */
    public long getIdleTime();
    public long getIdleTimeInternal();
  	/** Manager */
    public Manager getManager();
    public void setManager(Manager manager);    
  	/** 最大有限时间间隔，单位second */
	public int getMaxInactiveInterval();
    public void setMaxInactiveInterval(int interval);
    public void setNew(boolean isNew);
  	/** 认证凭证 */
    public Principal getPrincipal();
    public void setPrincipal(Principal principal);
    public HttpSession getSession();
  	/** 有效性 */
    public void setValid(boolean isValid);
    public boolean isValid();
  	/** Update the accessed time */
    public void access();
    public void addSessionListener(SessionListener listener);
    public void endAccess();
    public void expire();
  	/** 绑定的对象 */
    public Object getNote(String name);
    public Iterator<String> getNoteNames();
    public void recycle();
    public void removeNote(String name);
    public void removeSessionListener(SessionListener listener);
    public void setNote(String name, Object value);
    public void tellChangedSessionId(String newId, String oldId, boolean notifySessionListeners, boolean notifyContainerListeners);
    public boolean isAttributeDistributable(String name, Object value);
}
```

- 上面有四个SessionEvent常量，代表4个事件，用于Session事件监听

  ``` java
  public interface SessionListener extends EventListener {
      public void sessionEvent(SessionEvent event);
  }
  ```

- 一堆过期，访问等时间

- XXXNote，Note表示代表绑定在Session上的对象

- isAttributeDistributable表明还支持分布式

- XXXInternal与XXX区别在于XXXInternal不做有效性校验

Session对象存在于SessionManager中，而SessionManager关联在Context中。SessionManager通过getLastAccessedTime判断有效性，通过setValid设置有效性。每访问一个Session时，会调用其access来修改Session的最后访问时间。最后SessionManager会调用expire设置为过期，可以通过getSession获取一个经过Session外观类包装的HttpSession对象

### 9.1.2 StandardSession

StandardSession是Session接口的实现

```java
public class StandardSession implements HttpSession, Session, Serializable
```

- 支持序列化，以便支持分布式Session和Session在不同JVM的传输

- 构造器 必须绑定个SessionManager

  ```java
  public StandardSession(Manager manager)
  ```

- Session属性map

  ```java
  protected ConcurrentMap<String, Object> attributes = new ConcurrentHashMap<>();
  ```

- 外观类

  ``` java
  protected transient StandardSessionFacade facade = null;
  ```

- notesMap ，这里与Session属性map有区别，StandardSession实现了HttpSession接口，attributes用于该接口，而notes用于容器和管理器的一些内容

  ``` java
  protected transient Map<String, Object> notes = new Hashtable<>();
  ```

- getSession返回外观类

  ``` java
  public HttpSession getSession() {

  	if (facade == null){
  		if (SecurityUtil.isPackageProtectionEnabled()){
  			final StandardSession fsession = this;
  			facade = AccessController.doPrivileged(
  					new PrivilegedAction<StandardSessionFacade>(){
  				@Override
  				public StandardSessionFacade run(){
  					return new StandardSessionFacade(fsession);
  				}
  			});
  		} else {
  			facade = new StandardSessionFacade(this);
  		}
  	}
  	return (facade);

  }
  ```

- Session长时间未访问会设为过期，这个时间是maxInactiveInterval决定，为负则永不过期。expire用来设置过期

  1. 如果失效则立即返回
  2. 设置 expiring = true
  3. 通知相应监听器，context和session级别的
  4. SessionManager删除该session
  5. 回收，解除所有绑定，并设置 expiring = false和isValid =false

### 9.1.3 HttpSession

servlet容器使用此接口在HTTP客户端和HTTP服务器之间创建会话。 会话在指定的时间段内持续，跨越来自用户的多个连接或页面请求。 会话通常对应一个用户，谁可以访问一个网站多次。 服务器可以通过许多方式维护会话，例如使用Cookie或重写URL。

``` java

public interface HttpSession {

    public long getCreationTime();
    public String getId();
    public long getLastAccessedTime();
    public ServletContext getServletContext();
    public void setMaxInactiveInterval(int interval);
    public int getMaxInactiveInterval();
	@Deprecated
    public HttpSessionContext getSessionContext();
    public Object getAttribute(String name);
	@Deprecated
    public Object getValue(String name);
    public Enumeration<String> getAttributeNames();
	@Deprecated
    public String[] getValueNames();
    public void setAttribute(String name, Object value);
	@Deprecated
    public void putValue(String name, Object value);
    public void removeAttribute(String name);
	@Deprecated
    public void removeValue(String name);
    /**
     * 设置失效
     */
    public void invalidate();
    /**
     * Returns <code>true</code> if the client does not yet know about the
     * session or if the client chooses not to join the session. For example, if
     * the server used only cookie-based sessions, and the client had disabled
     * the use of cookies, then a session would be new on each request.
     */
    public boolean isNew();
}
```

注意@Deprecated的方法，被XXXAttribute替代

如果不支持session则isNew永远返回true

### 9.1.4 StandardSessionFacade

这个类是个外观类，代理了真正的StandardSession，不提供get方法，防止向下转型StandardSession访问敏感方法，如expire等



```java
public class StandardSessionFacade implements HttpSession {
	private final HttpSession session;
    public StandardSessionFacade(HttpSession session) 	  {
        this.session = session;
    }
    //...........
}
```

## 9.2 Manager

用来管理Session实例

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl446ig3xkj30l80int9u.jpg)

- 提供了关联Context的方法
- Session的创建删除查找
- 设置Session的过期时间，最大活跃数等属性
- load，upload持久化方法

ManagerBase提供了常用功能的实现，有两个子类，是StandardManager和PersistentManagerBase类

当Catalina运行是StandardManager将Session保存在内存中，关闭时存储到文件中，当再次启动时，将载入这些Session对象

PersistentManagerBase支持其他持久化方式的基类

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl44q3tfqmj30lm0chdi4.jpg)

### 9.2.1 ManagerBase 

```java
public abstract class ManagerBase extends LifecycleMBeanBase implements Manager{
  private Context context;
  protected SessionIdGenerator sessionIdGenerator = null;
  protected volatile int sessionMaxAliveTime;
  protected final Deque<SessionTiming> sessionCreationTiming =
            new LinkedList<>();

  protected final Deque<SessionTiming> sessionExpirationTiming =
            new LinkedList<>();
  protected final AtomicLong expiredSessions = new AtomicLong(0);
  protected int maxActiveSessions = -1;
  protected volatile int maxActive=0;
  protected long sessionCounter=0;
  private Pattern sessionAttributeNamePattern;
  private Pattern sessionAttributeValueClassNamePattern;
  private boolean warnOnSessionAttributeFilterFailure;
  protected Map<String, Session> sessions = new ConcurrentHashMap<>();
}
```

ManagerBase继承自LifecycleMBeanBase，也是tomcat的组件

- 一些Manager需要的状态量
- sessionAttributeValueClassNamePattern等用作Session对属性的过滤器
- sessions保存Session对象
- sessionCreationTiming和sessionExpirationTiming还不懂

#### 9.2.2.1 Tomcat的Session过期处理策略

tomcat容器实现类都继承了ContainerBase类,容器在启动的时候都会调用ContainerBase类的threadStart()方法,threadStart()方法如下:

``` java
protected void threadStart() {  
  
    if (thread != null)  
        return;  
    if (backgroundProcessorDelay <= 0)  
        return;  
  
    threadDone = false;  
    String threadName = "ContainerBackgroundProcessor[" + toString() + "]";  
    thread = new Thread(new ContainerBackgroundProcessor(), threadName);  
    thread.setDaemon(true);  
    thread.start();  
}  
```

StandardHost,StandardContext,StandardWrapper类的backgroundProcessorDelay的值都为-1,所以ContainerBackgroundProcessor线程都没有启动.而StandardEngine类在创建的时候 将backgroundProcessorDelay 的值设置为10.所以ContainerBackgroundProcessor线程在StandardEngine里启动,ContainerBackgroundProcessor线程代码如下:

``` java
protected class ContainerBackgroundProcessor implements Runnable {  
  
  public void run() {  
    while (!threadDone) {  
      try {  
        Thread.sleep(backgroundProcessorDelay * 1000L);  
      } catch (InterruptedException e) {  
        ;  
      }  
      if (!threadDone) {  
        Container parent = (Container) getMappingObject();//得到StandardEngine  
        ClassLoader cl =   
          Thread.currentThread().getContextClassLoader();  
        if (parent.getLoader() != null) {  
          cl = parent.getLoader().getClassLoader();  
        }  
        processChildren(parent, cl);  
      }  
    }  
  }  

  protected void processChildren(Container container, ClassLoader cl) {  
    try {  
      if (container.getLoader() != null) {  
        Thread.currentThread().setContextClassLoader  
          (container.getLoader().getClassLoader());  
      }  
      container.backgroundProcess();  
    } catch (Throwable t) {  
      log.error("Exception invoking periodic operation: ", t);  
    } finally {  
      Thread.currentThread().setContextClassLoader(cl);  
    }  
    Container[] children = container.findChildren();  
    for (int i = 0; i < children.length; i++) {  
      if (children[i].getBackgroundProcessorDelay() <= 0) {  
        processChildren(children[i], cl);  
      }  
    }  
  }  
  
}  
```

ContainerBackgroundProcessor每隔10秒执行一下线程,线程递归调用StandardEngine,StandardHost,StandardContext,StandardWrapper类的backgroundProcess()方法(backgroundProcess()方法在ContainerBase类中)

backgroundProcess()方法如下:

``` java
public void backgroundProcess() {  
      
    if (!started)  
        return;  
  
    if (cluster != null) {  
        try {  
            cluster.backgroundProcess();  
        } catch (Exception e) {  
            log.warn(sm.getString("containerBase.backgroundProcess.cluster", cluster), e);                  
        }  
    }  
    if (loader != null) {  
        try {  
            loader.backgroundProcess();  
        } catch (Exception e) {  
            log.warn(sm.getString("containerBase.backgroundProcess.loader", loader), e);                  
        }  
    }  
    if (manager != null) {//manager的实现类StandardManager  
        try {  
            manager.backgroundProcess();  
        } catch (Exception e) {  
            log.warn(sm.getString("containerBase.backgroundProcess.manager", manager), e);                  
        }  
    }  
    if (realm != null) {  
        try {  
            realm.backgroundProcess();  
        } catch (Exception e) {  
            log.warn(sm.getString("containerBase.backgroundProcess.realm", realm), e);                  
        }  
    }  
    Valve current = pipeline.getFirst();  
    while (current != null) {  
        try {  
            current.backgroundProcess();  
        } catch (Exception e) {  
            log.warn(sm.getString("containerBase.backgroundProcess.valve", current), e);                  
        }  
        current = current.getNext();  
    }  
    lifecycle.fireLifecycleEvent(Lifecycle.PERIODIC_EVENT, null);  
}  
```

在调用StandContext类的backgroundProcess()方法时, 会调用StandardManager的backgroundProcess()方法,该方法如下:

``` java
public void backgroundProcess() {  
    count = (count + 1) % processExpiresFrequency;  
    if (count == 0)  
        processExpires();  
}  
```

processExpiresFrequency的默认值为6,所以每6*10秒会调一次processExpires()方法,processExpires()方法如下:

```java
public void processExpires() {  
  
    long timeNow = System.currentTimeMillis();  
    Session sessions[] = findSessions();  
    int expireHere = 0 ;  
      
    if(log.isDebugEnabled())  
        log.debug("Start expire sessions " + getName() + " at " + timeNow + " sessioncount " + sessions.length);  
    for (int i = 0; i < sessions.length; i++) {  
        if (sessions[i]!=null && !sessions[i].isValid()) {//判断session是否有效,无效则清除  
            expireHere++;  
        }  
    }  
    long timeEnd = System.currentTimeMillis();  
    if(log.isDebugEnabled())  
         log.debug("End expire sessions " + getName() + " processingTime " + (timeEnd - timeNow) + " expired sessions: " + expireHere);  
    processingTime += ( timeEnd - timeNow );  
  
}  
```

该方法主要判断session是否过期,过期则清除.

整个过程看起来有点复杂,简单地来说,就是tomcat服务器在启动的时候初始化了一个守护线程,定期6*10秒去检查有没有Session过期.过期则清除.

### 9.2.3 StandardManager

StandardManager主要提供了持久化功能，在canalina启动或关闭时，会调用它的stop和start方法（继承至LifecycleMBeanBase），关闭时会在work目录下生成个SESSION.ser文件

``` java
public class StandardManager extends ManagerBase {
  protected String pathname = "SESSIONS.ser";
  
}
```

主要定义了load，unload， stopInternal，startInternal方法

### 9.2.4 PersistentManagerBase

是所有持久化Session管理器的父类

``` java
public abstract class PersistentManagerBase extends ManagerBase implements StoreManager{
      /**
     * Store object which will manage the Session store.
     */
    protected Store store = null;
}
```

对Session的增删改查都是通过Story接口

#### 9.2.4.1 StoreManager接口

``` java
public interface StoreManager extends DistributedManager {
    Store getStore();
    void removeSuper(Session session);
}
```

#### 9.2.4.2 Store接口

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fl49ichndrj30j007u74a.jpg)

```  java
public interface Store {
    public Manager getManager();
    public void setManager(Manager manager);
    public int getSize() throws IOException;
    public void addPropertyChangeListener(PropertyChangeListener listener);
    public String[] keys() throws IOException;
    public Session load(String id)
        throws ClassNotFoundException, IOException;
    public void remove(String id) throws IOException;
    public void clear() throws IOException;
    public void removePropertyChangeListener(PropertyChangeListener listener);
    public void save(Session session) throws IOException;
}
```

- **StoreBase**

  子类StoreBase也是组件，通过processExpires处理过期数据

  ``` java
  public abstract class StoreBase extends LifecycleBase implements Store 
  ```

- **FileStore**

  FileStore是文件处理，下面是save和remove操作，一个id对应一个文件

  ```java
  public void save(Session session) throws IOException {
    // Open an output stream to the specified pathname, if any
    File file = file(session.getIdInternal());
    if (file == null) {
      return;
    }
    if (manager.getContext().getLogger().isDebugEnabled()) {
      manager.getContext().getLogger().debug(sm.getString(getStoreName() + ".saving",
                                                          session.getIdInternal(), file.getAbsolutePath()));
    }

    try (FileOutputStream fos = new FileOutputStream(file.getAbsolutePath());
         ObjectOutputStream oos = new ObjectOutputStream(new BufferedOutputStream(fos))) {
      ((StandardSession)session).writeObjectData(oos);
    }
  }
  public void remove(String id) throws IOException {
    File file = file(id);
    if (file == null) {
      return;
    }
    file.delete();
  }
  ```

- **JdbcStore** 

  通过jdbc的方式，以remove为例

  ``` java
  public void remove(String id) throws IOException {
    synchronized (this) {
      int numberOfTries = 2;
      while (numberOfTries > 0) {
        Connection _conn = getConnection();
        if (_conn == null) {
          return;
        }
        try {
          remove(id, _conn);
          // Break out after the finally block
          numberOfTries = 0;
        } catch (SQLException e) {
          if (dbConnection != null)
            close(dbConnection);
        } finally {
          release(_conn);
        }
        numberOfTries--;
      }
    }
  }
  private void remove(String id, Connection _conn) throws SQLException {
    if (preparedRemoveSql == null) {
      String removeSql = "DELETE FROM " + sessionTable
        + " WHERE " + sessionIdCol + " = ?  AND "
        + sessionAppCol + " = ?";
      preparedRemoveSql = _conn.prepareStatement(removeSql);
    }
    preparedRemoveSql.setString(1, id);
    preparedRemoveSql.setString(2, getName());
    preparedRemoveSql.execute();
  }
  ```

  ​

