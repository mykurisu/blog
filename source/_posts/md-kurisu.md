---
title: 一个Markdown在线编辑器的开发全剖析
categories:
-   博客正篇
tags:
-   Javascript
-   Vue
date: 2020/03/17
---


>   还是讲讲背景吧，公司技术团队做了自己的公众号，不知为何笔者就成了小编之一。为了解放生产力，果断决定撸个可以一键生成微信推文的编辑器，也就是我们下面会讲到Markdown在线编辑器啦~

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/md-kurisu/md-1.png" />

如图，这就是编辑器的全貌，分为编辑区域与预览区域，其中预览区域可以通过菜单中的按钮实现隐藏或展示，方便不同的编辑需求。

##  编辑器

完成一个编辑器，听起来就十分困难呀，一开始笔者也是一脸懵逼，下面给大家说一下笔者的编辑器是如何一步步走上正轨的，开发这个编辑器大概经历了两个阶段，第一个阶段是按照自己的想象闭门造车，从零开始硬撸；第二阶段则是寻找业界成熟的编辑器解决方案，利用其完成编辑器模块。

### Textarea

一般需要做这种密集型的文本输入及编辑，我们脑海里第一个浮现出来的肯定是 **Textarea** 标签，但是平时见到的文本框都长这样：

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/md-kurisu/md-2.png" />

要怎么才能变得更像编辑器一点呢？首先，它得够大

```css
textarea {
    width: 100%;
    height: 100vh;
    /* ... */
    box-sizing: border-box;
    outline: none;
    resize: none;
}
```

weigth、height这些都是基操，那么最后的两个属性是啥呢？

**outline** 用于取消chrome下textarea的聚焦边框

**resize** 用于禁用文本框的缩放功能

有了这些属性之后，从外观上看，长得就比较像编辑器了，但是在实际使用中我们会发现一些莫名的波浪线，

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/md-kurisu/md-3.png" />

例如上图的这种，在我们文思泉涌时，看到一对波浪线心里应该不太好受，查了一下textarea和input这类输入功能的标签都有 **spellcheck** 属性，也就是语法检查，我们只需要在标签中将spellcheck置为false即可避免满屏波浪线的窘况。

如此一来，一个超简陋版的编辑器就算是完成了，但是在实际使用的上还是体验差了点，连最基础的Tab键缩进都没法实现(虽然后面还是头铁的用onkeydown写了个tab键缩进)，最后笔者选择看看自己常用的一些在线编辑器都是怎么实现它们的编辑区域的，这一看就彻底改变了开发方向，笔者也是第一次发现 **CodeMirror** 这种神器。

### CodeMirror

