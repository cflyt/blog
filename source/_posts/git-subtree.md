---
title: git管理公共库
date: 2016-12-30 15:44:18
tags:
categories:
type:
---

## **场景**

我们用git管理项目时经常会遇到这样的场景。

*   场景一：一个项目A仓库要引用另外的公共项目仓库作为一个子目录，比如是公共的lib库。要求引用lib仓库后， 项目A也是个完整仓库，在项目A中对lib项目所做的修改除了能push到A仓库中，也能push到lib仓库中；同时lib仓库更新后，A仓库也能获取lib的修改更新。
*   场景二：项目B变的很庞大臃肿，这时候你会想把其中一些目录分离独立成仓库管理，要求保留所有的更改历史和记录。

怎么做呢，git subtree登场了。

## **subtree和submodule**

在subtree出现之前，有个能达到相似的功能的命令submodule。git submodule用来项目切分和规划功能，使用起来复杂同时又是最具有迷惑性和破坏性。git submodule的实现原理，就是在本地保存一个 .gitsubmodules的文件，里面记录着子模块的信息。在进行git各项操作时，要相当注意区分主仓库和子仓库，不然很容易出问题。使用起来有很多的坑，门槛相对要高。
其中一篇诉说不用submodule的文章：
<https://codingkilledthecat.wordpress.com/2012/04/28/why-your-company-shouldnt-use-git-submodules/>
我认为用git submodule的好处就是主仓库和子仓库分离的比较清楚。
介绍使用submodule我认为比较好教程
<http://www.open-open.com/lib/view/open1396404725356.html>
<http://www.kafeitu.me/git/2012/03/27/git-submodule.html>
<http://git-reference.readthedocs.org/zh_CN/latest/Git-Tools/Submodules/>
subtree 早前是作为独立的开源项目独立存在的，后来被git收编成为官方的一部分。**subtree子命令很晚才集成到git中，请确保你的git版本（使用git –version查看） > v1.8.0.0**。subtree的实现方式并非包装了submodule的一系列操作序列，而是截然不同的另一种管理模型。 subtree是没有.gitsubmodule，通过改变分支指针的形式进行分离和合拼项目，相对对submodule方便不少。
官方推荐使用subtree代替submodule。
废话少说，下面就用subtree来开搞。

## **1 引用外部仓库**

### **1\.1 准备工作**

在github上建立两个仓库。

1.  wrapper：git@github.com:iamsummer/wrapper.git
2.  commonlib：git@github.com:iamsummer/commonlib.git

commonlib仓库是公共仓库， wrapper仓库需要导入commonlib仓库。

### **1\.2 相关命令和操作**

#### **1\.2.1 创建指向commonlib的远程仓库地址, 并创建与之关联的子目录**

需要用到两条命令

> git remote add [–f] < remote_name > < repository_url >

如果添加-f表示在添加远程仓库后，执行fetch操作。

> git subtree add --prefix={子目录名} <子仓库名> <子仓库对应分支> [--squash]

--prefix表示目录名， 运行该命令之后会创建一个目录，并将子仓库拷贝到该目录。 这个命令如果没有--squash，默认是将子仓库所有的历史修改log都保留下来的，如果不想保留，可以在该命令后面添加--squash参数。 --squash表示将子仓库所有改动log合并成一次commit。

下面执行这两条命令：

    git clone git@github.com:iamsummer/wrapper.git
    cd wrapper
    git remote add -f commonlib git@github.com:iamsummer/commonlib.git

执行git branch -a 和 git remote -v 可以看到远程仓库已经添加commonlib远程仓库。

![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2014/12/remote-add.png "remote-add")

执行

    git subtree add -prefix=lib commonlib master


执行完后，会发现在当前目录下创建了lib目录，这就是子仓库。 分别执行gitk 和 git log --graph，得到下图

![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2014/12/subtree-add.png  "subtree-add")
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2014/12/subtree-info.png "subtree-info")

从上图中可以看到子仓库和主仓库合并，而且可以清晰的看到子仓库作为一个分支合并过来了，并保存在lib子目录中。
这时候我们想把改动push到wrapper远程仓库，看看是否会把子仓库更新上去。

    git push origin master

你会发现lib目录push到远程的wrapper仓库了，也就是说wrapper仓库与一般的git仓库无异。 是的，无论你编辑wrapper原来的文件还是子目录lib里面文件，正常提交即可，跟脱离了远程子仓库一样。 但如果你要把lib目录的更改push到远程的子仓库，或者把远程子仓库的更新pull下来，还需要几步简单的操作。

#### **1\.2.2 从远程的子仓库获取更新到子目录中**

命令：

> git subtree pull --prefix={子目录路径} <远程仓库名> <branch> [--squash]
--squash参数同上面说的一样，带--squash表示将子仓库所有改动log合并成一次commit，否则保留子仓库改动历史。

我们先在远程仓库commonlib中做些修改并提交，然后我们用上面的命令来执行，看看wrapper下面的子仓库的更新效果。

    git subtree pull --prefix=lib commonlib master

