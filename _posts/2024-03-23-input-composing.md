---
layout: post
title: Input组件无法输入中文的诡异问题
date: 2024-03-23
Author: 天枫皓月 
tags: [技术,兼容性,输入]
---

## 问题描述
最近一个接入我们组件库的业务收到用户反馈，在iphone SE手机上存在多个输入框时，只有第一个输入框可以正常输入，剩余的输入框均无法输入（表现键盘上按任何按键都无法显示到输入框内，v-model值也没有更新）。业务组的同学找到了可复现的测试机，先进行了排查，并额外提供了一些信息:

### 关键信息
1. 特定机型
2. ~~多个输入框，第一个可以输入，后面的无法输入~~（经业务组同学排除）
3. 原生的input可以正常输入，使用我们提供的input组件无法输入
4. 金额输入框可以正常输入，普通文本输入框无法输入

## 分析定位

### 问题表现
业务组的同学已经帮排除了大部分的干扰因素，又加上现场有可复现的测试机，我经过简单的复现，发现在使用英文输入法时可以正常输入，这样进一步缩小了问题的范围，最终的问题表现可以描述为：
> 在iphone SE上(userAgent: Mozilla/5.0 (iPhone; CPU iPhone OS15_4 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148)，input组件在使用中文输入法（本质是**IME**，后面会展开来讲）时无法输入

具体的表现如下图

![bug-case](https://public.litong.life/blog/bug-case.GIF)

### 原因分析
通过对问题表现和关键信息的分析，可以得出两个结论：
1. 特定机型 -> 兼容性问题
2. 原生input正常 -> 问题出现在我们input组件的特有逻辑中

而我们的Input组件大部分输入行为与原生组件保持一致，对输入行为进行干预的只有扩展出的formatter和parser机制，这套机制可以通过formatter将v-model绑定的值进行格式化，再填充到input.value中，用户在输入框里看到的是格式化后的内容，而在输入框中进行输入，先经过parser将显示的值进行解析还原，再通过update:modelValue事件通知给v-model，v-model中接收到的是parser后的值。借由这套机制，可以扩展出很多定制化的输入框，比如：

| 类型 | 展示 |
|:----|:-----|
| 金额  | ![20240323172217](https://public.litong.life/blog/20240323172217.png){:width='300'} |
| 卡号  | ![20240323172806](https://public.litong.life/blog/20240323172806.png){:width='300'} |
| 有效期 | ![20240323173047](https://public.litong.life/blog/20240323173047.png){:width='300'} |
| 自定义 | ![20240323173222](https://public.litong.life/blog/20240323173222.png){:width='300'} |

这套机制的整体流程如下图所示：

![input-flow](https://public.litong.life/blog/input-flow.png?imageMogr2/quality/100/thumbnail/1200x){: width='700'}

从流程图上可以看到对input事件进行了拦截，并且在里面加工了input.value，这一步是最有可能对用户输入产生影响的，部分核心代码如下:
```ts
async function onInput(e: InputEvent) {
  // IME输入中间过程则直接返回，在输入完成后再进行处理
  if (e.isComposing) {
    return
  }
  const target = e.target as HTMLInputElement
  await nextTick()
  target.value = formattedValue.value
}
```
为了帮助大家更透彻的理解这里的问题，有必要先了解一下什么是IME。
##### 什么是IME
IME是Input Method Editor的缩写，详细解释见[维基百科IME](https://zh.wikipedia.org/wiki/%E8%BE%93%E5%85%A5%E6%B3%95){:target="_blank"}。

简单来讲，可以把输入法看作是一个编辑器程序，在input中进行输入时，每次触发按键会暂存在编辑器中(称为**composing**状态)，此时input.value中存入的是对应按键的字母，但并不会填入到输入框中，当IME通过暂存的按键匹配到对应的字符或手动结束composing状态(**compositionEnd**状态)(比如中文输入法通过空格或数字进行选字后)，再将结果填入到input中。在使用英文输入法时，每个按键对应一个字符，所以不存在暂存的问题，但使用中文、日文等输入法时，常常多个按键组合输入才能生成一个字符，如下图所示：

![20240323181011](https://public.litong.life/blog/20240323181011.png)

这个过程有一个关键问题，就是**每次按键都会触发input事件**，但只有在compositionEnd状态才会显示在input中，如果在**composing过程中**，通过代码**更改了input.value**，会导致暂存的按键被**重置**，IME永远无法拿到composing过程中已输入的按键来匹配字符，表现出来就是input中的值一直没有变化，也就是无法输入。而Input组件的bug刚好符合这几个特点:
1. 中文输入法
2. 在input事件中更改了input.value
3. 表现为无法输入

为了更直观的表现这个过程，我用playground做了一个示例。[点击查看](https://playcode.io/1811293){:target="_blank"}

![input-zh](https://public.litong.life/blog/input-zh.GIF){: width="500"}

对这个示例稍加改造，看看在input事件中通过代码更改input.value后的效果。[点击查看](https://playcode.io/1811293?v=2){:target="_blank"}

![input-zh-bug](https://public.litong.life/blog/input-zh-bug.GIF){: width="500"}

这个场景和业务反馈的问题一模一样!

但formatter机制要求必须在input事件中对input.value进行加工，这样是不是没有办法解决了呢？其实是有办法的，还记得前面核心代码的开头有一段判断逻辑，而这段代码就是用来处理上述问题的。e.isComposing为true时表示了这是IME输入的中间状态，这时候不用对input.value做任何处理。
```ts
if (e.isComposing) {
  return
}
```
既然已经处理了，为什么还会有bug呢？还记得前面原因分析时得出的两个结论吗？由于只在特定机型复现，说明有兼容性问题，我猜测在部分机器上InputEvent不支持isComposing属性，去caniuse查一下支持情况。

![20240323203941](https://public.litong.life/blog/20240323203941.png){: width="600"}

可以看出，Safari iOS是从16.4开始支持的，而复现的设备是15.4，确定问题是由于**InputEvent不支持isComposing属性**导致的。

### 解决方案
组合输入存在中间状态，这是输入法的固有机制，iOS不太可能没有考虑到，所以大概率iOS有相关的api，只是api的实现上有一定的差异。基于这个思路我去搜索了相关的文章，果然发现了这样一组事件。[详情见MSDN](https://developer.mozilla.org/en-US/docs/Web/API/Element/compositionstart_event){:target="_blank"}
> * compositionstart - The compositionstart event is fired when a text composition system such as an input method editor starts a new composition session.
> * compositionupdate - The compositionupdate event is fired when a new character is received in the context of a text composition session controlled by a text composition system such as an input method editor.
> * compositionend - The compositionend event is fired when a text composition system such as an input method editor completes or cancels the current composition session.

老规矩，先去查一下兼容性。可以看到很早的版本就支持了。

![20240323205545](https://public.litong.life/blog/20240323205545.png){: width="600"}

具体的解决思路，设置一个标识变量isComposing，在compositionstart事件触发时设为true，compositionend事件触发时设为false，input事件中判断这个标识变量为true时，则直接return。修改后的核心代码如下:
```ts
const isComposing = ref(false);

function onCompositionStart() {
  isComposing.value = true;
}
function onCompositionEnd() {
  isComposing.value = false;
}
onMounted(() => {
  inputRef.value.addEventListener('compositionstart', onCompositionStart);
  inputRef.value.addEventListener('compositionend', onCompositionEnd);
});

onUnmounted(() => {
  inputRef.value.removeEventListener('compositionstart', onCompositionStart);
  inputRef.value.removeEventListener('compositionend', onCompositionEnd);
});

function onInput(e) {
  if (e.isComposing || isComposing.value) {
    return;
  }
  // ....
}
```
##### 可以做的更好?
接下来修复、发补丁版本、业务验证通过，至此这个问题已经解决。但是否还有优化的空间？其实这段代码里还有一个问题，对于原生InputEvent已经支持了isComposing属性的浏览器来说，监听compositionstart和compositionend事件属于重复工作，可以仅对不支持composing属性的浏览器做处理，那关键问题就是如何判断是否支持呢？显然不能通过浏览器版本来判断，可以通过**Feature Detection(特性嗅探/特性检测)**来判断。思路就是既然isComposing是InputEvent的属性，那可以创建一个InputEvent实例，如果原生支持，isComposing属性的值应该为true/false，不支持则为undefined，通过判断实例上isComposing属性类型是否为boolean间接判断浏览器是否支持该特性。核心代码如下:
```ts
const supportComposing = typeof (new InputEvent('input')).isComposing === 'boolean';
if (!supportComposing) {
  onMounted(() => {
    inputRef.value.addEventListener('compositionstart', onCompositionStart);
    inputRef.value.addEventListener('compositionend', onCompositionEnd);
  });

  onUnmounted(() => {
    inputRef.value.removeEventListener('compositionstart', onCompositionStart);
    inputRef.value.removeEventListener('compositionend', onCompositionEnd);
  });
}

function onInput(e) {
  supportComposing && isComposing.value = e.isComposing;
  if (isComposing.value) {
    return;
  }
  // ....
}
```
### 影响范围
这个bug会影响所有需要输入中文的输入框，对于业务来讲是在核心路径上，对用户造成比较大的影响，需要对影响范围做一个评估。之前通过caniuse已经查到iOS16.3及以下不支持，再在苹果官网上查询iOS各个版本的使用率。[查看详情](https://developer.apple.com/support/app-store/){:target="_blank"}。
* 16以下占 - 4%
* 16-16.3的占比无法查到，粗略猜测 - 4%

大盘的占比在**4%~8%**左右，业务app的系统版本分布数据我目前还拿不到，只能用大盘数据做个粗略的评估。

![20240323204541](https://public.litong.life/blog/20240323204541.png){: width="300"}