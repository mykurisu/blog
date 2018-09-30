---
title: 从-1开始用Vue写一个活动页（二）
categories:
-   博客正篇
tags:
-   Javascript
-   Vue
-   CSS
-   微信相关
---

> 距离第一篇的发布过了好久，因为这次真的踩了不少坑..以后还是尽量不要新立项目好了，就跟立了个flag一样T T

[第一篇传送门--记从无到有的活动页开发-1](https://juejin.im/post/5a584a996fb9a01cb64eb868)

本篇主要内容有以下几点--
- 使用vue-router除了mode之外，可能还需要知道base属性
- 关于hash与history这两种模式的选择
- 关于微信授权的处理方式
- 域名限制下如何带着镣铐跳舞

## 使用vue-router除了mode之外，可能还需要知道base属性
首先介绍一下项目的背景，脱离了背景谈需求都在瞎逼逼，本次活动页是独立于主站项目(SPA)的新独立项目，但是由于对微信的强依赖，新项目还是需要挂在主域名下。

这有啥的，让运维同事帮忙配个nginx把主站的某个路径指向新项目目录就好呗~事实上确实是的，一开始配的域名是**m.kurisu.fm/my**(对..这个是假域名)凡是这个路径的都直接指向vue项目的目录，很完美跑起来了，这个也是微信的安全域名所以登录应该也是稳妥的。但是问题来了...它没法支付...这个域名并不是可支付的域名，所以我们必须要改动域名。

我们的可支付路径是**m.kurisu.fm/pay/:id**，只好再麻烦一下同事帮忙配个**m.kurisu.fm/pay/my**的路径了，好啦，restart之后一看，凉了。白屏，静态资源都加载出来了，#app也挂载上了，但是router-view那边没有显示。这是啥玩意？看了下vue-router的文档，发现了这个玩意--[应用基路径](https://router.vuejs.org/zh-cn/api/options.html#base)也就是说我们应该将router的配置修改一下

```js
let link = ''

if (window.location.host === 'm.kurisu.fm') {
  link = '/pay/my'
}

export default new Router({
  base: link, // 在此设定应用的基路径
  /*在window.location.host === 'm.kurisu.fm'的情况下，路由会将path为'/'的路径变为'/pay/my'，如果没有这个配置的话router还是会按'/'的路径去匹配路由，这样用户将会无法正常访问到页面*/
  routes: [
    {
      path: '/',
      name: 'JXGlobal',
      component: JXGlobal
    },
    {
      path: '/auth',
      component: Auth
    }
  ]
})
```

修改之后我们就可以愉快的进行下面的开发啦~而且还可以悄咪咪的在线上跑微信的各个功能[冷漠]

## 关于hash与history这两种模式的选择
说到了router就必然少不了对mode的讨论了，这里两种模式我都想说一说，因为实际开发中都<del>TM</del>用到了，最后我得出的结论是--**强烈建议使用hash路由**

<del>虽然最后我还是用了history的模式</del>由于项目的历史原因，此次开发还是使用了history的模式，具体原因会在下阐述。

### Hash模式路由
> vue-router 默认 hash 模式 —— 使用 URL 的 hash 来模拟一个完整的 URL，于是当 URL 改变时，页面不会重新加载。

使用这种模式的时候，大家会发现url会出现一个很奇葩的'#'，但是这个模式除了丑了点，真的没有啥问题。特别是在微信上面跑单页应用的时候，简直是良心之选。

但是为什么最后我没有采用这个模式呢？下面我还是想结合一下项目的实际，说说我选择的原因--

根本原因**支付授权目录限制是最多只能配置3个，超过3个将无法配置，并且路径深度不能大于2层**用了hash之后我们的url后面都会有'#/'这样就导致微信支付时匹配的目录有问题。经过测试，微信貌似之后根据'/'来判断路径层级的，所以如果使用了'#/'就代表天然少了一层路径，权衡之后还是放弃了使用hash模式

### History模式路由
关于History模式路由的[介绍](https://router.vuejs.org/zh-cn/essentials/history-mode.html)也已经烂大街了，它主要是依靠HTML5新增的API--history.pushState来实现的，具体实现就不在这里啰嗦了，日后争取出一篇router的详解。

由于种种原因（详情见上方Hash部分）我选择了history模式跑这个项目，在这期间也是遇到了一些坑的地方。

因为项目主要是跑在微信上的，总的来说坑的地方就是微信JSSDK对单页应用的兼容差强人意
-   在安卓与iOS上使用JSSDK会发现有些地方在两个系统内的表现会莫名的不太一样，比如对初始location的判断、支付时同一url会有不同的结果...
    -   iOS上微信不会根据history.pushState来更改当前的href这导致的问题可不少，复制链接是错误的、授权可能会不成功、支付会有问题；在安卓上则一点问题都没有。解决方案还是比较粗暴的，在iOS系统中，判定到需要和微信JSSDK交互的页面就强行刷新一下以保证交互的正常进行。

    -   测试支付的时候发现一个很有趣的现象，在iOS跑得妥妥的支付调起，到安卓就挂了。在安卓系统中，支付调起了，但是最终付款的弹窗却没出现。对比了两端的支付参数也是一模一样，问题出在哪呢？挣扎了许久，还是在网上找到了坑友们的回答--**安卓调起支付的url不允许'/'结尾，否则支付将会无法正常调起**

## 微信授权处理方式
由于router有两种模式，所以处理起来也有些许差异，不过总的来说还是一致的。

首先，我们最好有一个独立的页面(组件)来处理授权相关的事件，因为实际开发的情况是可能同时存在多种不同途径的授权，有一个独立页面有助于我们很好的与主业务解耦。

然后就是看看[微信的文档](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421140842)把授权的链接拼起来，这里需要注意的是redirect_uri必须要设定过白名单的域名，否则是没法正常发起授权的。

能正常的重定向到指定的url之后，恭喜你，你已经完成了授权处理的<del>1%</del>一大步啦。重定向的url会带有code(&state...)，我们只需要拿着这个code向后端请求登录数据即可。

在这一步，两种路由模式的处理方式就有些许不同了。

-   history模式下，我们可以直接通过$route.query来拿到相关的参数，然后可以继续后面的请求。这是没有将授权分离的做法，但是在这次的项目中，我还是分离出了一个单独的授权页面，这也给自己埋了不少坑。首先是router.push的用法，一开始测试的时候可以顺利的从微信授权的页面回来，但是回来就不动了，即不请求登录接口，也不重定向回原页面。后面发现是code被冲掉了，罪魁祸首就是<del>我自己</del>router.push，在push的时候没将参数带上。另外一个需要注意的是授权完成之后最好使用window.location.href来进行原页面的回跳，可以避开一些在iOS系统下jssdk的坑。

-   hash模式下，我们则不需要像history模式那样处处避坑，但是这里我们会发现无法用$route.query拿到想要的参数，那是因为从微信跳回来之后我们的url变成了这样-- **m.kurisu.fm/?code=1234#/** 发现问题了吗？'#/'加到了最后，导致$route没法拿到'#/'之前的code，这里只需要自己写个解析window.location.search的工具类就可以轻松解决啦。
```js
function urlQuery2Object (search) {
  let result = {}
  search.split('?')[1].split('&').forEach(function (part) {
    let item = part.split('=')
    result[item[0]] = decodeURIComponent(item[1])
  })
  return result
}
```

## 域名限制的情形下如何带着镣铐跳舞
由于发起支付的路径有所限制，活动页的二级页面甚至是主页都不能独占一层路由，这该怎么解决呢？最快捷的解决方案就是通过$route.query和watch对$route的监听来实现页面切换。

最后敲定的url是这样的**m.kurisu.fm/pay/my?:id&:sid**其中id是这次活动的id（可能会同时有多个活动使用同一套页面），sid则是每个活动下的子活动页面。
```
<div class="container__content" v-if="activity_init">
    <keep-alive>
        <jx-index :f_content="init_data" v-if="!$route.query.sid"></jx-index>
        <jx-subpage v-else></jx-subpage>
    </keep-alive>
</div>
```
在页面展示上我们只需判断页面是否存在sid就可以知道应该展示哪个template，然后对$route的监听则只需要在子活动页面的template中进行。
```js
watch: {
    //监听$route的变动
    '$route' (to, from) {
        if (to.query.sid && (to.query.sid !== from.query.sid)) {
            //对子活动页的缓存，若有缓存则优先使用缓存数据，并静默(无loading toast)请求新数据，优化页面体验
            if (window.sessionStorage.getItem(String(to.query.id) + String(to.query.sid))) {
                let data = window.sessionStorage.getItem(String(to.query.id) + String(to.query.sid))
                this.column = JSON.parse(data)
                this.page_init = true
                this.GetData(false)
            } else {
                this.GetData(true)
            }
        }
    }
}
```

## 总结
本系列文章主要还是记录实际开发过程中的一些解决方案，本篇主要是围绕在路由相关的问题以及微信在其中的兼容问题，希望能帮助大家和我规避日后的坑。

## 往期文章
-   [移动端自制音乐播放器--React](https://juejin.im/post/5a4e3af86fb9a01cba425d7d)
-   [移动端中踩过的关于日历&时间的坑](https://juejin.im/post/5a3f729c6fb9a044fb07f935)
-   [谈谈React--componentWillReceiveProps的使用](https://juejin.im/post/5a39de3d6fb9a045154405ec)

<img src="https://user-gold-cdn.xitu.io/2017/12/24/16087d7ac487f37c?w=375&h=524&f=png&s=118753" width=50% />
