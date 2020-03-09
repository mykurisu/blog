---
title: Fast-Nest -- 基于Nest.js的Node后端服务(Typescript)
categories:
-   博客正篇
tags:
-   Typescript
-   Node
-   Nest.js
date: 2020/01/15
---

>   久违的背景介绍:) 近期在做一个KOA内部管理系统的框架迁移，在迁移的过程中逐渐熟悉Nest.js的开发模式，对它的目录结构设计以及框架自带的装饰器支持颇有好感，固有将其整合成快速启动框架的想法。

本项目适用：

-   node简易应用初始化
-   node项目初学试验
-   typescript项目初学试验
-   有兴趣的童鞋亦可作为项目脚手架(项目内都是基础功能与结构，是可以作为简单的脚手架的)

##  食用前

-   可以先去项目内看看[Fast-Nest](https://github.com/mykurisu/fast-nest)
-   希望使用者有JavaScript基础，熟悉typescript更佳。
-   希望有简单node应用开发基础(可选)
-   希望能先了解Nest.js框架([传送门](https://github.com/nestjs/nest))

##  如何启动项目


```bash
git clone https://github.com/mykurisu/fast-nest.git

cd fase-nest

npm install

npm start
```

如代码所示，将本项目克隆至本地，安装好依赖即可启动服务，默认端口为**9999**，倘若需要调整请在根目录的**config.ts**中进行调整。

项目启动后，如果你没有启动本地的MongoDB会出现超时的报错，这时有两种方案：

①   将**share.module**中的mongo公共服务挂载删除

②   启动MongoDB，连接数据库的配置亦在**config.ts**内

##  这个项目内有什么料

-   <del>一套可运行的nest项目代码(废话)</del>
-   小而全的Node项目目录结构
-   按照核心、功能以及公共三个维度进行模块划分
-   核心模块
    -   内置格式化返回数据的拦截器
    -   内置格式化返回错误信息的过滤器
    -   内置简易日志中间件
-   公共模块
    -   内置MongoDB初始化服务
    -   内置腾讯云静态储存初始化服务
-   功能模块
    -   内置简易用户信息获取模块
    -   内置文件上传模块(支持上传到腾讯云)

##  这个项目将会怎么样

会持续维护。至于后续会接入什么模块，将视个人业余时间而定，初步设想会完善有用户模块的建设，争取能够初始化一个较为通用的用户模块。

##  最后

大家如果玩的舒服，希望能够star支持一下:)

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/end.png" width=50% />
