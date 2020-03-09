---
title: 记一次proto仓库的pre-commit开发
categories:
-   博客正篇
tags:
-   Git
date: 2020/02/01
---

>   背景介绍 -- 因业务需求，我们需要维护一个proto仓库用于前后端同学的数据结构定义，然而在仓库使用的过程中经常发生proto文件格式错误的情况，导致系统对应的页面会发生崩溃。所以，我们打算在proto仓库中启用pre-commit检验的方案，来降低这类事件发生的概率。

##  什么是pre-commit?

pre-commit大家可能不是熟悉，但是我们基本都享受过它带来的便利。pre-commit是Git自带有的hook，它可以在我们commit之前先对提交的内容进行遍历、检测亦或是其他操作。

举一个最常见的例子，在开发JS项目时，如果我们用的是社区提供的脚手架一般会都配有eslint，然后我们在提交代码时可能会因为某些代码风格不符合eslint的规范而提交失败。

这就是利用pre-commit可以办到的事，它可以确保提交到仓库内的代码风格一致。查看Git的文档可以知道，在Git不仅提供commit阶段的钩子，还有prepare-commit-msg、post-commit、pre-rebase...几乎在提交代码到推送代码的每个阶段都留有相应的钩子，我们可以按需进行触发。

##  如何触发pre-commit(如何触发Git Hook)

在每个git仓库根目录下都有一个隐藏的.git文件夹，内部有一个名为"hooks"的文件夹，顾名思义这个文件夹是用来存放各类钩子的。进去之后会发现，有很多.sample为后缀的文件，这些是git自带的钩子模板，里面内置了一些钩子的案例供大家参考，当我们编写好属于自己的钩子方法后，只需要将.sample后缀去掉即可。

