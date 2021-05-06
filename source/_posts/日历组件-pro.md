---
title: 日历组件-PRO
categories:
-   博客正篇
tags:
-   Vue
-   Javascript
date: 2021/4/01
---

> 这段时间忙着晋级的事情，都没心思总结博客了(借口)。本次想总结的是一个日历组件的迭代历程，做一个能给大家带来便利的日历组件是我一直以来的小目标，希望这次的短文能给大家带来些许帮助。

在年轻的时候写过这样一篇文章[基于Vue开发一个日历组件](https://www.mykurisu.xyz/2018/02/22/%E7%A7%BB%E5%8A%A8%E7%AB%AF%E8%87%AA%E5%88%B6%E6%97%A5%E5%8E%86/)，回头看看觉得还是比较粗糙，所以抽空优化迭代了一下。

## 逻辑优化

> 本次优化的重点是**抽离**及**易读**

### 抽离

原本是以vue为实现基础，开发的日历组件，但是生成日期的那部分逻辑并不与实现的框架相关，这一块逻辑是完全可以抽离出来的。于是就有了下面的改动：

```js
// Calendar.vue
import { getAllDaysForYear } from './calendar';
export default {
    // ...
    mounted() {
        getAllDaysForYear(2021);
    }
}
```

之前混在vue模板中的日期生成逻辑都被抽离出去了，模板只剩下界面交互的逻辑。

### 易读

> 每个变量都有其存在的意义，如果能很快弄清每个变量的用途，那么很容易就能读懂一段抽象逻辑。

在进行功能点开发之前，我总是习惯先梳理出两个切入方向

> - 我明确知道的是什么？
> - 我需要知道的是什么？

回到我们的功能中，我希望现在正在开发的函数可以输出制定年份的所有日期排布。

在这里面，我明确知道的是**每个月的天数**，而我需要知道的是**每个月的日期排布**。

那么我们就可以得出下面的逻辑：

```js
function getAllDaysForYear(year) {
    /**
     * monthData 每月数据 用于最后输出
     * daysInMonth 每个月的天数
     */
    const daysInMonth = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];

    // 对闰年二月天数特殊处理
    if ((year % 4 === 0 && year % 100 !== 0) || year % 400 === 0) {
        daysInMonth[1] = 29;
    }
    const monthData = new Array(12).fill(null);
}
```

问题又来了，`monthData`里面确实有12个数组，但是里面全是null啊！就算知道了每个月有多少天，好像也排不出整个日期顺序。

这里原理其实也很简单，就不卖关子了。我们只需要知道**每个月第一天是星期几**，那么整个月的日期排布自然就出来了。

```js
function getAllDaysForYear(year) {
    /**
     * monthData 每月数据 用于最后输出
     * daysInMonth 每个月的天数
     * specialDayInMonth 每个月第一天和最后一天的星期
     */
    const daysInMonth = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];

    // 对闰年二月天数特殊处理
    if ((year % 4 === 0 && year % 100 !== 0) || year % 400 === 0) {
        daysInMonth[1] = 29;
    }
    const monthData = new Array(12).fill(null);
    
    const specialDayInMonth = monthData.slice(0).map((m, i) => {
        return [new Date(year, i, 1).getDay()];
    });
    
    return monthData.map((m, i) => {
        return normalDaysCreator(daysInMonth[i]);
    });
}

function normalDaysCreator(days) {
    const normalDays = [];
    for (let i = 0; i < days; i++) {
        let obj = {
            content: i + 1,
            type: "normal",
        };

        normalDays.push(obj);
    }
    return normalDays;
}

export { getAllDaysForYear };
```

到这里，整个日历好像差不多搞定了。但是有很多日历好像是这么展示的↓↓

<img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d322d653d1004b5787d206cbee1950ba~tplv-k3u1fbpfcp-watermark.image" width="80%" />

如上图，我们还需要知道每个月前后若干天以确保能够填满日历的空缺，这应该怎么实现呢？

大家应该一下就能猜到，还是用`specialDayInMonth`就可以完成了，我们通过本月的第一天的星期可以往前推到上一个周日，并且用下一个月的第一天往后推到下一个周六，这么一来就可以将日历上的空缺给补上。

**但是**，最终我没有采用这个方案，在实现优雅与代码易读上，我选择了后者。我将`specialDayInMonth`变成了`specialDaysInMonth`(每个月第一天和最后一天的星期)。

```js
function getAllDaysForYear(year) {
    /**
     * monthData 每月数据 用于最后输出
     * daysInMonth 每个月的天数
     * specialDaysInMonth 每个月第一天和最后一天的星期
     */
    const daysInMonth = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];

    // 对闰年二月天数特殊处理
    if ((year % 4 === 0 && year % 100 !== 0) || year % 400 === 0) {
        daysInMonth[1] = 29;
    }
    const monthData = new Array(12).fill(null);

    const specialDaysInMonth = monthData.slice(0).map((m, i) => {
        return [
            new Date(year, i, 1).getDay(),
            new Date(year, i, daysInMonth[i]).getDay(),
        ];
    });

    return monthData.map((m, i) => {
        const month = [];
        const pre = preDaysCreator(
            daysInMonth[i === 0 ? 11 : i - 1],
            specialDaysInMonth[i][0]
        );
        const normal = normalDaysCreator(daysInMonth[i]);
        const next = nextDaysCreator(specialDaysInMonth[i][1]);
        return month.concat(pre, normal, next);
    });
}

function preDaysCreator(preLastDay, firstDay) {
    const preDays = [];
    for (; firstDay > 0; firstDay--) {
        let obj = {
            content: preLastDay--,
            type: "pre",
        };

        preDays.push(obj);
    }
    return preDays;
}

function nextDaysCreator(lastDay) {
    const nextDays = [];
    const count = 6 - lastDay;
    for (let i = 0; i < count; i++) {
        let obj = {
            content: i + 1,
            type: "next",
        };

        nextDays.push(obj);
    }
    return nextDays;
}

function normalDaysCreator(days) {
    const normalDays = [];
    for (let i = 0; i < days; i++) {
        let obj = {
            content: i + 1,
            type: "normal",
        };

        normalDays.push(obj);
    }
    return normalDays;
}

export { getAllDaysForYear };
```

## 自适应实现

> 本次优化的另一个重点 -- **自适应**

之前想着大家都是直接fork项目修改的，所以并没有特别在意自适应的问题。但是如果用在移动端中，这个特性还是比较重要的，故将其列入迭代目标。

> 在写CSS的时候，我总是想着能不能这么写`height: width`，可惜要不得。

但是转念一想，其实我们可以借助dom操作来实现，在本组件中，每个**日期区块**的宽高是一样的，所以我只需要在dom节点渲染之后，获取区块宽度，就可以实现`height: width`这种蜜汁操作了，于是有了下面的逻辑，在组件挂载的时候，会在区块渲染后立刻获取区块宽度并设置区块的高度。

```js
data() {
    //...
    blockHeight: "0px",
}
// ...
mounted() {
    // ...
    this.$nextTick(function() {
      this.blockHeight = document.querySelector(`.main__block-${this.calendarID}`).offsetWidth + "px";
    });
}
```

这样一来，只需要在组件外部包裹一层就可以控制日历的大小了，不需要在传入宽高相关的参数。

```html
<template>
  <div id="app">
    <div style="width: 300px;">
      <kurisu-calendar
        targetDate="2022/11/01"
      />
    </div>
    <div style="width: 375px;">
      <kurisu-calendar
        targetDate="2023/11/01"
      />
    </div>
  </div>
</template>
```


<img src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8a156ec758b4223969d256d8f30d90f~tplv-k3u1fbpfcp-watermark.image" width="80%" />

## 使用建议

> 本次迭代最后一个重点 -- 使用方式拓展

稍微研究了一下vue组件的打包，完成了两年前没有实现的梦想(??)，把这个日历做成了npm包并发布上去。

因为vue-cli上的步骤已经很详尽了，并且npm包发布相关的博客也数不胜数，在此我就不赘述发布流程了，大家想了解的话可以直接看仓库中的`package.json`(仓库地址在文末)。

① 直接集成

```js
// 安装依赖
// npm install kurisu-calendar

// main.js
import KurisuCalendar from 'kurisu-calendar';
import 'kurisu-calendar/kurisuCalendar.css';
Vue.use(KurisuCalendar);
// App.vue
<div style="width: 375px">
    <kurisu-calendar />
</div>
```

② Fork改造

这里提供的组件，样式或者功能可能都没法完美贴合大家的需求，这种时候就可以fork仓库按照想要的方式尽情改造，打包相关的可以看一下package.json，没有很复杂的操作。

核心代码：

- packages/calendar/src/calendar.js -- 负责日历日期相关的处理
- packages/calendar/src/calendar.scss -- 日历样式
- packages/calendar/src/Calendar.vue -- 日历组件

## 后续规划

- [ ] 升级Vue3
- [ ] 输出React版
- [ ] 输出小程序版
- [ ] 加入滑动切换特性
- [ ] 加入日历数据生成钩子(自定义日历数据)

欢迎⭐️⭐️本仓库[kurisu-calendar](https://github.com/mykurisu/kurisu-calendar)持续关注后续迭代。
