---
title: 从-1开始用Vue写一个活动页（一）
categories:
-   博客正篇
tags:
-   Javascript
-   Vue
-   CSS
-   微信相关
---

> 写在前面--为啥从-1，因为看到大大们都是从0开始起手的，身为切图仔就只能从-1开始起手啦。不扯啦..让我们进入正题。

关于Vue我感觉不需要再做介绍了，在掘金随便一搜都有一堆，不过如果大家实在没听说过，我还是十分建议直接进入[Vue的官方文档](https://cn.vuejs.org/)进行阅读。

## 项目背景
最近，接了一个活--一个完全独立的活动页开发，由于它跟公司原有的项目(基于React)几乎完全没有依赖关系，所以我决定采用Vue来开发这次的需求。前端项目的技术框架初步定为--Vue/Vue-router/axios/sass/webpack。看到vue和webpack相信大家一下就能想到vue-cli吧，对的，我们这次项目选用了vue-cli-webpack作为脚手架。

## vue-cli:居家必备
我们假设vue的起手难度是10个难度级别，但是当我们使用了vue-cli之后，我们会发现这个难度就变成了1个难度级别，(<del>好像我们的施法前后摇都被取消了一样</del>)安装好vue-cli之后我们就可以直接开始我们的生产了，无论是vue亦或是webpack的配置都不需要我们理会，并且还能够使用vue-template来加快开发速度。那么让我们开始搭建我们的脚手架吧--

```
$ npm install -g vue-cli

$ vue init webpack kurisu //'webpack'指定了我们希望项目是使用webpack进行打包的
$ cd kurisu
$ npm install
$ npm run dev
```

整个流程就是这么简单，前提是你需要安装Node(6.x版本以上)以及NPM(3.0版本以上)。现在我们有了真正意义上的脚手架了，接下来就要着手往上堆东西了。

## 我是谁？我在哪？我开发什么？我需要什么？
在项目伊始，相信大家都遇到过这个问题，我们所开发的项目到底需要怎么开发、它需要什么工具、或是有什么能够为它所用。

这回我需要开发的是一个移动端的活动页，移动端移动端，自然就少不了对屏幕的适配啦。以切图半年的经验，我选择rem作为页面开发的单位，其实就是借鉴了淘宝flexible的那一套。既然用到了这个，那么css的预编译器就少不了了，这里选择了之前一直都在用的scss，但是最近听说stylus也不错，小纠结了一会，最终还是选择现在看来开发效率最高的scss。

好了，回看上面那段话，好像已经给自己挖了不少坑了，让我们一个个来填。

- 移动端Html
    
    关于移动端的html要说的不多，最重要的这句在就好了--
    ```html
        <meta name="viewport" content="width=device-width,initial-scale=1,minimum-scale=1,maximum-scale=1,user-scalable=no"/>
    ```
- Rem布局

    Rem是啥？简单来说是css中的一个单位，以html的font-size作为单位的标准也就是说
    ```css
    html {
        font-size: 100px;
    }
    ```
    此时1rem也就等于100px，嗯嗯这些我都懂，但是他跟移动端开发有什么必然关系吗？可以说有也可以说无，用了rem作为开发单位可以提升开发效率，降低视觉稿还原难度，也能增强页面在不同屏幕上的适应能力。
    下面是在项目中的Global.scss的配置--
    ```css
    html {
        font-size: calc(100vw/7.5);
        /*有了这一行就代表html的font-size是动态计算的，它随着我们屏幕的大小(100vw)变化而变化*/
    }
    ```
    那么其中的7.5是什么呢？其实是这样的--100vw/750px，为什么是750px，因为这是标准iPhone7尺寸的两倍也就是俗称的二倍图，这样处理之后1rem就等于iPhone7手机页面中的100px，那为什么要除7.5呢？个人认为是为了方便书写。
    ```css
    @function px2rem($n) {
        @return $n/100*1rem
    }
    ```
    定义了这一个方法之后，我们想做设计稿(二倍图)中一个100x100px的正方形时可以这么写：
    ```css
    .somediv {
        width: px2rem(100);
        height: px2rem(100);
        /*px2rem(100) ==> 375px屏幕中的50px*/
    }
    ```
    不知道大家发现没有，这就代表着我们可以完全照搬设计稿中的所有参数，并且基本可以忽略不同尺寸屏幕的适配情况，因为rem的基础单位是会随着屏幕尺寸变化而变化的，但是有一点可能有问题,那就是**Retina屏的精度问题关于这里的适配大家可以去寻找相关的博文查看一下**，如果要细讲的话可能就偏题了。
- SCSS配置

    说了这么多次scss，好像还没讲应该怎么将scss接入webpack中。
    
    首先把相关的依赖装好--
    ```
    (c)npm install node-sass --save-dev  //安装node-sass
    (c)npm install sass-loader --save-dev  //安装sass-loader
    //本来应该还要装一个style-loader的但是一般我们用vue-cli之后都是有vue-style-loader的所以就不需要再折腾了。
    ```
    紧接着就配置一下我们的webpack.base.conf.js
    ```js
    module: {
        rules: [
            ...(config.dev.useEslint ? [createLintingRule()] : []),
            {
                test: /\.vue$/,
                loader: 'vue-loader',
                options: vueLoaderConfig
            },
            {
                test: /\.js$/,
                loader: 'babel-loader',
                include: [resolve('src'), resolve('test'), resolve('node_modules/webpack-dev-server/client')]
            },
            {
                test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
                loader: 'url-loader',
                options: {
                    limit: 10000,
                    name: utils.assetsPath('img/[name].[hash:7].[ext]')
                }
            },
            {
                test: /\.(mp4|webm|ogg|mp3|wav|flac|aac)(\?.*)?$/,
                loader: 'url-loader',
                options: {
                    limit: 10000,
                    name: utils.assetsPath('media/[name].[hash:7].[ext]')
                }
            },
            {
                test: /\.(woff2?|eot|ttf|otf)(\?.*)?$/,
                loader: 'url-loader',
                options: {
                    limit: 10000,
                    name: utils.assetsPath('fonts/[name].[hash:7].[ext]')
                }
            },
            //新增下面这一段即可，让webpack使用相关的loader去解析scss后缀的文件
            {
                test: /\.scss$/,
                loaders: ["style", "css", "sass"]
            }
        ]
    }
    ```
    这样就完成了在vue项目中引用scss的需求啦，还有一点需要注意的是该怎么在vue-template中引用，这里其实有几种不同的引用方法，就先介绍一下我的引用方式--
    ```css
    <style lang="scss" scoped>
        @import "./Index.scss";
    </style>
    ```
    个人倾向于一一对应的引用模式，每一个page我都会新建对应的文件夹，并将与之相关的东西都放到一起。
    ![vue文件目录](vue-1.png)
    与全局相关的样式则放在Global.scss中由App.vue引入，Utils.scss则是用来放一些**公用但不公共**的样式或方法。另外还有一种方式--将所有的scss文件都import进一个总的scss文件，然后在App.vue中引入这个总的scss文件即可。

## 舞台搭建完毕
至此，已经基本完成了vue项目的脚手架，我们可以很惬意的往项目里面填肉了。下一篇，将会结合具体开发的页面，记录一下途中遇到的问题和填过的坑。

## 往期文章
-   [移动端自制音乐播放器--React](https://juejin.im/post/5a4e3af86fb9a01cba425d7d)
-   [移动端中踩过的关于日历&时间的坑](https://juejin.im/post/5a3f729c6fb9a044fb07f935)
-   [谈谈React--componentWillReceiveProps的使用](https://juejin.im/post/5a39de3d6fb9a045154405ec)

<img src="https://user-gold-cdn.xitu.io/2017/12/24/16087d7ac487f37c?w=375&h=524&f=png&s=118753" width=50% />
