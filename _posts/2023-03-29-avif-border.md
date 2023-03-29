---
layout: post
title: avif格式图片兼容性诡异紫色边框问题
date: 2023-03-29
Author: 天枫皓月 
tags: [技术,avif,兼容性]
---

在部分安卓机上avif图片会出现诡异的紫色边框(cdn判断是支持avif格式的)

目前发现的机型有**pixel2**和**pixel3**

![](https://raw.githubusercontent.com/tonliver/tonliver.github.io/master/assets/imgs/202303291724306.png)
![](https://raw.githubusercontent.com/tonliver/tonliver.github.io/master/assets/imgs/202303291724283.png)

### 原因分析
TImage同学给出的结论
本质原因是因为AVIF解码得到YUV数据，YUV数据转RGB时，终端上应该是默认按8对齐填充的宽，所以导致不是8的整数倍的时候，右侧就出现了紫边。
图片宽度除以8，余数即为紫边的宽度

### 解决办法
通过thumbnail参数，返回一个最接近原图尺寸的，能被8整除的宽度的图片
可用以下公式进行计算
```
Math.round(originalWidth / 8) * 8
```
例如：下图的原尺寸是750，750 / 8 = 93.7 => 94 * 8 = 752，于是thumbnail的值就是752x

https://vfiles.gtimg.cn/vupload/20220919/596d2d1663581301955.png`?imageMogr2/thumbnail/752x`

**注意：但是如果是通过背景图写在css中的，并被webpack的loader编译后，会将`imageMogr2/thumbnail/752x`中的`/`url encode，导致参数识别失败，会返回原图尺寸**

```css
background-image:url('https://vfiles.gtimg.cn/vupload/20220919/596d2d1663581301955.png?imageMogr2/thumbnail/752x')
```
这种情况可以通过手动下载缩放后符合尺寸的图，再重新上传，这样就能得到不用参数的符合尺寸的图了。

```
https://vfiles.gtimg.cn/vupload/20220919/596d2d1663581301955.png?imageMogr2/thumbnail/752x/format/png
https://vip.image.video.qpic.cn/vupload/20230329/5ed2ea1680054190905.png
```