---
title: HDFS java api
date: 2018/3/15 20:46:25
category:
- 大数据
- hdfs
tag:
- hdfs
comments: true  
---

# 4 hdfs java开发

依赖：

```xml
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-client</artifactId>
    <version>2.9.0</version>
</dependency>

```

## 4.1 获取api中的客户端对象

```java
Configuration conf = new Configuration()
FileSystem fs = FileSystem.get(conf)

```

我们的操作目标是HDFS，所以获取到的fs对象应该是DistributedFileSystem的实例。get方法是从conf中的一个参数 fs.defaultFS的配置值判断用哪种文件系统。如果我们的代码中没有指定fs.defaultFS，并且工程classpath下也没有给定相应的配置，conf中的默认值就来自于hadoop的jar包中的core-default.xml，默认值为： file:///，则获取的是一个本地文件系统的客户端对象

## 4.2 DistributedFileSystem实例对象所具备的方法

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fm7iie3w5bj30lz0lkwgn.jpg)

## 4.3 HDFS客户端操作数据代码示例

文件操作

```java
public class HdfsClient {

	FileSystem fs = null;

	@Before
	public void init() throws Exception {

		// 构造一个配置参数对象，设置一个参数：我们要访问的hdfs的URI
		// 从而FileSystem.get()方法就知道应该是去构造一个访问hdfs文件系统的客户端，以及hdfs的访问地址
		// new Configuration();的时候，它就会去加载jar包中的hdfs-default.xml
		// 然后再加载classpath下的hdfs-site.xml
		Configuration conf = new Configuration();
		conf.set("fs.defaultFS", "hdfs://hdp-node01:9000");
		/**
		 * 参数优先级： 1、客户端代码中设置的值 2、classpath下的用户自定义配置文件 3、然后是服务器的默认配置
		 */
		conf.set("dfs.replication", "3");

		// 获取一个hdfs的访问客户端，根据参数，这个实例应该是DistributedFileSystem的实例
		// fs = FileSystem.get(conf);

		// 如果这样去获取，那conf里面就可以不要配"fs.defaultFS"参数，而且，这个客户端的身份标识已经是hadoop用户
		fs = FileSystem.get(new URI("hdfs://hdp-node01:9000"), conf, "hadoop");

	}

	/**
	 * 往hdfs上传文件
	 * 
	 * @throws Exception
	 */
	@Test
	public void testAddFileToHdfs() throws Exception {

		// 要上传的文件所在的本地路径
		Path src = new Path("g:/redis-recommend.zip");
		// 要上传到hdfs的目标路径
		Path dst = new Path("/aaa");
		fs.copyFromLocalFile(src, dst);
		fs.close();
	}

	/**
	 * 从hdfs中复制文件到本地文件系统
	 */
	@Test
	public void testDownloadFileToLocal() throws IllegalArgumentException, IOException {
		fs.copyToLocalFile(new Path("/jdk-7u65-linux-i586.tar.gz"), new Path("d:/"));
		fs.close();
	}

	@Test
	public void testMkdirAndDeleteAndRename() throws IllegalArgumentException, IOException {
		// 创建目录
		fs.mkdirs(new Path("/a1/b1/c1"));
		// 删除文件夹 ，如果是非空文件夹，参数2必须给值true
		fs.delete(new Path("/aaa"), true);
		// 重命名文件或文件夹
		fs.rename(new Path("/a1"), new Path("/a2"));

	}

	/**
	 * 查看目录信息，只显示文件
	 */
	@Test
	public void testListFiles() throws FileNotFoundException, IllegalArgumentException, IOException {
		// 思考：为什么返回迭代器，而不是List之类的容器
		RemoteIterator<LocatedFileStatus> listFiles = fs.listFiles(new Path("/"), true);
		while (listFiles.hasNext()) {
			LocatedFileStatus fileStatus = listFiles.next();
			System.out.println(fileStatus.getPath().getName());
			System.out.println(fileStatus.getBlockSize());
			System.out.println(fileStatus.getPermission());
			System.out.println(fileStatus.getLen());
			BlockLocation[] blockLocations = fileStatus.getBlockLocations();
			for (BlockLocation bl : blockLocations) {
				System.out.println("block-length:" + bl.getLength() + "--" + "block-offset:" + bl.getOffset());
				String[] hosts = bl.getHosts();
				for (String host : hosts) {
					System.out.println(host);
				}
			}
			System.out.println("--------------为angelababy打印的分割线--------------");
		}
	}

	/**
	 * 查看文件及文件夹信息
	 */
	@Test
	public void testListAll() throws FileNotFoundException, IllegalArgumentException, IOException {

		FileStatus[] listStatus = fs.listStatus(new Path("/"));

		String flag = "d--             ";
		for (FileStatus fstatus : listStatus) {
			if (fstatus.isFile())  flag = "f--         ";
			System.out.println(flag + fstatus.getPath().getName());
		}
	}
}

```

