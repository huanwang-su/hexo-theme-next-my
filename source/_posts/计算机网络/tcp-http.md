---
title: tcp/http
date: 2018/3/16 08:28:25
category:
- 计算机网络
tag:
- tcp
- 计算机网络
comments: true  
---

## 计算机网络 ##

### tcp ###
#### TCP报文格式 ####
![](http://www.2cto.com/uploadfile/2013/1022/20131022025345890.png)

- 序号：Seq序号，占32位，用来标识从TCP源端向目的端发送的字节流，发起方发送数据时对此进行标记。
- 确认序号：Ack序号，占32位，只有ACK标志位为1时，确认序号字段才有效，Ack=Seq+1。
- 标志位：共6个，即URG、ACK、PSH、RST、SYN、FIN等，具体含义如下：
- URG：紧急指针（urgent pointer）有效。
- ACK：确认序号有效。
- PSH：接收方应该尽快将这个报文交给应用层。
- RST：重置连接。
- SYN：发起一个新连接。
- FIN：释放一个连接。

确认方Ack=发起方Req+1，两端配对。 
#### TCP连接 ####
Transmission Control Protocol 传输控制协议
![](http://www.2cto.com/uploadfile/2013/1022/20131022025346218.png)
>
>  1. 第一次握手：Client将标志位SYN置为1，随机产生一个值seq=J，并将该数据包发送给Server，Client进入SYN_SENT状态，等待Server确认。
>  2. 第二次握手：Server收到数据包后由标志位SYN=1知道Client请求建立连接，Server将标志位SYN和ACK都置为1，ack=J+1，随机产生一个值seq=K，并将该数据包发送给Client以确认连接请求，Server进入SYN_RCVD状态。
>  3. 第三次握手：Client收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=K+1，并将该数据包发送给Server，Server检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，Client和Server进入ESTABLISHED状态，完成三次握手，随后Client与Server之间可以开始传输数据了

  **SYN攻击：**

　　SYN攻击就是Client在短时间内伪造大量不存在的IP地址，并向Server不断地发送SYN包，Server回复确认包，并等待Client的确认，由于源地址是不存在的，因此，Server需要不断重发直至超时，这些伪造的SYN包将产时间占用未连接队列，导致正常的SYN请求因为队列满而被丢弃，从而引起网络堵塞甚至系统瘫痪。
#### TCP释放 ####
四次挥手（Four-Way Wavehand）即终止TCP连接，就是指断开一个TCP连接时，需要客户端和服务端总共发送4个包以确认连接的断开。在socket编程中，这一过程由客户端或服务端任一方执行close来触发

![](http://www.2cto.com/uploadfile/2013/1022/20131022025350523.png)

1. 第一次挥手：Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态。
2. 第二次挥手：Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态。
3. 第三次挥手：Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态。
4. 第四次挥手：Client收到FIN后，Client进入TIME_WAIT状态，接着发送一个ACK给Server，确认序号为收到序号+1，Server进入CLOSED状态，完成四次挥手。

**为什么建立连接是三次握手，而关闭连接却是四次挥手呢？**

　　这是因为服务端在LISTEN状态下，收到建立连接请求的SYN报文后，把ACK和SYN放在一个报文里发送给客户端。而关闭连接时，当收到对方的FIN报文时，仅仅表示对方不再发送数据了但是还能接收数据，己方也未必全部数据都发送给对方了，所以己方可以立即close，也可以发送一些数据给对方后，再发送FIN报文给对方来表示同意现在关闭连接，因此，己方ACK和FIN一般都会分开发送。
<hr>
### UDP ###
UDP 是User Datagram Protocol的简称，提供面向事务的简单不可靠信息传送服务

- 关于何时，发什么数据应用层控制更为精细：*应用程序交付报文给UDP，UDP打包给网络层*
- 无需建立连接：*无需3次握手*
- 无连接状态：*udp不维护连接状态*
- 分组首部开销小：*20byte/8byte*

udp也可以可靠，但是要构建在应用程序中
#### udp报文 ####
![](http://blog.chinaunix.net/attachment/201304/27/26833883_13670536278Q7Q.png)

----------
### tcp与udp区别 ###
1. TCP协议面向连接，UDP协议面向非连接；
2. TCP协议传输速度慢，UDP协议传输速度快
3. TCP协议可靠数据传输，UDP协议可能丢包；
4. UDP程序结构较简单
5. 流模式和数据包模式

>通过TCP连接给另一端发送数据，你只调用了一次write，发送了100个字节，但是对方可以分10次收完，每次10个字节；你也可以调用10次write，每次10个字节，但是对方可以一次就收完.
>
>UPD是基于报文的，在接收的时候，每次最多只能读取一个报文，报文和报文是不会合并的，如果缓冲区小于报文长度，则多出的部分会被丢弃。

### 停等协议和流水线操作 ###
**停等协议**每发送完一个分组就停止发送，等待对方的确认。在收到确认后再发送下一个分组。
![](http://i.imgur.com/2Hu8auZ.png)
**流水线操作**
![](http://i.imgur.com/T5QYXv8.png)
>1. 必须增加序号范围
>2. 发送方和接收方必须缓存多个分组
>3. 处理丢失,损坏,延时, **回退N步(GBN),选择重传(SR)**

#### GBN:

发送的窗口大小为n，接受方的窗口仍然为1。具体看下面的图，这里假设n=9：      首先发送方一口气发送10个数据帧，前面两个帧正确返回了，数据帧2出现了错误，这时发送方被迫重新发送2-8这7个帧，接受方也必须丢弃之前接受的3-8这几个帧。      后退n协议的好处无疑是提高了效率，但是一旦网络情况糟糕，则会导致大量数据重发

只有一个计时器，只记录每一组发送的时间

#### SR:

后退n协议的另外一个问题是，当有错误帧出现后，总是要重发该帧之后的所有帧，毫无疑问在网络不是很好的情况下会进一步恶化网络状况。

接收端总会缓存所有收到的帧，当某个帧出现错误时，只会要求重传这一个帧，只有当某个序号后的所有帧都正确收到后，才会一起提交给高层应用。重传协议的缺点在于接受端需要更多的缓存。

每个分组都有计时器

SR发送方的事件与动作：

1. 收到ACK，倘若该分组序号在窗口内，则SR发送方将那个被确认的分组标记为已接收。如果该分组的序号等于send_base，则窗口基序号向前移动到具有最小序号的未确认分组处。如果窗口移动了并且有序号落在窗口内的未发送分组，则发送这些分组。

SR接收方的事件与动作：

1. 序号在［rcv_base，rcv_base+N-1]内的分组被正确接收。在此情况下，收到的分组落在接收方的窗口内，一个选择ACK被回送给发送方。并判断是否交付
2. 序号在［rcv_base-N,rcv_base-1]内的分组被正确收到（客户端重传）。在此情况下，必须产生一个ACK，即使该分组是接收方以前已确认过的分组。

窗口大小：发送方=接收方

#### TCP

TCP并不是每一个报文段都会回复ACK的，可能会对两个报文段发送一个ACK，也可能会对多个报文段发送1个ACK【**累计ACK**】

滑动窗口协议是**传输层进行流控**的一种措施，**接收方通过通告发送方自己的窗口大小**，从而控制发送方的发送速度，从而达到防止发送方发送速度过快而导致自己被淹没的目的。

ACK包含两个非常重要的信息：

1. **一是期望接收到的下一字节的序号n，该n代表接收方已经接收到了前n-1字节数据**
2. **二是当前的窗口大小m，如此发送方在接收到ACK包含的这两个数据后就可以计算出还可以发送多少字节的数据给对方，假定当前发送方已发送到第x字节，则可以发送的字节数就是y=m-(x-n).**
3. ​

### 可靠数据传输机制及其用途的总结

​    检验和：用于检测在一个传输分组中的比特错误

​    定时器：用于超时／重传一个分组，可能因为该分组（或其ACK）在信道中丢失了。由于当一个分组延时但未丢失（过早超时），或当一个分组已经被接收方收到但从接收方到发送方的ACK丢失时，可能产生超时事件，所以接收方可能会收到一个分组的多个冗余副本。

​    序号：用于为从发送方流向接收方的数据分组按顺序编号。所接收分组的序号空间的空隙可使接收方检测出丢失的分组。具有相同序号的分组可使接收方检测出一个分组的冗余副本。

​    确认：接收方用于告诉发送方一个分组或一组分组已经被正确滴接收到了。确认报文通常携带着被确认的分组或多个分组的序号。确认可以是逐个的或累积的，着取决于协议。

​    否定确认：接收方用于告诉发送方某个分组未被正确地接收。否定确认报文通常携带着未被正确接收到分组的序号。

​    窗口、流水线：发送方也许被限制仅发送那些序号落在一个指定范围内的分组。通过允许一次发送多个分组但未被确认，发送方的利用率可以在停等操作模式的基础上得到增加。我们很快会看到，窗口长度可根据接收方接收的缓存报文的能力、网络中的拥塞程度或两者情况来进行设置。

### TCP拥塞控制 ###
拥塞窗口(cwnd):指某一源端数据流在一个RTT内可以最多发送的数据包数。
指导性原则:

- A lost segment implies congestion, and hence, the TCP sender’s rate should be decreased when a segment is lost.(丢失报文即阻塞)
- An acknowledged segment indicates that the network is delivering the sender’s
  segments to the receiver, and hence, the sender’s rate can be increased when an
  ACK arrives for a previously unacknowledged segment(收到ack即非阻塞)
- Bandwidth probing.TCP’s strategy for adjusting its
  transmission rate is to increase its rate in response to arriving ACKs until a loss
  event occurs, at which point, the transmission rate is decreased.(带宽检测,收到ack加速,丢包减速)

拥塞控制算法:

1. 慢起动:cwnd小,然后依据ack快速增长(指数),到达拥塞最大值
2. 拥塞避免:一旦进入此状态,cwnd值大约为之前的一半,慢增长(+1)
3. 快速恢复:快重传算法规定，发送方只要一连收到三个重复确认就应当立即重传对方尚未收到的报文段，而不必继续等待设置的重传计时器时间到期。
  ![](http://i.imgur.com/seh4G0D.png)

为了防止cwnd增长过大引起网络拥塞，还需设置一个慢开始门限ssthresh状态变量。ssthresh的用法如下：

- 当cwnd<ssthresh时，使用慢开始算法。
- 当cwnd>ssthresh时，改用拥塞避免算法。
- 当cwnd=ssthresh时，慢开始与拥塞避免算法任意。

快重传

- 当发送方连续收到三个重复确认时，就执行“乘法减小”算法，把ssthresh门限减半。但是接下去并不执行慢开始算法。
- 考虑到如果网络出现拥塞的话就不会收到好几个重复的确认，所以发送方现在认为网络可能没有出现拥塞。所以此时不执行慢开始算法，而是将cwnd设置为ssthresh的大小，然后执行拥塞避免算法。


ACK延迟确认机制
接收方在收到数据后，并不会立即回复ACK,而是延迟一定时间。一般ACK延迟发送的时间为200ms，但这个200ms并非收到数据后需要延迟的时间。系统有一个固定的定时器每隔200ms会来检查是否需要发送ACK包。这样做有两个目的。

1. 这样做的目的是ACK是可以合并的，也就是指如果连续收到两个TCP包，并不一定需要ACK两次，只要回复最终的ACK就可以了，可以降低网络流量。
2. 如果接收方有数据要发送，那么就会在发送数据的TCP数据包里，带上ACK信息。这样做，可以避免大量的ACK以一个单独的TCP包发送，减少了网络流量。

## http ##

### 在TCP/IP协议栈中的位置

在TCP/IP协议栈中的位置HTTP协议通常承载于TCP协议之上，有时也承载于TLS或SSL协议层之上，这个时候，就成了我们常说的HTTPS。如下图所示：   

![img](http://www.blogjava.net/images/blogjava_net/amigoxie/40799/o_http%e5%8d%8f%e8%ae%ae%e5%ad%a6%e4%b9%a0-11.jpg)

默认HTTP的端口号为80，HTTPS的端口号为443。

### HTTP的请求响应模型

HTTP的请求响应模型HTTP协议永远都是客户端发起请求，服务器回送响应。无法实现在客户端没有发起请求的时候，服务器将消息推送给客户端。HTTP协议是一个无状态的协议，协议的状态是指下一次传输可以“记住”这次传输信息的能力。

### 工作流程

一次HTTP操作称为一个事务，其工作过程可分为四步：

1. 客户机与服务器需要建立连接
2. 客户机发送一个请求给服务器
3. 服务器接到请求后，给予相应的响应信息
4. 客户端接收服务器所返回的信息，然后客户机与服务器断开连接。

### HTTP/1.0和HTTP/1.1的比较

1. 建立连接方面：

   HTTP/1.0 每次请求都需要建立新的TCP连接，连接不能复用。HTTP/1.1 新的请求可以在上次请求建立的TCP连接之上发送，连接可以复用。优点是减少重复进行TCP三次握手的开销，提高效率。

   注意：在同一个TCP连接中，新的请求需要等上次请求收到响应后，才能发送。

2. Host域

   HTTP1.1在Request消息头里头多了一个Host域, HTTP1.0则没有这个域。

### HTTP之请求消息Request

请求行（request line）、请求头部（header）、空行和请求数据四个部分组成。

![img](https://upload-images.jianshu.io/upload_images/2964446-fdfb1a8fce8de946.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```http
GET /562f25980001b1b106000338.jpg HTTP/1.1
Host    img.mukewang.com
User-Agent    Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.106 Safari/537.36
Accept    image/webp,image/*,*/*;q=0.8
Referer    http://www.imooc.com/
Accept-Encoding    gzip, deflate, sdch
Accept-Language    zh-CN,zh;q=0.8


```

```http
POST / HTTP1.1
Host:www.wrox.com
User-Agent:Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; .NET CLR 2.0.50727; .NET CLR 3.0.04506.648; .NET CLR 3.5.21022)
Content-Type:application/x-www-form-urlencoded
Content-Length:40
Connection: Keep-Alive

name=Professional%20Ajax&publisher=Wiley

```

- Accept：浏览器可接受的MIME类型。
- Accept-Charset：浏览器可接受的字符集。
- Accept-Encoding：浏览器能够进行解码的数据编码方式，比如gzip。Servlet能够向支持gzip的浏览器返回经gzip编码的HTML页面。许多情形下这可以减少5到10倍的下载时间。
- Accept-Language：浏览器所希望的语言种类，当服务器能够提供一种以上的语言版本时要用到。
- Authorization：授权信息，通常出现在对服务器发送的WWW-Authenticate头的应答中。
- Connection：表示是否需要持久连接。
- Content-Length：表示请求消息正文的长度。post才需要，因为其他的方式正文为0
- Cookie：这是最重要的请求头信息之一
- From：请求发送者的email地址，由一些特殊的Web客户程序使用，浏览器不会用到它。
- Host：初始URL中的主机和端口。
- If-Modified-Since：只有当所请求的内容在指定的日期之后又经过修改才返回它，否则返回304"Not Modified"应答。
- Pragma：指定"no-cache"值表示服务器必须返回一个刷新后的文档，即使它是代理服务器而且已经有了页面的本地拷贝。
- Referer：包含一个URL，用户从该URL代表的页面出发访问当前请求的页面。
- User-Agent：浏览器类型，如果Servlet返回的内容与浏览器类型有关则该值非常有用。

### HTTP之响应消息Response

HTTP响应也由四个部分组成，分别是：状态行、消息报头、空行和响应正文。

``` http
HTTP/1.1 200 OK
Date: Fri, 22 May 2009 06:07:21 GMT
Content-Type: text/html; charset=UTF-8

<html>
      <head></head>
      <body>
            <!--body goes here-->
      </body>
</html>
```

状态行，HTTP协议版本号， 状态码， 状态消息 

消息报头

空行

响应正文

### HTTP请求方法

根据HTTP标准，HTTP请求可以使用多种请求方法。
HTTP1.0定义了三种请求方法： GET, POST 和 HEAD方法。
HTTP1.1新增了五种请求方法：OPTIONS, PUT, DELETE, TRACE 和 CONNECT 方法。

- GET     请求指定的页面信息，并返回实体主体。
- HEAD     类似于get请求，只不过返回的响应中没有具体的内容，用于获取报头
- POST     向指定资源提交数据进行处理请求（例如提交表单或者上传文件）。数据被包含在请求体中。POST请求可能会导致新的资源的建立和/或已有资源的修改。
- PUT     从客户端向服务器传送的数据取代指定的文档的内容。
- DELETE      请求服务器删除指定的页面。
- CONNECT     HTTP/1.1协议中预留给能够将连接改为管道方式的代理服务器。
- OPTIONS     允许客户端查看服务器的性能。
- TRACE     回显服务器收到的请求，主要用于测试或诊断

### KeepAlive

Connection: close/Keep-Alive

HTTP 1.0加上Keep-Alive header也可以提供HTTP的持续作用功能。

Keep-Alive: timeout=5, max=100 过期时间5秒 最多一百次请求

HTTP/1.0的默认情况下，是不会使用Keep-Alive的, HTTP/1.1持久连接将是默认的连接方式。



### HTTPS传输协议原理

以安全为目标的HTTP通道，简单讲是HTTP的安全版。即HTTP下加入SSL层，HTTPS的安全基础是SSL   

![img](https://images0.cnblogs.com/i/116165/201407/122142366922455.png)

### http的状态响应码

1**(信息类)：表示接收到请求并且继续处理
​	100——客户必须继续发出请求
​	101——客户要求服务器根据请求转换HTTP协议版本

2**(响应成功)：表示动作被成功接收、理解和接受

3**(重定向类)：为了完成指定的动作，必须接受进一步处理

4*(客户端错误类)：请求包含错误语法或不能正确执行

5**(服务端错误类)：服务器不能正确执行一个正确的请求

### URI 和 URL 区别

统一资源标志符URI就是在某一规则下能把一个资源独一无二地标识出来。

统一资源定位符URL是URI的子集，URL是以描述位置来唯一确定的。

假设所有的Html文档都有唯一的编号，记作html:xxxxx，xxxxx是一串数字，即Html文档的身份证号码，这个能唯一标识一个Html文档，那么这个号码就是一个URI。而URL则通过描述是哪个主机上哪个路径上的文件来唯一确定一个资源，也就是定位的方式来实现的URI。

### 解决HTTP无状态的问题

#### 通过Cookies保存状态信息

cookie在请求和响应的http首部，从服务器到客户端，再从客户端到服务器。服务器用cookie来指示会话ID，登录凭据等。

#### 通过Session保存状态信息

Session机制是一种服务器端的机制，服务器使用一种类似于散列表的结构（也可能就是使用散列表）来保存信息。客户端的请求里包含了一个session标识 - 称为 session id, 服务器就按照session id把这个 session检索出来使用

Session的实现方式：

1. 使用Cookie来实现

   服务器给每个Session分配一个唯一的JSESSIONID，并通过Cookie发送给客户端。
   当客户端发起新的请求的时候，将在Cookie头中携带这个JSESSIONID。

2. 使用URL回写来实现

   URL回写是指服务器在发送给浏览器页面的所有链接中都携带JSESSIONID的参数。Tomcat对Session的实现，是一开始同时使用Cookie和URL回写机制，如果发现客户端支持Cookie，就继续使用Cookie，停止使用URL回写。如果发现Cookie被禁用，就一直使用URL回写。

#### Cookie和Session有以下明显的不同点：

1. Cookie将状态保存在客户端，Session将状态保存在服务器端；
2. Cookies是服务器在本地机器上存储的小段文本并随每一个请求发送至同一个服务器。网络服务器用HTTP头向客户端发送cookies，在客户终端，浏览器解析这些cookies并将它们保存为一个本地文件，它会自动将同一服务器的任何请求缚上这些cookies。Session并没有在HTTP的协议中定义；
3. Session是针对每一个用户的，变量的值保存在服务器上，用一个sessionID来区分是哪个用户session变量,这个值是通过用户的浏览器在访问的时候返回给服务器，当客户禁用cookie时，通过URL回写机制
4. 服务器端的SESSION机制更安全些。因为它不会任意读取客户存储的信息。

### 浏览器输入网址发生了什么

1. 浏览器里输入网址

2. 浏览器查找域名的IP地址

   导航的第一步是通过访问的域名找出其IP地址。DNS查找过程如下：

   - 浏览器缓存 – 浏览器会缓存DNS记录一段时间。 不同浏览器会储存个自固定的一个时间（2分钟到30分钟不等）。
   - 系统缓存 – 如果在浏览器缓存里没有找到需要的记录，浏览器会做一个系统调用（windows里是gethostbyname）。这样便可获得系统缓存中的记录。
   - 路由器缓存 – 接着，前面的查询请求发向路由器，它一般会有自己的DNS缓存。
   - ISP DNS 缓存 – 接下来要check的就是ISP缓存DNS的服务器。
   - 递归搜索 – 你的ISP的DNS服务器从跟域名服务器开始进行递归搜索，从.com顶级域名服务器到Facebook的域名服务器。一般DNS服务器的缓存中会有.com域名服务器中的域名，所以到顶级服务器的匹配过程不是那么必要了。

   DNS递归查找如下图所示： ![img](http://igoro.com/wordpress/wp-content/uploads/2010/02/500pxAn_example_of_theoretical_DNS_recursion_svg.png)

3. 浏览器给web服务器发送一个HTTP请求

4. facebook服务的永久重定向响应

5. 浏览器跟踪重定向地址

6. 服务器“处理”请求

7. 服务器发回一个HTML响应

8. 浏览器开始显示HTML

9. 浏览器发送获取嵌入在HTML中的对象

10. 浏览器发送异步（AJAX）请求



