---
title: 一起了解一下JS的事件呗
categories:
-   博客正篇
tags:
-   Javascript
---

> 写在前面，这应该是JS中很基础的一部分吧，但是身为苦苦自学前端的切图仔，对其了解还是不够深刻，有必要写写~

本篇可能涉及以下内容--
-   一个弹窗&蒙版的点击
-   DOM的事件模型
-   JavaScript事件流
-   JavaScript事件委托
-   一个小DEMO

## 一个场景
相信下面这个弹窗大家都见过或者做过吧↓

<img src="dialog-1.jpeg" width="50%" />

我们假设弹窗底部没有可点击的按钮，我们需要点击弹窗外部的半透明层才可将弹窗隐藏，不知道大家在写这个需求的时候有没遇到过这样一种情况：

-   我们给半透明层添加的事件，但是点击弹窗区域也是可以触发的

为什么会这样，仿佛是点击穿透了一样。想要了解清楚这里面发生了什么，我们需要从DOM的事件模型开始讲起...

## DOM的事件模型
### DOM0级事件模型
依照个人的理解，DOM0级事件模型指的是绑定在DOM本身的事件属性，例如以下几种：
```html
<h1 onclick="alert(1)">点我</h1>
```
```js
// <h1 id="kurisu">点我</h1>
document.querySelector('#kurisu').onclick = function () {alert(1)}
```

### DOM2级事件模型
DOM2级事件模型定义了一种新的模式，给需要触发事件的元素绑定监听器--
```js
// <h1 id="kurisu">点我</h1>
document.querySelector('#kurisu').addEventListener('click', () => {
    // do something
})
```

上述两种模型都能达到点击触发事件的效果，但是DOM0级的同一事件类型会被后注册的覆盖，而DOM2级的事件监听则可以在一个DOM节点同时绑定多个同类事件，现在开发过程中一般都是会使用addEventListener来监听相应的事件，所以下面也将会围绕这种事件模型继续展开。

## JavaScript事件流
在默认的情况下，我们点击页面中的一个按钮，按钮会给我响应（点击事件的触发），同时这个点击会“往外”传递(e.target->...->window)直到window为止，在这条传递路径中如果还有其他点击事件也会被一同触发，还有一句话比较简洁--我们在点击这个按钮的同时，其实也点击了整个页面。

这个就是我们文章开头场景中出现问题的原因了，因为事件流的传递，导致你点击弹窗时，半透明层的点击事件也被触发了。但是有没有发现很器奇怪的事情，为什么点击半透明层的时候，弹窗的点击事件悄无声息捏？因为，默认情况下，事件流的传递方式是冒泡。

### 事件冒泡
现代浏览器默认的事件流模式，个人理解是从点击的DOM节点开始，一层一层向window传递，最后也止于window。

那么如果我们不想让事件继续分发给其他节点应该怎么办呢？很简单，在事件监听时禁止事件分发即可--event.stopPropagation()

### 事件捕获
与事件冒泡相反，有个事件流模式叫捕获，他是从window开始接收事件，再一步一步传入目标DOM节点。同样，在捕获事件中我们也可以通过event.stopPropagation()来禁止其分发事件，但是这样能够正常触发的就只有最靠近window的DOM节点的事件。

借用一张W3C的图片可以很形象的了解这两种模式--

<img src="eventflow.svg" width="50%" />

## JavaScript事件委托是个啥
因为事件冒泡和事件监听器的存在，使得我们可以使用事件委托。什么是事件委托？我想用一个例子说明会比较好，我们有一个ul，里面有数十个li，每一项li都有一个点击事件，这时候该怎么办？

一个个li去手机添加监听或者注册点击事件？不，这太不科学了。我们可以给ul绑定事件监听器，通过一个总的监听器来达到监听所有子项的效果。这样做不仅能够节省代码事件，甚至还可优化页面，绑定一个监听器，总比给每一个li都绑定好，对吧？

```js
function kurisu() {
    let target = document.querySelector('.all__content') // ul.all__content
    target.addEventListener('click', (e) => {
        if (e.target.className === 'item') { // li.item
            e.target.style.backgroundColor = '#eee'
        }
    })
}
```

由于事件冒泡的存在，我们绑定在ul上的监听器也能监听到在li往外冒泡的点击事件，现在我们只需要通过if来限定事件在冒泡到哪个阶段才执行相应的方法即可。

## 一个DEMO
这个demo包括JavaScript的两种事件流传递模式、禁止事件分发还有事件委托这三种情况，一直感觉JS事件监听这个玩意很难光靠文字说明清楚，我还是希望能用一个demo页来丰富我前面所写的内容。传送门在下方↓↓

[Codepen传送门](https://codepen.io/anon/pen/MQbPoq)
[Github传送门](https://github.com/mykurisu/mykurisu-demo/tree/master/eventlistener)

两个地方的代码是一样的，如何选择取决于大家怎么看比较方便~

<img src="https://user-gold-cdn.xitu.io/2017/12/24/16087d7ac487f37c?w=375&h=524&f=png&s=118753" width=50% />