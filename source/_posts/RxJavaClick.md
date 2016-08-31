---
title: 使用RxJava实现多次连续点击的事件监听
date: 2016-08-31 08:56:49
tags: [Android, RxJava, RxAndroid]
category: Android
---

说起响应试编程，要提到的当然是Rx系列的库了，Rx系列的库对于很多语言和平台的运用是非常广泛的，例如(.NET,Java, Scala, Clojure, JavaScript, Ruby, Python, C++, Objective-C/Cocoa, Groovy等等。而本篇将会记录如何使用RxJava对Android点击事件的监听以异步数据流的方式来进行处理，从而实现对多次点击事件的监听。
多次点击事件的监听在Android中应用还是比较广泛的，比如“再次点击关闭应用程序”，又比如7次连续点击开启开发者模式等等。常规的设计无非是定义一个变量来记录点击的时间差或者定义一个线程来重置连击标识，然而这样的设计写出来的代码并不好看，而且可读性不高不易扩展。但是使用RxJava来实现就不会有上述这些问题了，而代码非常简洁。
上示例图：
![多次点击示例](https://github.com/googlesamples/android-architecture/wiki/images/RxAndroidClick.png)
<!-- more -->

RxJava作为Rx系列在Java平台的实现用于Android开发在适合不过了，同时另外一个库RxAndroid作为Rx在Android平台特殊特性的支持也是必然要用上的。而RxBinding库提供了对View点击事件的Rx形式监听。
上面三个库是我们本次功能实现需要用上的，加上依赖：
```
dependencies {
    ...
    compile 'io.reactivex:rxandroid:1.2.1'
    compile 'io.reactivex:rxjava:1.1.9'
    compile 'com.jakewharton.rxbinding:rxbinding:0.4.0'
}

```

RxBind会对View的点击实现监听进行封装，并返回一个Observable作为观察者模式供Rx使用
```java
        final Button button = (Button) findViewById(R.id.bt);
        RxView.clicks(button).subscribe(new Action1<Void>() {
            @Override
            public void call(Void aVoid) {
                button.setText("button被点击");
            }
        });
```
而RxBing做的其实只是定义了一个Observable，在OnSubscribe中给View添加onClickListener并在onlick中调用subscriber的onNext，具体的实现这里不做赘述。
好了，现在button的监听已经是Rx形式的观察者模式了，而Rx的核心是异步数据流，同时提供了很多操作符可用对数据流中的数据进行处理。
这里我们使用buffer操作符对点击事件进行打包，我们提供一个以300毫秒为阀值的selector，并指定回调的线程为主线程：
```java
        Observable<Void> observable = RxView.clicks(button).share();
        observable.buffer(observable.debounce(300, TimeUnit.MILLISECONDS))
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<List<Void>>() {
                    @Override
                    public void call(List<Void> voids) {
                        button.setText("" + voids.size() + "连击");
                    }
                });
```
到这里其实我们已经实现我们的目标了，没错，就是这么简单粗暴。

那么接下来我们增加一些更加酷炫的功能来实现我们开篇时候的效果
从上面的代码我们可以看到，多次点击的监听回调是传入了一个Void类型的List，我们除了用上了List的size好像并没有其他用，所以我们可以用map操作符来对数据流进行处理
```java
                .map(new Func1<List<Void>, Integer>() {
                    @Override
                    public Integer call(List<Void> voids) {
                        return voids.size();
                    }
                })
```
通过map操作符将List类型的数据流转变为了一个代表List长度的Int类型的数据流，这样我们的回调就会变更为
```java
                .subscribe(new Action1<Integer>() {
                    @Override
                    public void call(Integer integer) {
                        button.setText("" + integer + "连击");
                    }
                });
```
这时候我们有了多次点击的回调监听，如果同时要对每次点击也加上监听呢，我们只需要增加一个doOnNext即可
```java
                .observeOn(AndroidSchedulers.mainThread())
                .doOnNext(new Action1<Void>() {
                    @Override
                    public void call(Void aVoid) {
                        label.setText("#" + label.getText());
                    }
                })
```
完工

贴上最终代码：
```java
        Observable<Void> observable = RxView.clicks(button).share();
        observable
                .observeOn(AndroidSchedulers.mainThread())
                .doOnNext(new Action1<Void>() {
                    @Override
                    public void call(Void aVoid) {
                        label.setText("#" + label.getText());
                    }
                })
                .buffer(observable.debounce(300, TimeUnit.MILLISECONDS))
                .map(new Func1<List<Void>, Integer>() {
                    @Override
                    public Integer call(List<Void> voids) {
                        return voids.size();
                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<Integer>() {
                    @Override
                    public void call(Integer integer) {
                        label.setText("");
                        button.setText("" + integer + "连击");
                    }
                });
```

引用前段时间看到的一句话：**自从用了Rx，吃个汤圆都想拿筷子串起来吃** 23333