通过流的方式访问hdfs

```java
/**
 * 相对那些封装好的方法而言的更底层一些的操作方式
 * 上层那些mapreduce   spark等运算框架，去hdfs中获取数据的时候，就是调的这种底层的api
 * @author
 *
 */
public class StreamAccess {
	
	FileSystem fs = null;

	@Before
	public void init() throws Exception {
		Configuration conf = new Configuration();
		fs = FileSystem.get(new URI("hdfs://hdp-node01:9000"), conf, "hadoop");
	}
	
	/**
	 * 通过流的方式上传文件到hdfs
	 */
	@Test
	public void testUpload() throws Exception {		
		FSDataOutputStream outputStream = fs.create(new Path("/angelababy.love"), true);
		FileInputStream inputStream = new FileInputStream("c:/angelababy.love");	
		IOUtils.copy(inputStream, outputStream);		
	}
	
	@Test
	public void testDownLoadFileToLocal() throws IllegalArgumentException, IOException{
		//先获取一个文件的输入流----针对hdfs上的
		FSDataInputStream in = fs.open(new Path("/jdk-7u65-linux-i586.tar.gz"));		
		//再构造一个文件的输出流----针对本地的
		FileOutputStream out = new FileOutputStream(new File("c:/jdk.tar.gz"));	
		//再将输入流中数据传输到输出流
		IOUtils.copyBytes(in, out, 4096);			
	}	
	
	/**
	 * hdfs支持随机定位进行文件读取，而且可以方便地读取指定长度
	 * 用于上层分布式运算框架并发处理数据
	 */
	@Test
	public void testRandomAccess() throws IllegalArgumentException, IOException{
		//先获取一个文件的输入流----针对hdfs上的
		FSDataInputStream in = fs.open(new Path("/iloveyou.txt"));
		//可以将流的起始偏移量进行自定义
		in.seek(22);	
		//再构造一个文件的输出流----针对本地的
		FileOutputStream out = new FileOutputStream(new File("c:/iloveyou.line.2.txt"));
		IOUtils.copyBytes(in,out,19L,true);		
	}
	
	/**
	 * 显示hdfs上文件的内容
	 */
	@Test
	public void testCat() throws IllegalArgumentException, IOException{	
		FSDataInputStream in = fs.open(new Path("/iloveyou.txt"));	
		IOUtils.copyBytes(in, System.out, 1024);
	}
}

```

获取一个文件的所有block位置信息，然后读取指定block中的内容

```java
@Test
	public void testCat() throws IllegalArgumentException, IOException{
		
		FSDataInputStream in = fs.open(new Path("/weblog/input/access.log.10"));
		//拿到文件信息
		FileStatus[] listStatus = fs.listStatus(new Path("/weblog/input/access.log.10"));
		//获取这个文件的所有block的信息
		BlockLocation[] fileBlockLocations = fs.getFileBlockLocations(listStatus[0], 0L, listStatus[0].getLen());
		//第一个block的长度
		long length = fileBlockLocations[0].getLength();
		//第一个block的起始偏移量
		long offset = fileBlockLocations[0].getOffset();
		
		System.out.println(length);
		System.out.println(offset);
		
		//获取第一个block写入输出流
//		IOUtils.copyBytes(in, System.out, (int)length);
		byte[] b = new byte[4096];
		
		FileOutputStream os = new FileOutputStream(new File("d:/block0"));
		while(in.read(offset, b, 0, 4096)!=-1){
			os.write(b);
			offset += 4096;
			if(offset>=length) return;
		};
		os.flush();
		os.close();
		in.close();
	}

```