> 提示：任何正确命名的可执行脚本都可以正常使用 —— 你可以用 Ruby 或 Python，或其它语言编写它们。  -- [自定义 Git - Git 钩子](https://www.git-scm.com/book/zh/v2/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-Git-%E9%92%A9%E5%AD%90)

![](https://user-gold-cdn.xitu.io/2020/1/11/16f94ff2db3f5014?w=1316&h=796&f=png&s=98024)

##  怎么编写一个可用的hook脚本

-   确定目标文件

在执行验证之前，首先应该需要确认验证的目标文件(被修改或新增的文件)。关于修改与新增的文件列表，我们可以通过git的diff方法来查出。

```bash
#!/bin/bash
STAGE_FILES=$(git diff --cached --name-only --diff-filter=ACM -- '*.proto')

if [[ $STAGE_FILES = "" ]] ; then
    echo "无需要检查的proto文件"   
    exit 0
fi
```

-   文件内容读取

如果改动的文件中有proto文件，则会进入下一个流程。我们先将检测到的改动文件打印出来，供提交者确认，随后进行验证流程。

```bash
echo "$STAGE_FILES 待检查"

for proto in $STAGE_FILES;do
    PROTO_TEXT=$(cat $proto)
    echo "$proto 检查中..."
done;

exit 0
```

-   文件内容推送至服务端

```bash
request=`curl -s -H "Content-type: application/json" -H "X-Proto-Commit: precommit" -X POST -d '{"proto_text":"'$PROTO_TEXT'","name":"'$proto'"}' http://localhost:7777/api/proto/preCommit`
```

-   服务端返回处理

```bash
STATUS=`echo -e "'$request'" | grep "error"`
if [[ $STATUS = "" ]];then
    echo "$proto 检查完成..."   
else
    echo $STATUS
    exit 1
fi
```

我们将会以第一步中得到的变更文件列表为循环体，将每个变更的proto文件中的内容都读取出来，并通过curl的方式将其发送到，负责提供检测能力的服务端，等待服务端返回检测结果。

看起来，一个简易的pre-commit-proto能力就完成了，但是在实际使用的时候发现，有某些proto文件提交到检测端必定会报错，就算这个文件本身并无任何错误，服务端仍然会返回错误。

经过多次测试，最后定位到是编码格式的问题，由于curl自带的encode的功能对中文的支持有限，所以我们决定将整个proto文件上传至服务端进行检测，以保证检测内容的绝对正确。

##  如何通过curl上传proto文件并验证?

由于直接读取推送文件内容的方式无法达到期望的效果，我们尝试将整个proto文件上传到服务端进行解析。

```bash
- request=`curl -s -H "Content-type: application/json" -H "X-Proto-Commit: precommit" -X POST -d '{"proto_text":"'$PROTO_TEXT'","name":"'$proto'"}' http://localhost:7777/api/proto/preCommit`
+ request=`curl -k -s -F "file=@$proto" http://localhost:7777/api/proto/preCommit`
```

我们只需将request变量后面的执行语句微调，即可将proto文件上传到指定地方。其中需要注意几个参数：

-   **-F**: 用来向服务器上传二进制文件。
-   **-k**: 指定跳过 SSL 检测。(可选，由于示例中不是https接口，所以加上了)
-   **-s**: 静默模式，不输出任何东西(可选，看自身需求)

将文件顺利上传之后就完成了代码提交端的任务了，剩下就是看服务端检测的结果。如果检测通过则退出pre-commit继续提交动作，反之则提示proto文件的错误的部分，并中断代码提交。

##  如何在一个新的克隆中启用pre-commit

到这，也许大家心中仍有个大大的问号。通篇的大前提是修改.git文件夹内的内容，但是既然是git项目，就肯定代表着除自己以外的人也能运行项目，这种情况下pre-commit就失效了。因为.git文件夹是属于本地的，没法存储到版本库中，我们要用别的方法将写好的pre-commit脚步装载到其本地的.git/hooks中。

```bash
#!/bin/bash
cp pre-commit .git/hooks/pre-commit
chmod 777 .git/hooks/pre-commit
echo 'protobuf预检初始化完成'
exit 0
```

我们逐行读一下上面的代码块，首先是声明bash脚本。紧接着将项目根目录的'pre-commit'文件复制到指定路径'.git/hooks/pre-commit'。

这还没完，如果没有修改文件权限，我们会发现git commit的时候仍是没有触发钩子，所以将'.git/hooks/pre-commit'修改成修改为777（可读可写可执行）。

##  后记

至此，一个简单的pre-commit钩子的开发与触发就完成了，git提供的其他钩子用法也大同小异，我们只需要选择适合需求的钩子，然后将开发好的脚本替换到hooks文件夹内即可。下面把完整的代码附上~

```bash
init.sh

#!/bin/bash
cp pre-commit .git/hooks/pre-commit
chmod 777 .git/hooks/pre-commit
echo 'protobuf预检初始化完成'
exit 0

pre-commit.sh

#!/bin/bash
STAGE_FILES=$(git diff --cached --name-only --diff-filter=ACM -- '*.proto')

if [[ $STAGE_FILES = "" ]] ; then
    echo "无需要检查的proto文件"   
    exit 0
fi

echo "$STAGE_FILES 待检查"

for proto in $STAGE_FILES;do
    PROTO_TEXT=$(cat $proto)
    echo "$proto 检查中..."
    request=`curl -k -s -F "file=@$proto" http://localhost:7777/api/proto/preCommit`
    CODE=`echo -e "'$request'" | grep "code" | awk -F ":" '{print $2}' | awk -F "," '{print $1}'`
    if [[ $CODE > 0 ]];then
        echo "预查服务故障，请重试"
        exit 1
    fi
    STATUS=`echo -e "'$request'" | grep "error"`
    if [[ $STATUS = "" ]];then
        echo "$proto 检查完成"
    else
        echo $STATUS
        exit 1
    fi
done;

exit 0
```

<img src="https://user-gold-cdn.xitu.io/2017/12/24/16087d7ac487f37c?w=375&h=524&f=png&s=118753" width=50% />
