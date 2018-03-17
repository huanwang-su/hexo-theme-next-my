---
title: tomcat classLoader
date: 2018/3/16 08:28:25
category:
- java框架
- tomcat
tag:
- tomcat 
- classLoader
comments: true  
---
# 8 ClassLoader

tomcat的classLoader是自定义的classLoader。每个容器要载入所需的servlet类。能够

- 定义访问规则，如果通过jvm的classLoader，则servlet能够访问所有的classpath下的所有class，而servlet只允许访问WEB-INF/classs目录及子目录下面的类
- 可以实现自动重载功能
- 缓存

tomcat的classLoader类接口是org.apache.catalina.Loader

## 8.1 java的classLoader

关于jvm的classLoader参考 [classLoader讲解](xxx) 

classLoader加载类时先查缓存，再从父加载器中寻找，最后自己再寻找

## 8.2 Loader接口

servlet及其相关类需要定义一些规制：只允许访问WEB-INF/classs目录及子目录下面的类，不能访问其他classpath的类

**tomcat的载入器指的是Web应用程序载入器，而不仅仅是类载入器。**载入器必须实现org.apache.catalina.Loader接口。在载入器的实现中，会使用一个自定义载入器，是WebappClassLoaderBase的实例

Loader还定义了对仓库的操作，WEB-INF/classs和WEB-INF/lib是作为仓库添加到载入器中的。

```java
public interface Loader {
    public void backgroundProcess();
    public ClassLoader getClassLoader();
    public Context getContext();
    public void setContext(Context context);
    public boolean getDelegate();
    public void setDelegate(boolean delegate);
    public boolean getReloadable();
    public void setReloadable(boolean reloadable);
    public void addPropertyChangeListener(PropertyChangeListener listener);
    public boolean modified();
    public void removePropertyChangeListener(PropertyChangeListener listener);
}
```

解读api：

- backgroundProcess，执行周期性任务，如加载等方法，被Context调用

- getClassLoader对应加载仓库下的类的classLoader，

  > 注意这里的Loader并未和文件路径绑定，而是抽象出仓库的概念

- **tomcat的载入器通常会和一个Context级别的容器相关联**，set/get

- 支持ReLoad

- modified() 绑定的仓库是否有变

- 是否委托给父加载器set/getDelegate();

一般Context是禁用自动重载功能的，可以通过如下更改

```xml
<Context path="/myApp" docBase="myApp" debug="0" reloadable="true"/> 
```

## 8.2 WebappLoader类 

``` java
public class WebappLoader extends LifecycleMBeanBase
    implements Loader, PropertyChangeListener 
```

LifecycleMBeanBase是LifecycleBase子类，实现了JmxEnabled接口，JmxEnabled提供基于jmx的管理

WebappLoader会创建一个WebappClassLoaderBase的实例作为其类载入器

WebappLoader的实现比较易懂

### 8.2.1 主要属性

``` java

    /**
     * 加载类的加载器
     */
    private WebappClassLoaderBase classLoader = null;

    /**
     * The Context with which this Loader has been associated.
     */
    private Context context = null;

    /**
     * The "follow standard delegation model" flag that will be used to
     * configure our ClassLoader.
     */
    private boolean delegate = false;


    /**
     * The Java class name of the ClassLoader implementation to be used.
     * This class should extend WebappClassLoaderBase
     * 根据这个类名反射生成classLoader并赋值
     */
    private String loaderClass = ParallelWebappClassLoader.class.getName();


    /**
     * The parent class loader of the class loader we will create.
     */
    private ClassLoader parentClassLoader = null;


    /**
     * The reloadable flag for this Loader.
     */
    private boolean reloadable = false;


    /**
     * The string manager for this package.
     * StringManager用于管理异常码，可以阅读下源码
     */
    protected static final StringManager sm =
        StringManager.getManager(Constants.Package);

    /**
     * Classpath set in the loader.
     */
    private String classpath = null;
    
    /**
    * The property change support for this component.
    */
    protected final PropertyChangeSupport support = new PropertyChangeSupport(this);
```
注意parentClassLoader和classLoader区别，parentClassLoader取自context.getParentClassLoader()，并传给classLoader，classLoader会根据 delegate判断是否让parentClassLoader加载类
### 8.2.2 get/setter
``` java
    /**
     * Return the Java class loader to be used by this Container.
     */
 
    @Override
    public void setContext(Context context) {
		//....
        if (this.context != null) {
            this.context.removePropertyChangeListener(this);
        }

        // Process this property change
        Context oldContext = this.context;
        this.context = context;
        support.firePropertyChange("context", oldContext, this.context);

        // Register with the new Container (if any)
        if (this.context != null) {
            setReloadable(this.context.getReloadable());
            this.context.addPropertyChangeListener(this);
        }
    }
```
以setContext为例，其他很像。
### 8.2.3 其他公有方法
``` java
    // --------------------- Public Methods
    /**
     * 更换线程的classLoader，再通过线程进行context.reload()
     * 这里的线程显然是系统分派的一个子线程，可能是context内部分配的
     */
    @Override
    public void backgroundProcess() {
        if (reloadable && modified()) {
            try {
                Thread.currentThread().setContextClassLoader
                    (WebappLoader.class.getClassLoader());
                if (context != null) {
                    context.reload();
                }
            } finally {
                if (context != null && context.getLoader() != null) {
                    Thread.currentThread().setContextClassLoader
                        (context.getLoader().getClassLoader());
                }
            }
        }
    }


    public String[] getLoaderRepositories() {
		//...
        URL[] urls = classLoader.getURLs();
        String[] result = new String[urls.length];
        for (int i = 0; i < urls.length; i++) {
            result[i] = urls[i].toExternalForm();
        }
        return result;
    }

    public String getLoaderRepositoriesString() {
        String repositories[]=getLoaderRepositories();
        StringBuilder sb=new StringBuilder();
        for( int i=0; i<repositories.length ; i++ ) {
            sb.append( repositories[i]).append(":");
        }
        return sb.toString();
    }

    @Override
    public boolean modified() {
        return classLoader != null ? classLoader.modified() : false ;
    }
```
### 8.2.4 组件周期方法

