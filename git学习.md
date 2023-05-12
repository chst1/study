---
title: git学习
date: 2019-07-20 17:57:13
description: git 是互联网公司必备工具, 更是it工作者必备技能, 这里主要讲解涉及如下内容.创建版本库, 版本回退, 工作区与暂存区, github使用, 分支管理, 远程仓库, 冲突合并.
tags:
    git
categories:
    work
mathjax:
    true
---

# <center/>git</center>

## 序

找实习两个月, 终于有公司肯要我这个菜鸡了. 似乎对于互联网公司来说第一个要学的就是git的版本控制了, 当然进去肯定会被要求赶快看一下,知道基本使用. leader发了份[廖雪峰的git](https://www.liaoxuefeng.com/wiki/896043488029600). 于是在了解基本指令后又自己按照网站学习了一下,包含一部分自己的理解.

## 创建版本库(**repository**)

1. 创建一个空目录 mkdir dir_name
2. 通过get init命令将目录变成git可管理的仓库.

此时会将目录变成空的版本库, 其中存在一个.git的隐藏目录, 用于跟踪管理版本库.

## 把文件添加到版本库

1. 首先将要添加的文件放到版本库目录下.
2. 使用`git add filename1 filename2 ...`添加文件到仓库.
3. 使用`git commit -m "explain words"`将文件提交到仓库.

`git commit`后面-m表示解释的参数, 指示出这次提交的内容.

例:

```git
git add test1.txt test2.txt test3.txt
git commit -m "test loads file"
```



## 更新文件和查询当前版本库状态

`git status`用来查看版本库当前状态,  如果没有任何更改则显示:

```git
On branch master
nothing to commit, working tree clean
```

当某个文件被更改而未add和commit时, 会显示文件发生变化通过add和commit进行更新:

```
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   readme.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

此时我们可以使用`git diff filename`来显示文件内容哪里发生了更改(只能在文件提交前查看):

```
diff --git a/readme.txt b/readme.txt
index f7249b8..7a8a2b4 100644
--- a/readme.txt
+++ b/readme.txt
@@ -1,2 +1,3 @@
 Git is a version control system
 Git is free software
+test change
```

当文件更改且add添加而未commit时, 会提示我们使用commit进行提交:

```
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	modified:   readme.txt
```

最后使用commit进行提交即完成更改, 提交后, status即恢复到初始状态.

```
On branch master
nothing to commit, working tree clean
```

## 版本回退

`git log`用来显示历史记录. 其会从最新一次更改显示到最初的更改.

例:

```
commit db0c719054994a2e6bab6367837077a628b1c4ee (HEAD -> master)
Author: chst1 <2322253097@qq.com>
Date:   Tue Jul 2 23:50:38 2019 +0800

    change 2

commit 76411db8fb0c504d70d0ca8dd5a674f81dc5773c
Author: chst1 <2322253097@qq.com>
Date:   Mon Jul 1 23:40:41 2019 +0800

    change readme

commit bd8b25edc8c33a8e134f992d570b7b50bca6361e
Author: chst1 <2322253097@qq.com>
Date:   Mon Jul 1 23:19:50 2019 +0800

    wrote a readme file
```

使用`git reset --hard HEAD^` 即可恢复到当前版本的上一个版本. 使用`--hard HEAD^^`恢复到上一个版本的上一个版本. 如果希望指定某个版本, 只需要在hard中指定版本号即可, 即`git log`输出的commit, 只需要输出其中部分位数即可. 当进行版本恢复时, 再次使用git log命令将会使得恢复版本后面的版本不存在记录了. 此时如果我们想要恢复到当前版本的下一个版本时, 我们就不知道--hard后边参数为何了, 此时我们可以使用`git reflog`命令来显示之前操作的每一个命令, 从中可以获得下一个版本的版本号, 之后使用`git reset --hard commit`恢复即可.

例:(git reflog)

```
76411db (HEAD -> master) HEAD@{0}: reset: moving to HEAD^ // 当前版本
db0c719 HEAD@{1}: commit: change 2 // 回退前最后版本
76411db (HEAD -> master) HEAD@{2}: commit: change readme
bd8b25e HEAD@{3}: commit (initial): wrote a readme file
```

git版本回退快是因为git内部有一个指向当前版本的HEAD指针, 当版本变换时只需要将指针指向对于版本即可.

## 工作区与暂存区

### 工作区

即为电脑中被git init初始化后的目录.

### 版本库

工作区的一个隐藏目录.git, 为git版本库.  .git存放了很多东西, 最重要的是stage(或叫index)的暂存区, 还有git自动创建的分支master以及指向master的一个指针HEAD.

![图示](https://s2.ax1x.com/2019/07/03/ZYKxYV.png)

git add是将文件修改添加到暂存区, `git commit`是将文件提交到当前分支. 当提交完成后, 暂存区不会被清理,(可能只是存在一个同步标志). 这就是为何git commit后使用git status显示

```
On branch master //指示当前分支
// 指示当前暂存区为空, 没有东西可以提交
nothing to commit, working tree clean
```

`git diff`默认比较的是工作区与暂存区(最后一次add).

`git diff --cached`比较暂存区与最新版本库(即最后一次commit的, 也就是分支中的)

`git diff commit_id`比较工作区与指定版本号区别

`git diff --cached commit_id`比较暂存区与指定版本号直接区别.

## 撤销修改

### 撤销工作区修改

```
git checkout -- file
```

该命令用于丢弃工作区的修改. 如果工作区更改还未放到暂存区, 则将文件恢复为和版本库一致. 如果已经add到暂存区了, 则将文件恢复到与当前暂存区的状态.

### 撤销暂存区修改

```
git reset HEAD file
```

该命令用于更改已经add到暂存区, 使用此命令可以将暂存区内file恢复到与版本库一致. 如果想要接着恢复工作区,则使用上面的撤销工作区的命令.

## 删除文件

首先在工作区中将要删除的文件删除, 而后使用git rm file_name删除文件, 而后git commit提交即可.

即:

```
rm file_name

git rm file_name

git commit -m "info"
```

还有一种情况是工作区中的文件被误删(`rm file_name`), 此时我们可以使用版本库中的文件对其进行恢复操作. 即使用`git checkout -- file_name`  即使用版本库中的文件(暂存区或版本库)对工作区文件恢复.

如果是使用`git rm file_name` 对文件删除, 则是先删除工作区文件, 在进行了一次add, 即将暂存区的文件也删除了, 此时如果想要恢复则应该先恢复暂存区的文件`git reset HEAD file_name`, 在恢复工作区文件 `git checkout -- file_name`

这里删除文件而后上传到远程仓库, 远程仓库中文件也会被删除.

## 远程仓库

这里主要介绍github的使用, 毕竟对个人来说, 主要使用场景就是github了.

### 创建SSH Key

在用户目录下(命令行中cd后ls -a)查看有没有.ssh目录, 如果有则查看是否存在id_rsa和id_rsa.pub文件. 已经存在则跳过下面步骤, 如果不存在, 则创建ssh key,在命令行中输入

```
$ ssh-keygen -t rsa -C "youremail@example.com"
```

youremail为自己的邮件地址.

## 在github上添加ssh

登录github, 在打开Account setting, SSH Keys页面, 点击Add SSH Key, 填写任意Title, 在Key中粘贴id_rsa.pub文件内容. 点击Add key即将当前电脑与个人github网站建立连接了.

### 添加远程库

登录github后点击`Start a project`,自己填写命名和描述. 点击创建, 默认会指导你如何操作. 如果本地已经存在仓库了, 只需要将本地仓库与github的仓库关联即可, 也就是运行如下代码

```
$ git remote add origin git@github.com:michaelliao/learngit.git
```

### 将代码上传到github

```
$ git push -u origin master
```

第一次需要-u, Git不但会把本地的`master`分支内容推送的远程新的`master`分支，还会把本地的`master`分支和远程的`master`分支关联起来，在以后的推送或者拉取时就可以简化命令。

从现在起，只要本地作了提交，就可以通过命令：

```
$ git push origin master
```

之后希望提交更改时, 先在本地完成add和commit, 即将本地的版本库更新, 再push到github即可.

### 从远程仓库克隆

cd到自己想要放置的目录, 使用

```
$ git clone git@github.com:michaelliao/gitskills.git
```

即可.



## 分支管理

### 创建与合并分支

创建版本库时会自动创建一个master分支, HEAD指向master, master才指向提交, 一开始的分支如下所示:

![1](https://s2.ax1x.com/2019/07/16/ZqSlvV.png)

当创建分支时, git新建一个指针, 指向与当前master所指向的相同提交, 当我们切换分支时, HEAD就会更改指向位置, 下图为切换到dev分支:

```
$ git checkout -b dev
```

等价于:

```
$ git branch dev // 创建分支
$ git checkout dev // 切换分支
```

![2](https://s2.ax1x.com/2019/07/16/ZqSM3q.png)

可以使用`git branch`显示工作目录下存在的分支以及当前所处分支

当我们切换到哪个分支时, 提交就会在那个分支进行, 即对于分支指向的提交前进, 而与别的分支无关, 下面为在创建的分支下提交的结果:

![3](https://s2.ax1x.com/2019/07/16/ZqSKCn.png)

即别的分支不变, 当前工作分支前移.

我们在dev中工作完成就可以将dev合并到master

```
git merge dev
```

![4](https://s2.ax1x.com/2019/07/16/ZqSn4s.png)

这里的合并是快速合并, 直接将master指向dev指向的地方. 

而后可以删除dev

```
$ git branch -d dev

$ git branch -D dev 强行删除
```

![5](https://s2.ax1x.com/2019/07/16/ZqSQg0.png)

### 存在问题

在创建分支后, 在分支中对工作区的文件更改而不提交时在切换到主分支时更改也能看到, 而如果在分支中更改了并且以及提交到版本库中, 此时回到主分支时是看不到工作区发生更改的.

个人理解, 由于工作区与暂存区对于各个分支来说是共有的, 同时在工作区应该存在一个各个分支共享指针表示当前工作区的更改是否提交, 如果提交了, 则每次打开工作区文件显示的都是当前所在分支的版本库中文件, 而如果没有提交, 则显示工作区元素文件. 这时当将主分支与另外的分支合并时, 由于工作区已经提交, 此时打开主分支显示的也是版本库中文件.(这里显示可能存在问题, 显示表示未变只是展示出来发生问题,这是不合理的, 更可能的应该是在每次操作之后按照上述规则对工作区文件进行更改). 这样可以解释的通,但却不知道对不对, 希望知道原因的大佬指正. 



### 合并时存储分支信息

当我们在合并后, 常常会将分支删除, 此时我们就不知道了分支合并前的信息, 为了保留合并前信息, 我们在合并时要关闭`Fast forward` 使用如下命令合并即可:

```
$ git merge --no-ff -m "merge with no-ff" dev
```

后面的-m是该命令调用了commit命令, 生成了一个存储合并前的文件.

如下展示区别:

```
*   bc742c3 (HEAD -> master) merge with no-ff
|\  
| * 1eb71b9 (dev) dev change2 // 此处是使用--no--ff模式下存储的分支信息
|/  
* 1f9002e dev change // 此处为直接合并, 无法看到合并前分支内容
*   0e9a735 merge info
|\  
| * 8fb45d3 dev change
* | 42bbb6d master change
|/  
*   d951ea7 conflict fixed
|\  
| * fd94b5f dev change
* | ca2dbf7 master change
|/  
* 3613205 kk
* 2360153 change del
* de50b7a dev change
* 00eb99a dev change

```



### 解决冲突

之前的合并均是在一个分支上没有变换, 另一个分支上发生了变换, 此时的合并规则较为简单, 当两个文件均发生了改变时, 会引起合并冲突.

此时在主分支合并时, git会告诉我们存在冲突, 而对应的工作区文件也会被git更改, 显示出冲突的位置, Git用`<<<<<<<`(当前工作区冲突内容)，`=======`(分割)，`>>>>>>>`(要合并的部分冲突内容)标记出不同分支的内容. 如下:

```
Git is a distributed version control system.
Git is free software distributed under the GPL.
Git has a mutable index called stage.
Git tracks changes of files.
<<<<<<< HEAD
Creating a new branch is quick & simple.
=======
Creating a new branch is quick AND simple.
>>>>>>> feature1
```

此时我们需要对该文件进行手动修复, 然后再提交一遍即可. 提交就相当于告诉git修复完成, 对当前分支进行更新, 但修复主分支不会影响另一个分支, 切换会另一个分支, 其内容依旧是其更改之后的样子.

```
$ git log --graph --pretty=oneline --abbrev-commit
```

上述代表可以显示合并过程, 像下面一样:

```
*   d951ea7 (HEAD -> master) conflict fixed
|\  
| * fd94b5f (dev) dev change
* | ca2dbf7 master change
|/  
* 3613205 kk
* 2360153 change del
* de50b7a dev change
* 00eb99a dev change
* f47bb03 (origin/master) change readme
* 7fe03d1 add del
* e2c8fff del del
* a8a2b06 test rm
* db0c719 change 2
* 76411db change readme
* bd8b25e wrote a readme file
```

### 存储当前工作区更改

当我们在进行一个任务进行到一半时, 接到另一个更加紧急的任务,此时我们需要先去处理第二个任务, 而第一个任务未完成,此时我们就要存储当前工作区状态, 而后去处理第二个任务. 因为工作区是共享的, 如果不存储并隐藏工作区更改, 在处理完第二个任务提交时会将第一个未完成的任务一起提交, 这就将导致严重的错误. 在完成第二个任务后可以进行恢复操作, 但恢复操作可能会涉及合并, 这是由于在隐藏时, 文件已经发生了变化, 就会出现合并冲突. 由于git存储的都是每次更改的变化, 因此十分适合该场景, 只需要将当前记录变化的文件隐藏,再打开一个新的即可.

#### 存储工作区

```
$ git stash
```

存储后使用`git status`查看工作区，就是干净的. 可能有多个优先级不同的任务进入, 此时我们可能会存储多次.

#### 恢复工作区

首先使用下属命令查看当前隐藏的状态列表.

```
$ git stash list
```

使用下述命令恢复指定隐藏状态

```
$ git stash apply stash@{n}
```

其中stash@{n}为`$ git stash list`输出的.

完成后可以将该id对应的隐藏状态删除

```
git stash drop stash@{n}
```

或者使用`git stash pop`，恢复的同时把stash内容也删了.(一个按照栈弹出).

恢复工作区不一定会发生冲突,这取决于隐藏的工作区与恢复前工作区之间是否存在冲突. 如果隐藏后直接在当前分支继续更改操作提交而后恢复极有可能出现冲突, 这是由于在隐藏时文件发生了更改, 与隐藏的更改不一致. 而如果是隐藏后在另一个工作区更改提交再切换回来恢复,则不会发生冲突, 这是由于在另一个工作区更改提交不会影响当前工作区内的内容,相当于当前工作区未发生变话,可以直接恢复.

### 多人协作

#### 查看远程仓库

远程仓库默认名字是`origin`, 使用如下命令查看:

```
$ git remote
```

查看详细详细使用:

```
$ git remote -v
origin  git@github.com:michaelliao/learngit.git (fetch)
origin  git@github.com:michaelliao/learngit.git (push)
```

如果没有推送权限就看不到`push` ．

#### 推送分支

```
$ git push origin master
```

`origin`指远程仓库名字, `master`表示要推送的分支名字.

#### 在本地根据远程仓库创建分支

```
$ git clone ...
```

只能是克隆`master`分支到本地, 要想克隆别的分支到本地需要使用如下命令:

```
$ git checkout -b dev origin/dev
```

`dev`为克隆下来分支名称, `origin/dev`表示从远程克隆哪个分支.

#### 抓取最新更改

```
$ git pull
```

默认抓取`origin/master`的更改, 当在本地不同非`master`工作区时, 应该首先建立指定分支之间连接:

```
$ git branch --set-upstream-to=origin/dev dev
```

而后在使用 `git pull`. 此时可能会发生合并冲突,在本地进行更改,而后提交即可.

#### 工作流程

1. 在本地更改代码.
2. 使用`git pull`合并其他人提交导致的冲突.
3. 使用`git push origin dev`提交. 如果出现错误则转到第二步.

#### rebase

在分支合并时, 使用`git merge`时, 会在最终的log中出现一个类似于环的图形(不是环, 有起点和终点), 但使用rebase进行合并时则不会出现, 具体参看[rebase][http://gitbook.liuhui998.com/4_2.html].

## 标签管理

### 创建标签

直接创建标签:

```
$ git tag tag_name
```

表示将当前`commit_id`的提交打上`v1.0`的标签.



指定`commit`创建标签:

```
$ git tag tag_name commt-id
```



在创建时添加说明文字:

```
$ git tag -a tag_name -m "info" commit_id
```

其中用`-a`指定标签名，`-m`指定说明文字.



查看所以标签:

```
$ git tag
```

输出的标签排序不是按照时间而是按名称.



查看标签信息:

```
$ git show tag_name
```

会展示对应提交的相关信息.



### 操作标签

删除:

```
$ git tag -d tag_name
```



推送标签到远程库:

```
$ git push origin tag_name
```



一次性推送全部尚未推送到远程的本地标签：

```
$ git push origin --tags
```



删除远程标签:

```
先在本地删除.
$ git tag -d tag_name
再删除远程仓库
$ git push origin :refs/tags/tag_name
```



## 自定义git

### 忽略特殊文件

当我们在提交时, 有些文件是无关紧要但变换很多的(如编译产生的中间文件或者服务器生成的log文件)文件, 显然版本控制不应该在意这些文件的话,此时我们就可以选择忽略特殊文件文件. 只用创建一个特殊的.gitignore文件, 将要忽略的文件写进去(按行), 再提交,git就会自动忽略这些文件. 对于不同编程需求官方提供了[忽略文件][https://github.com/github/gitignore], 我们只用在这个上面进行适合自己的修改即可.

### 配置别名

在输入命令时, 总会由于名字太长而不方便, 这时我们可以为命令配置一个简单的别名,此时我们只用输入简单的指令就能够实现一长串指令的操作岂不美滋滋. 语法如下:

```
$ git config --global alias.your_command origin_command
```

`--gloable`表示对当前试用.(否则只在当前版本库中能够使用.)

例:

```
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

试用`git lg`命令实现`git log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit`

每个仓库的git配置文件就在`.git/config`文件中.  而对当前用户的git配置则在用户目录下.gitconfig文件中.

```
[user]
	name = yourname
	email = youremail
[color]
	ui = true
[alias]
	lg = log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
```