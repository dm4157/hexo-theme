id: git1
title: git小南1
categories: teamwork
tags: [git,team]
---

> 请以解决问题为核心，不要为了用技术而用技术。

助你开始使用 git 的简易指南，木有高深内容

## Git仓库
Git拥有一个远端（origin）仓库，每个开发者拥有自己的本地（local）仓库。
因此Git的提交流程是：
1. 修改本地代码
2. 提交更新到本地库
3. 推送到远端库

这样的设计便于多人协作

## Hello Git
使用Git有两种开始，一种是检出已存在的项目继续开发，另一种是新建项目。
但无论哪一种，都要从安装Git开始。
### 安装
[去官网下载](https://git-scm.com/downloads)

### 新的开始
以mac系统为例

#### 创建本地Git仓库
```
# 新建目录把它当做Git仓库
cd ~
mkdir hello-git
cd hello-git

# 初始化Git仓库
git init

# 通过命令查看创建出的.git目录，和.gitignore文件
ll -a
```
### 提交代码
在`hello-git`目录下，随便怎么写带你文字。用下面的命令提交代码：
```
# 首先将改动的部分添加到提交清单

# 添加单个文件
git add first.java
# 添加所有文件（常用）
git add *

# 提交
git commit -m '我是本次提交的说明'
```
如果只是本地玩，以上就足够了。

### 志存高远
如果想存在远端服务器上，继续吧。
```
# 先给远端起个名字吧，调用起来也方便
git remote add <alias> <server>
# 举个栗子
git remote add blog git@github.com:dm4157/dm4157.github.io.git

# 从远端拉取内容
git pull <server> <branch>
# 举个栗子,获得blog库master分支的内容
git pull blog master

# 将本地库的提交（commit）推送到远端
git push <server> <branch>
```

### 再续前缘
如果是检出已存在的项目继续开发，就简单许多。
```shell
# 克隆， 没有目录的话就是放在当前目录
git clone <server> (<目标目录>)
```

## 进阶的提示
> 远端服务别名由blog 改为默认的 origin

### http(s)与ssh
Git仓库有两种类型的访问地址, 一种是http形式另一种是ssh形式。
`http`形式是通过账号密码访问的，与Web页面访问是一个道理。如果没有工具记住密码，只是命令行形式，那么每次push都需要输入账号密码的。
`ssh`形式可以通过提交ssh-key来认证计算机，进而在操作时省去账号密码的步骤。如果是习惯用命令形式的孩子，记得提交ssh-key到git服务提供商（一般在项目设置中有deploy key的设置）。
```
# 首先安装ssh， 自行查找方法
# 举个栗子，MAC
brew install ssh

# 生成ssh
ssh-keygen -t rsa -C "账号"

# 使用ssh，打印出来复制到服务提供商
cat ~/.ssh/id_rsa.pub
```

### 工作流
你的本地仓库由`git`维护的三棵“树”组成。第一个是你的`工作目录`，它持有实际文件；第二个是`缓存区（Index）`，它像个缓存区域，临时保存你的改动；最后是`HEAD`，指向你最近一次提交后的结果。
![gitwork](http://7xrbi6.com1.z0.glb.clouddn.com/img-gitwork.png)

### 分支-branch
分支是用来将特性开发绝缘开来的。在你创建仓库的时候，master 是“默认的”。在其他分支上进行开发，完成后再将它们合并到主分支上。
![分支](http://7xrbi6.com1.z0.glb.clouddn.com/img-branch.png)
```
# 创建一个叫做“feature_x”的分支，并切换过去：
git checkout -b feature_x
# 切换回主分支：
git checkout master

# 再把新建的分支删掉：
git branch -d feature_x

# 除非你将分支推送到远端仓库，不然该分支就是 不为他人所见的：
git push origin <branch>
```

### 更新与合并
要更新你的本地仓库至最新改动，执行：
```
git pull
```

以在你的工作目录中 获取（fetch） 并 合并（merge） 远端的改动。
要合并其他分支到你的当前分支（例如 master），执行：
```
git merge <branch>
```

两种情况下，git 都会尝试去自动合并改动。不幸的是，自动合并并非次次都能成功，并可能导致 冲突（conflicts）。 这时候就需要你修改这些文件来人肉合并这些 冲突（conflicts） 了。改完之后，你需要执行如下命令以将它们标记为合并成功：
```
git add <filename>
```

在合并改动之前，也可以使用如下命令查看：
```
git diff <source_branch> <target_branch>
```

### 标签-tag
在软件发布时创建标签，是被推荐的。这是个旧有概念，在 SVN 中也有。可以执行如下命令以创建一个叫做 1.0.0 的标签：
```
git tag 1.0.0 1b2e1d63ff
```

`1b2e1d63ff` 是你想要标记的提交 ID 的前 10 位字符。使用如下命令获取提交 ID：
```
git log
```

你也可以用该提交 ID 的少一些的前几位，只要它是唯一的。


### 替换本地改动
假如你做错事（自然，这是不可能的），你可以使用如下命令替换掉本地改动：
```
git checkout -- <filename>
```

此命令会使用 HEAD 中的最新内容替换掉你的工作目录中的文件。已添加到缓存区的改动，以及新文件，都不受影响。

假如你想要丢弃你所有的本地改动与提交，可以到服务器上获取最新的版本并将你本地主分支指向到它：
```
git fetch origin
git reset --hard origin/master
```

## 有用的提示
### 小命令
内建的图形化 git：
```
gitk
```
彩色的 git 输出：
```
git config color.ui true
```
显示历史记录时，只显示一行注释信息：
```
git config format.pretty oneline
```
交互地添加文件至缓存区：
```
git add -i
```

### 指南与手册
[Git 社区参考书](http://git-scm.com/book/zh/v2)
[如git思考](http://think-like-a-git.net/)
[GitHub 帮助](https://help.github.com/)
