---
title: 如何实现一个移动端极简PDF浏览页面
categories:
-   博客正篇
tags:
-   Javascript
-   PDF
date: 2019/8/15
---


近期沉迷...(不想再编了，就是偷懒了)

>   为什么会有这么个需求呢？因为我们的业务特性，用户会很经常的需要打开页面中的PDF进行浏览查阅，然而不同的手机在打开PDF的表现上各有千秋，特别是安卓机，大部分会直接将其下载到用户手机且没有明显提示。基于此，我们需要一个统一的PDF浏览页面。

##  关于“统一浏览”，我们应该怎么实现呢？

思路不外乎这两类 -- ① 将PDF转成图片，在用户访问时取出相应的图片集合并显示。② 在页面内下载PDF，并直接在页面内渲染。

-   在方案①中，用户看到的pdf实际上已经是一张张图片了，这种方式的兼容性极佳，几乎不会有不兼容的情况。但是这个方案也有一些不如人意的地方。

图片需要提前生成，如果等用户的访问请求到了再去拉取pdf并处理，前端的响应时长可能就不太可控了(甚至还有转换失败的可能)。

图片的存储，图片生成好之后还没完事，我们还需要定制存取规则，图片按规则存放，前端才能正常的取到图片并展示(还需要存储一份pdf的特征描述文件)。

-   方案②中，用户看到的则是转化成canvas的pdf，这种方式则是灵活性极佳，我们可以接收传递进来的任意pdf链接，然后直接下载并将其渲染成canvas，展示在页面上。

当然这个方式也是有缺陷的，无法保证它的兼容性达到“极佳”的水平(如果真的不兼容应该怎么办？页面可以提供下载按钮，用户点击后会直接下载到本地)，但是它的灵活性是无可替代的优势，我们最终选择的方案是在页面内直接渲染。

