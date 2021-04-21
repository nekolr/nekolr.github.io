---
title: 如何构建自定义的 jQuery
date: 2018/3/23 15:12:0
tags: [jQuery]
categories: [jQuery]
---

在我大三的时候就开始使用 jQuery 了，到今天这才不过两三年的时间，前端已经发生巨大的变化：ES 标准不断改进，各种框架、工具层出不穷等等。一些新的项目，新的需求已经完全不需要依赖 jQuery 了。

<!--more-->

回想我用 jQuery 的原因，我简单总结如下：  
- 强大的选择器
- 简化了 DOM 操作
- 处理了令人头疼的浏览器兼容性问题
- 封装良好的 AJAX
- 丰富的插件支持

但是随着 ES 标准的发展，各个浏览器厂商的支持，jQuery 时代遇到的这些问题在现在都几乎不是问题了。新的 DOM 标准借鉴 jQuery 加入许多新的方法；IE6 等浏览器逐渐退出历史舞台，以及主流浏览器对 ES 标准的支持，即便有兼容性问题，也可以使用 `polyfill/shim` 实现来解决。况且新的框架如 React、Angular 取代 jQuery 是完全不在同一层面的。

总的来说，jQuery 虽然已经过时，但是老当益壮，我们仍然可以拿来解决生产问题，当然我们这时就可以很大胆的思考是否有比 jQuery 更好的方案。

前面说了这么多，更多的是缅怀，接下来开始看问题。我们经常吐槽 jQuery 的庞大（也许与 Ext 相比反而很轻量）。最新的版本 jquery-3.3.1.js 大小有 265KB 之多，即便是压缩后的文件也有 80 多 KB。当然如果服务器开启 gzip，文件大小会小很多，但是有没有别的方式来缩减 jquery 的大小呢，比如说砍掉部分 jquery 的功能？

不瞒你说，博主之前还尝试过提取 jquery 的 ajax 方法，但是里面牵扯甚多，最终不得不放弃。一个偶然的机会发现 jquery 可以选择剔除自己不需要的模块，自己构建 jquery，在这里记录一下。

# 准备阶段
在开始前，你要先了解 `git`、`node.js`、`npm` 和 `grunt`，当然只需要了解它们是做什么的并安装在你的系统中就可以了。

# 构建
- 将 jquery 的仓库 clone 到本地
```
git clone git://github.com/jquery/jquery.git
```
- 使用 npm 安装依赖
```
cd jquery && npm install --registry=https://registry.npm.taobao.org
```
- 安装 grunt，或者直接使用命令 `npm run build` 会自动安装
```
npm install -g grunt-cli --registry=https://registry.npm.taobao.org
```
- 使用 grunt 命令去掉不需要的模块，这里用 ajax 举例
```
grunt custom:-ajax
```

现在你可以去 dist 目录看看了，你会发现构建出来的 jQuery 代码少了很多，至于可以剔除哪些模块，可以看 jQuery 的 github 仓库的说明：<https://github.com/jquery/jquery>

![缩减前](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/ppg.png)

![缩减后](https://cdn.jsdelivr.net/gh/nekolr/image-hosting@201911242020/2018/04/14/Nkr.png)
