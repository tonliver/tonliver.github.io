---
layout: post
title: 如何计算HTML元素中的文字宽度
date: 2024-04-11
Author: 天枫皓月
tags: [原创,技术]
---

## 背景描述
最近业务提了一个需求，在金额输入组件中，如果金额越出输入组件能显示的最大宽度，期望自动缩小字号。这个需求在金额输入这个场景是合理的，因为金额对位数是非常敏感的，如果超长进入滚动区域，用户看不到完整的位数，有可能导致输错，或者需要反复确认位数，体验非常不好。既然需求是合理的，那就想办法去实现它。

### 需求分析
这个需求是比较清晰的，是个典型的条件-结果式需求，一般会描述为：**在...条件下，就...**
* 条件 - 金额超出组件显示区域
* 结果 - 缩小字号

将需求翻译成技术语言

* 金额超出显示区域。也就是input出现横向滚动，我们知道html element的scrollWidth表示元素包含了滚动区域的总宽度,clientWidth是元素内可视宽度，出现滚动就是scrollWidth > clientWidth
* 缩小字号。通过给input.style.fontSize赋值，便可以改变字号。但是，我们应该应该缩小成多少，设计同学给了处理方式，比如正常是24px，超长后改成16px，但这是个具体的实现，我们在解决问题时，不能简单的写个if else把这两个值hardcode进去，需要对实现做进一步抽象，本质上是由调用方给一个规则，这个规则里包含了多个递减的字号，在判断超长后，自动的应用下一级字号，比如提供一个字号等级属性`['24px', '16px', '14px']`，理论上可以支持任意多级，这样就形成了一个比较通用的解决方案。

