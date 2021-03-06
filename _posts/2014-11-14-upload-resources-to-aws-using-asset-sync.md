---
layout: post
title: "Upload resources to AWS using asset sync "
author: benko
---

### 目的

网站的规模逐渐变大，页面加载速度逐渐变慢，其中图片加载的时间是主要的问题。选择把js css 图片这些文件放到aws有一下几个原因：

1. 减少本服务器的负载.
2. 我们网站拥有多个域名，即使是同一个产品，在不同子站显示，图片都要重新加载一遍。一旦统一使用aws，不同子站都是请求同一图片地址，浏览器可以做缓存.
3. 避免重复繁琐的图片上传管理操作，直接通过代码管理.


### 使用[asset sync](https://github.com/rumblelabs/asset_sync)

使用的时候进行简单的配置即可，其中遇到一个问题，插件在dev & staging都可以正确的运行，在production时遇到错误提示

```ruby
[2014-11-12T04:17:44.818010 #788] ERROR -- : uninitialized constant AssetSync (NameError)
```

大致的原因：

由于js css 与图片是能够正确上传到s3的，所以说明插件在 precomplice之后有正确的运行，但是启动unicorn的时候报错，可能是启动的时候不会加载 asset group 的插件, initializar 文件夹内的代码还会加载, 导致AssetSync gem 没有初始化，但是代码中却没有对这个进行判断。

修改如下：

```ruby
if defined?(AssetSync)
...
end
```

### 使用后发现的问题，暂时搁置

1、无法使用gzip。使用的s3服务器中没有cloudfront，导致无法识别HTTP HEAD中Content-Encoding:gzip的请求，导致使用只能请求到非gz后的 js 与 css 文件，网站整体的pagespeed反而下降了。

2、工作流程导致无法运用。前端工程师不使用rails server进行代码的编写，例如图片的url需要写成asset_host:XXXX,导致使用asset_host指向s3后，他们无法在本地进行开发工作。

3、代码中历史遗留的问题。旧代码中保留了不少直接使用public的图片，而public的图片不能被上传到s3中，不能直接使用asset sync，需要进行大量的修改。




