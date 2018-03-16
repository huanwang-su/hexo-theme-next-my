---
title: 云计算的三种服务模式
date: 2018/3/15 20:46:25
category:
- 大数据
- 云计算
tag:
- 云计算
comments: true  
---

# [云计算的三种服务模式：IaaS，PaaS和SaaS](http://www.cnblogs.com/beanmoon/archive/2012/12/10/2811547.html)

任何一个在互联网上提供其服务的公司都可以叫做云计算公司。其实云计算分几层的，分别是Infrastructure（基础设施）-as-a-Service，Platform（平台）-as-a-Service，Software（软件）-as-a-Service。基础设施在最下端，平台在中间，软件在顶端。别的一些“软”的层可以在这些层上面添加。

![](http://ww1.sinaimg.cn/large/0063bT3gly1fm8knoakiej30go0bg769.jpg)

### IaaS: Infrastructure-as-a-Service（基础设施即服务）

　　第一层叫做IaaS，有时候也叫做Hardware-as-a-Service，几年前如果你想在办公室或者公司的网站上运行一些企业应用，你需要去买服务器，或者别的高昂的硬件来控制本地应用，让你的业务运行起来。
　　但是现在有IaaS，你可以将硬件外包到别的地方去。IaaS公司会提供场外服务器，存储和网络硬件，你可以租用。节省了维护成本和办公场地，公司可以在任何时候利用这些硬件来运行其应用。
　　一些大的IaaS公司包括Amazon,
Microsoft, VMWare, Rackspace和Red Hat.不过这些公司又都有自己的专长，比如Amazon和微软给你提供的不只是IaaS，他们还会将其计算能力出租给你来host你的网站。

### PaaS: Platform-as-a-Service（平台即服务）

　　第二层就是所谓的PaaS，某些时候也叫做中间件。你公司所有的开发都可以在这一层进行，节省了时间和资源。
　　PaaS公司在网上提供各种开发和分发应用的解决方案，比如虚拟服务器和操作系统。这节省了你在硬件上的费用，也让分散的工作室之间的合作变得更加容易。网页应用管理，应用设计，应用虚拟主机，存储，安全以及应用开发协作工具等。
　　一些大的PaaS提供者有[Google AppEngine](http://venturebeat.com/2011/11/14/cloud-iaas-paas-saas/),Microsoft Azure，Force.com,Heroku，[Engine Yard](http://venturebeat.com/2011/08/23/engine-yard-acquires-orchestra/)。最近兴起的公司有[AppFog](http://venturebeat.com/2011/08/11/appfog-raises-8m-to-host-powerful-web-apps-in-the-cloud/), [Mendix](http://venturebeat.com/2011/10/31/mendix-grabs-13m-to-fuel-fast-enterprise-app-development/) 和 [Standing Cloud](http://venturebeat.com/2011/11/10/standing-cloud-cloud-app-management/)

### SaaS: Software-as-a-Service（软件即服务）

　　第三层也就是所谓SaaS。这一层是和你的生活每天接触的一层，大多是通过网页浏览器来接入。任何一个远程服务器上的应用都可以通过网络来运行，就是SaaS了。
　　你消费的服务完全是从网页如Netflix, MOG, Google Apps, Box.net, Dropbox或者苹果的iCloud那里进入这些分类。尽管这些网页服务是用作商务和娱乐或者两者都有，但这也算是云技术的一部分。
　　一些用作商务的SaaS应用包括Citrix的GoToMeeting，Cisco的WebEx，Salesforce的CRM，ADP，Workday和SuccessFactors。

### Iaas和Paas之间的比较

​    PaaS的主要作用是将一个开发和运行平台作为服务提供给用户，而IaaS的主要作用是提供虚拟机或者其他资源作为服务提供给用户。接下来，将在七个方面对PaaS和IaaS进行比较：

​    1) 开发环境：PaaS基本都会给开发者提供一整套包括IDE在内的开发和测试环境，而IaaS方面用户主要还是沿用之前比较熟悉那套开发环境，但是因为之前那套开发环境在和云的整合方面比较欠缺，所以使用起来不是很方便。
​    2) 支持的应用：因为IaaS主要是提供虚拟机，而且普通的虚拟机能支持多种操作系统，所以IaaS支持的应用的范围是非常广泛的。但如果要让一个应用能跑在某个PaaS平台不是一件轻松的事，因为不仅需要确保这个应用是基于这个平台所支持的语言，而且也要确保这个应用只能调用这个平台所支持的API，如果这个应用调用了平台所不支持的API，那么就需要对这个应用进行修改。
　3) 开放标准：虽然很多IaaS平台都存在一定的私有功能，但是由于OVF等协议的存在，使得IaaS在跨平台和避免被供应商锁定这两面是稳步前进的。而PaaS平台的情况则不容乐观，因为不论是Google的App Engine，还是Salesforce的Force.com都存在一定的私有API。
​    4) 可伸缩性：PaaS平台会自动调整资源来帮助运行于其上的应用更好地应对突发流量。而IaaS平台则需要开发人员手动对资源进行调整才能应对。
​    5) 整合率和经济性： PaaS平台整合率是非常高，比如PaaS的代表Google
App Engine能在一台服务器上承载成千上万的应用，而普通的IaaS平台的整合率最多也不会超过100，而且普遍在10左右，使得IaaS的经济性不如PaaS。
​    6) 计费和监管：因为PaaS平台在计费和监管这两方面不仅达到了IaaS平台所能企及的操作系统层面，比如，CPU和内存的使用量等，而且还能做到应用层面，比如，应用的反应时间（Response Time）或者应用所消耗的事务多少等，这将提高计费和管理的精确性。
​    7) 学习难度：因为在IaaS上面开发和管理应用和现有的方式比较接近，而PaaS上面开发则有可能需要学一门新的语言或者新的框架，所以IaaS学习难度更低。

|         | PaaS     | IaaS |
| ------- | -------- | ---- |
| 开发环境    | 完善       | 普通   |
| 支持的应用   | 有限       | 广    |
| 通用性     | 欠缺       | 稍好   |
| 可伸缩性    | 自动伸缩     | 手动伸缩 |
| 整合率和经济性 | 高整合率，更经济 | 低整合率 |
| 计费和监管   | 精细       | 简单   |
| 学习难度    | 略难       | 低    |

### 未来的PK

​    在当今云计算环境当中，IaaS是非常主流的，无论是Amazon EC2还是Linode或者Joyent等，都占有一席之地，但是随着Google的App Engine，Salesforce的Force.com还是微软的Windows Azure等PaaS平台的推出，使得PaaS也开始崭露头角。谈到这两者的未来，特别是这两者之间的竞争关系，我个人认为，短期而言，因为IaaS模式在支持的应用和学习难度这两方面的优势，使得IaaS将会在短期之内会成为开发者的首选，但是从长期而言，因为PaaS模式的高整合率所带来经济型使得如果PaaS能解决诸如通用性和支持的应用等方面的挑战，它将会替代IaaS成为开发者的“新宠”。