---
title: ClassLoader
date: 2018/3/16 08:28:25
category:
- jvm
tag:
- ClassLoader
- jvm
comments: true  
---

# ClassLoader

http://blog.csdn.net/briblue/article/details/54973413
## JAVA类加载流程
### Java语言系统自带有三个类加载器: 
- Bootstrap ClassLoader 最顶层的加载类，主要加载核心类库，%JRE_HOME%\lib下的rt.jar、resources.jar、charsets.jar和class等。另外需要注意的是可以通过启动jvm时指定-Xbootclasspath和路径来改变Bootstrap ClassLoader的加载目录。比如java -Xbootclasspath/a:path被指定的文件追加到默认的bootstrap路径中。我们可以打开我的电脑，在上面的目录下查看，看看这些jar包是不是存在于这个目录。 
- Extention ClassLoader 扩展的类加载器，加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件。还可以加载-D java.ext.dirs选项指定的目录。 
- Appclass Loader也称为SystemAppClass 加载当前应用的classpath的所有类。

我们上面简单介绍了3个ClassLoader。说明了它们加载的路径。并且还提到了-Xbootclasspath和-D java.ext.dirs这两个虚拟机参数选项。

### 加载顺序

1. Bootstrap CLassloder 
2. Extention ClassLoader 
3. AppClassLoader

BootstrapClassLoader、ExtClassLoader、AppClassLoader实际是查阅相应的环境属性sun.boot.class.path、java.ext.dirs和java.class.path来加载资源文件的。

	ClassLoader cl = Test.class.getClassLoader(); //ClassLoader is:sun.misc.Launcher$AppClassLoader@73d16e93 
	int.class.getClassLoader();//int.class是由Bootstrap ClassLoader加载的

### 每个类加载器都有一个父加载器
比如加载Test.class是由AppClassLoader完成，那么AppClassLoader也有一个父加载器，通过getParent方法

	ClassLoader is:sun.misc.Launcher$AppClassLoader@73d16e93
	ClassLoader's parent is:sun.misc.Launcher$ExtClassLoader@15db9742