[CodeMirror](https://github.com/codemirror/CodeMirror)是由JavaScript实现的专为浏览器而生的在线编辑器，它支持多种语言的编辑、高亮，还提供了丰富API和主题让我们自由定制自己的编辑器。

我们只需要将原先Textarea的一坨代码换成下面这一小段即可。

`
<codemirror :options="cmOptions" />
`

我们就已经得到一个带有Markdown高亮，且特定语言代码块也能有高亮的编辑器。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/md-kurisu/md-4.png" />

##  预览区域

预览区域的实现十分简单，接受外部的html字符串，然后通过**v-html**渲染出来即可。


```
<template>
    <div id="preview">
        <div :class="`content ${extensionName || ''}`" v-html="value"></div>
    </div>
</template>
```

也许大家会疑惑代码中的**extensionName**是干啥用的，这是后面插件导入用到的参数，我们在后面会介绍。

##  数据流通及处理

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/md-kurisu/md-5.png" />

整个编辑器的核心就是如此，用户输出数据，在Edit组件中收集并处理数据，随后将数据传递到Preview组件中进行渲染。

数据的输入不用多说，上文已将编辑框区域大致介绍了一遍，我们下面来讲讲Markdown的解析以及一些衍生的功能。

### Markdown解析

现成的Markdown解析有很多，笔者用过这两个[marked](https://github.com/markedjs/marked)、[showdown](https://github.com/showdownjs/showdown)，感觉都挺不错，文档清晰、支持拓展，就算预设的插件不足以支持自身需求也可以自己动手丰衣足食。

在本次编辑器开发中使用的是marked，大家可以结合自身情况选择合适的Markdown解析器。

```
<template>
    <div>
        <Code
            :value="codeValue"
            @change="handleChange"
        />
        <Preview
            :value="htmlValue"
        />
    </div>
</template>

<script>
handleChange({ value }) {
    this.codeValue = value;
    this.htmlValue = marked(value);
}
</script>
```

只需要一个**handleChange**方法就可以初步实现Markdown的实时预览了，为什么笔者还需要在Edit组件里面记录codeValue呢？主要是为了数据的一致性，既然是实时预览，那么我们两边的数据必然是一致的，不然会给用户带来困扰，所以我们需要在父组件对数据进行统一管理并按需分配。

### 插件开发

[marked.js的插件开发文档](https://marked.js.org/#/USING_PRO.md)

下面举一个正在使用的插件案例，这个插件负责完成公司公众号推文的特殊样式输出。

```js
import marked from 'marked'


const renderer = new marked.Renderer()

//  解析<h1>标签时需要将内容包裹在一个拥有特殊布局的<h1>标签中，并且不影响其他head标签的渲染
renderer.heading = function (text, level) {
    if (level === 1) {
        return `<h1 class="title"><p class="num">{{h1Title}}</p><p class="text">${text}</p></h1>
        `
    } else {
        return `<h${level}>${text}</h${level}>`
    }
}

//  上面renderer是在Markdown被解析成token时会调用的方法，下面这是Markdown解析完成后调用的方法
const parseCallback = (error, parseResult) => {
    if (error) {
        return ''
    }
    let h1Title = 0
    //  由于推文需要展示大标题的序号，所以在上面修改<h1>标签时留了占位符，在结果输出前将对应占位符换成序号即可
    parseResult = parseResult.replace(/{{h1Title}}/g, () => {
        return ++h1Title
    })
    return parseResult
}

const extension = {
    renderer,
    parseCallback
}

export default extension
```

最终我们通过这个小插件实现了下图的效果↓↓

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/md-kurisu/md-6.png" />

### 插件导入

下面简单的介绍一下，插件的导入方式。

最简单粗暴的，我们可以将插件直接写在**marked()**中

```js
this.htmlValue = marked(
    value,
    { renderer: () => {} }, //  renderer
    () => {}    //  parseCallback
);
```

也可以像上面一样，写到独立的js中，在相应的地方进行引用。要是更复杂一点，比如是可选择的插件，我们事先并不知道用户需要用到哪个插件，所以我们得支持动态的插件选取。

```js
handleExtensionChange(ex) {
    this.extensionName = ex;
    this.selectedExtension = null;
    if (ex) {
        this.selectedExtension = require(`../extensions/${ex}/index`).default;
        this.handleChange({ value: this.codeValue });
    }
}

handleChange({ value }) {
    this.codeValue = value;
    if (this.selectedExtension) {
        this.htmlValue = marked(
          value,
          { renderer: this.selectedExtension.renderer },
          this.selectedExtension.parseCallback
        );
    } else {
        this.htmlValue = marked(value);
    }
}
```

实现思路也比较简单，我们只需要在用户进行选择之后，将相应的插件内容赋值到**selectedExtension**中，并且手动调用一次编辑器的渲染过程，即可实装插件。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/md-kurisu/md-7.gif" />

### 本地储存

这个功能比较类似草稿功能，因为配套的后端还没开发，所以暂时将文章内容储存到本地。储存，最重要的是什么？储存的方式？储存的时间？笔者认为存储最重要的是确保唯一性，只有确保每条数据都是唯一的，才能谈后续的储存方式、储存时间等。

那么在编辑器中，我们有什么东西可以确保用户正在编辑的文章是唯一的呢？答曰：**时间**，用户第一次聚焦编辑区域的时间就是这篇文章的唯一标识，我们监听编辑区域的focus事件，可以精确的将用户聚焦的时间记录下来，并且将当前的路由替换为新的路径。

```js
handleFocus() {
    if (this.$route.params.articleID === "newArticle") {
        this.$router.replace(`/editor/${btoa(`${Date.now()}_md`)}`);
  }
}
```

有了唯一标识之后，我们就可以通过判断url中的**articleID**来对文章进行本地存储，在本编辑器中使用的是渐进式的本地存储方案[localforage](https://github.com/localForage/localForage)有兴趣的同学可以去它的项目里瞧瞧。有了本地存储之后，我们就可以通过读取url中的id来获取本地存储的文章内容，来实现草稿的功能。

```js
export default {

    mounted() {
        this.handlePageInit();
        
        this.saveTimer = setInterval(() => {
            this.handleSave();
        }, 30000);
    },

    methods: {
        handlePageInit() {
            const { articleID = "" } = this.$route.params;

            if (!articleID || articleID === "newArticle") {
                this.codeValue = Example
                this.htmlValue = marked(Example);
                return (this.inited = true)
            }

            localforage.getItem(articleID).then((res) => {
                if (res) {
                    const { value } = res
                    this.handleChange({ value });
                    message.success("草稿导入成功");
                }
                this.inited = true;
            });
        },

        handleSave() {
            if (!this.inited) {
                return;
            }

            const { articleID = "" } = this.$route.params;
            if (!articleID || articleID === "newArticle") {
                return;
            }
            const store = {
                title: this.codeValue.substring(0, 6),
                value: this.codeValue,
                time: Date.now()
            };
            localforage.setItem(articleID, store, () =>
                message.success("已自动保存")
            );
        },
    }

}
```

### 公众号适配

做这个编辑器的本意就是解放编辑公众号推文的劳动力，所以关于公众号的适配自然不能少。

对公众号编辑来说，最重要的就是编辑完的文章应该怎么复制过去。经常写推文的同学应该都接触过市面上的推文编辑器，他们都是通过直接复制HTML到公众号编辑框的方式，来完成特定格式推文的制作。所以我们实现的逻辑也当然是复制，但并不是复制编辑区域的文字，而是复制预览区域的所有内容。这个全选复制，可以自己实现也可以使用[clipboard](https://github.com/zenorocha/clipboard.js)这一方案，是十分成熟的浏览器复制解决方案。

复制的问题大概解决了，还有一个比较重要的问题就是文章排版样式，我们可以自定义一套符合自己需要的排版样式去覆盖公众号推文自身的样式，但是并不是所有的样式都可以被重写覆盖，具体的需要大家按需摸索，在我的代码中有正在使用的一套排版以供参考。

##  尾声

最后给大家介绍一下整篇的主角 --> [kurisu - Github](https://github.com/mykurisu/kurisu)，是自用的在线Markdown编辑器，想在线体验可以猛击[此处](https://md.mykurisu.xyz/editor/newArticle)（自用小水管服务器，大家悠着点）。

个人认为最好的几种使用方式：

-   和笔者一样，有负责一些团队或者小组的推文工作，可以clone项目到本地或者指定服务器，自己跑起来，然后根据需要更改插件与排版，做成自己专用的编辑器。
-   推文爱好者，如果仅仅只是自用，并且不想另起服务，可以直接使用笔者的在线版，并且可以在GitHub中提交自己的插件与排版。
-   尝鲜爱好者，如果只是想看看编辑器的构造，当然还是直接把代码拉下来比较好啦，由于后端还在开发，笔者认为前端能看的干货并不多，看官们轻拍就好。
