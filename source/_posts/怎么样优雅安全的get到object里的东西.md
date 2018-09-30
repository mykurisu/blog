---
title: 怎样优雅安全的Get到Object里的东西呢？
categories:
-   Lab小技巧
tags:
-   Javascript
---

> 写在前面--这是一篇短文，是平时逛各类奇怪网站看到的一个选题，感觉挺有趣的，就自己写了个方法。

## 不安全的取对象里的元素会怎么样？
**boom~**
```js
let someObject = {a: {b: {c: 'hah'}}}
console.log(someObject.a.b.c.d.e) 
// Uncaught TypeError: Cannot read property 'd' of undefined
```
浏览器立马报错，整个页面立刻停止加载，也就是我们俗称的，挂了:(

也许这个例子看起来会有点陌生，但是如果结合到我们从接口获取的数据，是不是就有点熟悉了？如果数据结构十分复杂的话，我们经常会面临这种不安全的情况。

## 应该如何安全的获取其中的元素呢？
我们定义一个safeGet函数，里面会传入我们的object以及获取元素的路径。

<img src="safeget.png" width="50%" />

本函数的核心是try catch的使用，根据传入的path，我们将其split成一个路径数组。

随后遍历数组中的路径，不断的用try去获取目标object的元素，并将获取到的子对象重新赋值给外部的a，倘若这个过程中catch到err我们会抛出undefined，而不是浏览器的报错。

最后倘若顺利的取到目标元素，我们将会return出结果。这就是一个优雅安全的safeGet方法~无论是作为冷知识还是用于实际开发都是不错的，如果你喜欢的话，不妨分享给其他砖友们:)