start过程：

1.  创建类加载器
2.  设置仓库
    - 将WEB-INF/classes目录传入类加载器的addRepository中
    - 将WEB-INF/lib目录传入类加载器的setJarPath中
3.  设置类路径
4.  设置访问权限
    - 设置访问目录的权限
5.  启动类加载器
6.  注册组件
7.  设置状态

``` java
    @Override
    protected void startInternal() throws LifecycleException {
    
        if (context.getResources() == null) {
            log.info("No resources for " + context);
            setState(LifecycleState.STARTING);
            return;
        }
    
        // Construct a class loader based on our current repositories list
        try {
    
            classLoader = createClassLoader();
            classLoader.setResources(context.getResources());
            classLoader.setDelegate(this.delegate);
    
            // Configure our repositories
            setClassPath();
    
            setPermissions();
    
            ((Lifecycle) classLoader).start();
    
            String contextName = context.getName();
            if (!contextName.startsWith("/")) {
                contextName = "/" + contextName;
            }
            ObjectName cloname = new ObjectName(context.getDomain() + ":type=" +
                    classLoader.getClass().getSimpleName() + ",host=" +
                    context.getParent().getName() + ",context=" + contextName);
            Registry.getRegistry(null, null)
                .registerComponent(classLoader, cloname, null);
    
        } catch (Throwable t) {
            t = ExceptionUtils.unwrapInvocationTargetException(t);
            ExceptionUtils.handleThrowable(t);
            log.error( "LifecycleException ", t );
            throw new LifecycleException("start: ", t);
        }
    
        setState(LifecycleState.STARTING);
    }
```
stop过程：

1. 设置状态
2. 停止并销毁类加载器
3. 注销组件

``` java

    @Override
    protected void stopInternal() throws LifecycleException {
    
        setState(LifecycleState.STOPPING);
    
        // Remove context attributes as appropriate
        ServletContext servletContext = context.getServletContext();
        servletContext.removeAttribute(Globals.CLASS_PATH_ATTR);
    
        if (classLoader != null) {
            try {
                classLoader.stop();
            } finally {
                classLoader.destroy();
            }
    
            // classLoader must be non-null to have been registered
            try {
                //....
                Registry.getRegistry(null, null).unregisterComponent(cloname);
            } catch (Exception e) {
                log.error("LifecycleException ", e);
            }
        }
    
        classLoader = null;
    }
```

这里有个注册操作：Registry.getRegistry.unregisterComponent和Registry.getRegistry.registerComponent

### 8.2.5 classpath解析和classLoader创建

classLoader创建通过loaderClass（String）反射创建

