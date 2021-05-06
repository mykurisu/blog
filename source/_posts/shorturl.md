---
title: 基于Node框架Nest实现的短链接服务
categories:
-   博客正篇
tags:
-   Node
-   Nest
-   Typescript
date: 2021/4/26
---

> 日常生活中能见到各种奇怪的短链接，每次点击跳转的时候，笔者都会觉得神奇，这短链是怎么将用户引导到正确页面的呢？

## 短链原理

**短链的原理就是以短博长**，那么这个短的字符串怎么才能变成一长串链接呢？难道是靠某些神奇的加密算法？并不是，我们只需要依赖key/value的映射关系就能轻松实现这个看似神奇的**以短博长**。

用一张图，大家就能清晰的看到我们访问短链的整个过程了。

![未命名文件.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29a951b5657d4f7b867e2d34dc536a0a~tplv-k3u1fbpfcp-watermark.image)

首先，我们会有一个长链接，通过短链服务的处理，通常会输出一个只有一层目录的URL，然后我们可以将获取的URL进行分发。

然后就到了用户侧，用户点击短链之后，先到达的并不是目标页面，而是短链服务。

短链服务会截取链接上的pathname，并将其当做key，到映射关系中查找对应的value。

如果查到不到对应的value，则表示这个短链不存在或者已失效；如果查询成功，则会由短链服务直接302到value中的目标链接，完成一次短链访问。

## 具体实现

