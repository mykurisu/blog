---
title: (prerender-spa-plugin)--微型Vue项目的SEO利器
categories:
-   博客正篇
tags:
-   Javascript
-   Vue
-   CSS
date: 2018/3/22
---

>   最近和同事写了个公司的PC官网，综合个人开发习惯、周期以及需求，我最终选择用vue-cli来快捷开发（因为之前已经写好了基于vue-cli的二次定制脚手架）。上线之后，老大说了一句，还是改回静态页吧，SPA的SEO太差啦。这...

本文会涉及到的内容--

-   使用prerender之前的境况介绍
-   使用prerender的姿势<del>(踩坑)</del>
-   小插曲:nginx的重定向
-   总结

##  使用prerender之前

我们的官网是特别“纯正”的vue-cli项目，也就是说这是个用webpack进行打包的单页应用。

在路由方面选择的是vue-router，mode是hash模式，因为并不需要考虑IE浏览器以及移动端浏览器<del>(特别是微信这个小妖精)</del>，所以并没有特别注意路由这一块的配置。

在页面开发方面，由于是两人开发，所以我倾向单个页面分离成多个组件的开发模式，这样既不互相干扰，后续改动或复用也相对灵活。

balabala...没过几天，我们的官网顺利上线啦，除了url里面会带一个"#"之外，貌似没啥不妥的地方。直到老大在群里吐槽url丑&非静态页，然后让我改用静态页T T

这样让我陷入沉思。

-   改用HTML/CSS重写一次？
-   用Nuxt进行SSR?
-   还有啥办法...

**也不知道为啥，突然想起来之前看过一个预渲染的webpack插件:)**

## prerender-spa-plugin登场

>   prerender-spa-plugin是什么？不太了解的童鞋可以传送门传走看看--[prerender-spa-plugin Github](https://github.com/chrisvfritz/prerender-spa-plugin) && [vue服务端渲染中的介绍](https://ssr.vuejs.org/zh/)

### history模式

将原来的hash模式改成了history模式，抛弃了丑丑的"#"并且也为prerender做铺垫，如果是hash模式的话可能没有办法正常的进行预渲染。

### webpack配置

因为我们用的是新版的vue-cli，webpack配置文件已经没有暴露出来了，我们需要通过修改vue.config.js来更改我们的webpack配置（其中应该是借助了webpack-merge插件来实现配置的整合）

```js
const path = require('path')
const PrerenderSpaPlugin = require('prerender-spa-plugin')
// ...
// ...
configureWebpack: {
    plugins: [
        new PrerenderSpaPlugin(
            // npm run build的输出目录
            path.resolve(__dirname, './raw'),
            // 需要进行预渲染的页面
            ['/', '/about'], {
                captureAfterTime: 5000,
                maxAttempts: 10,
            }
        )
    ]
}
```

上述配置大意是到相应的目录，根据router的信息将有关的页面预先加载渲染成静态页面，并且将静态页面以独立文件夹的形式保存下来。

下图是没有prerender的文件目录↓

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/prerender-0.png" />

可以看到，和普通的单页应用打包输出一样，一个index.html入口，动态渲染各个页面。

使用prerender之后的目录则是这样的↓

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/prerender-1.png" />

多出了一个about文件夹，里面有一个index.html文件，这里就是'/about'页面的静态页，这样我们通过url访问时就是直接访问about的静态页，而不需要再让浏览器编译这个页面。这样一来，搜索引擎就可以直接爬到我们页面的所有内容，而不是一个单页应用的启动页！

然后在根目录的index.html也变成'/'页面的静态页，这样一来就完成了我们官网页面的静态化，就是这么简单~因为prerender-spa-plugin的核心人员也是vue的核心人员，所以在vue项目里面用起来显得十分顺畅，不过如果是大型项目的话可能还是要往SSR方向去走。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/prerender-2.png" />

## 小插曲:nginx的重定向

在完成某个线上项目的重构之后需要注意一些"蝴蝶效应"--

以我们的官网为例，之前一些公司介绍被百科收录了，收录的来源链接是旧的静态页链接--"xxx/about.html"，一访问就凉了，页面已经不存在了。

一开始打算写个空的html负责跳转，但是后面想想这样不利于页面的seo，所以就想到是不是可以利用nginx的重定向解决这个问题。这里其实花了不少时间，刚开始写的时候是在location里面对相应的路径进行重定向，但是发现效果不理想。后面，东查查西查查，发现可以用rewrite来解决这个问题。

```
server {
    listen ......
    ......

    rewrite ^about.html https://targetlink.com/about/ permament;

    location / {...}
}
```

在nginx里面加入这个就可以将"xxx/about.html"这个链接永久重定向至目标url了，就不需要再写一个“跳板”页面了。

## 总结

关于SPA的SEO是永恒的话题，解决的方案其实不多，因为既然决定了使用SPA就代表该项目对SEO并没有强需求，但是如果是本文的背景下，我们就需要思考最便捷、无痛的解决方案。

SSR自然是首当其冲的方案，但是实在是伤筋动骨，得不偿失，不如重写成静态页。后面发现预渲染这个方案十分适合我们，所以就尝试了一下这个方案，在尝试这个方案的过程中，我也试了下gitlab的自动部署<del>这又是另外一个坑了</del>，在近期应该会有另外一篇文章来讲讲这个坑~

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/end.png" width=50% />