``` java
    /**
     * Create associated classLoader.
     */
    private WebappClassLoaderBase createClassLoader()
        throws Exception {
    
        Class<?> clazz = Class.forName(loaderClass);
        WebappClassLoaderBase classLoader = null;
    
        if (parentClassLoader == null) {
            parentClassLoader = context.getParentClassLoader();
        }
        Class<?>[] argTypes = { ClassLoader.class };
        Object[] args = { parentClassLoader };
        Constructor<?> constr = clazz.getConstructor(argTypes);
        classLoader = (WebappClassLoaderBase) constr.newInstance(args);
    
        return classLoader;
    }
    
    /**
     * Set the appropriate context attribute for our class path.  This
     * is required only because Jasper depends on it.
     */
    private void setClassPath() {
    
        // Validate our current state information
        if (context == null)
            return;
        ServletContext servletContext = context.getServletContext();
        if (servletContext == null)
            return;
    
        StringBuilder classpath = new StringBuilder();
    
        // Assemble the class path information from our class loader chain
        ClassLoader loader = getClassLoader();
    
        if (delegate && loader != null) {
            // Skip the webapp loader for now as delegation is enabled
            loader = loader.getParent();
        }
    
        while (loader != null) {
            if (!buildClassPath(classpath, loader)) {
                break;
            }
            loader = loader.getParent();
        }
    
        if (delegate) {
            // Delegation was enabled, go back and add the webapp paths
            loader = getClassLoader();
            if (loader != null) {
                buildClassPath(classpath, loader);
            }
        }
    
        this.classpath = classpath.toString();
    
        // Store the assembled class path as a servlet context attribute
        servletContext.setAttribute(Globals.CLASS_PATH_ATTR, this.classpath);
    }

    private boolean buildClassPath(StringBuilder classpath, ClassLoader loader) {
        if (loader instanceof URLClassLoader) {
            URL repositories[] = ((URLClassLoader) loader).getURLs();
                for (int i = 0; i < repositories.length; i++) {
                    String repository = repositories[i].toString();
                    if (repository.startsWith("file://"))
                        repository = utf8Decode(repository.substring(7));
                    else if (repository.startsWith("file:"))
                        repository = utf8Decode(repository.substring(5));
                    else
                        continue;
                    if (repository == null)
                        continue;
                    if (classpath.length() > 0)
                        classpath.append(File.pathSeparator);
                    classpath.append(repository);
                }
        } else if (loader == ClassLoader.getSystemClassLoader()){
            // Java 9 onwards. The internal class loaders no longer extend
            // URLCLassLoader
            String cp = System.getProperty("java.class.path");
            if (cp != null && cp.length() > 0) {
                if (classpath.length() > 0) {
                    classpath.append(File.pathSeparator);
                }
                classpath.append(cp);
            }
            return false;
        } else {
            log.info( "Unknown loader " + loader + " " + loader.getClass());
            return false;
        }
        return true;
    }
    
    private static final Log log = LogFactory.getLog(WebappLoader.class);

}
```
### 8.2.6 PropertyChangeXXX模型
``` java
	protected final PropertyChangeSupport support = new PropertyChangeSupport(this);

    @Override
    public void 			addPropertyChangeListener(PropertyChangeListener listener) 	{
		support.addPropertyChangeListener(listener);
	}

	@Override
    public void removePropertyChangeListener(PropertyChangeListener listener) {
        support.removePropertyChangeListener(listener);
    }
支持PropertyChange的模型基本都是这样，如下：
​``` java
 public class MyBean {
     private final PropertyChangeSupport pcs = new PropertyChangeSupport(this);

     public void addPropertyChangeListener(PropertyChangeListener listener) {
         this.pcs.addPropertyChangeListener(listener);
     }

     public void removePropertyChangeListener(PropertyChangeListener listener) {
         this.pcs.removePropertyChangeListener(listener);
     }

     private String value;

     public String getValue() {
         return this.value;
     }

     public void setValue(String newValue) {
         String oldValue = this.value;
         this.value = newValue;
         this.pcs.firePropertyChange("value", oldValue, newValue);
     }

     [...]
 }

```
由于该类实现了PropertyChangeListener，主要是监听Context的属性变化
``` java
    @Override
    public void propertyChange(PropertyChangeEvent event) {

        // Validate the source of this event
        if (!(event.getSource() instanceof Context))
            return;

        // Process a relevant property change
        if (event.getPropertyName().equals("reloadable")) {
            try {
                setReloadable
                    ( ((Boolean) event.getNewValue()).booleanValue() );
            } catch (NumberFormatException e) {
                log.error(sm.getString("webappLoader.reloadable",
                                 event.getNewValue().toString()));
            }
        }
    }
