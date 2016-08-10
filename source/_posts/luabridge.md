---
title: Android热更新/热补丁终极解决方案
date: 2016-08-10 14:17:23
tags: [Android, Lua, 热更新, NDK]
category: Android
---

在Android开发工作中热更新一直是个遗留问题，虽然GooglePlay或者苹果商店的应用审核中都是禁止App这么做的，但仍有大量的开发者想要通过热更新或者热补丁来对已经发布的应用进行更新，而不用重新打包发布一个新的版本，毕竟一个刚发布的应用如果因为一个小bug而让用户再次进行更新，一是用户体验不好，二是发布的过程太过繁琐。
之前也因为热更新的问题做过各种各样的尝试，包括H5，ReactNative，动态加载dex，使用Javascript和java交互等，单都因为各种原因造成不能在生产环境中使用。

|方案      |    问题| 
|:-------- | :--------| 
|H5/H5+  | 性能瓶颈在于设备WebView对html和JS的解析效率, 且WebView的加载是异步的，界面初始化时会空白一段时间，尝试使用腾讯TBS以及H5+SDK效果并不理想，尝试使用各种mobileUI框架依然存在卡顿帧数过低等各种不理想的问题。 |
|ReactNative     | RN是FaceBook提供的一个跨平台的开发框架，使用NodeJs来写JSX和CSS来进行布局和实现业务逻辑，门槛稍高(这不是问题)，使用RN实现的Android和iOS应用可用达到和原生应用相媲美的流畅度(也确实是原生的)，可以通过从服务器下载js文件来实现热更新，但是如果想要实现原生应用酷炫的效果则是不满足的，且自定义控件的事件传递会被RN容器干扰，RN本来是为iOS而设计的，近期才考虑兼容了Android，对android的支持也还暂时不充足。| 
|动态加载dex      | 在Dalvik虚拟机下的android环境这个方案是可行的，但是在android4.4之后google发布了新的虚拟机(ART)，因ART在app安装过程会对dex的bytecode进行优化并进一步解析成机器码，使得在程序运行期间hook代码变得更有难度，在不能放弃5.0及之后的用户的前提下，这个问题是致命的 |
|Javascript      | 使用jDK或者webview解析Javascript也能实现一部分的业务逻辑转移，但此方案的局限性太大，不能大规模的使用。 |

上面这些是我从接触热更新到目前为止所踩过的坑，可以看到暂时是没有任何一个方案是比较完善或者能用在生产环境的。
那么下面，高能来了。

<!-- more -->
事情的起因是前段事件突然想山寨一个app，当时是对MaterialDesign比较感兴趣，于是选了BiliBili客户端作为山寨的模板，这个客户端基本上涵盖了MD设计风格的核心概念，包括水波纹圆角卡片TabLayout等等。
上寨的期初阶段并没有在意太多，就是写UI嘛，和照着美工的效果图来一发是差不多的，看官方UI有什么元素就照着写一个，也确实没什么问题。
UI写得差不多了，需要抓取官方的API来给应用获取数据了，比如各种视频的列表，番剧更新等，这一抓不要紧，发现了有意思的东西，在官方app安装完成后首次打开时，我在抓包列表里看到了app在后台通过网络获取了很多.lua后缀的文件，鉴于鄙人在平时喜欢涉猎一些别的领域，对lua还是略有了解的，lua是一个动态脚本语言，常用做游戏开发或者和c/c++交互，在之前玩python的时候还特意了解了一下这个语言。

**那么问题来了，这个app下载这些lua脚本文件到底是用来做什么的？**

打开之后发现这些文件是被加密过的二进制文件，因为好奇就去google百度了一下两个关键词“android lua”，结果发现竟然有一个叫做luajava的库，这是由c编写的和lua交互的库并提供和java通讯的接口！
这是什么？这不就上面提到的第四个方案中的最后一个方案的优化版吗？
nodejs之所以快是因为用了google的V8引擎，Android肯定不能在App放一个V8用来解析js，因为太大了！
但是luajava呢？一共也才几百k，那么接下来就是见证奇迹的时刻了。

首先在github找到了这个项目的c语言和java源码，然后新建了JNI工程把这些东西编译了一遍，很顺利。
接下来就是各种尝试了，lua和java的相互调用，使用lua进行布局，使用lua实现android动画，使用lua实现数据列表...

随后把这些东西做成了一个库，可以gradle依赖并直接用在生产环境的

