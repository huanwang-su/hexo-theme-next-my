---
title: git内部原理
date: 2018-5-27 13:29:39
category:
- git
tag:
- git
comments: true  
---


# Git 内部原理

从根本上来讲 Git 是一套内容寻址 (content-addressable) 文件系统，在此之上提供了一个 VCS 用户界面。

## 底层命令 (Plumbing) 和高层命令 (Porcelain)

在一个新目录或已有目录内执行 `git init` 时，Git 会创建一个 `.git` 目录，几乎所有 Git 存储和操作的内容都位于该目录下。如果你要备份或复制一个库，基本上将这一目录拷贝至其他地方就可以了。

![](http://ww1.sinaimg.cn/large/0063bT3gly1frpwfi8pafj30l30923z2.jpg)  

- `hooks` 目录保存了客户端或服务端钩子脚本。gitweb系统或其他git托管系统会经常用到hook script。
- `info` 包含仓库的一些信息
- `logs`:保存所有更新的引用记录
- `objects`:该目录存放所有的Git对象，对象的SHA1哈希值的前两位是文件夹名称，后38位作为对象文件名。
- `refs`:具体的引用，Reference Specification，存储指向数据 (分支) 的提交对象的指针，这个目录一般包括三个子文件夹，`heads`、`remotes`和`tags`
- `HEAD` 文件指向当前分支
- `config`:这个是GIt仓库的配置文件
- `description`:仓库的描述信息，主要给gitweb等git托管系统使用
- `index`:这个文件就是我们前面提到的暂存区（stage），是一个二进制文件


## Git对象

Git 是一套内容寻址文件系统。它允许插入任意类型的内容，并会返回一个键值，通过该键值可以在任何时候再取出该内容。

初使化一个 Git 仓库并确认 `objects` 目录是空的，Git 初始化了 `objects` 目录，同时在该目录下创建了 `pack` 和 `info` 子目录，但是该目录下没有其他常规文件。

 Git 存储数据内容的方式──为每份内容生成一个文件，取得该内容与头信息的 SHA-1 校验和，创建以该校验和前两个字符为名称的子目录，并以 (校验和) 剩下 38 个字符为文件命名 (保存至子目录下)。

### tree (树) 对象

Git 以一种类似 UNIX 文件系统但更简单的方式来存储内容。所有内容以 tree 或 blob 对象存储，其中 tree 对象对应于 UNIX 中的目录，blob 对象则大致对应于 inodes 或文件内容。一个单独的 tree 对象包含一条或多条 tree 记录，每一条记录含有一个指向 blob 或子 tree 对象的 SHA-1 指针，并附有该对象的权限模式 (mode)、类型和文件名信息。

![img](https://gitee.com/progit/figures/18333fig0901-tn.png) 

