---
layout: post
title: 一次兼容性问题定位过程引发的思考
date: 2024-03-16
Author: 天枫皓月 
tags: [技术,兼容性,Debug]
---
## 背景

最近在负责一个基于vue3+typescript+vite的前端组件库的项目，我们团队负责为业务项目组提供组件库。由于项目性质，与我们对接的项目组有一些特点：
1. 各个项目组属于不同的团队，甚至是不同的公司，办公地点也不在一起，只能远程沟通。
2. 业务项目安全级别较高、开发环境的网络隔离。
3. 业务项目组在真机环境下无法自由使用VConsole等调试工具。
4. 项目组的进度比较紧，我们要快速解决问题，并且要尽可能少打扰项目组（次数和时间两个维度），我称为**最小介入原则**。

在这种条件下，我接到了一个项目组反馈的bug，而按照项目组提供的环境，无法复现此bug，因此定位bug的过程可以说是一波三折。

在问题解决后，我反思了整个过程，由于前端的代码是在用户端的浏览器/Webview中执行的，开发同学很难模拟出一个与用户侧一模一样的环境进行调试，因此经常会遇到“我这里是好的”这样的困境，而我这次遇到的便是典型的这类问题，经过复盘，我发现可以从里面沉淀出一些方法论，希望在以后遇到类似的问题可以帮助我们有迹可循。

## 问题描述
在2024.3.12接到了其中一个项目组的开发同学的反馈，原话是：
> 发现有Button和Grid组件有兼容问题，在华为P20和三星note 9上不渲染，请查看是什么问题

### 关键信息
通过这段描述，可以得到几个关键信息：
1. Button和Grid组件有问题
2. 组件不渲染
3. 在华为P20和三星note9上出现
4. 兼容性问题

### 信息筛选
但其实这些信息里有一些是反馈问题的同学的猜测，并不是对原始问题的描述，这些猜测很容易让我们产生**先入为主**的想法，干扰问题的定位，所以需要仔细加以甄别
1. 兼容性问题（这是反馈同学猜测的结论）
2. 不渲染（同样是结论，事实上反馈同学看到的现象是组件没有显示，没显示的原因有很多，未渲染、被其它元素遮挡、样式异常、display:none...，而他无法提供DOM结构或log信息，所以无法判定是否未渲染）

## 分析定位

### 迭代一
##### 问题表现
经过筛选后，我们可以得出真正的问题表现是，在华为P20和三星note9上，组件没有显示

##### 原因分析
在拿到这些关键信息后，我凭借经验对bug可能产生的原因做了初步的分析：
1. 可能性最大的是兼容性bug，导致组件报错未渲染（主要出于两点考虑：1. 这个项目组是接入我们组件库的第二个项目组，第一个项目并未发现类似的bug 2. 只在特定机型上存在问题）
2. 按钮渲染了，但由于未设安全距离，被底部小黑条遮挡（用户发给了我一张正常机型的和一张异常机型的手机屏幕照片，我看到按钮是在屏幕的最下面，所以猜测有这个可能）
3. 项目组的配置问题，导致vue组件名未被识别，被渲染成了自定义html标签
4. 项目组某些样式与组件样式冲突（可能性不大，但由于之前有项目组引用了tailwind基础样式，而这些css是对`*`选择器设置的，影响了组件的样式的先例，所以我把这个case考虑在内）

##### 解决思路
基于上面的分析，自然而然在脑中想到了对应的解决步骤：
1. 用我们的demo项目，从wetest上找两台同样的机型上进行复现，通过VConsole查看报错信息
2. 针对报错进行修复
3. 验证通过，发布fix版本
4. 项目组更新组件库，进行验证

### 迭代二
##### 问题表现
本以为就是个小bug，一首歌的时间就可以解决，可现实是在第一步就翻了车，我在项目组提供的两个机型上均无法复现!!!

##### 原因分析
冷静分析后，有两种可能：
1. 确实是兼容性bug，只是我的复现环境与项目组不一致
2. 不是兼容性bug，那上面初步分析中其他3种原因的可能性就增大了

