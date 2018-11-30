---
title: React中型项目的优化实践
categories:
-   博客正篇
tags:
-   Javascript
-   React
-   Webpack
date: 2018/6/13
---

>   写在前头--在公司搬砖也差不多一年了，眼看着项目越来越大，优化问题亟待解决。优化是一件很矛盾的事情，但是为了诗和远方，我们还是得走一趟坑坑洼洼的优化之路。

本文可能涉及的内容--

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/react-rebuild-00.png" />

##  项目介绍

整个项目大概有60+个页面，用到的组件大概150+，package里面的依赖大概有70+个，应该勉强算得上是一个中型的React的项目了。

下面给大家看看我们现在build一次项目的结果--

<img width="50%" src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/react-rebuild-01.png"/>

打包时间约150s，打包完之后的资源gzip之后约1.2m，尽管之前分离了一些公用依赖，但是index包的体积达到了600+还是令人难以接受的。

## 需要解决的问题 && 思考过的方案

开始优化之前，最重要的就是搞清楚我们到底要优化什么。确定了优化的目标才能着手思考优化方案，进而实施优化方案。结合对项目的bundle分析以及自身对项目的了解，我们初步可以定出以下几点优化方向--

### ① 体积瘦身

首先我们需要足够了解我们的项目，才能着手进行瘦身。在这里有一个很给力的工具可以推荐给大家[webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)

盗用一下github上的图

<img width="50%" src="https://user-gold-cdn.xitu.io/2018/6/11/163edebe5fa417ab?w=908&h=547&f=gif&s=3663774"/>

如图所示，我们可以很清晰的看到每个js文件里的module组成，还可以看到每个module的大小以及module的组成成分，这对我们分析代码冗余以及优化方向都能够提供很大的帮助。

具体食用方式也很简单--

<img width="50%" src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/react-rebuild-02.png"/>

这样一来我们就对项目有了一个比较具体的认识，大到项目的依赖一览，小到某个页面的组件引用都能在分析报告中找到。接下来就可以开始我们的瘦身之旅了。

#### 打团先找大哥

