---
title: 极简实现列表内容查看
categories:
-   博客正篇
tags:
-   Javascript
-   HTML
-   CSS3
date: 2019/06/15
---

>   最近在和各类文档文案展示斗智斗勇，在解决帮助列表静态化的同时，尝试了一下之前构思过的极简列表实现，发现竟然还不错。

本文可能涉及的内容--

-   模板渲染
-   比较冷门的CSS用法
-   一些小声哔哔

##  场景简介

假如我们有下面这组数据，需要将它们变成一个常规的Q&A页面，并且希望它是个`独立的、小型的`静态页。(如果有可能的话，尽量赋予它`通用化、模板化`的能力)

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/easylist-00.png" />

##  试试别想vue和react?

虽说vue和react给了我们很多惊喜，但是并不是所有的项目都适合使用它们，比如这次我们需要实现的 -- 静态化的文档列表查看页面，当然这并不是说vue、react不能静态化，而是有点杀鸡用牛刀的意味了。

##  如何实现?

好了，咱直接进入主题。虽然不打算用视图框架了，但是我们还是可以借助一下里面的思路，其实我们想要做的页面本质就是对一堆`已知数据`的`解析`以及`渲染`。基于此我们大概能构思这样的一个实现流程，`获取数据 -- 填充模板 -- 生成可用静态页`。

### ① 数据源

数据的源头千变万化，这里以我实践的情况来介绍。

我们会有许许多多条简短的问答条目存储在monogoDB中，根据某个tag，可以在库里取出一条或者多条数据，例如：

```json
[{"_id":"111","title":"问题一","body":{"text":"答案一"}},{"_id":"222","title":"问题二","body":{"text":"答案二"}},{"_id":"333","title":"问题三","body":{"text":"答案三"}},{"_id":"444","title":"问题四","body":{"text":"答案四"}},{"_id":"555","title":"问题五","body":{"text":"答案五"}}]
```

取出来很简单，但是我们要怎么塞进我们的页面中呢？<del>(通过接口呀!)</del>其实也不难，甚至只需要replace就能将数据放进去。假设我们的模板中有一段JS是这样的：

```js
//  ......
let FAQ_LIST = {{FAQ_LIST}};
//  ......
```

我们只需要将模板中的"{{FAQ_LIST}}"替换成我们的数据就好啦，那么，只要将上述取出的数据直接替换掉{{FAQ_LIST}}这里的占位字符串就好了吗？我们这么做了之后会发现，我们只塞了段`字符串`进去！

那应该咋办，既然替换进去成了字符串，那我们在替换之前就把它变成字符串不就好了？然后在后续的调用中，把它解析成数组就完美了。于是就有了下面这个流程：

```js
//...
const cursor = await col.find({ tag: 'xxx' })
const list = await cursor.toArray()
let template = fs.readFileSync(path.join(__dirname, '../templates.html'), 'utf-8')
const tp = template.replace(/{{FAQ_LIST}}/, JSON.stringify(list))
//...
```

没错，在填充模板之前，我们就将取得的数组变成了字符串，然后在模板实际调用时再将其`JSON.parse()`

### ② 页面布局

```html
<div class="wrapper">
        <div id="help-list"></div>
        <div class="help-content">
            <div id="help-content-container"></div>
            <div class="help-button">
                <div id="button-ok" class="button-item" onclick='handleButtonClick("ok");'>已解决</div>
                <div id="button-no" class="button-item" onclick='handleButtonClick("no");'>未解决</div>
            </div>
        </div>
    </div>
```

我们整个列表查看页的布局大致如上，大家会发现，这是不是少了点内容？是的，我们条目内容并没有出现在html中，它们将会在后续的JS处理被`插入`到相应的节点中。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/easylist-04.png" />

看到这里，也许大家会疑惑，这不是把所有的内容(问题标题&问题正文)混到一起了吗，怎么才能实现点击查看详情的效果呢？这就要借助我们神奇的CSS选择器了，具体请看下面对页面样式的介绍。

### ③ 页面样式

整个列表页面比较简单，一些整体的布局就不在赘述，大家可以点击后面的demo页面自行查看。现在比较倾向与介绍一下点击列表项实现原页面查看详情的实现↓↓

大家可以再去看看上面的页面节点图，会发现列表项的`href`是对应的详情的`id`，如果是`锚点功能`的话还可以理解，但是怎么做到类似`页面切换`的查看详情呢？看到这里是不是觉得很像我们写`单页应用`时用过的路由？

虽然不是同一个东西，但是我们可以把这看作一个极简易的单页路由。我们点击任意一个列表项，url都会带上#xxx字样，这就可以与css3的`target选择器`配合:

```css
#help-list:target {
    display: block
}

#help-list:target~.help-content {
    display: none
}

#help-content-container>.ww-section {
    display: none
}

#help-content-container>.ww-section:target {
    display: block
}
```

当url中带有`#help-list`锚点时，详情父节点`help-content`将会被隐藏，在未选中列表项时，详情模块的父节点`help-content-container`也是隐藏的状态，当我们点击某项列表项时，我们的url会带上相应的锚点，带有相应id的`ww-section`节点将会显示，由于锚点不再是`#help-list`，我们的列表也会一并隐藏。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/easylist-05.gif" />

也许上面的文字描述比较抽象，大家可以看看上面的GIF，关注`url`的变化。在刚进入页面的时候，如果判断到url没有带`hash`，我会重定向到带有`#help-list`的url，这样就能让列表显示出来。

点击列表中的某一项时，url同时也发生了变化，并且浏览器记录也增加了一条(可以模拟页面返回的效果)。

## 总结

前面啰嗦了很多，但通篇想表达的意思不多 -- 一些简单的页面还是可以用简单且高效的方案实现的。

在笔者的实际应用场景中，是需要在外部触发事件后，将库中相应的数据取出，并且塞进已编辑好的模板，最终直接上传到CDN，然后输出对应链接给使用方。

当然这是千万种方案中的一种，我们完全可以写个接口来获取数据并在页面中动态渲染......这里就当做是笔者的一次尝新实践~

测试页面附送 -- [猛击此处!!](https://blog-1252307419.cos.ap-beijing.myqcloud.com/easylist.html)
