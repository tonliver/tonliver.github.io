---
layout: post
title: 防抖与节流
date: 2023-06-12
Author: 天枫皓月 
tags: [原创,技术,技能优化]
---
## 什么是防抖(debounce)与节流(throttle)
大部分做过前端的同学应该对这两个词不陌生，为了加深理解，我们先看两个真实场景。

### 场景一
在百度、google的搜索框中输入内容时，根据输入内容有相关的suggestion出来，以google为例，打开控制台我们可以看到有很多后台请求，其中的q值便是我们输入的内容。
![search](https://s1.ax1x.com/2023/06/14/pCnBdBt.png)
从前端的角度我们来分析一下实现过程

1. 监听输出框的input事件
2. 在input事件处理函数中发起ajax请求，将input的value传给接口
3. 接口返回相关的关键词并展示为suggestion

下图中，我用关键词"什么"进行搜索，这里我用的是五笔，而"什么"的五笔码为"wftc"，再加上中文的确认输入（空格键），所以应该是触发5次input事件，图中也是向后台发起了5次请求。
![search2](https://s1.ax1x.com/2023/06/14/pCnrgwq.jpg)

但实际上输入**wftc**只是生成目的关键词"什么"的中间过程，我并不需要任何与wftc相关的suggestion，也就是说前4次对我来说都是无效建议，当然，从技术的角度，这前4个对后台的请求也是浪费。

那这个场景有没有什么办法可以优化呢？

拆分一下我的交互过程
1. 凭借肌肉记忆，几乎没有停顿的敲入wftc这四个字母
2. 敲空格生成汉字，此时我有一个较长时间的停顿，确认我的目标关键词"什么"正确出现在搜索框中
3. 观察相关的suggestion中有没有我想要的结果

从交互过程中可以看出，输入中间过程和最终输入结果最大的区别就是：中间过程是**快速输入**的，此时用户是专注在输入上的，并不太关注suggestion给的提示是什么，然后在用户阶段性的完成输入时，会有**停顿**来确认自己的输入，此时就会顺便看suggestion有没有提前把自己想要搜索的内容提示出来。

由此可以想到一个优化手段，我们设置一个时间阈值，当input事件快速连续触发，间隔时间小于这个阈值时，就认为是中间过程，舍弃执行处理逻辑，只有间隔时间大于阈值时，再执行真正的处理逻辑。

上面的描述中提到了间隔时间，自然而然的想到`setTimeout`，当事件触发时，起一个定时器，下次事件触发时发现如果已经有计时器，就将旧定时器清掉，然后新起一个，直到定时器等待时间结束，再执行处理逻辑。

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
  },
  methods: {
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
}
</script>
```
![防抖示例](https://i.mjj.rip/2023/06/15/13f1fb17e9af3bcfe2e94b543d1010e9.gif)
我们对这个场景做一下抽象，input事件的高频触发本质上是事件处理函数被频繁调用，而我们用定时器来取消调用，实际上是一个高阶函数，对处理函数包装后生成一个新的函数。

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

用这个封装好的防抖函数改写上面的例子，代码更简洁了，并且vue代码中只需要关注业务逻辑本身。当然，如果用了class风格的组件，还可以使用装饰器来包装，会更加优雅
```vue
<template>
  <input v-model="keyword" @input="handleInput" />
</template>

<script>
import debounce from './debounce.ts';

const WAIT_TIME = 200;

export default {
  data() {
    return {
      keyword: '',
      timer: null,
    };
  },
  methods: {
    handleInput: debounce(function () {
      this.search();
    }, WAIT_TIME),
    search() {
      // 发起请求
    }
  }
}
</script>
```

**官方定义：**触发事件后，在n秒后只能执行一次，如果在n秒内重复触发事件，则会重置等待时间。简单的说，当一个事件连续触发，只执行最后一次。

### 场景二
在大部分瀑布流的网站中，都有滚动到接近底部时，会自动加载更多内容的能力，这样既能避免用户刚进入首屏时就加载过多内容造成浪费，又可以在用户有意愿浏览更多内容时，最大程度在用户无感知的情形下，加载更多内容。

![滚动加载](https://i.mjj.rip/2023/06/15/397a9c5641bee27543e30c4a1599ebf2.gif)

为了方便下面举例计算，把大概的dom结构定义出来
```html
<div id="scroller">
  <ul id="scrollerContent">
    <li>一行毫无意义的内容</li>
    <li>一行毫无意义的内容</li>
    <li>一行毫无意义的内容</li>
    <li>一行毫无意义的内容</li>
  </ul>
</div>
```

从前端的角度分析一下具体实现
1. 监听出现滚动条的DOM的scroll事件
2. 在事件处理函数中计算视窗以下距离内容底部剩余高度(scrollerContent.clientHeight - scroller.scrollTop - scroller.clientHeight)
3. 判断步骤2中计算的剩余高度如果小于阈值则加载更多内容

在这个示例中我记录了一下scroll事件的触发频率，大概是10ms左右会触发一次，上面例子第二步中访问DOM的clientHeight、scrollTop这些属性都会引起回流，对性能有比较大的损耗。
![scroll事件触发频率](https://s1.ax1x.com/2023/06/15/pCuDch8.png){: height="300"}

再从用户的角度来分析一下交互过程，触发加载更多是否需要精准到10ms级别的响应，用户的诉求是只要看完底部的内容，不用等待就可以继续看到更多的内容，用户是边浏览边滚动的，百毫秒级别的响应对用户来说基本就满足无感知了。

这里技术实现和用户诉求之间就出现了不一致的点，而这个不一致的点就是优化的方向，我们并不需要在每次触发scroll事件时都去做计算，只需要按我们规定的时间间隔做计算即可。

用vue做一个简单的实现
```vue
<template>
  <div class="scroller" ref="scroller" @scroll="handleScroll">
    <ul class="scroller-content" ref="scrollerContent">
      <li>无意义的内容</li>
      <li>无意义的内容</li>
      <li>无意义的内容</li>
      <li>无意义的内容</li>
    </ul>
  </div>
</template>

<script>
  const REMAIN_HEIGHT_TO_LOAD_MORE = 50;
  const WAIT_TIME = 200;
  export default {
    data() {
      return {
        scrollerHeight: 0,
        scrollerContentHeight: 0,
        timer: null,
        lastExecutionTime: 0,
      };
    }

    mounted() {
      this.calculateScrollerHeight()
    },

    methods: {
      calculateScrollerHeight() {
        this.scrollerHeight = this.$refs.scroller.clientHeight;
        this.scrollerContentHeight = this.$refs.scrollerContent.clientHeight;
      }
      
      handleScroll() {
        if (this.timer) {
          clearTimeout(this.timer);
          this.timer = null;
        }
        const remainTime = Date.now() + WAIT_TIME - this.lastExecutionTime;
        if (remainTime <= 0) {
          this.tryLoadMore();
        } else {
          this.timer = setTimeout(() => {
            this.tryLoadMore();
            this.timer = null;
          }, remainTime)
        }
      }
  
      willLoadMore() {
        const remainHeight = this.scrollerContentHeight - this.scrollerHeight - this.$refs.scroller.scrollTop;
  
        return remainHeight <= REMAIN_HEIGHT_TO_LOAD_MORE;
      }
  
      tryLoadMore() {
        this.lastExecutionTime = Date.now();
        if (this.willLoadMore()) {
          this.loadMore();
        }
      }
  
      loadMore() {
        // 加载更多的逻辑
      }
    }
  };
</script>
```
![节流示例](https://i.mjj.rip/2023/06/15/5b39850a9f5df5ca71f646a23fbd1390.gif)

对这个场景也可以做进一步抽象，给定一个原始函数，经过包装后生成新的函数，这个函数在一定时间内多次调用，但只会以固定时间间隔来执行原函数，在这时间间隔内的调用会被忽略

用ts实现一下
```ts
type Func = (...args: any[]) => any;
/**
 * @param {Func}   fn     原始函数
 * @param {number} wait   等待时间间隔
 */
function throttle(fn: Func, wait: number): Func {
  let timer: ReturnType<typeof setTimeout> | null;
  let lastExecutionTime = 0;

  return function(this: any, ...args: any[]) {
    if (timer) {
      clearTimeout(timer);
      timer = null;
    }
    const remainTime = Date.now() + wait - lastExecutionTime;

    if (remainTime <= 0) {
      lastExecutionTime = Date.now();
      fn.apply(this, args);
    } else {
      timer = setTimeout(() => {
        lastExecutionTime = Date.now();
        fn.apply(this, args);
        timer = null;
      }, remainTime);
    }

  }
}
```
这个实现就是**函数节流(throttle)**

使用节流函数改写上面的例子
```vue
<template>
  <div class="scroller" ref="scroller" @scroll="handleScroll">
    <ul class="scroller-content" ref="scrollerContent">
      <li>无意义的内容</li>
      <li>无意义的内容</li>
      <li>无意义的内容</li>
      <li>无意义的内容</li>
    </ul>
  </div>
</template>

<script>
  import throttle from './throttle.ts';

  const REMAIN_HEIGHT_TO_LOAD_MORE = 50;
  const WAIT_TIME = 200;
  export default {
    data() {
      return {
        scrollerHeight: 0,
        scrollerContentHeight: 0,
        timer: null,
        lastExecutionTime: 0,
      };
    }

    mounted() {
      this.calculateScrollerHeight()
    },

    methods: {
      calculateScrollerHeight() {
        this.scrollerHeight = this.$refs.scroller.clientHeight;
        this.scrollerContentHeight = this.$refs.scrollerContent.clientHeight;
      }
      
      handleScroll: throttle(function () {
        this.tryLoadMore();
      }, WAIT_TIME),
  
      willLoadMore() {
        const remainHeight = this.scrollerContentHeight - this.scrollerHeight - this.$refs.scroller.scrollTop;
  
        return remainHeight <= REMAIN_HEIGHT_TO_LOAD_MORE;
      }
  
      tryLoadMore() {
        if (this.willLoadMore()) {
          this.loadMore();
        }
      }
  
      loadMore() {
        // 加载更多的逻辑
      }
    }
  };
</script>
```

**官方定义：**当一个函数连续多次被调用，在每个固定的时间周期内只执行一次

### 防抖与节流的异同
防抖与节流都是对函数高频调用时的性能优化手段，主要区别是防抖每次在时间阈值内调用都会重置等待时间，最终函数只会执行一次；而节流却是在每一个时间周期内都会确保执行一次。

### 代码
上面两个例子的完整代码https://github.com/tonliver/fun/tree/master [传送门](https://github.com/tonliver/fun/tree/master)