> 原料: [Fast-Nest](https://github.com/mykurisu/fast-nest)脚手架、Redis

整个实现分拆成3个部分：

### ① 接收长链接

```typescript
@Post('/createUrl')
async createUrl(
    @Body('url') url: string,
    @Body('type') type: string,
) {
    const shortUrl = await this.shorturlService.createUrl(url, type);
    return {
        shortUrl,
    };
}
```

在服务中创建一个`createUrl`接口，接收`url`已经`type`字段，并将其传入`shorturlService`中，等待短链接生成然后输出。

### ② 生成shortKey

```typescript
async createUrl(url: string, type: string = 'normal') {
    const urlKey = await this.handleUrlKey();
    const dataStr = JSON.stringify({
        url,
        type
    });
    await this.client.set(urlKey, dataStr, type === 'permanent' ? -1 : 300);
    return `${Config.defaultHost}/${urlKey}`;
}

private async handleUrlKey(count?: number): Promise<string> {
    const _count = count || 1;
    const maxCount = Config.maxRetryTimes;
    if (_count >= maxCount) throw new HttpException('超过重试次数，请重新生成链接', HttpStatus.INTERNAL_SERVER_ERROR);
    const urlKey: string = Math.random().toString(36).slice(-4);
    const _url = await this.client.get(urlKey);
    if (_url) {
        return await this.handleUrlKey(_count + 1);
    }
    return urlKey;
}
```

首先通过`Math.random().toString(36).slice(-4)`获取4位随机字符串，这个将会作为短链的pathname。

在进行映射之前，我们需要对其进行唯一性判断，虽然出现的可能性不大，但是还是需要防范短链覆盖这类的问题。本服务的解决方案是重试生成，如果短链值不幸重复时将会进入重试分支，服务将会内置可重试次数，如果重试的次数超过配置的字数，本次转换将会返回失败。

除了`url`，`createUrl`方法还接受一个`type`字段，这里涉及特殊短链的特性。我们短链有三种模式：

- normal - 普通短链接，将会在规定时间内失效
- once - 一次性短链接，将会在规定时间内失效，被访问后自动失效
- permanent - 长期短链接，不会自动失效，只接受手动删除

生成`urlKey`之后，将会与`type`一起转成字符串储存到redis中，并输出拼接好的短链接。

### ③ 接收短链接并完成目标重定向

```typescript
@Get('/:key')
@Redirect(Config.defaultIndex, 302)
async getUrl(
        @Param('key') key: string,
    ) {
    if (key) {
        const url = await this.shorturlService.getUrl(key);
        return {
            url
        }
    }
}

// this.shorturlService.getUrl
async getUrl(k: string) {
    const dataStr = await this.client.get(k);
    if (!dataStr) return;
    const { url, type } = JSON.parse(dataStr);
    if (type === 'once') {
        await this.client.del(k);
    }
    return url;
}
```

用户侧会获得一个类似`http://localhost:8000/s/ku6a`的链接，点击之后相当于是给短链接服务发送了一个GET请求。

服务接收到请求之后获取链接中key字段的值，也就是`ku6a`这个字符串，利用它查找Redis中的映射关系。

这里有两个分支，一个是在Redis中无法查询到相关的值，服务则认为短链接已经失效会直接return，因为`getUrl`返回了空值，重定向装饰器会将本次请求重定向到默认的目标链接中。

如果在Redis中顺利查到相关的值，则会读取其中的`url`和`type`字段，如果type为once则代表这个是一次性链接，会主动触发删除方法，最终都会返回目标链接。

## 额外功能

### 利用日志系统输出报表

使用短链接时，大概率都会需要相关的数据统计，怎么样在不使用数据库的前提下进行数据统计呢？

在本服务中，我们可以通过对落地日志文件的扫描，完成当日短链访问的报表。

在生成短链接的时候加上**urlID**字段进行统计区分并主动输出日志，如下：

```ts
async createUrl(url: string, type: string = 'normal') {
    const urlKey = await this.handleUrlKey();
    const urlID = UUID.genV4().toString();
    const dataStr = JSON.stringify({
        urlID,
        url,
        type
    });
    this.myLogger.log(`createUrl**${urlID}`, 'createUrl', false);
    await this.client.set(urlKey, dataStr, type === 'permanent' ? -1 : 300);
    return `${Config.defaultHost}/${urlKey}`;
}
```

然后在用户点击短链接时获取该短链接的**urlID**字段，并主动输出日志，如下：

```ts
async getUrl(k: string) {
    const dataStr = await this.client.get(k);
    if (!dataStr) return;
    const { url, type, urlID } = JSON.parse(dataStr);
    if (type === 'once') {
        await this.client.del(k);
    }
    this.myLogger.log(`getUrl**${urlID}`, 'getUrl', false);
    return url;
}
```

这么一来我们将能够在服务的logs目录中获得类似这样的日志：

```
2021-04-25 22:31:03.306	INFO	[11999]	[-]	createUrl**3f577625-474a-4e30-9933-e469ce3b0dcf
2021-04-25 22:31:38.323	INFO	[11999]	[-]	getUrl**3f577625-474a-4e30-9933-e469ce3b0dcf
2021-04-25 22:31:39.399	INFO	[11999]	[-]	getUrl**3f577625-474a-4e30-9933-e469ce3b0dcf
2021-04-25 22:31:40.281	INFO	[11999]	[-]	getUrl**3f577625-474a-4e30-9933-e469ce3b0dcf
2021-04-25 22:31:40.997	INFO	[11999]	[-]	getUrl**3f577625-474a-4e30-9933-e469ce3b0dcf
2021-04-25 22:31:41.977	INFO	[11999]	[-]	getUrl**3f577625-474a-4e30-9933-e469ce3b0dcf
2021-04-25 22:31:42.870	INFO	[11999]	[-]	getUrl**3f577625-474a-4e30-9933-e469ce3b0dcf
2021-04-25 22:31:43.716	INFO	[11999]	[-]	getUrl**3f577625-474a-4e30-9933-e469ce3b0dcf
2021-04-25 22:31:44.614	INFO	[11999]	[-]	getUrl**3f577625-474a-4e30-9933-e469ce3b0dcf
```

之后我们只需要以`createUrl`的日志为索引，对`getUrl`类型的日志进行计数，即可完成链接与点击数的报表，如果还需要其他维度的报表只需要在输出日志的时候带上即可，或者修改日志中间件中的日志范式。

## 使用方式

根据上述的流程，笔者写了一个比较简易的短链服务，大家可以开箱即用。

[shorturl(欢迎大家Star⭐️⭐️)](https://github.com/mykurisu/shorturl)

**具体启动方式**

> 首先请确保有可用的redis，否则无法顺利启动服务。

```
git clone https://github.com/mykurisu/shorturl.git

cd shorturl

npm install

npm start
```

**可用配置修改**

与短链相关的配置收束在根目录的`config.ts`中。

```js
serverConfig: {
    port: 8000,
},
redis: {
    port: 6379,
    host: '0.0.0.0',
    db: 0,
},
cacheType: 'redis',
defaultHost: 'http://localhost:8000/s',
defaultIndex: 'http://localhost:8000/defaultIndex',
```

| 配置 | 默认值 | 配置用途 |
| ---- | ---- | ---- |
| serverConfig.port | 8000 | 服务启动端口 |
| redis.port | 6379 | redis端口 |
| redis.host | 0.0.0.0 | redis服务地址 |
| redis.db | 0 | redis具体储存库表 |
| cacheType | redis | 短链储存模式，接受memory/redis |
| maxRetryTimes | 5 | 生成短链接最大重试次数 |
| defaultHost | http://localhost:8000/s | 短链接前缀 |
| defaultIndex | http://localhost:8000/defaultIndex | 短链接失效后重定向地址 |

**内置接口**

| 接口路由 | 请求方式 | 接口参数 | 接口用途 |
| ---- | ---- | ---- | ---- |
| /s/createUrl | POST | url: string, type?: string | 短链接生成接口 |
| /s/deleteUrl | POST | k: string | 删除短链接接口 |
| /s/:key | GET | none | 目标链接获取 |

## 拓展

### ① 储存降级策略

[shorturl](https://github.com/mykurisu/shorturl)是有本地储存方案的，也就是说我们是可以监听Redis的状态，如果断开连接时就临时将数据储存到内存中，以达到服务降级的目的。当然我们也可以直接使用内存来储存短链内容，在`config.ts`配置中可以进行更改。

### ② 不仅仅是短链接服务

让我们脱离短链接这个束缚，其实[shorturl](https://github.com/mykurisu/shorturl)本身已经是一个微型存储服务了，我们完全可以进行二次开发，输出更多的模块以支撑更多样的业务。

## 小结

整个短链接服务其实非常简单，麻烦的是服务的搭建，也就是迈出的第一步。笔者也是在无数次**最初一步**中挣扎，最终积累了[fast-nest](https://github.com/mykurisu/fast-nest)这么一个脚手架，希望能帮助到有同样境遇的同学。

另外，附上本文的服务源码 -- [shorturl](https://github.com/mykurisu/shorturl)(欢迎大家Star⭐️⭐️)