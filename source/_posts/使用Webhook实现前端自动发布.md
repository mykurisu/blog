---
title: 使用Webhook实现前端自动发布
---

> 喜闻乐见的背景故事时间--承接[[prerender-spa-plugin]--微型Vue项目的静态化利器](https://juejin.im/post/5ab31b8cf265da239706c2df)，官网上线之后，就开始琢磨，每次改动都得上服务器部署一下，是不是有点麻烦了，是时候该整个自动化部署惹:)然后就开始自己挖坑自己填啦。

本文将会涉及的内容--

- Webhook是啥？什么时候该使用它？
- 该怎么利用Webhook解放我们的双手？
- 小结

## Webhook是啥？什么时候该使用它？

以我司为例，我们团队内部使用Gitlab作为代码仓库，所以以下内容都是在Gitlab中进行实践的，当然在Github上其实也是大同小异的。

Webhook顾名思义，其实就是一钩子。当我们在Gitlab上做出某些特定操作时，可以触发钩子，去进行一些我们事先设定好的脚本，以达到某些特定功能（例如--前端项目自动发布）。

也许熟悉Gitlab的同学会说了，这不就是CI吗，为什么好好的CI放着不用要去搞webhook呢？确实，CI也能完成一样的功能，但是也要结合业务实际状况而言。该项目并非公司核心项目，跑CI的runner需要有Gitlab控制权限的同事帮忙配置，本着不骚扰同事的原则，个人认为应该自己动手丰衣足食，既然有简单便捷的webhook为什么不用呢？

什么时候才应该去使用它呢？个人认为至少要有这几点：

- 项目不是一成不变的
- 你是项目的负责人
- 你有权限进入部署项目的服务器
- <del>你能挤得出时间来踩坑</del>

## 该怎么利用Webhook解放我们的双手？

<img src="https://user-gold-cdn.xitu.io/2018/3/25/1625dc023af783b9?w=1366&h=1106&f=png&s=196389" />

上图是Gitlab中有关webhook的配置页面，当我们成功进行了[Trigger]内部的某些或某些操作时，比如**Push events**--成功push了一次代码，无论是向哪个分支push，都可以触发hook......

之后Gitlab将会自动的替我们向URL中的链接发去POST请求，这里可以是脚本也可以是服务，只要能够成功接收来自Gitlab的请求即可。在本次实践中，我起了一个node服务来帮助完成自动化部署。

那么就有一个问题了，请求那个URL就会触发node服务中的部署流程，那么万一有人一直在玩那个接口，会不会把服务搞挂或是把网站搞挂呢？还真有可能...所以Gitlab给了我们设置Secret Token的机会，我们检测到请求有带有这段特殊的token才能认为本次请求是安全可接受的。

>  It will be sent with the request in the X-Gitlab-Token HTTP header.

这里需要注意一下的是，实际上我们并不会找到**X-Gitlab-Token**这个请求头，我们只会匹配到**x-gitlab-token**这个字段，别问我为什么知道的，大家注意避坑就好:)

下面讲讲在服务器上我们是怎么接收Gitlab的请求并且执行部署的--

```js
const exec = require('child_process').exec
const express = require('express')
const app = express()

let isLocking = false

app.post('/deploy', function (req, res) {
    let headers = req.headers
    let cmdStr = 'cd ... && git fetch origin && ...'
    if (!isLocking && headers['x-gitlab-token'] === 'xxx') {
        isLocking = true
        exec(cmdStr, function (err, stdout, stderr) {
            if (err) {
                // ...
                console.log('error:' + stderr);
            } else {
                // ...
                console.log(stdout)
                isLocking = false
            }
        })
    }
    // ......
})

app.listen(1234, '0.0.0.0', function () {
    console.log(`listening on port 1234`)
})
```

在项目部署的机器上，跑了这个简单的node服务，大意是当Gitlab POST一个请求过来时，我们进行鉴权，随后通过node去执行一段命令行语句，根据执行的结果调用不同的方法。这里有几个坑点--

### node服务搭建

贪图方便，依赖了一下express，当然我们是可以用http模块来完成这些操作的，另外监听端口的时候需要确认当前端口是否可被外网访问。

### node执行服务器命令行

我们可以使用node中'child_process'模块的exec方法来执行命令行语句，将定义好的命令行语句**字符串**传入exec方法作为第一个参数即可，后面一个参数则是执行结果的回调，依照业务需要设计即可。

### 发布鉴权&&发布状态

通过Gitlab请求中带来的token来鉴定本次请求是否能够触发下面的部署，并且需要设定一个"发布中"的状态，防止多次请求带来的各种无法预知的后果。

上面的代码只是最基本的发布服务，在里面可以做任何事，发布完成或失败的各种通知，甚至还可以对提交的代码进行检测，倘若不符合规范还可以拒绝本次自动部署......只要想踩坑，一切皆有可能~这样反映出webhook的灵活，只需对git做一个简单的操作，就可以推倒多米诺骨牌，完成一些意想不到的操作。

## 小结

本切图仔第一次踩这方面的坑，希望对萌新们有所帮助，也希望大大们轻拍。接下来，可能会有一个基于python的**自动部署完成后微信通知**脚本，敬请期待，当然也可能没有:)毕竟只是个切图仔，搬完砖就该睡了> <

<img src="https://user-gold-cdn.xitu.io/2017/12/24/16087d7ac487f37c?w=375&h=524&f=png&s=118753" width=50% />