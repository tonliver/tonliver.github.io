---
layout: post
title: 一行HTML注释引发的血案
date: 2024-04-20
Author: 天枫皓月
tags: [原创,技术,Vue,Bug]
---

## 背景
前几天接到业务方反馈的一个bug，大致的场景是有一个页面使用了我们的input和number-keyboard组件，页面路由回退的时候会报错，导致路由无法跳转。经过一番沟通，我得到的原始反馈信息如下：
1. 复现步骤的录频（由于项目的安全要求，这里不能使用原视频，我用一个demo还原了视频中的操作步骤）
![复现视频](https://public.litong.life/blog/IMB_D7xmli.GIF)
2. 有错误信息的VConsole截图
![20240421101924](https://public.litong.life/blog/20240421101924.png){:width="500"}
3. 业务同学的反馈描述
> **场景那边的数字键盘有问题
>
> input插槽或数字键盘都会导致问题
>
> PC上正常，手机上有问题

### 信息筛选
这些反馈里，有一些是问题的表现，有一些是业务开发同学基于自己的经验下意识给出的结论，这些结论不一定都是正确的，如果把这些结论当成问题表现，就会产生先入为主的想法，干扰问题排查的方向，一定要仔细甄别。

| 信息              | 表现/结论 |
| :----------------| :--------|
| 操作视频          | ✅ 表现   |
| VConsole截图      | ✅ 表现   |
| input插槽导致问题  | ❌ 结论   |
| 数字键盘导致问题    | ❌ 结论   |
| PC正常，手机有问题  | ✅ 表现   |

## 分析定位
通过对上面的信息进行分析，初步猜测出几个结论：

| 结论                             | 理由                              | 排查方法 |
| :-------------------------------| :---------------------------------| :------|
| input组件存在bug                 | 业务方用减法去除了页面上的其他组件，只留了input和number-keyboard组件依然会引发bug | 继续做减法，只放input组件，如果能够复现，则结论成立 |
| number-keyboard存在bug          | 没拉起数字键盘时，页面可以正常返回，而input聚焦拉起键盘后，返回报错<br/>(这也是业务给出数字键盘有问题这一结论的主要原因) | 只放number-keyboard组件，如果能够复现，则结论成立 |
| 可能是组件销毁时存在bug            | 路由回退会触发组件销毁(查看过路由回退代码，没有可疑的逻辑) | 将路由回退替换成v-if触发组件销毁逻辑，如果能够复现，则结论成立 |
| 路由回退逻辑存在bug | 视频中是路由回退操作时出现报错 | 将路由回退替换成v-if触发组件销毁逻辑，如果不能复现，则结论成立 |
| 兼容性问题                        | PC正常，手机有问题 | 用手机复现 |

由于有VConsole的错误信息，可以看到是访问了null的parentNode属性导致报错，如果在代码中能搜到parentNode，大概率可以直接找到产生bug的地方，但是搜索后，代码中并没有parentNode，这个问题应该是无法简单的通过阅读代码逻辑就发现的。而上面几个排查方法都依赖把问题复现出来，于是我便快速写了个demo项目尝试复现问题。

因为会触发JS报错，在报错的地方设个断点，分析调用栈，不出意外的话应该很快就能定位到问题的根源，但既然出现了这篇文章，说明不出意外的话应该是出意外了。Demo写出来后，不管是PC端，还是手机上，都一切正常，控制台也没有任何报错信息，我怀疑是浏览器内核版本不够低，于是用了低于业务方内核的版本，依然无法复现。

### 困境
无法复现也就意味着上面列出的猜测无法得到验证，又加上无法通过梳理代码逻辑定位，排查一度陷入了困境，这个问题就像三体人的水滴一样，找不到突破口。

### 突破口

英语中有一句谚语`The devil is in the details`。如果说找不到突破口，说明一定有什么细节被忽略了。先把先前的种种猜测都清空，回到业务反馈的原始信息中，我反复观看了复现视频，思考这几条线索之间的关联。

> 阿克琉斯之踵 - PC正常，手机有问题

当我看到业务开发同学提的最后一点“**PC正常，手机有问题**”，之前下意识的根据这点判断是兼容性问题，可是PC和手机端除了兼容性，还有没有其它的差异。PC端一般是开发同学在本地运行的的dev server，而视频中用手机访问的是编译打包后部署到测试环境的代码，我又看了下VConsole的报错信息，果然是打包后的代码。
![20240421152637](https://public.litong.life/blog/20240421152637.png){:width="400"}

那会不会是dev server中的代码和打包后的代码逻辑不一致？乍一看似乎不可能，这里引出我很喜欢的一句话：

> 排除一切不可能，剩下的即使再不可思议，那也是真相

接下来，大胆假设，小心求证。验证方法很简单，执行`npm run build`，将demo项目打包，然后用whistle代理到本地的dist目录，用手机访问，**问题复现了!!!**。既然不是兼容性问题，那PC上也有可能复现，在PC上看果然也能复现。

### 验证结论
接下来就是去验证我们之前猜测的结论

| 结论                     | 是否复现 |
| :-----------------------| :-------|
| input组件存在bug         | ✅       |
| number-keyboard存在bug  | ❌       |
| 可能是组件销毁时存在bug    | ✅       |
| 路由回退逻辑存在bug       | ❌       |
| 兼容性问题               | ❌       |

经过一轮验证后，基本可以确定是input组件在销毁时存在bug。但还记得视频中如果没弹出数字键盘，是能够正常返回的，这应该说明是数字键盘触发的bug，这不是与验证的结论矛盾了吗？其实在数字键盘弹出前，还发生了一件事，那就是input获取了焦点，**input组件的状态发生了变化**，过于关注数字键盘，而忽略了input的改变。如果从一开始就把数字键盘引发bug当成表现，不去怀疑这个结论的正确性，我们的排查方向就会完全被误导。

### 最小化复现场景
既然确定是input销毁导致的问题，就可以把demo简化为最小化的复现场景：
1. 使用input组件
2. 更改input组件状态
3. 销毁input组件

## 开始调试
通过控制台上的错误信息，可以判断这个报错位置不是组件库的代码里的，应该是处于**更底层的库**，比如Vue。这种报错是无法通过console.log来定位的，更有效的方式是通过设置断点，跟踪调用栈，最终找到错误的根源。可以总结为以下步骤：

1. 触发bug，找到直接报错点。
2. 在直接报错点设置断点。
3. 重新触发bug，观察断点处的变量值，如果是正常值，恢复代码执行，直到变量出现异常值（即触发bug的值）。
4. 回溯调用栈，分析每个函数的执行逻辑，找到异常变量产生的源头。

下图是chrome devtools的源码调试面板，箭头标注的两个面板是需要重点关注的，一个是本地变量，一个是函数调用栈。
![20240421170334](https://public.litong.life/blog/20240421170334.png)

### 找到直接报错点
直接上操作视频
<video controls width="100%" src="https://public.litong.life/blog/%E5%BD%95%E5%B1%8F2024-04-21%2018.12.23.mov"></video>

从视频上可以看到，最后一次调用remove时，el为null，之后控制台便出现了报错信息，这里便是回溯调用栈的点。同时通过断点，也确认了另一个猜测，这个异常是由更底层的vue抛出来的，下一步回溯需要分析vue的代码，而vue的代码是混淆过的，不利于阅读，需要引入sourcemap来映射到源码上。关于sourcemap的使用，我会在另一篇文章中专门介绍。[[如何调试vue源码]](/debug-vue-source-code)

### 回溯调用栈
回溯指的是从设定的断点开始，沿着函数调用链路，往回/后查看每个函数的执行情况。代码逻辑是顺序执行的，比如`A -> B -> C`，如果在C处出现了不符合预期的结果，常规调度手段要先去B处打log，再次执行看log信息。而断点调试可以在C处设置断点，让程序暂停执行，保持完整上下文的情况下，进入到函数B中，因为有完整的上下文，此时能够直接看到B中所有变量的值。

还是继续上操作视频
<video controls width="100%" src="https://public.litong.life/blog/2024-04-21 19.47.04.mov"></video>

从视频的最后可以看到，在直接报错点的上游调用栈中，调用Teleport.ts中remove方法时，children为`@slot 输入框前置内容`的vnode的el为空。
![20240421195316](https://public.litong.life/blog/20240421195316.png)

而这个`@slot 输入框前置内容`，是input组件中的一行注释，虽然还不清楚什么原因一行注释会导致vnode的el为null，大胆猜测是它引起了bug。
![20240421204747](https://public.litong.life/blog/20240421204747.png){:width="300"}

再小心求证，将这行注释删掉 -> 打包组件库 -> 在demo项目安装 -> 尝试复现，果然报错消失。虽然很难以置信，但确实是这一行注释导致了报错。至此终于知道怎么修复这个bug，可以先发版消除对业务的影响。

### 追本溯源
虽然bug暂时修复了，但我只是想办法绕开了这个报错，为什么一行注释会导致这个bug，这个问题并没有得到答案，我决定深入Vue源码中寻找答案。

之前通过回溯调用栈，知道了是Teleport的remove方法，在remove那个注释vnode时发生报错，原因是这个vnode的el属性为null，所以接下来就是找到el属性为null的原因。那这里的突破点是什么？还记得上面的最小复现步骤中的第2步吗？- 更改input组件状态。更改Vue组件的响应式状态，也就意味着触发组件的更新。触发组件更新是产生bug的必要条件，那就可以大胆猜测是更新时产生了这个异常的vnode，然后再小心求证。

还是从Teleport.ts入手，通过阅读源码，发生Teleport有以下几个方法:
1. remove销毁时调用
2. process更新时调用(通过函数签名以及代码中的注释判断)

![20240422035922](https://public.litong.life/blog/20240422035922.png){:width="500"}
![20240422040302](https://public.litong.life/blog/20240422040302.png){:width="500"}
![20240422040720](https://public.litong.life/blog/20240422040720.png){:width="500"}

在process的update content逻辑分支处设置断点，然后更改组件状态，看是否会执行到断点处。经过验证，发现组件状态更新时，确实会调用Teleport的process方法
<video controls width="100%" src="https://public.litong.life/blog/2024-04-22 04.57.25.MOV"></video>

上面的视频中使用了单步调试(让代码一行一行的执行)，发现执行到`traverseStaticChildren`函数的时候，有两个参数，分别是
* n1 - 旧vnode
* n2 - 新vnode

再结合函数名，这个函数应该是用于传送静态子节点。

![20240422051847](https://public.litong.life/blog/20240422051847.png){:width="600"}

使用断点调试的`Step into next function call`，可以进入到`traverseStaticChildren`函数内部

![20240422052441](https://public.litong.life/blog/20240422052441.png){:width="600"}

可以发现这个函数主要是逻辑就是把旧vnode.el赋值给新vnode.el，代码中处理了ElementNode、TextNode，而对于CommentNode仅在__DEV__模式处理，而prod模式不处理，所以新的vnode.el是默认值null。这和我们的bug触发条件完全吻合，由于从源码中了解到触发机制，这应该是vue的通用bug，与组件库的逻辑无关，可以对复现步骤进一步做减法，验证不使用组件库是否能够复现: [[查看最小复现步骤]](https://play.vuejs.org/#__PROD__eNqVU8tu2zAQ/JUtL7FRW2qbnFzJ6AM5tIe2aHoUUCjSWmJCkQRJKTYM/XuX1CN2kQTIRSJ3Z7jD5eyRfdY66lpkG5bYwnDtwKJr9TaTvNHKODiCwR30sDOqgQuCXnycc19Vo8dEFPuNP8rnCyUtnVSrhwBJ/RkLZ1pcZjKJh0JUgjYOGy1yh7RL6qvtjUNtN0lMSwooQV+ARPBtaxG0UWVbOK4kmFY63iA0qsQkpvwpzqHAoI/LINGrOgPlZQk5OKVBYIcCatcIKFTToAyk6YAzUlHnskLI5QGsI8UeSBytJLFW4AyvKjTgaoRWlwQ4Y5donTKHR8aYTeJwycTHoVvzXZqxqW8Zg9jnbs34b52ju38qBC/uT3DpLhcWM/ZUkYETCp20mq2Ys/RIO15Fd1ZJev6jV5oxT+UCzU/t+2wztoGQ8blcCPXwPcT8W66meFFjcf9E/M7ufSxjvwxaNB1JnHMuNxW6IX198wP3tJ6T9KqtIPQLyd9olWi9xgH2pZUlyT7BBbXfglG5rP7Y671DaadLeaEe2Qd8xsgivpXPXf1R7mV0FXiZ7KmLk71eOT/DfBSKXEyWGcbjHc3GEOeyMJiTkVNYLCHdTsCoy0WL8DaF9y+NUck7+o0cusNxLtT38KyZpqJko7n+yFsMfh5svzz11DxpTtERt6o8EJ3ib9brcZzW69F7AzBsBoH/G/Jvh8Y/D7WSehx9uGT9P3XFlug=){:target="blank"}
1. 写一个只有一个响应式状态的简单组件Comp(为了触发更新)。
2. 在Comp组件中使用teleport，teleport中添加html注释。
3. 更改Comp组件状态，触发update。
4. 在父组件上使用v-if销毁Comp组件。
5. prod模式报错，dev模式正常。

依然报错，再次验证了这次bug的根源就在此。后面也给vue官方提交了issue和修复bug的MR，还在等待回复。

## 收获
1. 通过这次bug的定位过程，对vue的源码有了更深入的了解，尤其是组件的更新和销毁流程。
  - 更新流程
  ![teleport更新流程](https://public.litong.life/blog/teleport更新流程.png){:width="400"}
  - 销毁流程
  ![teleport销毁流程](https://public.litong.life/blog/teleport销毁流程.png){:width="600"}
2. Vue在dev和prod环境会加载不同的js文件，两个文件的实现会有差异。
```js
// vue主入口文件index.js
'use strict'
if (process.env.NODE_ENV === 'production') {
  module.exports = require('./dist/vue.cjs.prod.js')
} else {
  module.exports = require('./dist/vue.cjs.js')
}
```
## 几个有用的鸡汤
这次bug排查中，出现了几个人颠覆原有认知的点。在面对复杂问题时，知识和技巧固然重要，但更重要的是思维方式和认知模式，就像大海中航行一样，技巧决定船的速度，而思维方式决定了船的方向，如果方向错误，越快的速度只会让我们越快的远离目标。

* 仔细甄别事实和结论，未被验证的结论，不管看上去有多可信，都有可能是不成立的。
  - 弹出数字键盘后销毁组件才会报错，因此是数字键盘引发的bug。
* The devil is in the details.(魔鬼藏匿于细节中)
  - PC正常，手机有问题 —— 不是兼容性问题，而是vue dev和prod模式逻辑不同。
* 大胆假设，小心求证。
* 排除一切不可能，剩下的即使再不可思议，那也是真相。
  - vue dev和prod模式逻辑不同
  - 一行html注释也会引发bug
* 追本溯源。通过高明的技巧绕过问题固然有用，但更好的做法是深入了解问题产生的根源和原理，因为你无法确定这次绕过问题所走的捷径终点是不是另一个陷阱。