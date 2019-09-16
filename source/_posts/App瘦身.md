---
title: App瘦身
date: 2019-09-02 17:07:21
tags: iOS
---

## 1 分析当前ipa的组成

一般一个ipa会包含：

1） 资源文件

- 本地文件：数据、配置、数据库等等
- 字体文件
- 图片资源

2)  源代码

通过生成linkmap文件，分析源代码生成的编译文件的大小。在Build Settings中Write Link Map File设置为Yes (记住release时候不要设置为Yes)。

 

编译之后会在build目录中生成两个LinkMap文件： XXX-LinkMap-normal-i386和XXX-LinkMap-normal-x86_64，分别代表在模拟器中32位和64位指令集生成的LinkMap文件。关于LinkMap的文件详细结构解释可以参考：http://blog.cnbang.net/tech/2296/

LinkMap会包含每个可执行文件的偏移量及大小，所以可以很方便的知道每个可执行文件的大小。可以使用LinkMap分析工具：https://github.com/huanxsd/LinkMap

![img](https://images2017.cnblogs.com/blog/746857/201709/746857-20170906191955007-1995207350.png)

 

## 2 资源瘦身

### 无用的图片文件

查找无用的图片文件，使用LSUnusedResources（https://github.com/tinymind/LSUnusedResources）

![img](https://github.com/tinymind/LSUnusedResources/raw/master/LSUnusedResourcesExample.gif)

 

### 无损压缩图片

使用ImageOptim（https://github.com/ImageOptim/ImageOptim）进行png文件的无损压缩

![img](https://images2017.cnblogs.com/blog/746857/201709/746857-20170906192458647-804731014.png)

 

### WebP图片压缩

WebP是Google提供的一种图片编码格式，通常情况下WebP格式的图片是原始JPG/PNG图片的1/3，所以对于重度依赖图片显示的应用，转换使用WebP可以节省大量的网络传输数据和时间。对于APP瘦身，使用WebP格式可能是一种方式，可以使用WebP格式的图片替代现有的图片资源，可以一定程度的节省空间。

使用WebP转换工具（https://developers.google.com/speed/webp/docs/precompiled）尝试转换了几张较大的图片，效果如下：

![img](https://images2017.cnblogs.com/blog/746857/201709/746857-20170906192710054-1516701797.png)

 

iOS原生并不支持WebP格式加载，需要引入SDWebImage/WebP，详细可以参考：http://blog.devzeng.com/blog/ios-webp-usage.html

```
`NSString *path = [[NSBundle mainBundle] pathForResource:``@"logo"` `ofType:``@"webp"``];``NSData *data = [[NSData alloc] initWithContentsOfFile:path];``UIImage *img = [UIImage sd_imageWithWebPData:data];``self.imageView.image = img;`
```

使用WebP格式的图片，似乎就抛弃了iOS @2x @3x按照设备加载对应图片的机制，所以应该还可以删除所有@2x图片，不过加载速度比原生较慢。

 

## 3 代码瘦身

### AppCode代码静态检查

AppCode提供了非常强大的代码静态检查工具，使用Inspect Code，可以找到很多代码优化的地方。可以参考这篇介绍：[AppCode inspections for your code perfection](https://blog.jetbrains.com/objc/2014/01/appcode-inspections-for-your-code-perfection/)

 

### 清除无用代码

AppCode搜索出来的无用的Class，会有误报需要仔细检查每一个报错的代码。

使用Fui（https://github.com/dblock/fui）查找发现下列无用文件，同样需要double check避免误删

 

**清除无用的Import**

Fui（https://github.com/dblock/fui）可以用于查找无用的import，同时也提供[xcfui](https://github.com/jcavar/xcfui) 可以和Xcode集成。

 

**清除无用的Method**

\1. 基于AppCode的扫描定期做清理

\2. 这篇文章提供了一个很好的思路可以一键删除无用方法：http://www.jianshu.com/p/a53480ad0364

 

**查找相似的代码**

使用SameCodeFinder (https://github.com/startry/SameCodeFinder)可以查找到相似的代码，最后一位数字代表两个文件的海明距离，数字越小说明两个文件越类似。

 

 

**清理其他无用的代码** 

\1. 已经下线的陈旧代码，AB试验已经下线的代码

\2. 通过转H5、Hybrid或者RN实现的Native功能，可以定期清理

\3. 一些非核心Hybrid或者RN模块，可以考虑不要打包进入APP，通过动态下发的方式获取

\4. 代码的重构，UI组件、业务逻辑的重用等等

##  

## 4 一些参考文章

- iOS可执行文件瘦身：http://blog.cnbang.net/tech/2544/
- iOS APP瘦身实践：http://www.jianshu.com/p/c94dedef90b7，资源优化、编译器配置优化、可执行文件优化
- 滴滴出行iOS端瘦身实践: http://gmtc.geekbang.org/#schedule, 提供了查找无用图片的工具、WebP图片压缩、基于clang plugin实现查找无用代码（https://github.com/kangwang1988/XcodeZombieCode）、查找类似代码（https://github.com/startry/SameCodeFinder）
- [基于clang插件的一种iOS包大小瘦身方案](http://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&mid=2651112856&idx=1&sn=b2c74c62a10b4c9a4e7538d1ad7eb739)
- [减小ipa体积之删除frameWork中无用mach-O文件](http://jaq.alibaba.com/community/art/show?articleid=229)