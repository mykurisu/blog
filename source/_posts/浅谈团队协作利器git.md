---
title: 萌新(我)的Git备忘录
categories:
-   博客正篇
tags:
-   Git
-   Gitlab
date: 2018/4/17
---

>   又是喜闻乐见的背景时间--最近也开始接触面试了，发现很多童鞋对git十分陌生，甚至听到git有点恐慌，不过这样无可厚非，毕竟如果是自己单干的话确实可能接触不到协作层面的情境。<del>对的，比如说我</del>

**本文只适合我等萌新，大佬可能就不需要继续看下去啦，如果是帮忙纠正错误的话就十分欢迎~**

本文会涉及到的内容--

-   刚进入新的工作群该用git干啥
-   开始<del>接客</del>接活之后改用git干啥
-   一起<del>接客</del>接活时该怎么保证安全(?)
-   小结

## 写在前头

还是和我之前的风格一样，这里只可能出现业务型的干货，并不会在此过多的原理陈述(一是怕误人子弟，二是感觉没必要)，git作为一种工具，我们能在工作中用得顺手即可。

**注意：本文的开发机器为Mac，代码仓库为Gitlab**

## 刚进入新的工作群该用git干啥

进入新公司或是加入新的工作群，最重要的事情当然是要拿到当前业务的代码，这是进行后续开发工作的基础。就我们当前的开发情况来说，我们需要做以下几件事--

### ①注册账号，与管理仓库的同事沟通，开通相关项目的权限

一般在这个时候，都会要求大家提供一串秘钥--本机的SSH Key，如果之前有在本机生成过SSH Key的话我们可以在终端输入

```bash
touch cat ~/.ssh/id_rsa.pub
```

这样终端便会显示一串秘钥，之后只需把秘钥告知同事即可，为什么要做这件事呢？因为就算我们有观看项目的权限，我们的机器也没有将其克隆下来的权限，只有将我们的SSH Key记录到gitlab中，我们才可以克隆项目。

### ②克隆代码

```
进入相应的目录
cd my_project
在本目录下将远端代码克隆到本地
git clone ssh://xxx/xx/xx.git
```


## 开始<del>接客</del>接活之后改用git干啥

### ①创建自己的开发分支

正常情况下，我们是不会在master分支下进行开发的，这既不安全也不优雅(一般也不会允许非管理员在master分支提交代码)。所以我们就需要创建一个属于自己的开发分支，它与master的代码时刻保持同步，提交代码时merge到master分支即可。

```
git branch 
//确认在master分支
git checkout -b xxx(自定义分支名)
//从所在分支，checkout出一个新的分支，这个新分支拥有所在分支的所有代码
git push -u origin xxx 
// 将本地分支推往远程仓库
```

输入这三行之后，我们就拥有了属于自己的分支，因为这个分支是从master切出来的，所以它的代码与master完全一致，之后我们在此分支开发即可。

### ②管理自己的源码

在IDE里面愉悦的搬砖一段时间之后，要记得劳逸结合！我们最好要养成定时提交代码的习惯，最好不要码了一整天之后才想起来要提交代码，就我个人而言，在完成需求中的某个小需求时我便会提交或推送一次代码。

```
git add .  or  git add /xxx/...
//将本次改动全部加入缓存区
git commit -m ""  or  git commit
//将代码提交到本地代码库，并附带相关注释
```

### ③时刻同步master分支

维持一个项目的稳定性，其中一个要素就是保证代码的纯净，所以我们需要保证我们的开发分支需要时刻与master保持同步，我们团队采用的rebase的方案。

```
git fetch origin
// 发现远程仓库最新的代码
git rebase origin/master
//同步远程仓库master分支
git push
//将你的代码推到远程仓库
or
git push -f origin xxx
//如果无法push则可以选择下面这个方式**(前提是要确保本地的代码是绝对的完整)**
```

在这里我们需要注意的是要记得fetch远端的最新代码，然后才进行rebase。紧接着就是rebase时对冲突的应对--

```
git fetch origin
// 发现远程仓库最新的代码
git rebase origin/master
//同步远程仓库master分支
----conflict----
//rebase时发生冲突
git status
//定位冲突文件
//...处理冲突...
git add .
//提交冲突处理到缓存区，一定不能commit
git rebase --continue
//继续rebase
or
//如果无法处理冲突则退出rebase复原代码
git rebase --abort
```

### ④提交自己的第一个PR

为了保证代码的纯净，最理想的方式是只有一个有权限的管理员进行分支与master之间的合并，这里用到的就是我们常说的PR(Pull Request)，简单来说，就是向项目管理员提出一个merge的请求，管理员同意之后则会将我们的代码合并到master。

```
git fetch origin
git rebase origin/master
git push //...ok
```

将本地改动推送到远端代码库后，我们需要进入gitlab相应的项目，创建PR(图中Merge Request选项，也就是Github中的Pull Request)

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/git-1.jpeg" width="50%" />

确认相关改动无误后我们可以创建PR等待管理员review，创建好之后又有改动怎么办？没关系的，我们只需要按照老方法将代码推送到服务器，PR内容会自动更新至最新。


## 一起<del>接客</del>接活时该怎么保证安全

背景--有一个比较大的需求，需要两个人共同开发，并且需要这个需求最终共同上线。

对于我来说，这种情况不太常见，但是也想过相关的方案，在这里之前的rebase方案就不太够用了，需要区分主从分支。假设进行开发的童鞋为A和B--

```
// A童鞋
// on master
git checkout -b A
git push -u origin A

// B童鞋
git fetch origin
git checkout A
git checkout -b B
git push -u origin B

// 开发中...
// A童鞋
git add .
git commit
git fetch origin
git rebase origin/master
git push

// B童鞋
git add .
git commit
git fetch origin
git rebase origin/A
git push
```

这样开发可以使得AB两位的代码都能时刻与master分支保持同步，并且B还可以获得A最新的开发代码，在共同开发某样功能的时候应该会十分有用。最后B提交一个PR即可把两人的代码合并，再由A发起合并到master的PR即可。


## 小结

其实这就是我个人开发时候的备忘录，一些基本开发流程还是有的，如果有特殊的需求的话，可以上网找找Git相关的教程~在此不会过多的涉猎。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/end.png" width=50% />
