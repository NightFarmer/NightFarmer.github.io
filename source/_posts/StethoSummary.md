---
title: 使用来自Facebook的调试神器Stetho对Android应用程序进行调试
date: 2016-11-17 10:48:57
tags: [Android, 干货]
category: Android
---

UI、网络交互以及本地数据存储在当前的手机应用程序开发中已经是不可缺少的三个元素了，然而如何在开发过程中快速调试这三个元素则成为提高开发效率的必然途径。
在不是用任何辅助工具的情况下，我们罗列一下这三个元素的的普通调试方式：
**UI**：若是静态的UI我们只需在在AS中实时预览即可，而如果是动态UI，比如代码生成或者根据用户操作实时生成这些情况，我们则不能比较直观的查看到UI的层次结构。
**网络交互**：若是简单的监听返回数据，我们或许可以通过Log打印出来，简单的GET请求也可以在浏览器上输入URL进行测试，但是遇见参数复杂或者数量庞大的网络请求的情况下，上述两种方式都不能优雅的解决我们的问题。
**本地数据**：Android的本地数据包括数据库和SharedPreference（SP），查看数据库数据无非两种方式，一种是使用Android平台的查看工具进行查看，第二种这是把sql文件拷贝到电脑上使用工具进行查看，而数据库数据变更后我们则需要再次拷贝一份。而SP的数据除了查看对应的xml文件，则只能通过写一份工具代码来遍历查看了。
>> 如果你的开发效率卡在上面这些瓶颈上，那么Stetho则是解决这个问题的“银弹”。

<!-- more -->

### 集成Stetho
在[官网](http://facebook.github.io/stetho/)中查看最新的版本号
#### 在AS中增加依赖
```
  // Gradle dependency on Stetho 
  dependencies { 
    compile 'com.facebook.stetho:stetho:1.4.1' 
  } 
```
#### 在Application中添加初始化
```java
 public class MyApplication extends Application {
   public void onCreate() {
     super.onCreate();
     Stetho.initializeWithDefaults(this);
   }
 }
```
不要忘了把MyApplication配置到AndroidManifest.xml文件中
#### 使用chrome打开Stetho映射的设备
使用Stetho我们需要一个Chrome浏览器或者一个Chrome内核的浏览器
连接真机或使用模拟器运行集成Stetho之后的App，并使用Chrome打开chrome://inspect/#devices
![选择设备](http://facebook.github.io/stetho/static/images/inspector-discovery.png)
这时候我们会通过看到已经连接了的设备，而设备是通过Stetho和Chrome进行连接
点击**inspect**，会弹出一个新的窗口，我们将通过这个窗口来对应用程序进行调试，如果你看的是一个空白的窗口，那么可能是遇到GFW了，尝试下载最新版的Chrome或者找个梯子试试。
### 解析UI
打开**Element**页签，我们会看到类似于下图的界面
![界面元素解析](http://facebook.github.io/stetho/static/images/inspector-elements.png)
Stetho把app中的布局元素解析之后在chrome中通过xml的形式进行映射，点击界面右上角的预览图标，我们还能看打界面UI情况的实时投影
在这个界面中，允许我们修改简单的xml元素来对界面进行微调，并实时预览调整效果。
### 查看数据库
打开**Resources**页签中的WebSQL菜单，在这个菜单中我们能看到app中SQLite的所有数据库，不管是使用sql还是使用orm框架，在这里都是能够看到的。
![数据库解析](http://facebook.github.io/stetho/static/images/inspector-sqlite.png)
同时，这个界面还支持输入sql查询语句对数据库进行精确查询
### 查看SP
打开**Resources**页签中的LocalStorage菜单，在这个菜单中我们能看到app中所有的sp以及数据
图略23333

### 集成OKHTTP3，追踪网络请求
如果要使用Stetho的网络监听功能，需要加入额外的模块，这里以OKHttp3为例
#### 在AS中增加依赖
首先加入OKHttp3的依赖
```
compile 'com.squareup.okhttp3:okhttp:3.4.2'
```
再加入Stetho针对OKHttp3的扩展依赖
```
compile 'com.facebook.stetho:stetho-okhttp3:1.4.1'
```
#### 给OKHttpClient添加Stetho拦截器
使用builder形式创建OKHttpClient，并添加拦截器
```java
        OkHttpClient okHttpClient = new OkHttpClient.Builder()
                .addNetworkInterceptor(new StethoInterceptor())
                .build();
```
使用上述发方式生成的client进行网络请求就可以在Stetho中监听到所有的请求数据了。
![网络请求及数据监听](http://facebook.github.io/stetho/static/images/inspector-network.png)
### 使用JS终端远程注入执行代码
这是一个非常强大的功能，允许在chrome的js终端里输入js代码映射为app中的java的代码，并实时执行。
#### 在AS中增加依赖
```
compile 'com.facebook.stetho:stetho-js-rhino:1.4.1'
```
#### 在Application中添加初始
```java
Stetho.initialize(Stetho.newInitializerBuilder(context)
        .enableWebKitInspector(new InspectorModulesProvider() {
          @Override
          public Iterable<ChromeDevtoolsDomain> get() {
            return new DefaultInspectorModulesBuilder(context).runtimeRepl(
                new JsRuntimeReplFactoryBuilder(context)
                    // Pass to JavaScript: var foo = "bar";
                    .addVariable("foo", "bar")
                    .build()
            ).finish();
          }
        })
        .build());
```
#### 在chrome的js终端中输入js
在chrome的js终端中可以拿到app运行时的context
测试：输入测试代码
```JavaScript
importPackage(android.widget);
importPackage(android.os);
var handler = new Handler(Looper.getMainLooper());
handler.post(function() { Toast.makeText(context, "Hello from JavaScript", Toast.LENGTH_LONG).show() });
```
此时我们可以看到正在运行的app中弹出了一个toast，内容则是Hello from JavaScript。
测试中的context实际上是MyApplication
![js代码注入](http://facebook.github.io/stetho/static/images/inspector-js.png)
然而这个高级特性在开发过程中用到的其实并不多，附上官方文档地址：
[stetho-js-rhino的官方文档地址](https://github.com/facebook/stetho/blob/master/stetho-js-rhino/README.md)

### Realm数据库扩展
Stetho官方只给出了对SQLite数据库的支持，所以我们在使用realm数据库的时候并不能获得对数据库的监听的，但方法总是有的，github上有个开源项目能让Stetho也能对realm数据库进行查看
唯一遗憾的是在使用了这个库的时候Stetho会失去对SQLite数据库的支持，当然我们在同一个项目中也不可能同时使用两种数据库。
这里附上该项目的[地址](https://github.com/uPhyca/stetho-realm)