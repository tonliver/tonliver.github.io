---
layout: post
title: 防抖与节流
date: 2023-06-12
Author: 天枫皓月 
tags: [技术,技能优化]
---
## 什么是防抖(debounce)与节流(throttle)
大部分做过前端的同学应该对这两个词不陌生，为了加深理解，我们先看两个真实场景

### 场景一
在百度、google的搜索框中输入内容时，根据输入内容有g一些相关的suggestion出来，以google为例，打开控制台我们可以看到有很多后台请求，其中的q值便是我们输入的内容
![search](https://s1.ax1x.com/2023/06/14/pCnBdBt.png)
从前端的角度我们来分析一下实现过程

1. 监听输出框的input事件
2. 在input事件处理函数中发起ajax请求，将input的value传给接口
3. 接口返回相关的关键词并展示为suggestion

下图中，我用关键词"什么"进行搜索，这里我用的是五笔，而"什么"的五笔码为"wftc"，再加上中文的确认输入（空格键），所以应该是触发5次input事件，图中也是向后台发起了5次请求
![search2](https://s1.ax1x.com/2023/06/14/pCnrgwq.jpg)

但实际上输入**wftc**只是生成目的关键词"什么"的中间过程，我并不需要任何与wftc相关的suggestion，也就是说前4次对我来说都是无效建议，当然，从技术的角度，这前4个对后台的请求也是浪费

那这个场景有没有什么办法可以优化呢？

想象一下我的交互过程
1. 凭借肌肉记忆，几乎没有停顿的敲入wftc这四个字母
2. 敲空格生成汉字，此时我有一个较长时间的停顿，确认我的目标关键词"什么"正确出现在搜索框中
3. 观察相关的suggestion中有没有我想要的结果

从交互过程中可以看出，输入中间过程和最终输入结果最大的区别就是：中间过程是**快速输入**的，此时用户是专注在输入上的，并不太关注suggestion给的提示是什么，然后在用户阶段性的完成输入时，会有**停顿**来确认自己的输入，此时就会顺便看suggestion有没有提前把自己想要搜索的内容提示出来

由此可以想到一个优化手段，我们设置一个时间阈值，当input事件快速连续触发，间隔时间小于这个阈值时，就认为是中间过程，舍弃执行处理逻辑，只有间隔时间大于阈值时，再执行真正的处理逻辑。

上面的描述中提到了间隔时间，自然而然的想到`setTimeout`，当事件触发时，起一个定时器，下次事件触发时发现如果已经有计时器，就将旧定时器清掉，然后新起一个，直到定时器等待时间结束，再执行处理逻辑

用vue快速实现一下
```vue
<template>
  <input v-model="keyword" @input="handleInput" />
</template>

<script>
const WAIT_TIME = 200;

export default {
  data() {
    return {
      keyword: '',
      timer: null,
    };
  }
  handleInput() {
    if (this.timer) {
      clearTimeout(this.timer)
    }
    this.timer = setTimeout(() => {
      this.search();
    }, WAIT_TIME)
  }
  search() {
    // 发起请求
  }
}
</script>
```
我们对这个场景做一下抽象，input事件的高频触发本质上是事件处理函数被频繁调用，而我们用定时器来取消调用，实际上是一个高阶函数，对处理函数包装后生成一个新的函数

用代码来实现一下
```ts
type Func = (...args: any[]) => any
/**
 * 生成防抖函数
 * @param {Func} fn 原始函数
 * @param {number} wait 等待时间阈值，毫秒为单位
 * @returns {Func}
 */
function debounce(fn: Func, wait: number): Func {
  let timer: ReturnType<typeof setTimeout> | null = null;
  return function(this: any, ...args: any[]) {
    if (timer) {
      clearTimeout(timer);
    }
    timer = setTimeout(() => {
      fn.apply(this, args);
      timer = null;
    }, wait);
  }
}
```
事实上，这个就叫做函数防抖(debounce)

**官方定义：**触发事件后，在n秒后只能执行一次，如果在n秒内重复触发事件，则会重置等待时间。简单的说，当一个事件连续触发，只执行最后一次。

### 节流（待续）
