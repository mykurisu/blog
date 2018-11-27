---
title: Object.assignPro-XMax(Lab小技巧-002)
categories:
-   Lab小技巧
tags:
-   Javascript
date: 2018/9/24
---

**标题和内容可能无关(微笑**

>   Object.assign()相信大家都不陌生，就算平时没怎么用，也可能有在各类文章中见过它。在如此高的曝光率背后，我们的assign是不是能承接我们所有的寄托了？貌似是不能的:(

最近在开发产品配置的后台，经常要对表单数据进行合并，所以经常用到我们的主角**Object.assign**，用起来也十分方便，直到有一次遇到这个问题--

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--002-1.png" width="50%"/>

两个对象合并之后，结果却不符合预期，assign方法的后参数对象覆盖了前参数对象，按照对这个方法的直观理解，是应该输出图中的第一个结果才对。

深感疑惑的打开了MDN，看到了关于Object.assign的这样一句解释:

>   针对深拷贝，需要使用其他方法，因为 Object.assign()拷贝的是属性值。假如源对象的属性值是一个指向对象的引用，它也只拷贝那个引用值。

这样就能解释上面那个不符预期的结果了，Object.assign()拷贝的是属性值，如果属性值是对象，它只会拷贝引用值，也就造成了“覆盖”的假象了。既然知道了原因，我们是否可以搞一个assign的升级版，让它把引用值的属性也进行拷贝，刚好看到MDN里面有关于它的polyfill，代码如下--

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--002-2.png" width="50%"/>

不难发现，assign的核心就是枚举对象的键值进行比对，然后得出拷贝的结果，那么我们的改造重心应该也是循环的这块内容。几经折腾，写了个assignPro方法--

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--002-3.png" width="50%"/>

在新的方法中，将之前循环赋值的地方换成了一个新的方法**handleR**，在里面将会对传入的对象进行递归解构赋值，检测到传入的键值是非数组对象，则将其作为参数再传入handleR方法中，直到遍历完所有的对象。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/cool/cool--002-4.png" width="50%"/>

最终使用assignPro可以使拷贝结果达到预期~**请注意，这个玩意并没有在生产中运用，大家抱着看看的心态就好了**，当然如果能够继续完善，比如对数组的处理...最后应该还是可以投入生产使用的。

最后顺带的讲一下拷贝吧，assign可以看做单层的深拷贝，如果真的想深拷贝某个对象，最方便的方法就是**用JSON.stringify把对象转成字符串，再用JSON.parse把字符串转成新的对象**。大家有兴趣的话可以试试看~

这次的小技巧到这就结束啦，大家遇到assign结果不符合预期的时候也不用慌，因为这是正常操作:)

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/end.png" width=50% />
