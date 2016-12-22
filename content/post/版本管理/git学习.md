+++
keywords = [
  "git"
]
description = "git"
author = "author"
draft = false
type = "post"
date = "2016-12-08T17:05:34+08:00"
title = "git学习"
tags = [
  "git"
]
topics = [
  "topic 1",
]

+++

参考资料：  
https://git-scm.com/book/zh/v2  
http://igit.linuxtoy.org/contents.html  

git最基础操作
----

配置用户

```
git config --global user.email "user@gmail.com"
git config --global user.name "user"
```

本地创建git管理的项目

```
mkdir git_study
git init
git status
git log
```

创建文件并提交

```
echo "hello Git" > readme.txt
git add readme.txt
git status
git commit -m "project init"
git log
```

更改文件并提交

```
echo "Git is Cool" >> readme.txt
git status
git diff
git add readme.txt
git status
git commit -m "Git is Cool"
```

在上边的操作我们可以发现，在一个Git项目中文件的状态大概分成下面的两大类，而第二大类又分为三小类：

1. 未被跟踪的文件（untracked file）
2. 已被跟踪的文件（tracked file）
    1. 被修改但未被暂存的文件（changed but not updated或modified）
    2. 已暂存可以被提交的文件（changes to be committed 或staged）
    3. 自上次提交以来，未修改的文件(clean 或 unmodified)

git是如何使用sha来管理内容
----

首先git的所有数据都保存在`.git`目录下
在这个目录下
```
find .git/objects

......
.git/objects/e9/13915d3d5bd4830fdd7a13d73a896a09bd3478
.git/objects/ee
.git/objects/ee/4e9782bf8b320c60a8d5d7aa070d47d752dba1
.git/objects/ee/e94f02c4c32247c7aa5b87829f37693be90be4
.git/objects/ef
......
```
没有看到任何一个文件叫做readme.txt，那么文件存在哪里了呢？

执行`git log`查看git历史  

```
git log
  
=============================================  
commit e581b637ae12cc7c9975af04dc2bcc58a93b1274
Author: user <user@gmail.com>
Date:   Thu Jan 24 15:46:53 2013 +0800

    Git is Cool
=============================================
```  
可以看到commit上有一串编号，这个就是这次提交的sha编号  
我们可以用`git cat-file`看看这个编号对应的内容   

```
git cat-file -p e581b637ae12cc7c9975af04dc2bcc58a93b1274  

=============================================
tree f07fcbff9c87b1a3c3d4393184e2011744c7966e
parent 1e7bd8de9ca7a624364a986243c9d26b4efe5ffa
author user <user@gmail.com> 1359013613 +0800
committer user <user@gmail.com> 1359013613 +0800

Git is Cool
=============================================
```

可以看到tree、parent有与之前类似的编码  
我们继续用`git cat-file`看看它们的内容  

```
git cat-file -p f07fcbff9c87b1a3c3d4393184e2011744c7966e  

=============================================
100644 blob 49ec0d692868a0d17f3e663314e7601a55ae766a    readme.txt
=============================================
```

从这个结果里我们又看到一个sha码  

```
git cat-file -p 49ec0d692868a0d17f3e663314e7601a55ae766a

=============================================
hello Git
Git is Cool
=============================================
```
其实就是readme.txt的内容
所以之前一个命令的结果，就可以解释为blob类型的数据，数据对应一个sha码，数据的文件名是readme.txt

```
git cat-file -p 1e7bd8de9ca7a624364a986243c9d26b4efe5ffa  

=============================================
tree be2e44a49d0da5a3e1913b91d234eea7b0b610fe
author user <user@gmail.com> 1359013080 +0800
committer user <user@gmail.com> 1359013080 +0800

project init
=============================================
```
然后我们就找到了项目初建的地方


分支的简单实用
----
创建分支

```
git branch develop
```

查看所有分支

```
git branch
```

切换分支

```
git checkout develop
```

合并分支develop到master

```
git checkout master
git merge develop
```

保留合并之前的分支信息  
```
git merge develop --no-ff
```

删除已合并的分支

```
git branch －d develop
```

强制删除分支

```
git branch －D develop
```

查看历史
----
`git log`命令可以用来查看历史，配合参数可以看到不同的内容

```
git log --oneline --graph --decorate  
git log --pretty=format:'%h : %s' --topo-order --graph  
git log --pretty='%h : %s' --topo-order --graph  
git log --graph  
git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
```

为了方便使用，我们可以创建alias, 编辑`.git/config`，并添加下边的内容  

```
[alias]
     lg = log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
```

或者执行命令  

```
git config --global alias.lg "log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

然后查看日志

```
git lg  
git lg -p  
git lg --all
```

标签的使用
----
列出标签

```
git tag -l
```

创建标签

```

```

删除标签

```
```

远程服务器的管理
----

显示当前项目有几个远程库

```
git remote -v
```

增加新的远程库

```
git remote add gitcafe 'git@gitcafe.com:user/project.git'
git push -u gitcafe master

git remote add gitshell 'git@gitshell.com:user/project.git'
git push -u gitshell master
```

远程库改名

```
git remote rename gitcafe gitcafe1
git remote rename gitshell gitshell1
```

删除远程库

```
git remote rm gitcafe
git remote rm gitshell
```

一些所谓的奇技淫巧
----

修改git传输字节限制

```
git config http.postBuffer  524288000
```


找回reset后丢失某些节点

```
$ git reflog
b5ee6a7 HEAD@{0}: reset: moving to b5ee6a76f40e3dc4170b544ecf4b895b54e01863
390744b HEAD@{1}: commit (amend): need ref host instead of a utm param list
3da3e39 HEAD@{2}: checkout: moving from newfeature2 to newfeature2
3da3e39 HEAD@{3}: checkout: moving from ptmind_etl_japan_250million to newfeature2
1861139 HEAD@{4}: checkout: moving from ptmind_etl_china_test to ptmind_etl_japan_250million
$ git cherry-pick 390744b
```

git https使用用户名密码

```
git clone https://username:passowrd@gitshell.com/user/project.git
```

最后的最后
----
使用git，一定要常看它的版本树，了解项目的更新合并情况。  
为此选择一款好的gui客户端是很有必要的。
