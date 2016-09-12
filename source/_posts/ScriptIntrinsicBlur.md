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

使用官方api实现高斯模糊处理我们需要用到`android.renderscript.ScriptIntrinsicBlur`类以及方法, 而这个类只能在api17(android4.2)及以上的版本中才能使用, 在android4.0版本仍在大量使用的今天,这样当然不能满足我们的需求, 而google也为这个类提供了向下兼容的方法:
在Module的build文件中增加以下属性:
```gradle
android{
    ...
    defaultConfig{
         ...
        renderscriptTargetApi 19
        renderscriptSupportModeEnabled true
    }
}
```
这样在AndroidStudio中就加入的V8包中的RenderScript的支持, 如果是使用Eclipse开发环境则需要手动把v8的jar包以及so文件加入到lib中, 如今google已经放弃了对eclipseADT的支持, 也没必要抱着Eclipse不放了.

这时我们只需要将RenderScript以及ScriptIntrinsicBlur和其他类的引用改为v8中的即可完成低版本的兼容, 如:`android.support.v8.renderscript.ScriptIntrinsicBlur`

现在就可用使用官方的api来进行图片的处理了, 下面记录的api的使用方法以及每个注释:
```java
/**
     * 处理bitmap为高斯模糊图片
     * @param context 上下文
     * @param image   图片源
     * @param radius  模糊程度 0到25之间
     * @param scale   图片缩放比例, 该值越小越节省内存,模糊程度越敏感,0到1之间
     * @return 模糊的图片
     */
    public static Bitmap blurBitmap(Context context, Bitmap image, float radius, float scale) {
        // 计算图片缩小后的长宽
        int width = Math.round(image.getWidth() * scale);
        int height = Math.round(image.getHeight() * scale);
        // 将缩小后的图片做为预渲染的图片。
        Bitmap inputBitmap = Bitmap.createScaledBitmap(image, width, height, false);
        // 创建一张渲染后的输出图片。
        Bitmap outputBitmap = Bitmap.createBitmap(inputBitmap);
        // 创建RenderScript内核对象
        RenderScript rs = RenderScript.create(context);
        // 创建一个模糊效果的RenderScript的工具对象
        ScriptIntrinsicBlur blurScript = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs));
        // 由于RenderScript并没有使用VM来分配内存,所以需要使用Allocation类来创建和分配内存空间。
        // 创建Allocation对象的时候其实内存是空的,需要使用copyTo()将数据填充进去。
        Allocation tmpIn = Allocation.createFromBitmap(rs, inputBitmap);
        Allocation tmpOut = Allocation.createFromBitmap(rs, outputBitmap);
        // 设置渲染的模糊程度, 25f是最大模糊度
        blurScript.setRadius(radius);
        // 设置blurScript对象的输入内存
        blurScript.setInput(tmpIn);
        // 将输出数据保存到输出内存中
        blurScript.forEach(tmpOut);
        // 将数据填充到Allocation中
        tmpOut.copyTo(outputBitmap);
        return outputBitmap;
    }
```

将上面这个方法加入到项目中即可完美得处理图片的高斯模糊效果了, 而官方api的使用方法到这里也基本介绍结束. 接下来会对开篇的拓展实现进行记录.

#### 使用RxJava对SeekBar事件流进行缓存
我们知道图片的处理是比较耗时的, 将大量的图片处理放在UI线程中必然会造成界面的卡顿, 如果我们使用常规的方法来实现开篇的效果也肯定会遇到这个问题:
```java
        seekBar.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
            @Override
            public void onProgressChanged(SeekBar seekBar, int i, boolean b) {
                try {
                    Bitmap bitmap1 = blurBitmap(MainActivity.this, bitmap, 25f*i/100, 0.4f);
                    img_result.setImageBitmap(bitmap1);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void onStartTrackingTouch(SeekBar seekBar) {}

            @Override
            public void onStopTrackingTouch(SeekBar seekBar) {}
        });
```

seekBar的onProgressChanged方法是在UI线程中回调的, 而我们的blurBitmap方法传入了原始bitmap并进行高斯模糊处理并返回新的bitmap并更新UI也是在UI线程处理的, 我们拖动seekBar时会有大量的onProgressChanged回调涌入UI线程, 这时候seekBar的卡顿会非常的严重.