AppClassLoader的父加载器是ExtClassLoader,但通过getParent方法获取ExtClassLoader的父加载器空指针异常,注意父加载器不是父类,先看加载器类图
![](http://img.blog.csdn.net/20170211112754197?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYnJpYmx1ZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

URLClassLoader的源码中并没有找到getParent()方法。这个方法在ClassLoader.java中。

	public abstract class ClassLoader {

		// The parent class loader for delegation
		// Note: VM hardcoded the offset of this field, thus all new fields
		// must be added *after* it.
		private final ClassLoader parent;
		// The class loader for the system
			// @GuardedBy("ClassLoader.class")
		private static ClassLoader scl;
	
		private ClassLoader(Void unused, ClassLoader parent) {
			this.parent = parent;
			...
		}
		protected ClassLoader(ClassLoader parent) {
			this(checkCreateClassLoader(), parent);
		}
		protected ClassLoader() {
			this(checkCreateClassLoader(), getSystemClassLoader());
		}
		public final ClassLoader getParent() {
			if (parent == null)
				return null;
			return parent;
		}
		public static ClassLoader getSystemClassLoader() {
			initSystemClassLoader();
			if (scl == null) {
				return null;
			}
			return scl;
		}
	
		private static synchronized void initSystemClassLoader() {
			if (!sclSet) {
				if (scl != null)
					throw new IllegalStateException("recursive invocation");
				sun.misc.Launcher l = sun.misc.Launcher.getLauncher();
				if (l != null) {
					Throwable oops = null;
					//通过Launcher获取ClassLoader
					scl = l.getClassLoader();
					try {
						scl = AccessController.doPrivileged(
							new SystemClassLoaderAction(scl));
					} catch (PrivilegedActionException pae) {
						oops = pae.getCause();
						if (oops instanceof InvocationTargetException) {
							oops = oops.getCause();
						}
					}
					if (oops != null) {
						if (oops instanceof Error) {
							throw (Error) oops;
						} else {
							// wrap the exception
							throw new Error(oops);
						}
					}
				}
				sclSet = true;
			}
		}
	}

getParent()实际上返回的就是一个ClassLoader对象parent，parent的赋值是在ClassLoader对象的构造方法中，它有两个情况： 
1. 由外部类创建ClassLoader时直接指定一个ClassLoader为parent。 
2. 由getSystemClassLoader()方法生成，也就是在sun.misc.Laucher通过getClassLoader()获取，也就是AppClassLoader。直白的说，一个ClassLoader创建时如果没有指定parent，那么它的parent默认就是AppClassLoader。

  private ClassLoader loader;
  public Launcher() {
        // Create the extension class loader
        ClassLoader extcl;
        try {
            extcl = ExtClassLoader.getExtClassLoader();
        } catch (IOException e) { }
       try {
        //将ExtClassLoader对象实例传递进去
            loader = AppClassLoader.getAppClassLoader(extcl);
        } catch (IOException e) {}
  }
  public ClassLoader getClassLoader() {
        return loader;
    }

代码已经说明了问题AppClassLoader的parent是一个ExtClassLoader实例。ExtClassLoader的parent为null。

那么BootstrapClassLoader呢？

#### Bootstrap ClassLoader是由C++编写的。
Bootstrap ClassLoader是由C/C++编写的，它本身是虚拟机的一部分，所以它并不是一个JAVA类，也就是无法在java代码中获取它的引用，JVM启动时通过Bootstrap类加载器加载rt.jar等核心jar包中的class文件，之前的int.class,String.class都是由它加载。JVM初始化sun.misc.Launcher并创建Extension ClassLoader和AppClassLoader实例。并将ExtClassLoader设置为AppClassLoader的父加载器。Bootstrap没有父加载器，但是它却可以作用一个ClassLoader的父加载器。比如ExtClassLoader。

#### 双亲委托
一个类加载器查找class和resource时，是通过“委托模式”进行的，它首先判断这个class是不是已经加载成功，如果没有的话它并不是自己进行查找，而是先通过父加载器，然后递归下去，直到Bootstrap ClassLoader，如果Bootstrap classloader找到了，直接返回，如果没有找到，则一级一级返回，最后到达自身去查找这些对象。这种机制就叫做双亲委托。 

![](http://img.blog.csdn.net/20170210192931505?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYnJpYmx1ZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 重要方法
loadClass()
JDK文档中是这样写的，通过指定的全限定类名加载class，它通过同名的loadClass(String,boolean)方法。

	protected Class<?> loadClass(String name,  boolean resolve) throws ClassNotFoundException

一般实现这个方法的步骤是 

1. 执行findLoadedClass(String)去检测这个class是不是已经加载过了。 
2. 执行父加载器的loadClass方法。如果父加载器为null，则jvm内置的加载器去替代，也就是Bootstrap ClassLoader。这也解释了ExtClassLoader的parent为null,但仍然说Bootstrap ClassLoader是它的父加载器。 
3. 如果向上委托父加载器没有加载成功，则通过findClass(String)查找

如果class在上面的步骤中找到了，参数resolve又是true的话，那么loadClass()又会调用resolveClass(Class)这个方法来生成最终的Class对象。 

	protected Class<?> loadClass(String name, boolean resolve)throws ClassNotFoundException{
	    synchronized (getClassLoadingLock(name)) {
	        // 首先，检测是否已经加载
	        Class<?> c = findLoadedClass(name);
	        if (c == null) {
	            long t0 = System.nanoTime();
	            try {
	                if (parent != null) {
	                    //父加载器不为空则调用父加载器的loadClass
	                    c = parent.loadClass(name, false);
	                } else {
	                    //父加载器为空则调用Bootstrap Classloader
	                    c = findBootstrapClassOrNull(name);
	                }
	            } catch (ClassNotFoundException e) {
	                // ClassNotFoundException thrown if class not found
	                // from the non-null parent class loader
	            }
	
	            if (c == null) {
	                // If still not found, then invoke findClass in order
	                // to find the class.
	                long t1 = System.nanoTime();
	                //父加载器没有找到，则调用findclass
	                c = findClass(name);
	
	                // this is the defining class loader; record the stats
	                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
	                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
	                sun.misc.PerfCounter.getFindClasses().increment();
	            }
	        }
	        if (resolve) {
	            //调用resolveClass()
	            resolveClass(c);
	        }
	        return c;
	    }
	}

### 自定义ClassLoader

自定义步骤

1. 编写一个类继承自ClassLoader抽象类。
2. 复写它的findClass()方法。
3. 在findClass()方法中调用defineClass()。

defineClass()

这个方法在编写自定义classloader的时候非常重要，它能将class二进制内容转换成Class对象，如果不符合要求的会抛出各种异常。

一个ClassLoader创建时如果没有指定parent，那么它的parent默认就是AppClassLoader。

	public class DiskClassLoader extends ClassLoader {

		private String mLibPath;

		public DiskClassLoader(String path) {
			// TODO Auto-generated constructor stub
			mLibPath = path;
		}
	
		@Override
		protected Class<?> findClass(String name) throws ClassNotFoundException {
			// TODO Auto-generated method stub
	
			String fileName = getFileName(name);
	
			File file = new File(mLibPath,fileName);
	
			try {
				FileInputStream is = new FileInputStream(file);
	
				ByteArrayOutputStream bos = new ByteArrayOutputStream();
				int len = 0;
				try {
					while ((len = is.read()) != -1) {
						bos.write(len);
					}
				} catch (IOException e) {
					e.printStackTrace();
				}
	
				byte[] data = bos.toByteArray();
				is.close();
				bos.close();
	
				return defineClass(name,data,0,data.length);
	
			} catch (IOException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
	
			return super.findClass(name);
		}
	
		//获取要加载 的class文件名
		private String getFileName(String name) {
			// TODO Auto-generated method stub
			int index = name.lastIndexOf('.');
			if(index == -1){ 
				return name+".class";
			}else{
				return name.substring(index)+".class";
			}
		}
	
	}

调用

	//创建自定义classloader对象。
	DiskClassLoader diskLoader = new DiskClassLoader("D:\\lib");
	//加载class文件
	Class c = diskLoader.loadClass("com.frank.test.Test");

## 关键字 路径
BootStrap ClassLoader、ExtClassLoader、AppClassLoader都是加载指定路径下的jar包。如果我们要突破这种限制，实现自己某些特殊的需求，我们就得自定义ClassLoader，自已指定加载的路径，可以是磁盘、内存、网络或者其它。

常见的用法是将Class文件按照某种加密手段进行加密，然后按照规则编写自定义的ClassLoader进行解密，这样我们就可以在程序中加载特定了类，并且这个类只能被我们自定义的加载器进行加载，提高了程序的安全性。

## Context ClassLoader 线程上下文类加载器

	public class Thread implements Runnable {

	/* The context ClassLoader for this thread */
	   private ClassLoader contextClassLoader;
	
	   public void setContextClassLoader(ClassLoader cl) {
		   SecurityManager sm = System.getSecurityManager();
		   if (sm != null) {
			   sm.checkPermission(new RuntimePermission("setContextClassLoader"));
		   }
		   contextClassLoader = cl;
	   }
	
	   public ClassLoader getContextClassLoader() {
		   if (contextClassLoader == null)
			   return null;
		   SecurityManager sm = System.getSecurityManager();
		   if (sm != null) {
			   ClassLoader.checkClassLoaderPermission(contextClassLoader,
													  Reflection.getCallerClass());
		   }
		   return contextClassLoader;
	   }
	}

每个Thread都有一个相关联的ClassLoader，默认是AppClassLoader。并且子线程默认使用父线程的ClassLoader除非子线程特别设置。