贴上效果：
```java
public class TestActivity extends AppCompatActivity {

    private LuaState luaState;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_recycler_view);

        luaState = LuaStateFactory.newLuaState();
        luaState.openLibs();
        luaState.LdoString(LuaBridge.loadAssets(this, "TestActivity.lua"));

        luaState.getField(LuaState.LUA_GLOBALSINDEX, "onCreate");
        luaState.pushJavaObject(this);
        luaState.pushJavaObject(((ViewGroup) findViewById(android.R.id.content)).getChildAt(0));
        luaState.pushJavaObject(savedInstanceState);
        luaState.call(3, 0);
    }
}
```
这里定义了一个Activity并加载了一个lua脚本文件，这文件放在了assets下，当然如果用于热更新的话需要后台从服务器下载，存储在本地，并使用文件系统来加载。
TestActivity.lua
```lua
-- TODO 初始化
function onCreate(activity)
    activity:setTitle("这个界面完全由Lua编写")
    initUI(activity)
    addListener(activity)
end
```
我们在lua文件里定义了和activity生命周期相同命名的oncreate，如有需要可以扩展onResume/onActivityResult等其他方法。

新建一个button
```lua
function initUI(activity)
    local id = luajava.bindClass("com.nightfarmer.luabridge.sample.R$id")
    local layout = activity:findViewById(id.layout)

    button2 = luajava.newInstance("android.widget.Button", activity)
    button2:setText("Lua初始化的button")
    layout:addView(button2)
end
```

添加点击事件，show一个toast
```lua
function addListener(activity)
    local id = luajava.bindClass("com.nightfarmer.luabridge.sample.R$id")
    local launchBt = activity:findViewById(id.myLaunchButton)
    local buttonProxy = luajava.createProxy("android.view.View$OnClickListener", button_cb)
    launchBt:setOnClickListener(buttonProxy)
end
button_cb = {}
function button_cb.onClick(eventView)
    local Toast = luajava.bindClass("android.widget.Toast");
    Toast:makeText(eventView:getContext(),"按钮被点击，通过lua发起toast",0):show()
end
```

创建并执行一个动画
```lua
button3_cb={}
function button3_cb.onClick(eventView)
    local Toast = luajava.bindClass("android.widget.Toast");
    Toast:makeText(eventView:getContext(),"按钮被点击，通过lua发起toast",0):show()

    ObjectAnimator = luajava.bindClass("com.nightfarmer.luabridge.sample.ObjectAnimator")
    local animator = ObjectAnimator:ofFloat(button2, "rotation",0,360)
    animator:setDuration(1000)
    animator:start()
end
```

用java代码可用实现的功能基本上在lua脚本中都可以实现，下面贴上一个通过扩展实现的一个RecyclerView的数据列表
```java
function onCreate(activity, rootView)
	activity:setTitle("使用Lua脚本构建RecyclerView")
	context = activity
        local recyclerView = luajava.newInstance("com.nightfarmer.luabridge.sample.recyclerview.LuaRecyclerView", activity)
	local linearManager = luajava.newInstance("android.support.v7.widget.LinearLayoutManager", activity)
	recyclerView:setLayoutManager(linearManager)
	rootView:addView(recyclerView)
	
	local adapter = luajava.createProxy("com.nightfarmer.luabridge.sample.recyclerview.LuaRecyclerAdapter",rec_adapter)
	recyclerView:setAdapter(adapter)
end

rec_adapter={}
function rec_adapter.onCreateViewHolder(parent, viewType)
	local button1 =	luajava.newInstance("android.widget.TextView",context)
	button1:setPadding(50,50,50,50)
	button1:setText("根据网络Lua脚本动态生成的button.....一可赛艇")
	parent:addView(button1)
	local holder = luajava.newInstance("com.nightfarmer.luabridge.sample.recyclerview.LuaViewHolder", button1)
	holder:putView("btn1", button1)
	return holder
end

function rec_adapter.onBindViewHolder(holder, position)
	holder:getView("btn1"):setText("yoo"..position)
end

function rec_adapter.getItemCount()
 	return 1000
end

```

是不是很刺激，如果把Activity全部使用lua来开发并把文件放在服务器上，那么就是可用实现应用全功能的热更新了。
当然这样做的开销也是比较大的，在实际使用中可用定义一个BaseActivity，在打开activity的时候判断这个activity是否有可用的lua脚本文件，来实现用lua脚本动态的替换java代码，比如app发布之后发现某个activity有一个小bug，那么针对这个activity写一个lua脚本就可用让所有联网的用户来执行lua脚本而替换掉有bug的代码。
当然如果是首页等活动性比较大的界面完全可以lua来写，比如儿童节教师节情人节什么的，在服务器上放一个lua脚本文件就搞定了。而且用户看到的是原生应用的效果，而不是H5这样的廉价感。


食用方法：
Step 1. 添加build文件
```
allprojects {
	repositories {
		...
		maven { url "https://jitpack.io" }
	}
}
```
Step 2. 添加依赖
```
dependencies {
	compile 'com.github.NightFarmer:LuaBridge:1.0.2'
}
```

源码及sample：https://github.com/NightFarmer/LuaBridge
