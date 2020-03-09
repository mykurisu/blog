---
title: nestjs项目初探
categories:
-   博客正篇
tags:
-   Typescript
-   Node
-   Nest.js
date: 2020/01/20
---

>   最近在重构一个内部使用的配置平台，主要重构平台的后端部分，目标是使用nest框架替换原有的koa框架。跌跌撞撞了几天，把项目慢慢的重构好了，在重构的过程中慢慢体会到nest框架的优势。

本文将会介绍nest框架，以及为什么我们选择使用这个框架（关于迁移过程遇到的问题可以关注后续的文章）。

进入正题，**为什么要使用nest来重构原有的node项目呢？**

##   Nest的项目结构清晰明了

如果大家没接触过nest，可以结合[nest官网](https://nestjs.com/)以及[fast-nest 一个基于nest的快速启动项目](https://github.com/mykurisu/fast-nest)来构建一个初印象。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/nest/module.png" />

如图，这是官网示例项目的目录结构，可以发现它和我们传统的node项目有些不同，在目录结构我们就能看到模块化影子。先是主入口main.ts，后是主模块app.module.ts，最后是挂载在app中的cat模块。

这与我们之前接触的大部分项目都不太一样，我们之前接触到的node项目，有些是将controller层汇集到一个router文件中处理，有些是分开了多个路由文件最后再汇总。再说说service层，在我接触过的node项目中，有些是没有将service层分离出来的，在controller层就把所有的事情完成并返回。

与之相比，nest的项目结构就比较有意思了。cat模块是个独立的目录，与app.module同级，在这个目录中包含了与cat相关的controller、service甚至是接口约定，最后暴露出cat.module.ts，注入到app中，完成挂载。**在这种风格的目录结构中，我们可以清晰的看出整个项目的脉络，无论是初识项目或是项目迭代，都可能很容易的找到切入点。**

从**关注点分离**的角度来说，我们以往的node项目结构大都是根据技术点来分离我们的关注，与controller相关的放一块，与service相关的放一块，接口约定放一块等等。而nest项目则是从业务层面进行分离，服务于同一个业务的内容构成一个模块放到同一个目录中。要说这两种分离方案哪个更好，倒是比较难说，它们各有优劣，**但是我们一般的服务都是为各种业务提供支持的，为何不考虑专注于业务呢:)**

##  Nest对Typescript支持完备

Nest自身就是通过typescript开发的，所以我们初始化完一个nest项目后可以直接使用ts来进行后续的开发，不需要再做额外的js转ts操作。

由于Nest天生支持typescript这一特性，使得它对[装饰器](https://www.tslang.cn/docs/handbook/decorators.html)的使用易如反掌，我们只需要在tsconfig中加入experimentalDecorators项即可在项目中使用装饰器。

也许大家会疑惑，用装饰器就用呗，有啥好特地拿出来说？

这就要结合Nest自带的注解式开发说起：

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/nest/decorate.png" />

如图，我们可以通过装饰器来完成对请求参数的获取，例如我们想获取请求头中的某一项，我们可以这么写

```js
@Controller('/test')
export class TestController {

    @Post('/create')
    async createUser(
        @Headers('name') name: string,
        @Body('id') id: string
    ) {
        // ...
    }
    
}
```

在上述controller中，我们为 **/test/create** 路由下的方法获取了请求头中 **name** 的值，以及请求body中 **id** 的值，相较于未使用装饰器的写法有以下几点小优势。

-   代码量明显减少，利于代码整洁
-   service所需参数一目了然，如果只是想知道某个service需要什么参数，我们只需要在对应的controller中就能读到
-   自定义装饰器，用于获取自己拓展的请求参数

##  开源表现及项目兼容程度

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/nest/issue.png" />

进到Nest的github中不难发现，项目的commit提交数以及issue解决数都十分优秀。我们使用某个开源项目除了项目本身的稳定，一般也很注意这个项目的活跃程度，而活跃程度的评判标准我觉得commit以及issue都可以当做衡量的标准。

看到这，也许有些同学会问了，如果有一个现成的express项目，可以直接把Nest套上去吗？

答案是肯定的。Nest本身应该属于更上一层的抽象框架，在项目底层当然是可以使用express的，并且Nest还提供了配套的适配包 **@nestjs/platform-express** ，安装依赖后在初始化时将express对象传入Nest的启动器中，就可以使用express进行业务开发了。

##  尾声

关于Nest的初探到此已经将近完结，对笔者而言这个框架给我带来的更多是一种规整感。它赋予项目极其解耦且灵活的模块化能力，并不需要我们额外去规划一套约定，只要使用了它，我们的项目自然而然会变成模块化的项目。相信随着后续的使用，我可以总结更多关于Nest的使用干货，感谢大家的阅读。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/end.png" width=50% />
