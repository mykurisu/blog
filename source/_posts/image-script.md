---
title: 借助Node实现博客素材自动上传
categories:
-   博客正篇
tags:
-   JavaScript
-   Node
date: 2020/9/22
---

> 相信大家在本地用Markdown写博客的时候都遇到过这个情况：文章写好了，但是图片都是本地的引用路径，不能直接copy到线上博客，需要自行手动上传。

饱受手动上传的痛苦之后，本搬砖工决定写个小脚本解放双手。本脚本依赖腾讯云的静态存储，无论用的是哪种静态储存，本质都是一样的，就是运用的API可能有细微的差别。

下面，给大家介绍一下自动上传图片素材的流程：

**下文均已此目录结构为基础↓↓**

> 每篇博客都有独立的文件夹，文件夹里面放博客正文以及用到的素材。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/4f51672fa003624a55edf683021bc5da.png" />

## ① 获取Markdown文件

```js
const fs = require('fs')
const path = require('path')
// 获取指令参数
const args = process.argv
const file = args.pop();
const mdPath = path.join(__dirname, `./${file}`)
const folderPath = mdPath.split('/').slice(0, -1).join('/')
const content = fs.readFileSync(mdPath, { encoding: 'utf-8' })
```

这个阶段主要用到fs、path模块，首先通过**process.argv**获取指令中的参数，从上述代码可以看出这个脚本默认指令的最后一个传参是Markdown的路径及名称。

由于我每篇博客都有其独立的文件夹，所以需要通过**split**来获取博客所处文件夹的路径。最后再通过**readFile**方法来读取Markdown中的具体内容。

然后就该进入解析Markdown的环节了。

## ② 解析Markdown文件

解析Markdown的插件有不少，我选了之前一直在用的[marked](https://github.com/markedjs/marked)，它比较轻便，适合这种小项目使用。

```js
const marked = require('marked')
const html = marked(content)
```

在解析Markdown的过程中，有一个问题困扰着我 -- 该什么时候将图片过滤出来呢？

为此我想了三个方案：

### ① 通过正则匹配，把img标签以及'\!\[\]\(\)'写法的内容匹配出来

### ② 在解析过程中将图片相关的token过滤出来

其中方案①很轻松就被②打败了，marked的解析确实支持将Markdown解析成一组组token，并输出。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/1c65b0972d135458576a95b778fafdde.png" />

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/6a34957908cb8a88e5af6c4f1ad21cd6.png" />

但是在实际使用中发现，img标签的图片写法与Markdown原生的图片写法，解析出来的token并不一致，一个是HTML类型，一个是Paragraph类型，并且Paragraph类型往往伴随着无尽的嵌套。这样处理起来也不是那么的舒服。

那么我们能否像在浏览器一样，通过DOM去识别图片并取出图片的链接呢？

虽然我们是node环境，但是我们确实可以操作DOM。

下面就讲一下最终的实现方案。

## ③ 利用JSDOM获取图片路径

在网上搜索一下，不难看到[jsdom](https://github.com/jsdom/jsdom)的身影，全世界的前端同学都一样，想在node环境中也能操作一下熟悉的dom。

```js
const jsdom = require('jsdom')
const { JSDOM } = jsdom
const dom = new JSDOM(html)
const document = dom.window.document
const imgList = document.querySelectorAll('img')
```

将我们上一步解析出来的HTML字符串转化成DOM模式，再通过**querySelectorAll**就能简单便捷的获取文章内所有的图片节点。

随后再对每个图片进行上传操作，即可完成我们的自动上传功能了。

```js
imgList.forEach((img) => {
    const src = img.src || ''
    if (!src || /http|myqcloud/.test(src)) return
    const imgFullSrc = path.join(folderPath, src)
    const imgName = path.basename(imgFullSrc)
    const imgExt = path.extname(imgName)
    const data = fs.readFileSync(imgFullSrc)
    const md5Name = `${CryptoJS.MD5(data.toString('base64'))}${imgExt}`
    promises.push(uploader(md5Name, data, src))
})
```

为了避免图片重复处理，在脚本里简单的加上了路径判断，条件是**http**以及**myqcloud**，如果图片的src里面存在这些字样则不会继续处理。

为了避免图片路径重复，脚本将图片的MD5值作为了上传的文件名，另外这样做也方便图片的版本替换，旧素材也不用担心会被覆盖或者丢失。

处理完之后就应该正式进入上传流程了，可以看到**imgList**每项的处理最后都会向一个**promises**数组里面push一个**uploader**方法，这个**uploader**就是我们的上传主流程。

## ④ 腾讯云初始化

```js
const COS = require('cos-nodejs-sdk-v5')
const cos = new COS({ SecretId, SecretKey });
```

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/619f31e8f098c0c40fce38442e5ce0b2.png" />

在谈论上传操作之前，我们得先将腾讯云的sdk初始化好。整个初始化十分简单，只用将我们控制台中查看到的SecretId、SecretKey输入即可完成初始化。

## ⑤ 图片上传

初始化完sdk之后，通过cos的putObject方法，就可以将图片上传到制定的bucket中。因为图片是批量上传的，所以需要改造一下sdk的putObject方法，将其改为promise的模式，为后续的promise.all调用做准备。

```js
// ......
function uploader(fileName, fileBuffer, oldSrc) {
    return new Promise((resolve) => {
        cos.putObject({
            Bucket,
            Region,
            Key: fileName,
            StorageClass: 'STANDARD',
            Body: fileBuffer,
        }, function(err, data) {
            if (!err && data) {
                resolve({ oldSrc, newSrc: `https://${data.Location}` })
            }
            resolve(null)
        });
    })
}
```

为了图方便，就没有做异常分支的处理了，就算出错了也是按resolve处理，不过可以再后续的回调事件中进行甄别，本文就不多赘述。

上传成功后，腾讯云那边会返回文件的访问路径**data.Location**，但是这个路径是不带协议的，所以在输出的时候我手动给它加上了协议，并且附上了图片在Markdown内的路径，为下一步替换处理做准备。

```js
Promise.all(promises).then((res) => {
    let newContent = content
    res.forEach((item) => {
        if (item && item.newSrc) {
            newContent = newContent.replace(item.oldSrc, item.newSrc)
        }
    })
    const mdName = path.basename(mdPath).split('.')[0]
    fs.writeFileSync(mdPath.replace(mdName, `${mdName}-${Date.now()}`), newContent)
})
```

批量上传完成之后，将会进入路径替换环节，因为在resolve的时候我将图片的原路径也一并输出了，所以在处理promise的输出时，只需要将**oldSrc**替换为**newSrc**即可，随后输出新的Markdown文件。此处亦可覆盖源文件，因为脚本内做了图片路径甄别，所以不用担心重复上传的问题。

## 小结

至此，一个自动检测博客内容并上传图片的小脚本就完成了，我们只需要输入：

```
node ./image.js image-script/content.md
```

即可开始上传流程，并且会输出替换图片链接后的Markdown文档。

除了可以不用手动上传图片以外，将图片上传到腾讯云还可以方便我们将博客发布至微信公众号上，因为在公众号中直接使用腾讯云的静态资源是不会被拦截的，我们不需要再在公众号编辑界面重新传图并校对。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/end.png" />