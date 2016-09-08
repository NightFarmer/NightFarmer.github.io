---
title: 给应用增加动态切换主题配色的功能
date: 2016-09-07 15:56:43
tags: [Android]
category: Android
---

最近发现BiliBili客户端有个非常个性的功能, 提供了夜间模式/少女粉/姨妈红/胖次蓝/基佬紫等主题配色供用户切换选择, 不但丰富了用户了的视觉体验, 而且一定程度的增加了产品的收入, 是的你没看错,BiliBili客户端的很多配色主题都是要花费硬币的. 这么酷炫又实用的功能当然不能错过了!
上图:
![颜色主题效果](http://nightfarmer.github.io/public/static/image/ColorTheme.gif)
<!-- more -->

切换主题这个功能也算是一个骨灰级的东西了, 就拿最熟悉的QQ来说很久之前就有了切换皮肤的功能, 当我们这里讨论的只是配色而不是皮肤, 本篇的目的在于用最精简的代码来实现切换整个应用配色的功能.

动手之前在github上扒到了哔哩哔哩关于他们主题的开源框架[MagicaSakura](https://github.com/Bilibili/MagicaSakura), '魔法小樱'浓浓的二次元气息啊, 这些都不是问题, 问题是这个Sakura有点肿.. 为什么这么说呢? 通过源码发现这个框架是对所有用到的View全部重定义一遍, 并实现了切换配色的interface, 这个会造成所有View都要因为配色问题重新定义一个副本, 并在所有用到的地方使用这个新的View. 
MagicaSakura具体的实现这里不做讨论, 一是因为肿, 二是这样的轮子重复造起来也没意思.

所以接下来本篇会记录一个更加简洁的方式,来实现开篇的效果.

其实,google已经为我们提供了一个切换主题的方法setTheme, 不过如果想要这个方法来实现我们的功能, 可能会存在一些缺陷. 这个方法的作用是在activity内的view在初始化之前指定运行时的主题, 意思在这个方法调用之前的Activity如果已经初始化完成那么是没有用的, 只有在Activity在创建完成并且contentview没有初始化的时候调用才会生效. 
那么这个方法不能用吗? 其实不然, activity有一个recreate方法, 会让当前界面重新构建一次, 这时候再构建的时候指定theme就可以变更主题了. 不过这个方法调用时会造成界面严重的闪烁, 用户体验非常不好, 而且存在于Activity栈中的activity也仍然是旧的样式. 只是简单的调用recreate方法并不能实现我们要的效果.

那么如何让recreate能够平滑过度并且对之前已经存在的activity进行重建将会是接下来要记录的重点

>> 本篇的代码片段摘录自我的一个kotlin写的项目, 代码皆为kotlin实现, 鉴于kotlin可读性比较高就不再翻译了

首先定一个类ColorTheme做为我们切换主题的工具类, 同时实现ActivityLifecycleCallbacks接口.
```kotlin
object ColorTheme : Application.ActivityLifecycleCallbacks {

    var activityList = arrayListOf<Activity>()//创建一个Activity栈,用于记录所有创建的Activity
    val ThemeChacheKey = "themeCacheData"
    var style: Int = 0

    fun setStyle(activity: Activity, @StyleRes style: Int, data: Bundle? = null) {
        ColorTheme.style = style
        Observable.from(activityList)
                .skipLast(1)//跳过栈顶的activity,也就是
                .subscribe {
                    it.recreate()//将栈中的所有activity全部重建,如此可应用新主题配色
                }
        val intent = Intent(activity, activity.javaClass)
        intent.putExtra(ThemeChacheKey, data)
        activity.startActivity(intent)
        activity.overridePendingTransition(0, 0);//移除activity跳转动画
        activity.finish()
    }
    //Activity入栈,并指定theme
    override fun onActivityCreated(activity: Activity?, savedInstanceState: Bundle?) {
        activity?.let {
            activityList.add(activity)
            val styleRes = ColorTheme.style
            activity.setTheme(styleRes)
        }
    }
    //activity出栈
    override fun onActivityDestroyed(activity: Activity?) {
        activity?.let {
            activityList.remove(activity)
        }
    }
    ...
```

上面这段代码实现了对当前界面应用新theme的处理以及对已经打开的activity应用新theme的处理.
通过对调用的activity再次创建,并finish掉当前的activity可用规避recreate方法的严重闪烁问题,需要注意的是对通过的intent传递并回显界面中的临时数据.
栈中的数据则是通过对活动activity的记录,并依次调用这些activity的recreate方法进行重建,因为这些activity是被压入栈中不显示的,所以使用recreate方法并不会出现闪烁问题. 需要注意的是需要在可能被调用recreate的activity中复写onActivitySaveInstanceState方法用来处理activity被销毁终极时的数据回显问题.

给这个类加上初始化方法
```kotlin
    fun init(application: Application, style: Int = 0) {
        application.registerActivityLifecycleCallbacks(this)
        this.style = style
    }
```

我们定义这个类之后在我们的Application的onCreate里初始化即可
```kotlin
class CoderApplication : Application() {

    private val TAG = PluginApplication::class.java.simpleName

    override fun onCreate() {
        super.onCreate()
        ColorTheme.init(this,R.style.Theme_Default)
        ...
    }

```

需要切换主题配色的地方如此调用即可:
```kotlin
    ColorTheme.setStyle(activity, style)
```

开发UI的地方需要注意, 所有用到需要主题配色的地方都需要使用?attr/来指定颜色, 并在每个主题的style中指定attr的值:
UI指定颜色
```xml
...
    <TextView
        android:background="?attr/colorPrimary"
        android:textColor="?attr/colorPrimaryText"
        android:text="测试文本" />
...
```
两个主题配色
```xml
    <style name="Theme_Blue" parent="BaseTheme">
        <item name="colorPrimary">#0000FF</item>
        <item name="colorPrimaryText">#FFFFFF</item>
        ...

    <style name="Theme_Red" parent="BaseTheme">
        <item name="colorPrimary">#FF0000</item>
        <item name="colorPrimaryText">#FFFFFF</item>
        ...
```

为了更方便的使用, 我把上述的实现做了gradle依赖, 使用方法:

gradle

/build.gradle
```
repositories {
    ...
    maven {
        url "https://jitpack.io"
    }
}
```
/app/build.gradle
```
dependencies {
    ...
    compile 'com.github.NightFarmer:Themer:1.0.0'
}
```


>> 本工程使用kotlin编译, kotlin和java同为jvm平台语言, 所以java开发同样可以正常使用.


github: https://github.com/NightFarmer/Themer
<div style="width: auto; max-width: 600px;">
	<div class="github-widget" data-repo="NightFarmer/Themer"></div>
	<script type="text/javascript" src="/about/GithubRepoWidget.js"></script>
</div>




