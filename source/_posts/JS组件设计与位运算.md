---
title: 前端组件设计--位运算的妙用
categories:
-   博客正篇
tags:
-   Javascript
-   设计模式
date: 2018/10/24
---

>   Hello好久不见。跳槽之后一直没什么时间总结记录这段时间的见闻或实践，好不容易挤出点时间，今天想记录一下最近的一个组件设计&开发历程，**该组件的开发环境是小程序**，但是我认为这个思维是通用的~

本文可能涉及内容--

-   需求简介
-   位运算与表单组件的结合
-   组件设计小结

##  需求简介

我们大概需要完成这样一个表单组件↓↓

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/component-01.png" width="50%" />

由图大致可以判断，我们应该将此组件拆成两块，一个是tab选项区，一个是下方的附加选项区。

**为什么需要分拆？**

-   最主要的原因是项目自身原因，之前就有tab的组件，我们可以直接引用。

-   当然，就算我们自身项目没有这个组件，我也是建议拆分开来，因为在表单的开发中，这种tab选择的形式可以说太常见了，拆分开来肯定是有益的，并且它的功能十分单一，很适合作为一个独立组件。

**如何用数据结构描述组件？**

静态结构大概明了，那么我们应该怎么用数据来描述这个组件呢？接下来就来考虑一下组件的数据结构，开始之前先明确几个前提--

>   「tab数目不定、tab与附加项内容有联动、附加项数目不定」

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/component-02.png" width="50%" />

上图是我在第一版中构思的数据结构，制定该规则的原因如下--

-   由于tab数目不定，所以配置成数组由前端进行列表渲染最为合适(附加项的设定也同理)。

-   因为**tab与附加项内容有联动**，所以我将附加项直接与tab内联，切换tab的时候直接切换相应的附加项数据即可。

-   由于存在多个tab和多个附加项，所以必定会有组合排列的问题。在第一版方案中，我采用的是map的处理方式，将选中的value值拼接到一起，然后在valueMap中获取最终的value值。(不同业务不同方式，如果不需要处理组合问题，可直接输出所选的所有value值)

接下来逻辑就很顺畅啦，我们只需要获取用户最终选择的value拼接值就可以在valueMap中得到目标数据。但是这个方案是有瑕疵的，我忽略了**配置复杂度**，这样在前端代码中可以很轻松的处理逻辑。但是会为后续的配置带来无穷后患，随着方案数量的增加，组合的情况会飞速增长。为了避免这个坑，我需要对valueMap做些优化。

**valueMap的优化方案**

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/component-03.png" width="50%" />

如图中代码所示，摈弃原来键值一一对应的方式，采用根据tab项带出value可选范围的模式。然后通过对选中选项的处理得出value可选范围数组的下标。这样在配置起来会比上一种方案好一点，相对前端的逻辑会稍微复杂一点，但前端的代码只需要我们写一次，而配置则可能是无数次。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/component-04.png" width="50%" />

新方案的前端逻辑大概是这样，等等...怎么看起来这么别扭？这段逻辑竟然完全和业务搅在一起了，万一新增附加项目岂不是爆炸了？万一附加方案间也出现了组合情况不也爆炸了:(

那岂不是代表这种方案更不靠谱？**我们是不是可以试试将其与业务解耦？**

##  位运算与表单组件的结合

**JavaScript位运算简介**

位运算就是直接对整数在内存中的二进制位进行操作，与是否处于JavaScript环境并没有什么关联，但我们可以借助一下这种位运算的思维。[有关JavaScript位运算的相关文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Bitwise_Operators)，在本文中我们主要会使用按位移动的操作符来达到我们的目的。

**如何将二者结合一起？**

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/component-05.png" width="50%" />

直接上代码，我们看图说话--

在新的逻辑中，我们已经看不到对addon项value的依赖，取而代之是二进制形式的位置下标。假设我们有这样一个数组：

`
let A = [A-value-0, A-value-1, A-value-2, A-value-3]
`

分别对这种四种选择情况的取值--

| 选择情况 | 值 | 数组下标 | 二进制下标 |
| ------ | ------ | ------ | ------ |
| 不选 | A-value-0 | A[0] | 00 |
| 选择第一个 | A-value-1 | A[1] | 01 |
| 选择第二个 | A-value-2 | A[2] | 10 |
| 全选 | A-value-3 | A[3] | 11 |

这样就很简单明了了，看二进制下标那一列不难发现，其中0表示未选，1表示已选，并且可通过1的位置，知道已选或未选的项目是哪一项。

再看看数组下标与二进制下标的关系，将二进制下标转为十进制后，正好就是所需的数组下标。

回头看看图中的代码，上述的取值思路正是forEach内部所干的事情：
>       遍历addon数组中被checked的项，根据其index得出属于该项的二进制下标，再将遍历而来的所有二进制下标做OR运算得出最终的二进制下标后转为真正的数组下标，从而得出最终的value值。
通过这种方案，我们可以在完全不关心addonList中的项个数或是每项值的情况下，得出我们想要的组合以及取到该组合对应的值，在我们配置项目的时候只需要留心每个value的顺序即可，无论配置多少项对前端的逻辑也不会产生影响。

**将此案例中的addonList个数更改也是一样可行的，比如有3项的情况：**

| 选择情况 | 值 | 数组下标 | 二进制下标 |
| ------ | ------ | ------ | ------ |
| 不选 | A-value-0 | A[0] | 000 |
| 选一 | A-value-1 | A[1] | 001 |
| 选二 | A-value-2 | A[2] | 010 |
| 一、二 | A-value-3 | A[3] | 011 |
| 选三 | A-value-4 | A[4] | 100 |
| 一、三 | A-value-5 | A[5] | 101 |
| 二、三 | A-value-6 | A[6] | 110 |
| 全选 | A-value-7 | A[7] | 111 |

更多情况就不再赘述，大家可以在console里面自己尝试一下~

##  组件设计总结

通过这次的表单组件设计，对前端的组件设计有一点点思考：

-   独立
    -   组件功能单一，radio就应该负责radio选择，不要加上一些与业务相关的逻辑或文案。
    -   组件尽量不要过于复杂，既然分拆了组件就不要想着一个组件能完成所有的事情。
-   可复用
    -   在设计组件的时候除了满足当前的需求，最好还要考虑其通用性。倘若发现它不具有通用性，就不如写在业务里，不需要单独抽离。
    -   当然，在设计组件时不可能要求使用组件的场景要百分百一致，让组件的适应、拓展能力变强，有助于组件的健壮。
-   中立
    -   尽可能的少于业务耦合--保证组件中立。
    -   这一点看看就好，一般都是骨感的现实:)
-   易用
    -   尽量能做到使用者只需要考虑输入以及输出。

<img src="https://blog-1252307419.cos.ap-beijing.myqcloud.com/end.png" width=50% />
