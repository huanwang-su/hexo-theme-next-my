---
title: java io 
date: 2018/3/16 08:28:25
category:
- java基础
- java基础
tag:
- nio 
comments: true  
---

## IO流 ##
所有的输入类都继承自InputStream,输入类都继承自OutputStream

### InputStream ###
InputStream的作用是标志那些从不同数据起源产生输入的类。这些数据起源包括（每个都有一个相关的InputStream子类）：

-  字节数组
-  String对象
-  文件
-  “管道”，它的工作原理与现实生活中的管道类似：将一些东西一端置入，它们在另一端输出。
-  一个由其他种类的流组成的序列，以便我们将其统一收集合并到一个流内。
-  其他数据集，如Internet连接等。

![](http://i.imgur.com/2jlcRWb.png)

### outputStream的类型 ###
这一类别包括的类决定了我们的输入往何处去：一个字节数组（但没有String；假定我们可用字节数组创建一个）；一个文件；或者一个“管道”。

![](http://i.imgur.com/l8sEpQ7.png)

### 过滤器 ###

FilterInputStream 和FilterOutputStream （这两个名字不十分直观）提供了相应的装饰器接口，用于控制一个特定的输入流（InputStream）或者输出流（OutputStream）。

FilterInputStream类型

![](http://i.imgur.com/fNo9IC4.png)

FilterOutputStream 的类型

![](http://i.imgur.com/OTvGCar.png)

## File类 ##
代表文件或者目录

### 目录列表器 ###
列出File 对象，调用list()，会获得File 对象包含的一个完整列表。然而，若想对这个列表进行某些限制，就需要使用一个“目录过滤器”

	public interface FilenameFilter {
		boolean accept(文件目录, 字串名);
	}

把accept()方法提供给list()方法，使list()能够“回调”accept() ，从而判断应将哪些文件名包
括到列表中。

举例：匿名内部类
		
	//: DirList3.java
	// Building the anonymous inner class "in-place"
	import java.io.*;
	public class DirList3 {
		public static void main(final String[] args) {
			try {
				File path = new File(".");
				String[] list;
				if(args.length == 0)
					list = path.list();
				else
					list = path.list(
						new FilenameFilter() {
							public boolean accept(File dir, String n) {
								String f = new File(n).getName();
								return f.indexOf(args[0]) != -1;
							}
						});
				for(int i = 0; i < list.length; i++)
					System.out.println(list[i]);
			} catch(Exception e) {
				e.printStackTrace();
			}
		}
	} ///:~

### 顺序目录列表 ###
将list提为成员变量，再进行排序list

### 检查与创建目录 ###
File 类并不仅仅是对现有目录路径、文件或者文件组的一个表示。亦可用一个File 对象新建一个目录，甚至创建一个完整的目录路径——假如它尚不存在的话。亦可用它了解文件的属性（长度、上一次修改日期、
读/写属性等），检查一个File 对象到底代表一个文件还是一个目录，以及删除一个文件等等。

具体查看jdk文档

## 输入流 ##
### 1. 从文件输入 ###
为打开一个文件以便输入，需要使用一个FileInputStream，同时将一个String 或File 对象作为文件名使用。为提高速度，最好先对文件进行缓冲处理，从而获得用于一个BufferedInputStream 的构建器的结果句柄。为了以格式化的形式读取输入数据，我们将那个结果句柄赋给用于一个DataInputStream 的构建器。DataInputStream 是我们的最终（final）对象，并是我们进行读取操作的接口。
### 2. 从内存输入 ###
这一部分采用已经包含了完整文件内容的String s2，并用它创建一个StringBufferInputStream（字串缓冲输入流）——作为构建器的参数，要求使用一个String，而非一个StringBuffer）。随后，我们用read()依次读取每个字符，并将其发送至控制台

### 3. 格式化内存输入 ###
StringBufferInputStream 的接口是有限的，所以通常需要将其封装到一个DataInputStream 内，从而增强它的能力。。然而，若选择用readByte()每次读出一个字符，那么所有值都是有效的，所以不可再用返回值来侦测何时结束输入。相反，可用available()方法判断有多少字符可用。

	import java.io.*;
	public class TestEOF {
		public static void main(String[] args) {
			try {
				DataInputStream in =
				new DataInputStream(
					new BufferedInputStream(
						new FileInputStream("TestEof.java")));
				while(in.available() != 0)
					System.out.print((char)in.readByte());
			} catch (IOException e) {
				System.err.println("IOException");
			}
		}
	} ///:~

### 4. 行的编号与文件输出 ###
标志DataInputStream 何时结束的一个方法是readLine()。一旦没有更多的字串可以读取，它就会返回null。每个行都会伴随自己的行号打印到文件里。

## 输出流 ##
两类主要的输出流是按它们写入数据的方式划分的：一种按人的习惯写入，另一种为了以后由一个
DataInputStream 而写入。RandomAccessFile 是独立的，尽管它的数据格式兼容于DataInputStream 和DataOutputStream。

### 保存与恢复数据 ###
PrintStream 能格式化数据，使其能按我们的习惯阅读。但为了输出数据，以便由另一个数据流恢复，则需用一个DataOutputStream 写入数据，并用一个DataInputStream 恢复（获取）数据。

为了保证任何读方法能够正常工作，必须知道数据项在流中的准确位置，因为既有可能将保存的double 数据作为一个简单的字节序列读入，也有可能作为char 或其他格式读入。所以必须要么为文件中的数据采用固定的格式，要么将额外的信息保存到文件中，以便正确判断数据的存放位置。

## 从标准输入中读取数据 ##
	import java.io.*;
	public class Echo {
		public static void main(String[] args) {
			DataInputStream in =
			new DataInputStream(
				new BufferedInputStream(System.in));
			String s;
			try {
				while((s = in.readLine()).length() != 0)
					System.out.println(s);
			// An empty line terminates the program
			} catch(IOException e) {
				e.printStackTrace();
			}
		}
	} ///:~

## 重导向标准I O ##
Java 1.1 在System 类中添加了特殊的方法，允许我们重新定向标准输入、输出以及错误IO 流。此时要用到下述简单的静态方法调用：

- setIn(InputStream)
- setOut(PrintStream)
- setErr(PrintStream)

例子

	import java.io.*;
	class Redirecting {
		public static void main(String[] args) {
			try {
				BufferedInputStream in =
				new BufferedInputStream(new FileInputStream("Redirecting.java"));
				// Produces deprecation message:
				PrintStream out =new PrintStream(new BufferedOutputStream(new FileOutputStream("test.out")));
				System.setIn(in);
				System.setOut(out);
				System.setErr(out);
				BufferedReader br =	new BufferedReader(new InputStreamReader(System.in));
				String s;
				while((s = br.readLine()) != null)
					System.out.println(s);
				out.close(); // Remember this!
			} catch(IOException e) {
				e.printStackTrace();
			}
		}
	} 
## 压缩 ##

## 序列化 ##
Serialization（序列化）是一种将对象以一连串的字节描述的过程；反序列化deserialization是一种将这些字节重建成一个对象的过程。

什么情况下需要序列化 

1. 当你想把的内存中的对象保存到一个文件中或者数据库中时候；
2. 当你想用套接字在网络上传送对象的时候；
3. 当你想通过RMI传输对象的时候；

如果我们想要序列化一个对象，首先要创建某些OutputStream(如FileOutputStream、ByteArrayOutputStream等)，然后将这些OutputStream封装在一个ObjectOutputStream中。这时候，只需要调用writeObject()方法就可以将对象序列化，并将其发送给OutputStream（记住：对象的序列化是基于字节的，不能使用Reader和Writer等基于字符的层次结构）。而反序列的过程（即将一个序列还原成为一个对象），需要将一个InputStream(如FileInputstream、ByteArrayInputStream等)封装在ObjectInputStream内，然后调用readObject()即可。


	public class MyTest implements Serializable
	{
		private static final long serialVersionUID = 1L;
		private String name="SheepMu";
		private int age=24;
		public static void main(String[] args)
		{//以下代码实现序列化
			try 
			{
				ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("my.out"));//输出流保存的文件名为 my.out ；ObjectOutputStream能把Object输出成Byte流
				MyTest myTest=new MyTest();
				oos.writeObject(myTest); 
				oos.flush();  //缓冲流 
				oos.close(); //关闭流
			} catch (FileNotFoundException e) 
			{		 
				e.printStackTrace();
			} catch (IOException e) 
			{
				e.printStackTrace();
			} 
			fan();//调用下面的  反序列化  代码
		}
		public static void fan()//反序列的过程
		{	 
	        ObjectInputStream oin = null;//局部变量必须要初始化
			try
			{
				oin = new ObjectInputStream(new FileInputStream("my.out"));
			} catch (FileNotFoundException e1)
			{		 
				e1.printStackTrace();
			} catch (IOException e1)
			{
				e1.printStackTrace();
			}      
	        MyTest mts = null;
			try {
				mts = (MyTest ) oin.readObject();//由Object对象向下转型为MyTest对象
			} catch (ClassNotFoundException e) {
				e.printStackTrace();
			} catch (IOException e) {
				e.printStackTrace();
			}     
	         System.out.println("name="+mts.name);    
	         System.out.println("age="+mts.age);    
		}
	}
深复制

	public Object deepCopy() throws IOException, ClassNotFoundException{
        //字节数组输出流，暂存到内存中
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        //序列化
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(this);
        ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bis);
        //反序列化
        return ois.readObject();
    }
