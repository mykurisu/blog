---
title: 小程序中WXS(Lab小技巧-003)
categories:
-   Lab小技巧
tags:
-   Javascript
-   小程序
---

>   小程序出现也有一段时日了，随着生态的日益健壮，小程序也慢慢成为相对成熟的平台。今天想和大家分享一下小程序中相对冷门但有用的东西--WXS

最近都在写小程序，遇到这样一个tab组件的需求，大概意向如下图

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--003-2.png?q-sign-algorithm=sha1&q-ak=AKIDdFh1DFpsRyLXYinXO6d0DGuNfnlYfwa4&q-sign-time=1543160044;1543160944&q-key-time=1543160044;1543160944&q-header-list=&q-url-param-list=&q-signature=b9686ab14976dbb231648f1f48f6c3ff832b2c0a" width="50%" />

背景介绍，**该tab组件将会由两个现成的组件拼凑而成，分别是tab-head与tab-body**，并且tab-haed组件只接受字符串组成的数组。废话少说，先上代码↓↓

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--002-4.png?q-sign-algorithm=sha1&q-ak=AKIDdFh1DFpsRyLXYinXO6d0DGuNfnlYfwa4&q-sign-time=1543159939;1543160839&q-key-time=1543159939;1543160839&q-header-list=&q-url-param-list=&q-signature=9c34f202e1b89a92f85f66c0e71edc7ea86e8bcb" width="50%" />

如图所示，我们将会得到一个list数组用于描述整个tab组件，按照我们一贯的处理方式，大概会在获取到list数据之后进行title的分离(分离的原因请参考背景介绍)，以供视图层的tab-head使用。但是，不知道大家有没想过，js里面其实应该是处理数据逻辑的，这种有关视图层面的处理在可行的情况下应该交由视图层自行处理。

这里所说的“可行方案”就是我们小程序中的WXS了，既然主角出场了，我们就顺便介绍一下~

引用微信小程序官方文档的介绍--

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--003-2.png?q-sign-algorithm=sha1&q-ak=AKIDdFh1DFpsRyLXYinXO6d0DGuNfnlYfwa4&q-sign-time=1543160044;1543160944&q-key-time=1543160044;1543160944&q-header-list=&q-url-param-list=&q-signature=b9686ab14976dbb231648f1f48f6c3ff832b2c0a" width="50%" />

简单理解一下，这是一个类似JavaScript但又不是JavaScript的语言，它的运行速度将会比JavaScript更快，他可以与WXML配合使用。

那么，在我的这个需求里，WXS应该有什么样的作用呢？

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--003-4.png?q-sign-algorithm=sha1&q-ak=AKIDdFh1DFpsRyLXYinXO6d0DGuNfnlYfwa4&q-sign-time=1543160059;1543160959&q-key-time=1543160059;1543160959&q-header-list=&q-url-param-list=&q-signature=3ab95c09fa98029fe9380418aa87c9438ac6c159" width="50%" />

正如前面分析list结构的时候所说，WXS可以在视图层提供一些数据的处理能力，如上图，在WXS中构造了一个循环获取列表中每项标题的方法，并将这些标题放入数组然后return出来供视图使用。这样我们就可以不用再在js中额外处理关于标题这一块的数据整合。关于WXS具体使用方式，可以参照微信小程序官方文档中关于WXS的文档。

**注意**：在使用WXS的时候踩过一个坑，当然这也怪自己，WXS中只能支持到ES5的写法，ES6的写法在模拟器以及一些较新的手机上都可正常运行，但是到了低版本的旧手机就会让整个小程序直接崩掉。所以大家在用WXS的时候要谨记，一定不能贪图方便使用ES6的语法。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/end.png?q-sign-algorithm=sha1&q-ak=AKIDdFh1DFpsRyLXYinXO6d0DGuNfnlYfwa4&q-sign-time=1543159886;1543160786&q-key-time=1543159886;1543160786&q-header-list=&q-url-param-list=&q-signature=50e602fedfd1f96f14a753cd09aad472ea5b915d" width=50% />