当我们第一次看到bundle的分析报告时，总能找到一些出乎意料的“大个子”，如果是必不可缺的依赖则没办法，但如果是一些可以被取代的依赖就有别的说法了。这里刚好可以看看之前我[对create-react-app中moment.js依赖的处理](https://juejin.im/post/5a3f729c6fb9a044fb07f935#heading-1)，如果处理顺利的话可以很直观的看到bundle大小的变化。

总结来说，如果有小的并且满足需求的依赖可以替换，请不要迟疑；但如果没有可满足的依赖，可以尝试自己造一个轮子，当然后者需要结合自身状况考虑。

#### 分散站位

相信大家在开发网站的时候都用到了不少依赖，但是这些依赖在输出之后是和业务代码打包在一起的，这个明显不符合我们的预期。面对这些基本不会变更的依赖，我们更倾向它们能够主动抱团并且远离我们的业务代码。

这时候，就该使用webpack的CommonsChunkPlugin（貌似在wp4中已经被别的插件替代了，在此我们先不讨论wp4），它可以帮助我们将一些指定的module打包进指定的bundle里。具体使用方案可以[参考wp官网中的相关介绍](https://webpack.docschina.org/plugins/commons-chunk-plugin)，有一个坑点就是--倘若希望负责集合依赖bundle的文件名在打包时不变，则需要生成manifest。

#### 不要将页面都放到一个篮子里（一）-- 页面分离

我们可以将一些低频页面彻底拉出项目，拿我们的项目来说，一共60+个页面，用户大多都是只会访问其中的几个或十几个页面，不可能将所有的页面都访问一遍。这样一来就必然会有一些页面的访问频率相比之下会十分低下，该怎么处理这些页面呢？这里有几种方案：

-   脱离框架重写相关页面并重新部署
-   Copy项目代码，在其他地方重新跑一次并部署，原项目就可以删除不需要的页面
-   上一方案的简化版，复刻项目环境，跑一个新的纯净项目并部署，将原项目低频页面“剪切”到新的项目中

以上三种方案个人觉得各有优劣，第一个方案`很简单`，适用场景就是类似Q&A的静态页，可以将其脱离项目，写成静态页部署在其他的地方，但是不好的地方就是可能没法复用原项目组件。

方案二呢则是`时间至上`的方案，可以做到快速迁移，但是不好的地方在于迁移出去的页面其实还是塞在一个篮子里，只不过换了个新的篮子罢了。

方案三则是`质量至上`的方案，以时间作为成本，换来一个新的“低频页面项目”。具体要使用哪种方案，我觉得也是根据当前项目状况而定，不追求最完美，追求最合适。


### ② 首屏加载

首屏加载，大概是优化的永恒话题，所有的优化都避不开这一个话题，因为只有它能最直观的让“大家”都感受到我们这次优化的成果。对于用户来说，认为网页首屏很快的标准其实很单一，就是一打开页面，看了多久的白屏。`所以我们需要做的就是弱化用户对白屏的感知`，围绕这一点，个人认为，首屏加载这一优化可以有两个方向：一个是速度，另一个是体验。

#### 引入加载占位

其实这个就是前段时间很火的“骨架屏”，我们可以在页面真正被渲染出来之前，先给用户看到一个“假的”页面,等到某个时间节点（例如数据已经准备完毕...）就将真正的内容替换上去。这里有一个我写的很不走心的例子:)

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/react-rebuild-01.gif" />

在这个优惠券列表页面我的处理方案是，初始化页面的时候就渲染3个列表项骨架，等待接口数据返回就将真实内容替换上去。

在我们的首屏其实也是类似的，我们可以根据首屏的展示结构，做一个匹配的骨架组件，然后按需求进行展示即可，这样可以有效减少用户看到白屏的时间。下面是我这个骨架的代码，优化的空间很大，不过由于优先级不是很高，所以就没有进行迭代了。

<img width="50%" src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/react-rebuild-03.png"/>

大概结构就是这样，样式方面很粗暴，因为每一项都是独立的一个组件，直接可以用absolute定位堆砌一个简洁的占位列表项。里面那个类似进度条的效果则是通过css3的animation实现的，我们可以将每个block的背景色变成渐变的，然后通过background-positon的变化来达到图中的效果。

#### 图片懒加载

这应该是个老生常谈的优化方向了，原理大概是将视图之外的图片都用同一个占位图进行占位，将其真正的图链接存在data-*中，通过监听滚动来判断图片是否进入视图中，来控制img标签src的值。具体的实现很多地方都能搜到，大家可以根据自身情况，按需选择。

#### 不要将页面都放到一个篮子里（二）-- 懒加载

>   其实在整个优化过程中我的重心是放在这个地方的，其他的都是半路上想到的...

让我们回想一下，上面我们讲过**将低频页面分离**，那么，必然就有会那么几个访问量十分高的页面，那么对于这几个页面应该怎么办呢？

因为访问频率高，所以我们可以认为这些页面与我们的核心业务是强相关的，所以将其分离就显得不那么划算了（很可能会出现维护多套代码的窘况）。

但是这样高频页面才是优化的重点区域呀，应该怎么办呢？面对这样页面我们还是可以使用懒加载大法（页面懒加载 || 组件懒加载 || 依赖懒加载）。

想要在js层面实现各类懒加载，我们都需要借助webpack中的特性**Code Splitting**，它可以将我们本来打包在一起的js分解成一块一块，并能达到按需加载并使用的效果。

-   页面懒加载

因为我们使用了react-router，所以我们可以使用react-router的getComponent轻松达到页面懒加载这一需求。如下图所示，将mainpage这样引入route的话，在打包的时候会将其分离成一个独立的js。

<img width="50%" src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/react-rebuild-05.png"/>

-   组件懒加载 && 依赖懒加载

组件和依赖的懒加载也是十分简单的，如下图这样写就能达到懒加载的效果，但如果我们使用了babel则需要修改一下babel的配置，让它能够顺利解析动态import()的语法。

<img width="50%" src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/react-rebuild-06.png"/>


### ③ 打包提速

我们通常的优化都是为了用户而优化，但其实为了我们自身能够良好的开发体验，也应该为开发人员优化优化开发体验，打包优化则成了不二之选。

#### 使用DLL为打包保驾护航

由于时间原因，在公司的项目中并没有尝试使用DLL，但是看到网上有不少同学都推荐介绍了它，所以我选择在此提及一下~[有关于webpack DLL的文档](https://webpack.docschina.org/plugins/dll-plugin/)

#### 将webpack版本从2.0 --> 4.x

由于项目是在差不多一年多以前正式启动的，所以接手的时候是webpack1.x，在刚接手的时候为了懒加载硬是升级到了2.x。

但是到现在发现，2.x好像也不够用了，毕竟已经落下了两个大版本了，更新之后的新特性、新功能或是新优化都应该成为我将项目迁移至新版本的动力。wp4具体的配置细节，在掘金上就见到过挺多同学介绍的，这里我想介绍一下，我是怎么将旧项目迁移到wp4的：

在开始进行版本迁移之前，我设想了两个方案，一个是在原有项目上直接升级并修改配置；第二个方案是新建webpack4项目，搭建好之后将业务代码迁移过来。

经过对成功率以及时间成本的评估，我最后选择的是第二个方案。那么这个新建的项目应该完善到什么程度才能进行迁移呢？我个人是经过以下几个步骤--

-   搭建项目骨架

这回的项目和之前的都不太一样了，我们没有借助大神们的脚手架来搭建项目骨架了，我们需要自己从零开始一点一点的摸索webpack的用法以及新旧版本的差异。

关于一些基础的知识以及配置十分推荐查阅[webpack官网的文档](https://webpack.docschina.org/concepts/)以及一些之前参考过的文章[webpack4-用之初体验](https://juejin.im/post/5adea0106fb9a07a9d6ff6de)、[webpack 4.0.0-beta.0 新特性介绍](https://juejin.im/entry/5a7abcc26fb9a063523ddbaf)。

相信大家看完这些之后都会对wp的配置有基本的认知，紧接下来就是建目录、装依赖巴拉巴拉。最终我们会得到一个这样的目录结构--

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/react-rebuild-07.png" width="50%" />

-   写个Hello World!

我们应该如何判断将项目代码迁入新项目的时机呢？很简单，当这个新项目可以正常的调试或打包一个相应框架的Hello World即可。

拿我们的项目来说，搭建完项目目录以及一些基础配置之后，接下来就是完全模拟原有项目的技术栈，在新的项目中写几个简单的demo页。当然这些demo页并不是随便写的，是带有目的性的，按我这次的经历来说，我写了这么几个文件index.js、App.js、Hello.js、Global.scss、router.js。

剥开非核心依赖，我们最核心的依赖其实就是react & react-router & sass，只要webpack能够正确的解析es6和sass我们就能很大程度的还原旧项目的环境。（babel中与jsx相关的配置在package里）

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/react-rebuild-08.png" width="50%" />


-   仔细研读package

上面的demo完成之后，我们新项目就初具雏形了，接下来我们就需要将旧项目package.json迁移到新项目中，这里需要注意的几点是：

① "scripts"中的指令要注意，我们要看里面的每条指令分别有什么作用，然后再思考应该怎么在新项目中写一个功能一样的指令。

② "dependencies" && "devDependencies"旧项目的依赖也应该无缝迁移过来，不过我们可以趁这个机会把没有用到的依赖剔除出去。

③ "babel" || "autoprefixer"等辅助工具的配置也应该与旧项目保持一致。

-   迁移项目&修修补补

上述步骤都跑通之后，就能删掉原有的demo，将旧项目的所有业务代码都迁移过来。接下来就看着报错，一个一个修复即可。这里遇到这么几个坑，困扰了我许久。页面很顺利的迁移过来了，依赖补全之后也顺利的跑起来了。

但是在dev环境下切换页面老是会404。相信大家看到这里就懂了，我用了history模式的路由，在devServer中应该要加上这样一行配置

```js
devServer {
    historyApiFallback: true
}
```

好啦，404消灭之后又有新的状况了，静态资源老是引用不到，这是为啥？其实这是因为我们在output的时候没有设置publicPath引起的，在dev的webpack.config中我的output是这样配置的

```js
output: {
    path: path.resolve('dist'),
    publicPath: '/'
},
// ...
devServer = {
    contentBase: './dist',
    port: 9000,
    historyApiFallback: true
}
```

这个问题解决之后，我们的开发环境算是还原的差不多了。接下来就该踩踩打包的坑了，我遇到的第一个问题就是，打包完成之后，文件夹里面只有打包输出，index.html咋不见了...这说好的不太一样。后面发现是少了[copy-webpack-plugin](https://github.com/webpack-contrib/copy-webpack-plugin)

```js
const CopyWebpackPlugin = require('copy-webpack-plugin')
const config = {
    // ...
    plugins: [
        new CopyWebpackPlugin([{from: 'public', to: ''}])
    ]
}
```

加上这个依赖之后，我们public文件夹的内容就会乖乖的在dist里面出现。但紧接着又出现新的问题了，第二次打包的时候，怎么dist没有被清空呢？和上面一样，年轻的我少用了一个插件[clean-webpack-plugin](https://github.com/johnagan/clean-webpack-plugin)

```js
const CleanWebpackPlugin = require('clean-webpack-plugin')
const CopyWebpackPlugin = require('copy-webpack-plugin')

let cleanOptions = {
    root: path.join(__dirname, '..'),
    verbose: true,
    dry: false
}

const config = {
    // ...
    plugins: [
        new CleanWebpackPlugin(['dist'], cleanOptions),
        new CopyWebpackPlugin([{from: 'public', to: ''}])
    ]
}
```

完成上述配置后，每次打包webpack都会清空dist文件夹，并且在打包完成之后，将public中的内容复制到dist。好了看来应该可以了，但是在本地开了个服务器跑页面的时候发现，各种静态资源404。这又是什么玩意？实话说在这里踩坑时间是最多的，但是解决方案又是令人窒息的简单...都怪自己没有好好看文档

```js
const CleanWebpackPlugin = require('clean-webpack-plugin')
const CopyWebpackPlugin = require('copy-webpack-plugin')

let cleanOptions = {
    root: path.join(__dirname, '..'),
    verbose: true,
    dry: false
}

const config = {
    output: {
        filename: 'static/js/[name].[chunkhash:8].js',
        chunkFilename: 'static/js/[name].[chunkhash:8].chunk.js',
        publicPath: 'http://localhost:5000/' // !!!这里一定要使用绝对路径，不然就会被坑到
    }
    // ...
    plugins: [
        new CleanWebpackPlugin(['dist'], cleanOptions),
        new CopyWebpackPlugin([{from: 'public', to: ''}])
    ]
}
```

## 总结

好了，不知道有多少同学会看到这里，先谢谢大家看我在这唠叨一堆~各类优化的方案在网上看了好多好多，但是好像大家都只讲方案没有涉及实践，等到自己真正去玩的时候才发现，其实优化没有想象中那么简单，要兼顾原有的，又要尽量使用更新更好的，很多时候都会在夹缝中取舍。

其实，能够优化的还有很多很多，请求方面、业务方面甚至是代码写法...都是可以优化的，但是这些怎么能一蹴而就呢？还是得走一步，看一步，选择最适合自家项目的优化方案才是最佳方案~

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/end.png" width=50% />
