---
layout: post
title: InputEvent丢失inputType
date: 2024-09-10
Author: 天枫皓月
tags: [技术,兼容性,输入]
---

## 问题描述
### 业务场景
当input使用了自定义键盘，点击删除键时，无法正常删除输入框中的内容；而使用系统键盘时，表现正常。

![card-input](https://public.litong.life/blog/card-input.gif)

## 复现环境
ios < 16.5

## 原因分析
经过调试，发现在ios部分机型上InputEvent的inputType为空字符串，判断键入删除操作的逻辑失效，导致无法删除。

![20240910181250](https://public.litong.life/blog/20240910181250.png)

Sail组件库有一个设计原则，就是尽量与系统原生行为保持一致。这里为了让input在使用了自定义键盘后，行为与使用系统键盘一致，按下删除键时，会手动创建一个InputEvent事件，并由input元素dispatch出去。这样对于调用者来说，不管是系统键盘，还是自定义键盘，在按下删除键时，如下代码都会有一致的行为。
```ts
input.addEventListener('input', (e: InputEvent) => {
  console.log(e.inputType); // deleteContentBackward
});
```
### 问题本质 
当**ios < 16.5**，通过`new InputEvent('input', { inputType: 'deleteContentBackward' })`手动创建的事件，监听后`event.inputType`为空字符串，而期望得到`deleteContentBackward`。经实测，不仅是`deleteContentBackward`，所有的inputType值都会变为空字符串。

#### 代码示例
[演练场传送门](https://playcode.io/1993768){:target="blank"}

```vue
<template>
  <button @click="clicked">触发事件</button>
  <input ref="el" @input="onInput" />
</template>

<script lang="ts" setup>
const el = ref();

function clicked() {
  const event = new InputEvent('input', { inputType: 'deleteContentBackward' });
  el.value.dispatchEvent(event);
}

function onInput(e: InputEvent) {
  console.log(e.inputType); // ios < 16.5为空字符串,其它版本为'deleteContentBackward'
}
</script>
```

通过查阅MDN、caniuse、w3c文档可以看到inputType是InputEvent的标准属性，各主流浏览器在很早均实现了该接口，并无兼容性问题。

[caniuse传送门](https://www.w3.org/TR/uievents/#inputevent){:target="blank"}

![caniuse](https://public.litong.life/blog/20240910165045.png)

[W3C传送门](https://www.w3.org/TR/uievents/#inputevent){:target="blank"}

![w3c](https://public.litong.life/blog/20240910170041.png)

[MDN传送门](https://developer.mozilla.org/en-US/docs/Web/API/InputEvent/inputType)

![MDN](https://public.litong.life/blog/20240910172728.png)

#### 实测表现
* 系统键盘，所有系统下都能正常读取`event.inputType`。
* 手动dispatch的InputEvent事件，Android > 4.4.4(与caniuse描述一致)，ios >= 16.5能正常读取到。
* 手动dispatch的InputEvent事件，ios < 16.5获取到空字符串。

由此判断应该是ios系统的bug，苹果在16.5修复了此问题。

## 解决方案
通过文档可以发现InputEvent还有一个属性`data`，用来设置输入的内容，可以借助这个属性来传递inputType。

![20240910173017](https://public.litong.life/blog/20240910173017.png)

### 代码
以删除操作为例

```ts
// 创建事件
const event = new InputEvent('input', {
  inputType: 'deleteContentBackward',
  data: 'deleteContentBackward',
});

el.dispatchEvent(event);

// 监听事件
el.addEventListener('input', (e: InputEvent) => {
  if (isDeleteContentBackward(e)) {
    // 业务逻辑
  }
})

/**
 * 判断是否是删除事件
 */
const isDeleteContentBackward = ({ inputType, data }: InputEvent) => {
  // 如果inputType有值，则优先用inputType判断
  if (inputType === 'deleteContentBackward') {
    return true;
  }
  /**
   * 当inpuType为空时，通过e.data === 'deleteContentBackward'做二次兼容判断
   */
  if (!inputType && data === 'deleteContentBackward') {
    return true;
  }

  return false;
};
```