决定方案之后，我们该考虑的就是如何实现了，关于pdf转canvas的方案在网上一搜就有一堆，但是核心基本都指向pdf.js，所以我们自然也是朝着[mozilla/pdf.js](https://github.com/mozilla/pdf.js)前进。

##  pdf.js怎么玩?

其实大家点进去项目仓库都能看到使用指南，基础的部分就不再赘述，在这比较倾向介绍一下在实践中遇到的各种问题。

### ① - 移动端自适应

按指南上的配置，我们在页面里写下这样的配置，

```js
var viewport = page.getViewport({
    scale: 1
});
var canvas = document.getElementById('the-canvas');
var context = canvas.getContext('2d');
canvas.height = viewport.height;
canvas.width = viewport.width;

var renderContext = {
  canvasContext: context,
  viewport: viewport
};
page.render(renderContext);
```

我们会得到这样的效果

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/pdf-viewer/01.png" />

我们会发现，PDF显示不全并且也不像是移动端的显示模式。用户想要看到完整的内容只能通过放缩，这未免体验太差了。

既然显示的结果是内容过大，那我们能否在渲染的时候就将其缩小呢？在此之前我们先看看viewport里面得到的是什么内容。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/pdf-viewer/03.png" />

结合打印出的内容也API文档上的介绍，我们大致可以知道这个得到的是PDF的尺寸，因为我们传入的scale是1，所以我们应该得到的是“一倍图”PDF的尺寸。到此，身为切图仔突然有了一点思路 -- 这和我们在web端处理小于12px字体的方案有点相似。

当我们需要在页面显示小于12px的字体时，我们有一个方案就是将那部分字体大小先放大一倍(假如需要10px的字体，我们会先得到20px的字体，然后再transform: scale(0.5, 0.5))，然后在将其缩小一倍，然后处理它的位置。

思路有了，我们要怎么将其运用到PDF浏览里面呢？我们尝试将所有的canvas都缩小，效果如下。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/pdf-viewer/05.png" />

这是怎么回事呢？PDF显示的位置偏移的十分离谱，这就不得不说一下我们的transfrom-scale，我们在进行变形时，css会默认将其放缩的基点放在整项的中间，就等于我们在设计时会说到的中心放缩。

所以我们在进行放缩类的操作后，需要进行一下变形基点的设定，也就是transform-origin属性。

```css
canvas {
    transform: scale(0.5, 0.5);
    transform-origin: 0 0;
}
```

解决了位置偏移的问题后，又有新的问题出现了 -- 这个PDF缩放后太小了，没法铺满屏幕，那么我们是否可以通过与页面宽度得出一个关系，让其可以铺满屏幕呢？

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/pdf-viewer/06.png" />

就拿我们上面的测试PDF来说，scale为1的时候，PDF宽度为750，然而视窗是iPhone8 plus的尺寸，所以将canvas缩小一半会令其无法横向铺满视窗。现在有两个方案，一个是动态的transform放缩尺寸，另一个则是getViewport时动态计算scale数值。从css对小数数值的兼容性考虑，最终我选择了后者。

```js
//  初始scale数值
var scale = 1;

//  获取PDF在“一倍图”时的尺寸
var viewport = page.getViewport({
    scale: scale
});

//  获取body宽度
var width = document.body.clientWidth;

/** 
 * width / viewport.width > 1
 * 视窗 > PDF一倍宽度最终得到scale > 2
 * 反之则会得到小于等于1的scale
 * 最终再*2是为了得到更清晰的渲染
*/
scale = scale * width / viewport.width * 2

//  重新定义scale之后再次getViewport
viewport = page.getViewport({
    scale: scale
});
```

至此，我们已经可以渲染出一个比较易于阅读的PDF浏览页了，也许看上面的文字不太能够理解我在胡言乱语什么，所以做了一张图，希望能够更好的表述我的思路 -- 

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/pdf-viewer/04.png" />

### ② - PDF内的印章显示不出来？

完成上述的动作后，一个可阅读的PDF页面已经完成得差不多了，但是对比原件之后惊奇的发现，印章没了。

去翻了一下项目的issue发现这是个有一定年份的问题了，看了下源码还是作者有意将电子签名及印章隐藏的，不知是出于安全考虑还是优先级排期，这个issue一直没被解决。

最终的解决方案也异常简单粗暴，只需找到源码设置HIDDEN属性的代码，将其注释即可。

### ③ - PDF里面有特殊字体会一片空白？

在调试其他PDF文档是发现，有些页面会是一片空白，一开始以为原件就是如此，但是对照之后发现这一页是用了特殊字体。

在搜索解决方案的时候看到**getDocument**时有这样一个参数**disableFontFace**，这个参数的默认值是false，看起来将其设为true就可以使用默认字体了。事实上并不是的，这个参数是负责控制是否使用内置的字体渲染器来渲染。

随着搜索的深入，看到这样一个解决方案 -- 

```js
pdfjsLib.getDocument({
    url: path,
    cMapPacked: true,
    cMapUrl: 'https://unpkg.com/pdfjs-dist@2.2.228/cmaps/'
})
```

里面涉及到**cMapPacked**和**cMapUrl**两个参数，前者表明用到的cmap是二进制类型的，后者这是设定cmap的请求地址。个人对这一块配置的理解是，如果遇到不支持的字体，将会去指定的地址获取默认字体的bcmap用于渲染替代特殊字体的默认字体。

参考链接 -- 

[getDocument参数介绍](https://mozilla.github.io/pdf.js/api/draft/global.html#getDocument)

[what-is-bcmap?有关bcmap的解释](https://stackoverflow.com/questions/32764773/what-is-a-pdf-bcmap-file)

### ④ - 渲染性能调优

因为之前都是用自己的手机进行调试的，所以一直没感觉到卡顿的情况，借用了测试机之后发现，打开页数较多的PDF会出现卡顿情况。总结了一下，原因大概是并行了过多的渲染。一开始的写法是在getDocument之后拿到PDF页数直接for循环将所有的page同时输出。

```js
//  伪代码
pdfjsLib.getDocument({ url: 'xxx' }).promise.then(function (pdf) {
    numPages = pdf._pdfInfo.numPages
    for (var i = 0;i < numPages;i++) {
        pdfCreator(pdf, i + 1)
    }
});
```

既然同时输出会引起卡顿，能否优化成一张一张顺序渲染呢？自然是可以的，我们可以通过递归的方式将PDF一页页输出。

```js
var numPages = 0;
var renderFlag = 0;
//  ......
pdfjsLib.getDocument({ url: 'xxx' }).promise.then(function (pdf) {
    numPages = pdf._pdfInfo.numPages
    pdfCreator(pdf, 1)
});

function pdfCreator(pdf, index) {
    //  ......
    pdf.getPage(index).then(function (page) {
        //  ......
        page.render(renderContext).promise.then(() => {
            renderFlag = index
            if (renderFlag < numPages) {
                pdfCreator(pdf, renderFlag + 1)
            }
        });
    })
}
```

##  小结

一个功能相对完整的PDF浏览页面就完成了，还是有需要后续优化的地方，例如page.render的容错处理、PDF下载功能，甚至还可以新增懒加载功能。

感谢大家的阅读，最后留下页面的访问模式，大家可以传入自己的PDF链接进行浏览。(待补充)
