---
layout: post
title: 如何在真机上调试网页
date: 2024-05-25
Author: 天枫皓月
tags: [原创,技术,调试,远程,真机,devtools]
---
# 背景
在PC端使用H5技术开发和调试时，由于主流浏览器Chrome、Safari、Firefox的devtools已经做的比较成熟，提供了较好的开发体验。但在移动端开发时，无法直接使用devtools。虽说H5技术具有跨平台性，理论上可以在PC上开发，直接运行于移动端，但事实是移动端webview内核的实现并非与PC端一模一样，有很多兼容性的问题只能在移动端重现，因此有一些场景不可避免的需要在移动端真机上进行开发和调试。那有没有办法让移动端H5开发和PC上有一样的体验呢？答案就是——Remote Debugging(远程调试)
# Remote Debugging
## 原理
本文重点介绍如何使用，关于devtools的工作原理，后面有时间会专门介绍。
![20240525212520](https://public.litong.life/blog/20240525212520.png)

## Android
### 一. 打开手机USB调试功能
1. 设置 -> 关于手机 -> 系统版本。
2. 连续点击***系统版本***，一般5次以内就会弹出“已进入开发者模式”的toast。
3. 设置 -> 系统 -> 开发人员选项 -> 打开***USB调试***

![enable-dev-mode](https://public.litong.life/blog/enable-dev-mode.gif)

### 二. USB连接手机与电脑
1. 用数据线连接手机与电脑
2. 手机上弹出如下对话框，一定要选确定/允许。
![enable-use-debug](https://public.litong.life/blog/enable-use-debug.jpg){:width="300"}
3. 电脑上也会弹出授权对话框，同样要选允许。
![allow-connect](https://public.litong.life/blog/allow-connect.png){:width="300"}

### 三. 开始调试
1. 手机chrome浏览器打开待调试页面。
2. 在PC chrome浏览器地址栏输入`chrome://inspect/#devices` -> 找到待调试页面 -> 点击`inspect`打开devtools。
![20240527134335](https://public.litong.life/blog/20240527134335.png){:width="500"}
![20240527134524](https://public.litong.life/blog/20240527134524.png){:width="700"}
3. 用chrome devtools调试
![android-remote-debugging](https://public.litong.life/blog/android-remote-debugging.gif)

## iOS
### 一. 开启调试工具
1. 手机端: 设置 -> Safari -> 高级 -> Web Inspector
![enable-web-inspector](https://public.litong.life/blog/enable-web-inspector.jpg){:width="350"}
2. PC端: 打开Safari -> 菜单 -> Safari浏览器 -> 设置 -> 高级 -> 显示网页开发者功能
![20240527150700](https://public.litong.life/blog/20240527150700.png){:width="600"}

### 二. USB连接手机与电脑
### 三. 开始调试
1. 手机Safari浏览器打开待调试页面。
2. 打开PC safari浏览器 -> 开发 -> 选取对应手机 -> 点击待调试页面。
![20240527152701](https://public.litong.life/blog/20240527152701.png){:width="600"}
3. 用safari devtools调试
![ios-remote-debugging](https://public.litong.life/blog/ios-remote-debugging.gif)

## 用chrome devtools调试手机safari网页
我身边大部分前端开发同学都习惯用chrome开发，我个人也觉得chrome devtools要更好用一点。我再介绍一种用chrome devtools调试手机端safari网页的方法。

涉及到两个关键工具
1. remotedebug-ios-webkit-adapter
2. ios-webkit-debug-proxy

### 原理
![20240527154739](https://public.litong.life/blog/20240527154739.png){:width="700"}

### 安装
```bash
brew update
brew unlink libimobiledevice ios-webkit-debug-proxy usbmuxd
brew uninstall libimobiledevice ios-webkit-debug-proxy usbmuxd   
brew install --HEAD usbmuxd
brew install --HEAD libimobiledevice
brew install ios-webkit-debug-proxy

npm i remotedebug-ios-webkit-adapter@next -g
npm update remotedebug-ios-webkit-adapter -g
```

### 启动
```bash
remotedebug-ios-webkit-adapter is listening on port 9000
```

### 配置
![20240527161021](https://public.litong.life/blog/20240527161021.png){:width="500"}

### 开始调试
配置好之后和在android chrome上远程调试一样，可以看到待调试页面出现在了列表中，点击inspect。
![20240527161400](https://public.litong.life/blog/20240527161400.png){:width="350"}

![ios-chrome-devtools](https://public.litong.life/blog/ios-chrome-devtools.gif)