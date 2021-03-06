---
title: 前端页面全局锁(Lab小技巧-004)
categories:
-   Lab小技巧
tags:
-   Javascript
date: 2018/12/23
---

>   看到页面上有个按钮不知大家是否有疯狂点击的冲动？请善待我们前端开发，不要轻易多次点击页面上的按钮（开玩笑~

在网页开发的过程中，秉着<del>保护自己</del>不信任用户的原则，我们有必要在某些会被频繁触发的按钮或者热区加上“锁”，这里的锁指的是短时间内不允许多次点击按钮。

##  首先，有必要说一下重复点击这个事

它到底会导致怎样的后果？举一个常见的栗子：

商品详情页中的购买按钮，倘若没有对用户的点击行为作出相应的限制，可能会产生以下结果--
-   用户可能会重复下单并生成多张订单
-   点击频率过大把下单接口刷爆了
-   还可能会出现未知的体验性问题
-   ......

这只是其中一个会涉及用户点击的场景，试想在一个较为复杂的表单页面可能会有很多的可点击项，如果不在全局的层面对点击加以限制，可能会对整个页面造成不可估量的影响。

##  那么，应该怎么锁？

核心很简单--在调用方法前或执行方法前将锁注册好，下次调用方法时去查看锁是否已释放，如果释放则照常运行，否则跳出方法不再往下执行。

下面让我们跟着代码来看，这个锁应该怎么实现--

<img src='https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--004-02.png' />

首先，我们先对全局锁进行一些基础变量的定义，为了方便锁状态的还原，在最开始定义了defalutLockOption，也就是全局锁方法的默认数据。紧接着是lockOption，后面对于锁操作所需的数据都在这里取或者修改.reloadOption则是对锁状态复原的方法，具体变量含义在图里都有展示。

<img src='https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--004-03.png' />

上面是全局锁最核心的功能，当然就是“上锁”这个操作啦。它接受两个参数--是否自动释放锁、自动释放锁的时间。但是大家会发现，在设定释放时间的时候我还是写了10000ms，这样做是为了避免某些没能手动取消锁导致的页面无法点击情况。

介绍一下第一个判断的条件，如果**lockOption的endTime有值**并且已经**过了释放锁的时间**最后是**当前锁的状态是锁上的**。满足这样一系列的条件，我们认为这个锁是“可释放”或“已释放”的。所以在调用lock()时会重置锁的配置，并且让_lockStatus = false（表明此次调用不在上锁状态，可以继续往下执行）。

紧接着下一个判断条件，_lockStatus实际指的是调用lock()时全局锁的实际状态，倘若在调用lock()时，锁在释放状态则会将锁的状态更改，并设定好释放锁的时间。随后return _lockStatus **注意，这里并非return lockOption.lockStatus**。

光看文字可能有点绕，我总结了一张示意图，完整的展示事件未锁--锁--释放锁的过程:

<img src='https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--004-01.png' />

##  该怎么使用它？

<img src='https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--004-04.png' />

上面只是其中的一种情景，实际上所有的可点击区域都可以用这种模式限制触发频率。点击[这里](https://code.h5jun.com/rajek/edit?console,output)，源码以及实现效果都可以直观的看到~如果绕住了可以配合前面的解析看看。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/end.png" width='50%'/>