```
### 8.2.7 JmxEnabled接口实现
该接口实现的是注册的功能，暂且不展开

可以结合父类LifecycleMBeanBase和WebappLoader简单了解

## 8.3 类加载器 WebappClassLoaderBase

``` java
public abstract class WebappClassLoaderBase extends URLClassLoader
        implements Lifecycle, InstrumentableClassLoader, WebappProperties, PermissionCheck 
```

WebappClassLoaderBase继承自jdk的URLClassLoader

- 默认情况下，此类加载器遵循规范要求的委派模型。 将先查询系统类加载器，然后查询本地存储库，然后才能委托给父类加载器查询。

  - 这允许Web应用程序覆盖除J2SE以外的任何共享类。

    > J2SE是系统类加载器加载的，Web应用程序能覆盖父加载器加载的

  - 对从未从webapp存储库加载的JAXP XML解析器接口，JNDI接口和servlet API的类提供特殊处理

  - delegate属性修改查询本地存储库和委托给父类加载器的顺序。

- 由于Jasper编译技术的限制，任何包含servlet API类的存储库都将被类加载器忽略。

- 类加载器生成源URL，其中包含从JAR文件加载类时的完整JAR URL，即使在JAR中包含类时也允许在类级别设置安全权限。

- 本地存储库按照通过初始构造函数添加的顺序进行搜索。

- 不进行安全校验除非设置了安全管理器

### 8.3.1 重写父类的loadClass

加载具有指定名称的类，使用以下算法进行搜索，直到找到并返回类。 如果找不到类，返回ClassNotFoundException。

- 调用findLoadedClass（String）来检查是否已经加载了该类。 如果有，则返回相同的Class对象。
- 如果委托属性设置为true，请调用父类加载器的loadClass（）方法（如果有）。
- 调用findClass（）在本地定义的存储库中查找此类。
- 调用我们的父类加载器的loadClass（）方法（如果有的话）。

如果使用上述步骤找到类，并且resolve标志为true，则该方法将在生成的Class对象上调用resolveClass（Class）。

> resolveClass :链接指定的类。 这个（误导性的）方法可以被类加载器用来链接一个类。 如果类c已经被链接，那么这个方法只是返回。 否则，类将按照Java™语言规范的“执行”一章所述进行链接。**resolveClass 应该是和findLoadedClass相对应的**

``` java
public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {

	synchronized (getClassLoadingLock(name)) {
		if (log.isDebugEnabled())
			log.debug("loadClass(" + name + ", " + resolve + ")");
		Class<?> clazz = null;

		// Log access to stopped class loader
		checkStateForClassLoading(name);

		// (0) Check our previously loaded local class cache
		clazz = findLoadedClass0(name);
		if (clazz != null) {
			if (log.isDebugEnabled())
				log.debug("  Returning class from cache");
			if (resolve)
				resolveClass(clazz);
			return (clazz);
		}

		// (0.1) Check our previously loaded class cache
		clazz = findLoadedClass(name);
		if (clazz != null) {
			if (log.isDebugEnabled())
				log.debug("  Returning class from cache");
			if (resolve)
				resolveClass(clazz);
			return (clazz);
		}

		// (0.2) Try loading the class with the system class loader, to prevent
		//       the webapp from overriding Java SE classes. This implements
		//       SRV.10.7.2
		String resourceName = binaryNameToPath(name, false);

		ClassLoader javaseLoader = getJavaseClassLoader();
		boolean tryLoadingFromJavaseLoader;
		try {
			// Use getResource as it won't trigger an expensive
			// ClassNotFoundException if the resource is not available from
			// the Java SE class loader. However (see
			// https://bz.apache.org/bugzilla/show_bug.cgi?id=58125 for
			// details) when running under a security manager in rare cases
			// this call may trigger a ClassCircularityError.
			// See https://bz.apache.org/bugzilla/show_bug.cgi?id=61424 for
			// details of how this may trigger a StackOverflowError
			// Given these reported errors, catch Throwable to ensure any
			// other edge cases are also caught
			tryLoadingFromJavaseLoader = (javaseLoader.getResource(resourceName) != null);
		} catch (Throwable t) {
			// Swallow all exceptions apart from those that must be re-thrown
			ExceptionUtils.handleThrowable(t);
			// The getResource() trick won't work for this class. We have to
			// try loading it directly and accept that we might get a
			// ClassNotFoundException.
			tryLoadingFromJavaseLoader = true;
		}

		if (tryLoadingFromJavaseLoader) {
			try {
				clazz = javaseLoader.loadClass(name);
				if (clazz != null) {
					if (resolve)
						resolveClass(clazz);
					return (clazz);
				}
			} catch (ClassNotFoundException e) {
				// Ignore
			}
		}

		// (0.5) Permission to access this class when using a SecurityManager
		if (securityManager != null) {
			int i = name.lastIndexOf('.');
			if (i >= 0) {
				try {
					securityManager.checkPackageAccess(name.substring(0,i));
				} catch (SecurityException se) {
					String error = "Security Violation, attempt to use " +
						"Restricted Class: " + name;
					log.info(error, se);
					throw new ClassNotFoundException(error, se);
				}
			}
		}

		boolean delegateLoad = delegate || filter(name, true);

		// (1) Delegate to our parent if requested
		if (delegateLoad) {
			if (log.isDebugEnabled())
				log.debug("  Delegating to parent classloader1 " + parent);
			try {
				clazz = Class.forName(name, false, parent);
				if (clazz != null) {
					if (log.isDebugEnabled())
						log.debug("  Loading class from parent");
					if (resolve)
						resolveClass(clazz);
					return (clazz);
				}
			} catch (ClassNotFoundException e) {
				// Ignore
			}
		}

		// (2) Search local repositories
		if (log.isDebugEnabled())
			log.debug("  Searching local repositories");
		try {
			clazz = findClass(name);
			if (clazz != null) {
				if (log.isDebugEnabled())
					log.debug("  Loading class from local repository");
				if (resolve)
					resolveClass(clazz);
				return (clazz);
			}
		} catch (ClassNotFoundException e) {
			// Ignore
		}

		// (3) Delegate to parent unconditionally
		if (!delegateLoad) {
			if (log.isDebugEnabled())
				log.debug("  Delegating to parent classloader at end: " + parent);
			try {
				clazz = Class.forName(name, false, parent);
				if (clazz != null) {
					if (log.isDebugEnabled())
						log.debug("  Loading class from parent");
					if (resolve)
						resolveClass(clazz);
					return (clazz);
				}
			} catch (ClassNotFoundException e) {
				// Ignore
			}
		}
	}

	throw new ClassNotFoundException(name);
}
```

同理，getResourceAsStream和getResource也是这种顺序

**findClass和loadClass区别**

- loadClass是public，findClass是private的
- loadClass被其他类用来加载类，而findClass一般用来被loadClass调用查找该classLoader解析类的方法

> 在ClassLoader的抽象类中
>
> ``` java
> protected Class<?> loadClass(String name, boolean resolve)
> 	throws ClassNotFoundException
> {
> 	synchronized (getClassLoadingLock(name)) {
> 		// First, check if the class has already been loaded
> 		Class<?> c = findLoadedClass(name);
> 		if (c == null) {
> 			long t0 = System.nanoTime();
> 			try {
> 				if (parent != null) {
> 					c = parent.loadClass(name, false);
> 				} else {
> 					c = findBootstrapClassOrNull(name);
> 				}
> 			} catch (ClassNotFoundException e) {
> 				// ClassNotFoundException thrown if class not found
> 				// from the non-null parent class loader
> 			}
>
> 			if (c == null) {
> 				// If still not found, then invoke findClass in order
> 				// to find the class.
> 				long t1 = System.nanoTime();
> 				c = findClass(name);
>
> 				// this is the defining class loader; record the stats
> 				sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
> 				sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
> 				sun.misc.PerfCounter.getFindClasses().increment();
> 			}
> 		}
> 		if (resolve) {
> 			resolveClass(c);
> 		}
> 		return c;
> 	}
> }
> ```
>
> 1. Invoke `findLoadedClass(String)` to check if the class   has already been loaded.  
> 2.  Invoke the `loadClass` method   on the parent class loader.  If the parent is `null` the class   loader built-in to the virtual machine is used, instead.  
> 3.  Invoke the `findClass(String)` method to find the   class. 
>
> 所以我们一般只需要重写 `findClass(String)` 即可

### 8.3.2 启动方法start()

将/WEB-INF/classes和/WEB-INF/lib作为两个本地厂库

``` java
public void start() throws LifecycleException {

	state = LifecycleState.STARTING_PREP;

	WebResource classes = resources.getResource("/WEB-INF/classes");
	if (classes.isDirectory() && classes.canRead()) {
		localRepositories.add(classes.getURL());
	}
	WebResource[] jars = resources.listResources("/WEB-INF/lib");
	for (WebResource jar : jars) {
		if (jar.getName().endsWith(".jar") && jar.isFile() && jar.canRead()) {
			localRepositories.add(jar.getURL());
			jarModificationTimes.put(
					jar.getName(), Long.valueOf(jar.getLastModified()));
		}
	}

	state = LifecycleState.STARTED;
}

```