执行完后执行gitk， git log --graph看分支变化。

![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2014/12/subtree-pull.png "subtree-pull")
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2014/12/subtree-pull-2.png "subtree-pull-2")

从图上发现，子仓库一直作为一个分支存在的，子仓库更新会在子仓库分支上移动，最后跟wrapper的master分支进行合并，回归wrapper的分支。

#### **1\.2.3 从子目录push到远程的仓库**

需要命令

> git subtree push --prefix={path/to/subdir} <remote_name> <branch>

我们现在wrapper下面修改子目录lib里面的文件，并commit。然后运行上面命令

> git subtree push --prefix=lib commonlib master

使用gitk或者git log --graph查看分支变化， 你会发现子仓库的分支并没有变化。 而且commonlib的远程仓库push成功。

到这里，你应该已经明白git subtree是如何pull和push了，可以方便使用你的公有库了。

### **1\.3 恢复子仓库信息**

如果别人clone你包含子仓库的wrapper库后，怎么恢复子仓库信息，以便继续同步更新远程的子仓库？ 执行下面的操作：

    git clone  git@github.com:iamsummer/wrapper.git
    cd wrapper
    git pull -s subtree commonlib master 或者直接 git subtree pull --prefix=lib commonlib master

执行完后，你会发现即使子仓库没有更新，子树和wrapper仓库还是做了一次合并的commit。
-s subtree是指采用subtree的合并策略。关于subtree的合并策略解释有

> subtree This is a modified recursive strategy. When merging trees A and B, if B corresponds to a subtree of A, B is first adjusted to match the tree structure of A, instead of reading the trees at the same level. This adjustment is also done to the common ancestor tree.

然后你就可以愉快的和远程的子仓库交互啦。

## **2 分离子目录成独立仓库**

在原先是一个完整的仓库下，随着项目的不断积累和扩张，会变的臃肿和庞大，你可能会有单独管理子目录的需求。 下面介绍如何分离git仓库。
准备一个要分离的仓库，parent仓库，下面有son和daughter这个子目录。现在要把son目录分离出去。

git clone git@github.com:iamsummer/parent.git

clone后，现在我们来操作。

### **2\.1 在本地创建一个子分支**

需要命令

> git subtree split --prefix=<name-of-folder> --branch <name-of-new-branch> # 将需要分离的目录的提交日志分离到一个独立的分支上

执行

    cd parent
    git subtree split --prefix=son --branch son #将本地的son目录提交日志分离创建son子分支。

使用git branch查看，发现有son分支存在。 使用gitk或者git log --graph并没发现创建新的子分支。切换到son分支，git log查看提交，你会发现只有跟son目录相关的提交记录。 是的,son的提交历史已经剥离了，但是没影响到现有的分支。

### **2\.2 在本地创建一个repo用于存储上面分离的branch**

    cd 到parent目录外
    mkdir son
    cd son
    git init
    git pull ../parent son #git pull </path/to/big-repo> <name-of-new-branch>

执行完后，son目录的内容就是parent子目录son的内容，git log得到的log只是跟son目录相关的提交记录。

### **2\.3 推送本地repo到远程的仓库**

在github上创建一个远程仓库son： git@github.com:iamsummer/son.git

执行

    git remote add origin git@github.com:iamsummer/son.git
    git push origin -u master


子目录已经作为一个仓库push到远程。

### **2\.4 清理目录**

如果原仓库parent要保留son目录，这步可以不做。 这一步可以有两种方式：

*   ①直接删除son目录，然后提交，保留son的历史log。
*   ②彻底删除son目录，包括历史log。 使用命令：
    > git filter-branch -f --index-filter "git rm -r -f -q --cached > --ignore-unmatch **son**" --prune-empty HEAD 注意上面的son， son是要删除的目录。这个命令会rewrite历史，请谨慎使用。

### **2\.5 将分离的目录作为一个子仓库**

如果parent已经分离出去的son目录还想以子仓库的形式进行管理。 需要执行上面的清理目录，然后执行下面步骤：

    git remote add son git@github.com:iamsummer/son.git
    git subtree add --prefix=son son master

结果如下图：

![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2014/12/split-add.png "split-add")
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2014/12/split-pull1.png  "split-pull")

可以看到远程的son进行一次合并的commit。值的注意的是在分离son之前，如果son目录下面的修改提交跟其他修改是一并提交的，在分离son时，会自动的进行一次commit，也就像上图看到一样，commit的版本不一样。 以后要更新son子仓库，就可以使用上面介绍的方法啦。

## **总结**

对项目进行拆分与合并，在项目管理中是很常见的，上面介绍的都是本人亲测可行的方案。同时欢迎大家对于不对之处，多提出意见。

## **参考文档**

<http://git-reference.readthedocs.org/zh_CN/latest/Git-Tools/Subtree-Merging/>
<http://blog.nwcadence.com/git-subtrees/>
<http://manpages.ubuntu.com/manpages/utopic/man1/git-subtree.1.html>
