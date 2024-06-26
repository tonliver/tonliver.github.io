---
layout: post
title: 异常设计
date: 2023-03-21
Author: 天枫皓月 
tags: [原创,技术,异常处理]
---

今天给组里的同学们分享了《异常设计与防御性编程》，程序员善于消灭异常，而惧于制造异常，以致于在工作中，常见到一大段代码被`try catch`包裹，事实上异常也是我们程序的一部分，在特定的场景，不仅不能消灭异常，还要刻意的设计抛出异常。

### 为什么需要设计异常

之所以要设计异常，我认为主要由两个原因
1. 代码是分层架构的
2. 异常只能在合理的层次处理，大部分时候抛出异常和处理异常不在同一个抽象层级。

如下图，多个模块处于不同的抽象层次，假设C模块出现了预期外的行为，但自己又无法决定如何处理，只有调用方B模块希望这个预期外的行为弹一个toast提示用户，这个C模块就需要通过某种方式告诉B模块，这就是抛出异常。

![模块](/assets/imgs/module.png)

举一个具体点的例子，我们有一个movieService模块，用于请求影片列表，现在有两个页面都在调用这个模块

**页面A**

影片列表不是关键模块，如果列表数据获取异常，则可不显示这个模块

**页面B**

影片列表是关键模块，如果列表数据为空则不显示，如果获取列表失败，则需要自动重试

movieService处于较底层，它的职责是请求列表数据，知道数据请求成功或是失败，但它无法决定如何处理。而页面A和B根据业务场景，知道数据获取失败时该如果处理。这种场景movieService就应该只关注如何获取数据，如果获取到的数据不符合预期，那就抛出异常，由上层决定如何处理。

### 异常的分类

1. 黑名单 - 决不允许出现的情况
    - 参数校验
    - 环境检测
2. 白名单 - 出现了预期外的情况
    - 外部依赖
    - 接口的ret/retcode/code

如下图，被除数为0，这是绝不允许出现的情况，所以当接受到被除数为0时，需要明确的抛出异常

```typescript
/**
 * 除法运算
 * @param {number} a 被除数
 * @param {number} b 除数
 */
function divide(a: number, b: number) {
  if (b === 0) {
    throw new Error('除数不能为0');
  }

  return a / b;
}
```

### 需要设计异常的情况

1. 绝不允许出现的情况
2. 当前层级无法处理的预期之外的情况
3. 发生了无法忽略的错误，需要通知程序其他部分进行处理

### 合理的处理异常

既然设计异常是必需的，那如何合理的处理异常呢，我觉得主要从两方面考虑

##### 时机

1. 恰当的层级
2. 在第一个前提下，尽量接近异常发生的层级

##### 常用的方法

- 提示。往往是在接近用户的层级，需要给用户明确的提示，由用户来决定下一步操作。
- 兼容。比如设计一个Boolean类型的参数时，往往会赋一个默认值，用户即使传了true/false之外的值，也会强行做类型转换，使用代码继续运行下去。
- 记录日志。
- 降级。比如一个模块数据获取失败，可以不显示这个模块，或用缓存/兜底的数据进行展示。
- 传递。对异常进行二次加工，然后继续将异常抛出。

### 总结

不要轻意的吞掉异常，除非确定当前是合适的抽象层级。异常可以通过中断程序告诉我们程序出现了预期外的行为，而吞掉异常后，我们不知道预期外的行为发生，但事实上它却一直在发生，这才是毁灭性的灾难。