既然onProgressChanged是按顺序触发的事件, 那么使用RxJava来把这些事件当做事件流来处理再适合不过了.
首先我们要将seekBar的事件包装成一个Observable, 我们模仿RxBinding对OnSeekBarChangeListener进行封装:
```java
public class SeekBarOnChangeSubscribe implements Observable.OnSubscribe<Float> {
    SeekBar seekBar;
    public SeekBarOnChangeSubscribe(SeekBar seekBar) {
        this.seekBar = seekBar;
    }
    @Override
    public void call(final Subscriber<? super Float> subscriber) {
        seekBar.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
            @Override
            public void onProgressChanged(SeekBar seekBar, int i, boolean b) {
                if (!subscriber.isUnsubscribed()) {
                    subscriber.onNext(i * 1f);
                }
            }
            @Override
            public void onStartTrackingTouch(SeekBar seekBar) {}

            @Override
            public void onStopTrackingTouch(SeekBar seekBar) {}
        });

        //unSubscribe监听,并将seekBar的监听置空
        subscriber.add(new MainThreadSubscription() {
            @Override
            protected void onUnsubscribe() {
                seekBar.setOnSeekBarChangeListener(null);
            }
        });
    }
}
```
创建一个工具类让我们的封装看起来更加有builder的感觉
```java
    public class MyBinder{
        public static Observable<Float> bind(SeekBar seekBar){
             return Observable.create(new SeekBarOnChangeSubscribe(seekBar));
        }
    }
```

调整我们的监听:
```java
        MyBinder.bind(seekBar)
                .map(new Func1<Float, Bitmap>() {
                    @Override
                    public Bitmap call(Float aFloat) {
                        Bitmap bitmap1 = null;
                        float radius = 25f * aFloat / 100;
                        bitmap1 = blurBitmap(MainActivity.this, bitmap, radius, 0.4);
                        return bitmap1;
                    }
                })
                .subscribe(new Action1<Bitmap>() {
                    @Override
                    public void call(Bitmap newImage) {
                        img_result.setImageBitmap(newImage);
                    }
                });
```
现在我们已经成功的把回调监听改为了RxJava模式的响应试监听
但是这样仍然不满足我们的需求, 因为他本质上和之前是一样的, 阻塞UI线程并调用blurBitmap方法.
我们需要给map和subscribe方法分别指定.observeOn(Schedulers.computation())和.observeOn(AndroidSchedulers.mainThread())的调度线程. 
```java
        MyBinder.bind(seekBar)
                .observeOn(Schedulers.computation())
                .map(new Func1<Float, Bitmap>() {
                    @Override
                    public Bitmap call(Float aFloat) {
                        Bitmap bitmap1 = null;
                        float radius = 25f * aFloat / 100;
                        bitmap1 = blurBitmap(MainActivity.this, bitmap, radius, 0.4);
                        return bitmap1;
                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<Bitmap>() {
                    @Override
                    public void call(Bitmap newImage) {
                        img_result.setImageBitmap(newImage);
                    }
                });
```
这时候看起来像那么回事了, 跑一下看看如何?
当然, 你会遇到MissingBackpressureException! 为何呢, 我们将图片处理放入子线程并在ui线程更新了UI啊. 问题不是出在这里, 其实当我们在子程序中处理图片时这个子线程依然是处于阻塞状态的, 而UI线程中产生的事件仍在不断的涌入子线程, 这时候RxJava并不知道该如何缓存这些事件, 而我们只需要告诉RxJava一个缓存方式即可.
这里我们使用最普通的缓存方式onBackpressureBuffer(), 这个方法会将来不及处理的事件统统依次缓存到一个队列中, 在子线程中的上一个事件处理完毕后依次进入处理.

贴上最终代码:
```java
        MyBinder.bind(seekBar)
                .onBackpressureBuffer()
                .observeOn(Schedulers.computation())
                .map(new Func1<Float, Bitmap>() {
                    @Override
                    public Bitmap call(Float aFloat) {
                        Bitmap bitmap1 = null;
                        float radius = 25f * aFloat / 100;
                        bitmap1 = blurBitmap(MainActivity.this, bitmap, radius, 0.4);
                        return bitmap1;
                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<Bitmap>() {
                    @Override
                    public void call(Bitmap newImage) {
                        img_result.setImageBitmap(newImage);
                    }
                });
```