写了个简单的demo，效果如下。[查看完整代码](https://playcode.io/1833319){:target="_blank"}

![响应式输入](https://public.litong.life/blog/IMB_CazAlt.GIF)

### 新的问题
输入内容超长时，字号缩小的功能是实现了，那如果删除内容，删到剩余内容也不会超长时，期望字号恢复该如何实现？如果将上面的思路反过来，当没有滚动时，就自动应用下一级字号，快速实现一下。 

[查看完整代码](https://playcode.io/1833319?v=2){:target="_blank"}

![IMB_Ss4kzB](https://public.litong.life/blog/IMB_Ss4kzB.GIF)

可以看到字号会来回横跳，显然不符合预期，详细分析下原因。在超长字号被缩小后，内容就能全部显示在输入框内，但这时候并非刚好缩小到临界宽度，输入框中还有较多的输入空间，所以这时候是满足scrollWidth === clientWidth条件的，这时候哪怕是继续输入内容，也不会出现滚动，内容增多了反而恢复上一级更大的字号，显然是不合理的。回到现象本身，分析下整个过程：
1. 字号变小后，再触发input，满足scrollWidth === clientWidth，于是字号变大。
2. 变大后的字号肯定会超长，下一次input触发时字号又会缩小。
3. 缩小后滚动消失，再次触发input时回到过程1。

这个过程会往复循环，在界面上表现出的就是字号变大变小反复横跳。

### 原因分析
导致这个问题的本质是，决定文字要不要缩小，取决于以当前字号为基准文字内容的总宽度；而决定文字要不要放大，取决于**以放大后字号**为基准文字内容的总宽度，而以输入框是否有滚动条来判断，始终是以**当前显示的字号**为基准。要想修正这个问题，就必须在判断是否要放大时，用放大后的字号去显示，再看有没有滚动，但这样会导致用户看到字号变化抖动，在体验上是不可接受的。因此需要一种方案，**在不改变当前显示字号的前提下，计算出以其它字号为基准时内容总宽度**，这也是这篇文章的主角。

## 如何计算HTML元素中的文字宽度
这里的计算不是指获取已经渲染在页面上的文字内容的宽度，而是以一种纯函数的形式，传入一个字符串，就可以计算出它在某一个HTML容器内的显示后总宽度，函数签名类似于
```ts
type calculateTextWidth = (text: string) => number
```
既然是计算显示在HTML容器内的文字宽度，那就需要考虑文字渲染在HTML中时影响宽度的因素有哪些：
* font-family
* font-size
* font-weight
* letter-spacing
* word-spacing

这些因素也需要作为参数传入计算函数，于是函数签名变为

```ts
interface FontOptions {
  family: string;
  size: string;
  weight?: number|string;
  letterSpacing?: string;
  wordSpacing?: string;
}

type calculateTextWidth = (text: string, options: FontOptions) => number
```

### 方案一
能想到的最直接，最简单的方案，就是找到字号和单个字符宽度的关系，就可以用类似于`width = c * fontSize * text.length`这样的公式计算。但很可惜，受字体的影响，字号和单个字符之间并没有稳定的对应关系。下图可以比较直观的表现出来，这些文字font-size都是14px，比例字体同字体下不同的字符宽度可能不一致，不同字体下同样的字符宽度也可能不一致。

[查看完整代码](https://playcode.io/1832166){:target="_blank"}

![字体](https://public.litong.life/blog/20240412090140.png)

### 方案二
在页面上创建一个不可见的inline元素，将FontOptions中的样式设置到这个元素上，再将文字内容填入，最后计算这个元素的宽度。这个方案有几个关键的技术点：

* 不可将inline元素设为`display:none`，如果不占布局，获取到的宽度始终为0。
* 要对这个inline元素设置`white-space:nowrap;`，否则文字可能换行，导致无法计算出期望的宽度。
* 可以用一个div包裹这个inline元素，然后给div加上如下样式，这样做既可以保证元素不可见，也可以让div脱离文档流，减少回流。

  ```css
  {
    position:absolute;
    top:0;
    left:0;
    width:0;
    height:0;
    overflow:hidden;
    z-index:-9999;
  }
  ```

[查看完整代码](https://playcode.io/1811293){:target="_blank"}

### 方案三
通过canvas context的measureText方法可以直接获取到文字的宽度。[查看MSDN](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/measureText){:target="_blank"}

使用这个方案有几个关键技术点：
* 给context.font设置值的时候，遵循css font简写规则，需要注意font各属性的顺序，否则会出现属性不生效的问题。[查看MSDN](https://developer.mozilla.org/en-US/docs/Web/CSS/font){:target="_blank"}
![20240412184049](https://public.litong.life/blog/20240412184049.png){:width="500"}
* wordSpacing和letterSpacing有兼容性问题，iOS全系不支持，Chrome>=99。[查看CanIUse](https://caniuse.com/?search=wordSpacing){:target="_blank"}

部分核心代码如下

[查看完整代码](https://playcode.io/1811293){:target="_blank"}

```ts
const text = '请计算我的宽度';
const canvas = document.createElement('canvas');
const context = canvas.getContext('2d');
context.font = 'bold 14px monospace';

const { width } = context.measureText(text);
console.log(width);
```

### 性能对比
这两个方案分别涉及了DOM操作和canvas，都属于性能开销比较大的操作，那会不会产生性能瓶颈，我对这两种方案做了一些性能测试，后面为了方便起见，将两个方案简称为：

* 方案二：DOM实现
* 方案三：Canvas实现

##### 性能测试一
测试目的：研究调用次数和耗时之间的关系
测试方法：随机生成100个字符，分别用两个方案做1000/10000/30000次计算，重复10次，取平均值，所得测试数据如下图：
![20240412192714](https://public.litong.life/blog/20240412192714.png){:width="500"}

测试结论：对100个字符做1000次计算，Canvas实现耗时15ms，DOM实现耗时62ms，随着调用次数增多，调用次数和耗时基本上呈O(n)的关系。

##### 性能测试二
测试目的：研究字符长度和耗时之间的关系
测试方法：随机生成10/100/1000/10000个字符，分别用两个方案做1000次计算，重复10次，取平均值，所得测试数据如下图：
![20240412193322](https://public.litong.life/blog/20240412193322.png){:width="500"}

测试结论：对1000个字符做1000次计算，Canvas实现耗时6ms，DOM实现耗时41ms，随着字符数增多，字符数和耗时基本也呈O(n)的关系。

##### 性能测试三
测试目的：研究在不同浏览器的耗时情况
测试方法：随机生成100个字符，在不同浏览器中做2000次计算，重复10次，取平均值，所得测试数据如下图：
![20240412194738](https://public.litong.life/blog/20240412194738.png)
测试结论：
对100个字符做2000次计算，取耗时最高的三种浏览器，发布时间均超过5年，Canvas实现耗时900ms以下，DOM实现耗时1800ms以下，换算成单次计算耗时分别在0.5ms和1ms以内。而随着系统的更新，越邻近的浏览器性能表现越好。

* Mac Safari12.2.1 发布时间***2019.3***
* iPhone8/iOS11 - 发布时间***2017.9***
* 小米Redmi Note8/Android9 Chrome120 - 发布时间***2019.8***

**结论：通过以上三组数据，可以发现不管是调用次数，还是字符数，都和耗时呈O(n)关系；单次调用都能在1ms内完成；并且随着系统的更新，性能表现越好；Canvas实现性能要优于DOM实现。基本可以确定无性能瓶颈。**

### 是否可以更好？
这两个方案有一部分开销是用于创建Canvas或DOM，这个对象只是用于辅助计算的容器，而创建对象的开销远大于将对象保存在内存中的开销，因此可以用空间换时间的策略，把创建出来的Canvas或DOM作为单例，保存在内存中，所有计算都共享这个Canvas或DOM对象。

##### 优化后的性能测试
测试目的：对比使用单例和不使用单例的耗时
测试方法：随机生成100个字符，分别用两种方案的有单例和无单例做1000/5000/10000次计算，重复10次，取平均值，所得测试数据如下图：

![20240412202631](https://public.litong.life/blog/20240412202631.png)
![20240412202640](https://public.litong.life/blog/20240412202640.png)
![20240412202733](https://public.litong.life/blog/20240412202733.png)

测试结论：可以看出不管是Canvas还是DOM实现，使用单例后性能均提升50%左右

### 最终方案
通过上面的数据可以得出结论，使用单例模式的Canvas实现性能最优，DOM实现性能上也完全满足需求，但由于iOS全系不支持canvas context的wordSpacing和letterSpacing属性，为了保证最大程度的兼容性，可以根据特性检测结果，优先使用Canvas实现，如果不支持，则降级为DOM实现，核心代码如下：
```ts
const supportSpacing = CanvasRenderingContext2D.prototype.hasOwnProperty('letterSpacing')
  && CanvasRenderingContext2D.prototype.hasOwnProperty('wordSpacing');

const createCalculator = () => {
  return supportSpacing ? new CanvasTextWidthCalculator() : new DomTextWidthCalculator();
}
```

### 应用场景
1. 文章开始的响应式输入框方案（下篇分享完整实现）
![IMB_vi1T4m](https://public.litong.life/blog/IMB_vi1T4m.GIF)
2. 突破css的`text-overflow:ellipsis;`限制，实现可控的任意位置文字超长打点

更多应该场景有待小伙伴们发现。
