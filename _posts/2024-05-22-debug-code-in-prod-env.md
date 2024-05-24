---
layout: post
title: 如何调试生产环境js代码
date: 2024-05-22
Author: 天枫皓月
tags: [原创,技术,调试,源码,sourcemap]
---
# 背景
这是一道在大厂前端面试中出现频率很高的“八股”题，虽然被打上“八股”的标签，但我个人认为这是一个非常实用，而且是合格的前端必须掌握的技巧之一。

前端由于其固有的技术特性，导致生产环境代码难以调试：
* 生产环境的代码是混淆压缩过的，缺乏可读性。
* JS代码是运行在用户浏览器中的，开发人员对运行时的代码没有控制权。

那么有没有办法让我们在生产环境通过devtools也能看到源码，并且设置断点进行调试呢？答案就是sourcemap。
# Sourcemap（源映射）
前端同学应该对sourcemap不陌生，我就不重复介绍了，推荐一篇文章[[What are source maps]](https://web.dev/articles/source-maps){:target="blank"}。

效果如下图，加载的是打包后的js，但借助sourcemap可以看到源码，并设置断点进行调试。

![debug-sourcemap](https://public.litong.life/blog/debug-sourcemap.gif)

## 如何使用
### 1. 添加sourceMappingURL注释
这是构建工具（vite、webpack、rollup等）开启了sourcemap后默认的处理方式。通过sourceMappingURL告诉devtools sourcemap文件路径。值得注意的是这里的路径可以是相对路径、在线url或file协议的本地路径。

![20240523160803](https://public.litong.life/blog/20240523160803.png)
#### 优点
构建工具天然支持

#### 缺点
需要将sourcemap文件和混淆后的代码一起发布出去，所有人都可以通过devtools看到源码，这是不可接受的。

### 2. 隔离部署sourcemap文件
上一个方法关键的问题是所有用户都能访问到sourcemap文件，那我们就稍加改进，把外网资源与sourcemap分离部署，将sourcemap部署在内网，这样只有内网的开发者可以访问到sourcemap，进行源码调试，而外网用户依旧只能看到混淆后的代码。

![网络隔离sourcemap](https://public.litong.life/blog/网络隔离sourcemap.png){:width='600'}

实现也非常简单，就是把生成的sourceMappingURL路径设置为内网地址。我以Vite的配置为例：
```ts
export default defineConfig({
  build: {
    sourcemap: true,
    rollupOptions: {
      output: {
        sourcemapBaseUrl: 'https://private-domain.com'
      }
    }
  }
}
```

生成的效果是这样
![20240524050840](https://public.litong.life/blog/20240524050840.png){:width="600"}

#### 优点
构建工具天然支持

#### 缺点
* 需要将sourcemap部署在内网，增加了部署和服务器成本。
* 内网所有用户都能访问到sourcemap，依旧存在安全风险。

### 3. 使用鉴权网关
基于方案2做进一步优化，引入鉴权网关，只有鉴权通过的内网用户才能访问到sourcemap文件

![部署分离(鉴权网关)](https://public.litong.life/blog/部署分离(鉴权网关).png){:width="600"}

#### 优点
* 前端部分与方案2一样。
* 鉴权网关对开发者透明。
* 灵活，可按个人、团队、角色等任意颗粒度实现精准权限控制。
* 通用解决方案，适合有一定规模的公司或团队。

#### 缺点
* 对公司基础建设有一定要求，对小团队或个人开发者不适用。

### 4. 使用devtools本地添加sourcemap
对于参与人数较少的项目，也可以不用将sourcemap文件发布到服务器，只需要生成到本地，然后通过devtools加载本地的sourcemap文件，这样sourcemap就仅对个人可见。

首先要改构建配置，只生成sourcemap文件，而不在js中添加sourceMappingURL注释
```ts
export default defineConfig({
  build: {
    sourcemap: 'hidden'
  }
}
```
#### Chrome
![20240524162553](https://public.litong.life/blog/20240524162553.png)

##### 优点
* 不需要将sourcemap文件发布到服务器。
* 不依赖sourceMappingURL注释，可随时替换sourcemap文件。

##### 缺点
* 只能在页面加载完成后替换，并且刷新页面后会丢失，无法调试刷新页面后自动执行的代码逻辑。

#### Safari
Safari devtools没有此功能╮(╯▽╰)╭

### 5. Local overrides(本地文件覆盖)
#### 原理
不上传sourcemap文件，利用devtools的本地文件覆盖功能，创建一个sourcemap的请求，并将本地的sourcemap文件作为请求内容返回。
#### Chrome
很可惜Chrome无法拦截Developer resources，所以也无法本地覆盖
![20240525044415](https://public.litong.life/blog/20240525044415.png){:width="700"}

#### Safari
![safari-local-overrides](https://public.litong.life/blog/safari-local-overrides.gif)

#### 优点
* 不需要将sourcemap文件发布到服务器。
#### 缺点
* Chrome不支持

### 6. HTTP sourcemap header
通过给混淆后js文件的请求http header添加sourcemap字段来指定sourcemap文件。详情见[MSDN SourceMap](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/SourceMap){:target="blank"}

最早是由firefox支持的，而Chrome和Safari都是最近的版本才开始支持，目前还无法大规模应用，就当了解一下。

![20240525052932](https://public.litong.life/blog/20240525052932.png){:width="700"}

![20240525054358](https://public.litong.life/blog/20240525054358.png){:width="700"}
#### Chrome
经实测，Chrome无法生效╮(╯▽╰)╭

#### Safari
Safari可以正常使用

### 7. 代理工具
代理工具有很多，像fiddler、whistle、charles。我个人强烈推荐[whistle](https://wproxy.org/whistle/){:target="blank"}，基于node实现的，可以直接通过npm安装，一看就是前端嫡系工具。

用代理工具管理sourcemap和devtools的本地覆盖的原理是类似的，都是通过更改网络请求让浏览器找到本地的sourcemap文件，但代理工具更加灵活和强大。话不多说，上实战。

1. 可以看到服务器上是没有这个sourcemap文件的。
![20240525060406](https://public.litong.life/blog/20240525060406.png){:width="600"}
2. 添加whistle规则，把这个请求代理到本地的sourcemap文件上。
![20240525060606](https://public.litong.life/blog/20240525060606.png)
3. 可以正常访问到sourcemap文件了。
![20240525060832](https://public.litong.life/blog/20240525060832.png){:width="600"}
![20240525061409](https://public.litong.life/blog/20240525061409.png){:width="600"}

但这样还不够优雅，如果一个项目里有多个js，就需要为每个js添加规则。
#### 使用通配符
将这个项目下所有的以.js.map结尾的请求都代理到本地dist目录下的对应文件

```bash
^page.works996.icu/sm/**.js.map /Users/litong/workspace/source-map-demo/dist/$1.js.map
```
#### 优点
* 灵活、通用，有无限的发挥空间。
#### 缺点
* 依赖whistle这个代理工具
* 有一丢丢学习成本。

## 总结
最后对上面提到的各个方法的优缺点做个更直观的对比，大家可以根据实际情况选用适用的方案。

![20240525070143](https://public.litong.life/blog/20240525070143.png){:width="700"}