##### 解决思路
针对上面的两种可能性，对应的解决步骤：
1. 放弃还原复现环境，直接在业务项目中引入VConsole，获取报错信息。（这是最快最有效的方法，但受限于项目组的约束，这条路行不通）
2. 继续还原复现环境。Webview内核除了与手机型号有关，还和操作系统版本有关，之前的复现环境只考虑到了手机型号，需要进一步确认项目组两台手机的系统版本。
3. 如果是安全区域的问题，需要项目组调整组件的位置，确认组件是否正常渲染。
4. 如果不是兼容性bug，那就可能是外部影响（比如项目组的构建配置、第三方库冲突等等），这类问题都必须在业务项目中逐点排查，会耗费双方大量的时间，这是最坏的情况，我们要尽量避免这种情况的发生。要快速排除外部影响的可能性，就需要把组件库放到一个没有外部依赖的页面中，让项目组去访问，如果组件正常，说明是业务项目中的某些功能影响到了组件；如果组件仍然不正常，说明是组件本身的问题。

思路确定后，我开始并行处理2-4点：
1. 询问项目组两部手机的系统版本。华为P20(Android 9)，三星note9(Android 10)。
2. 让项目组帮排除下安全区域的可能性。刚好项目组之前尝试过，还是不能正常显示。
3. 提供可以排除外部影响的demo页面。这里考虑到时间和准确性，我采取了两个措施。
  - 给我们现有的storybook加上VConsole。好处是可以立即提供给项目组验证，缺点是storybook本身也是一套有大量依赖的重型框架，如果storybook中的组件正常，可以确定是业务项目产生的影响，但如果storybook不正常，仍然无法确定是组件库的问题。
  - 用`npm create vue`的最小配置创建一个新项目，仅安装组件库并引入VConsole。好处是可以排除所有的外部影响，缺点是需要创建和部署站点，稍微需要一点时间。

##### Tips
在执行的时候，我是先询问了项目组系统版本，自己在wetest上用一样的机型和系统版本，访问了我们的storybook，仍旧无法复现。然后才和项目组沟通，提供了第3个方案，在他尝试访问storybook的同时，我开始用create vue创建新的demo项目。这里也是最小介入原则的一个运用。

### 迭代三
##### 问题表现
事情并没有想象的那么顺利，项目组访问storybook表现为白屏，正如上一步分析的，这个结果我们没法确定是组件库的问题。

##### 原因分析
但也有好的一面，因为业务项目和storybook是两套完全不同的框架，交叉点并不多，确定的是两者都使用了我们的组件库，出现了类似的问题，说明大概率是组件本身的问题。这样至少让我们离上面评估的最坏的情况又远了一步。

##### 解决思路
剩下的就是用我新建的demo项目，如果不出意外的话，这次我就可以拿到DOM结构和log信息了。

