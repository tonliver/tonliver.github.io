---
layout: post
title: 关于justify-content的坑
date: 2024-10-11
Author: 天枫皓月
tags: [兼容性,CSS]
---

# 问题描述
前两天接到业务反馈，nav-bar组件在**部分机型**上右侧图标**位置异常**。

大概的表现如下图所示

<img src="https://public.litong.life/blog/20241011164818.png" height="80" />

# 问题分析
通过反馈可以提取到两条有用的信息。
1. 部分机型
2. 位置异常

基于以往的经验，从这两条信息可以快速得出定位问题的大致方向。

* 只在部分机型复现 -> 有可能是兼容性问题 -> 浏览器内核版本 -> UA
* 位置异常 -> 样式问题 -> CSS

于是让业务提供了UA，浏览器内核版本是chrome 69，属于比较旧的版本，更有可能出现兼容性问题。

<img src="https://public.litong.life/blog/20241011165838.png" width="400" />

有了内核版本就可以去下载对应版本的浏览器进行复现，很顺利的复现了。既然是样式问题，就优先查看CSS，这两个icon的父容器使用了flex布局，并且设置了`justify-content: end;`让子元素从结束位置排列，但从表现来看，在有问题的浏览器中，子元素是从起始位置排列的，不符合预期。因此基本可以大胆猜测问题是`justify-content: end`引起的。

<img src="https://public.litong.life/blog/20241011161856.png" width="400" />

上面的只是猜测，仍然需要小心求证，先通过查阅MDN，确认下我对这个属性的用法是否有误解。

> end
>
> The items are packed flush to each other toward the end edge of the alignment container in the main axis.

> flex-end
>
> The items are packed flush to each other toward the edge of the alignment container depending on the flex container's main-end side. This only applies to flex layout items. For items that are not children of a flex container, this value is treated like end.

通过文档可以看到用法应该是没有问题的，但同时发现还有另一个属性值`flex-end`，是专门用于flex布局的，在我们的场景中使用`flex-end`更适合。按文档的说法，如果不是flex布局的子元素，会与`end`表现一致，但并未提到flex布局中不能使用`end`，而且在较新版本的浏览器中`end`表现确实是符合预期的。

确认了用法无误，再看看这个属性是否有版本兼容问题，通过查看caniuse，发现`end`是在`chrome >= 93`及`ios safari >= 15.3`开始支持的，业务反馈的机型是`chrome 69`，确实还未支持。

<img src="https://public.litong.life/blog/20241011175919.png" width="500" />

再看属性值`flex-end`在caniuse中没有特殊说明，那应该是和flex支持justify-content属性同时支持的。

* chrome >= 21
* ios safari >= 7

<img src="https://public.litong.life/blog/20241011180708.png" width="500" />

这样的话，把`end`换成`flex-end`就应该能解决这个bug了。

# 结论
## 验证
写了个小demo，再次验证上面的推测和结论。

### iOS safari 14 VS. safari 15.4

<img src="https://public.litong.life/blog/20241011153112.png" width="500" />

### Android chrome 76 VS. chrome 114

<img src="https://public.litong.life/blog/20241011181949.png" width="500" />

## 结论
1. ios >= 15.4和chrome >= 93，flex布局中，`justify-content: end;`与`justify-content: flex-end;`表现一致，都符合预期。
2. ios < 15.4和chrome < 93，flex布局中，`justify-content: end;`无法生效，`justify-content: flex-end;`符合预期。
3. 所有版本的grid布局，`justify-content: end;`都符合预期。

**因此，对`justify-content`设置值时，为了保障最大程度的兼容性，如果是flex布局，使用`flex-start`或`flex-end`，其它布局方式则使用`start`、`end`、`left`或`right`**。