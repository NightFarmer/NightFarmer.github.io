---
title: 使用v8兼容包对图片进行高斯模糊处理/毛玻璃效果
date: 2016-09-09 15:58:15
tags: [Android, 特效, RxAndroid]
category: Android
---

android中实现毛玻璃效果的方法比较多, 有用java实现图片处理算法的, 也有把算法用c/c++实现并用jni调用的, 而实现毛玻璃的开源库在github上也有不少. 
其实google的官方sdk中也为我们提供了这样的工具, 本着能用官方尽量不自己实现,能自己实现尽量不用第三方的原则, 官方的实现方式当然是要尝试一下的.
同时, 本例中的拖拽进度和图片的处理以及回显是通过RxJava放在不同的进程中处理的, 如果不熟悉Rx框架可以补一下, RxJava用于异步操作以及事件流的处理非常好用.
上图:
![高斯模糊示例](http://nightfarmer.github.io/public/static/image/BlurDemo.gif)
<!-- more -->