### 迭代四
##### 问题表现
既然写到了迭代四，说明不出意外的话，应该是出意外了。这个demo项目又双叒叕白屏了！！！但好在VConsole出来了，项目组看到了DOM结构。
![20240319085145](http://public.litong.life/blog/20240319085145.png?imageMogr2/quality/100/thumbnail/400x)

##### 原因分析
可以看到整个app节点是空的，也就是说vue app没有挂载到DOM上，可能的原因有两个：
1. 代码报错，中断了后续的执行（项目组说VConsole中没有报错，所以可以排除）
2. JS代码压根没执行（我当时心里一直循环播放着一句话：不可能！！！绝对不可能！！！）
> 但福尔摩斯有一句名言：“排除一切不可能，剩下的即使再不可思议，那也是真相”
冷静分析后，还是想到几种可能性:
1. 构建的时候script标签没被正确注入。但其实可能性很小，create vue最少配置创建的新项目，如果有这么明显的bug，前端圈早炸锅了。
2. 存在我不知道的知识盲区。

于是我去看了下构建后的html，发现了注入的script标签是这样的
```html
<script type="module" crossorigin src="./assets/index-DPuj3le7.js"></script>
```
`type="module"`，只有支持es6的浏览器才能正常执行，这样一切就都说的通了，业务组的浏览器不支持es6，这个demo项目使用最新的vite构建，默认只支持现代浏览器，所以加了type="module"的JS代码没有执行，而组件库中大概率是使用了es6中才有的方法，而我们在打包的时候没有做polyfill，所以代码报错，组件未渲染。

查看了vite的官网，也验证了这个结论，如果要支持旧浏览器，需要引入这个插件
> ### @vitejs/plugin-legacy 
>Vite's default browser support baseline is Native ESM, native ESM dynamic import, and import.meta. This plugin provides support for legacy browsers that do not support those features when building for production.

##### 解决思路
最终答案呼之欲出，我们快速验证一下，引入了插件对项目进行重新构建，果然增加了一些polyfill和nomodule的script
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <link rel="icon" href="./favicon.ico">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Vite App</title>
    <script src="https://unpkg.com/vconsole@latest/dist/vconsole.min.js"></script>
    <script>new window.VConsole()</script>
    <script type="module" crossorigin src="./assets/index-DPuj3le7.js"></script>
    <link rel="stylesheet" crossorigin href="./assets/index-DqayVCv2.css">
    <script type="module">import.meta.url;import("_").catch(()=>1);(async function*(){})().next();if(location.protocol!="file:"){window.__vite_is_modern_browser=true}</script>
    <script type="module">!function(){if(window.__vite_is_modern_browser)return;console.warn("vite: loading legacy chunks, syntax error above and the same error below should be ignored");var e=document.getElementById("vite-legacy-polyfill"),n=document.createElement("script");n.src=e.src,n.onload=function(){System.import(document.getElementById('vite-legacy-entry').getAttribute('data-src'))},document.body.appendChild(n)}();</script>
  </head>
  <body>
    <div id="app"></div>
    <script nomodule>!function(){var e=document,t=e.createElement("script");if(!("noModule"in t)&&"onbeforeload"in t){var n=!1;e.addEventListener("beforeload",(function(e){if(e.target===t)n=!0;else if(!e.target.hasAttribute("nomodule")||!n)return;e.preventDefault()}),!0),t.type="module",t.src=".",e.head.appendChild(t),t.remove()}}();</script>
    <script nomodule crossorigin id="vite-legacy-polyfill" src="./assets/polyfills-legacy-cC7ibzT-.js"></script>
    <script nomodule crossorigin id="vite-legacy-entry" data-src="./assets/index-legacy-DFMm9gGC.js">System.import(document.getElementById('vite-legacy-entry').getAttribute('data-src'))</script>
  </body>
</html>
```
简单部署后，让项目组的同学访问了一下，果然看到了期望已久的报错信息

![20240319093622](http://public.litong.life/blog/20240319093622.png)

### 迭代五
##### 原因分析
现在问题比较明确了，组件库在代码里使用了Object.fromEntries，而这个方法在低版本浏览器中不支持，导致代码报错。

我们可以通过这个网站查到指定的特性在各个平台的chrome的支持版本
[https://chromestatus.com/features](https://chromestatus.com/features)
![20240319100211](http://public.litong.life/blog/20240319100211.png?imageMogr2/quality/100/thumbnail/300x)
通过VConsole，看到项目组的webview内核版本是70，确实不支持Object.fromEntries
![20240319100619](http://public.litong.life/blog/20240319100619.png)

##### 解决思路
根源已经找出来了，剩下的就是考虑如何修复，可选的方案有：
1. 在组件库中引入polyfill。
  - 优点: 简单。
  - 缺点: 可能引入多余的polyfill，导致包体积增大。全局的polyfill有可能对组件库之外的模块造成污染。
2. 使用babel-plugin在编译阶段引入polyfill和runtime。
  - 优点: 自动处理，可以按需polyfill，作用于组件库内部，对外无污染。
  - 缺点: 包体积增大，编译方式的变更会引发整个组件库变更，需要全面回归验证。
3. 自己实现或者使用第三方的库（比如lodash）替换代码中调用的Object.fromEntries
  - 优点: 改动小，影响范围可控，对包体积几乎无影响
  - 缺点: 只修复了Object.fromEntries，无法保证其它API的兼容性

综合考量后，现阶段采用方案3，主要是对项目组影响最小，避免大范围的回归测试。后期可以用babel-plugin进行处理，避免有漏掉的API兼容性问题。

##### 问题修复
1. 从线上分支创建一个hotfix分支
2. 修改代码
3. 预发布一个beta补丁版本
4. 在demo项目中更新，部署后让项目组同学验证，组件终于可以正常渲染了
![20240319103656](http://public.litong.life/blog/20240319103656.png?imageMogr2/quality/100/thumbnail/400x)
5. 将hotfix分支合并回主分支，发布正式版本
6. 让项目组安装正式版本，在业务项目中验证通过

## 复盘
至此问题已经修复，但这里面始终有一个迷团没有解开，就是我用和项目组同样的机型和系统，都无法复现。作为一个老前端，直觉告诉我，看看能否通过userAgent发现什么，打开VConsole查看了我在wetest上找的云真机chrome版本竟然是**99**!!!

![20240319110012](http://public.litong.life/blog/20240319110012.png?imageMogr2/quality/100/thumbnail/400x)

为什么同样的机型和系统版本，chrome版本会相差这么多，我肯定是哪里存在知识盲区，难道是我打开云真机的姿势不对，于是我再次打开了wetest的云真机列表，发现有一列硬件型号，它甚至排在系统版本的前面，这个型号编码在userAgent中也有体现，难道说它会影响浏览器内核版本，我感觉越来越接近真相了。

![20240319111440](http://public.litong.life/blog/20240319111440.png)

验证的方法很简单，选同样机型和系统版本的手机，通过VConsole查看浏览器内核版本

| 机型      | 硬件型号  | 系统版本   | 浏览器内核版本 |
|:---------|:---------|:---------|:-------------|
| P20      | EML-AL00 | Android9 | Chrome99     |
| P20      | EML-L29  | Android9 | Chrome70     |
| P20 lite | ANE-AL00 | Android9 | Chrome57     |
| P20 lite | ANE-LX1	| Android9 | Chrome118    |
| P20 lite | ANE-LX3	| Android9 | Chrome122    |

由此可见，浏览器内核版本是由**机型 + 硬件型号 + 系统版本**三者共同决定的，这个知识盲区导致了我一直没有找到准确的复现场景，使得后续的定位过程这么曲折。

### 问题定位的通用模型
上面问题定位的过程，每一次迭代我都是在重复几个步骤，对这些步骤加以抽象就可以表示为下图，对于大部分的问题定位，都会经历这些步骤。

![debug-model](http://public.litong.life/blog/debug-model.png?imageMogr2/quality/100/thumbnail/600x){: width='500'}

而我们把所有迭代叠加到一起，会发现整个解决问题的过程其实是一棵决策树，我们通过问题表现来获得输入，再通过原因分析生成新的决策节点，然后通过解决方案筛选节点，最后验证获取新的问题表现，这样形成闭环反复迭代，最终使问题得以解决。

![tree](http://public.litong.life/blog/strategy-tree.png?imageMogr2/quality/100/thumbnail/600x)

### 更多的思考
虽然最终问题得以解决，但也暴露出两个问题:
1. 我个人对硬件型号会影响浏览器版本的知识盲区，导致走了很多弯路。
2. 缺失日志上报功能。如果有日志收集和上报能力，我们可以通过日志直接获取到用户端的环境信息（机型、系统、浏览器userAgent），不仅能快速发现是由于浏览器内核版本过低导致的兼容性问题，还能将最小介入原则发挥到极致。

### 工具
在解决问题的过程中，合理的使用工具可以事倍功半，我在这里整理出在解决这个问题过程中用到的几个工具

| 名称                                             | 功能描述                                                                      | 是否免费 |
|:------------------------------------------------|:-----------------------------------------------------------------------------|:------- |
| [wetest](https://wetest.qq.com/)  | 使用云平台上的真机调试，有海量的机型 | 收费(新用户赠送60分钟试用时长) |
| [vconsole](https://github.com/Tencent/vConsole) | 在移动端网页中显示类似于chrome devtools的小工具，可以查看控制台信息、系统信息、网络请求等 | 免费 |
| [caniuse](https://caniuse.com/)  | 查看某一特性在各个浏览器中支持的版本 | 免费 |
| [chromeStatus](https://chromestatus.com/feature) | 查看某一特性在各个平台的chrome上开始支持的版本（比caniuse更准确）| 免费 |
| [chromeRelease](https://chromereleases.googleblog.com/) | 搜索指定版本chrome的发布时间 | 免费 |
| [browserstack](https://www.browserstack.com/) | 兼容性调试的神器，与云真机类似，不过是专门针对浏览器的，可以选择任意系统下、任意版本的主流浏览器（Chrome、Safari、Firefox、Edge、Opera），并且**支持调试运行在localhost上的项目**，唯一缺点是收费 | 收费(每个浏览器版本60秒试用时长) |
