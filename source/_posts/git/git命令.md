---
title: git命令
date: 2018-5-27 15:07:00
category:
- git
tag:
- git
comments: true  
---


# Git 命令

git中文件内容并没有真正存储在索引(*.git/index*)或者提交对象中，而是以blob的形式分别存储在数据库中(*.git/objects*)，并用SHA-1值来校验。 索引文件用识别码列出相关的blob文件以及别的数据。对于提交来说，以树(*tree*)的形式存储，同样用对于的哈希值识别。树对应着工作目录中的文件夹，树中包含的 树或者blob对象对应着相应的子目录和文件。每次提交都存储下它的上一级树的识别码。

![img](https://images0.cnblogs.com/blog/66979/201412/191735587664863.png) 

## 基本用法

![img](https://images0.cnblogs.com/i/66979/201406/262308046017546.jpg)

- `git add files` 把当前文件放入暂存区域。
- `git commit` 给暂存区域生成快照并提交。
- `git reset -- files` 用来撤销最后一次`git add files`，你也可以用`git reset` 撤销所有暂存区域文件。（操作对象是HEAD）
- `git checkout -- files` 把文件从暂存区域复制到工作目录，用来丢弃本地修改。（目的是working Directory）

![img](https://images0.cnblogs.com/i/66979/201406/262311265392839.jpg)

 

- `git commit -a `相当于运行 `git add` 把所有当前目录下的文件加入暂存区域再运行。`git commit`.
- `git commit files` 进行一次包含最后一次提交加上工作目录中文件快照的提交。并且文件被添加到暂存区域。
- `git checkout HEAD -- files` 回滚到复制最后一次提交。

### diff

![img](https://images0.cnblogs.com/i/66979/201406/262338188364937.jpg)

### commit

提交时，git用暂存区域的文件创建一个新的提交

![img](https://images0.cnblogs.com/i/66979/201406/262341091173993.jpg)

更改一次提交，使用 `git commit --amend`，git会使用与当前提交相同的父节点进行一次新提交，旧的提交会被取消，可以让你==合并你暂存区的修改和上一次commit==，amend不是修改最近一次commit, 而是整个替换掉他. 对于Git来说是一个新的commit. 

![img](https://images0.cnblogs.com/i/66979/201406/262345359613448.jpg)

### checkout

checkout命令用于从历史提交（或者暂存区域）中拷贝文件到工作目录，也可用于切换分支。

![img](https://images0.cnblogs.com/i/66979/201406/262348217421730.jpg)

注意当前分支不会发生变化(HEAD指向原处)。

**当不指定文件名**，而是给出一个（本地）分支时，那么*HEAD*标识会移动到那个分支

![img](https://images0.cnblogs.com/i/66979/201406/262350274921880.jpg)

`git checkout -b name`来创建一个新的分支

- `git checkout .` 或者 `git checkout -- <file>` 命令时，会用暂存区全部或指定的文件替换工作区的文件。这个操作很危险，会清除工作区中未添加到暂存区的改动。
- `git checkout HEAD .` 或者 `git checkout HEAD <file>` 命令时，会用 HEAD 指向的 master 分支中的全部或者部分文件替换暂存区和以及工作区中的文件。这个命令也是极具危险性的，因为不但会清除工作区中未提交的改动，也会清除暂存区中未提交的改 动。
- `git checkout -f`  =等价=> `git checkout HEAD .`

### reset

reset命令把当前分支指向另一个位置，并且有选择的变动工作目录和index。也用来在从历史仓库中复制文件到index，而不动工作目录。

![img](https://images0.cnblogs.com/i/66979/201406/262359481174654.jpg)

- `--hard` 重置HEAD返回到另外一个commit，重置index和working directory
- `--soft`  重置HEAD返回到另外一个commit
- `--mix` 默认参数，重置HEAD返回到另外一个commit，重置index

如果给了文件名, 那么工作效果和带文件名的`checkout`差不多，除了索引被更新。

![img](https://images0.cnblogs.com/i/66979/201406/270002318834820.jpg)

### merge

合并分支，索引必须和当前提交相同。如果当前提交是另一个分支的祖父节点，就导致*fast-forward*合并。指向只是简单的移动，并生成一个新的提交

![img](https://images0.cnblogs.com/i/66979/201406/270005447425305.jpg)

否则就是一次真正的合并。默认把当前提交(ed489 如下所示)和另一个提交(33104)以及他们的共同祖父节点(b325c)进行一次三方合并。

结果是先保存当前目录和索引，然后和父节点33104一起做一次新提交。

![img](https://images0.cnblogs.com/i/66979/201406/270006531338299.jpg)

### rebase

合并把两个父分支合并进行一次提交，提交历史不是线性的。衍合在当前分支上重演另一个分支的历史，提交历史是线性的。

![img](https://images0.cnblogs.com/i/66979/201406/270008484762671.jpg)

上面的命令都在*topic*分支中进行，而不是*master*分支

# GIT中版本的保存

### 文件状态

　　![img](https://images0.cnblogs.com/blog/168097/201308/07173755-e9cd02568038429e9546269a231adc94.png)

![img](https://images0.cnblogs.com/blog/168097/201308/07174428-5c25023c8f6a4cd5b758281f5e4dfa23.png)

- git directory就是我们的本地仓库.git目录，里面保存了所有的版本信息等内容。
- working driectory，工作目录，就是我们的工作目录，其中包括未跟踪文件及已跟踪文件，而已跟踪文件都是从git directory取出来的文件的某一个版本或新跟踪的文件。
- staging area，暂存区，不对应一个具体目录，其时只是git directory中的一个特殊文件。

当我们修改了一些文件后，要将其放入暂存区然后才能提交，每次提交时其实都是提交暂存区的文件到git仓库，然后==清除暂存区。==

# Git 常用命令

![img](https://pic002.cnblogs.com/img/1-2-3/201007/2010072023345292.png)