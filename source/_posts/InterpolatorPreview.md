---
title: 自定义view展示官方Interpolator并扩展自定义Interpolator
date: 2016-09-12 18:15:54
tags: [Android, 动画]
category: Android
---

现在动画在Android开发中使用得越来越频繁, 一个新发布的app如果没有一点酷炫的效果还真拿不出手, 然而说到动画必然会联系到动画差值器Interpolator, 这是一个让动画执行起来不会闲的突兀或者平淡乏味的东西, 有了它我们的动画会显得更加生动和具有物理特性.
而实际开发过程中如果需要某个差值器的情况下, 一般是找一些曲线图片然后依靠想象挑选或者直接挨个尝试来得到最合适的, 这些做法虽然可行但终究还是比较麻烦, 说直白点就是比较low, 而且在官方提供的差值器不满足我们的需求的时候, 我们需要开发自己的差值器, 在这种情况下就更需要一种非常直接的方法来展示差值器的效果了. 
本篇则基于这样的需求驱动, 自定义了View来展示曲线以及差值器的执行效果, 把官方所有的差值器都放进去跑起来, 当需要的时候直接挑选即可. 而自定义的Interpolator也可用来测试以及查看差值曲线.
图:
![Interpolator示例1](http://nightfarmer.github.io/public/static/image/InterpolatorDemo1.gif) ![Interpolator示例1](http://nightfarmer.github.io/public/static/image/InterpolatorDemo2.gif) ![Interpolator示例1](http://nightfarmer.github.io/public/static/image/InterpolatorDemo3.gif)
<!-- more -->

#### 自定义View展示Interpolator

在动手之前, 我们先要了解一下Interpolator接口, 在android中所有的差值器都需要实现Interpolator接口, 而Interpolator接口继承了TimeInterpolator接口. 实现这个接口只需要实现一个方法`float getInterpolation(float var1)`, 这个方法会在动画执行期间不断传入0-1之间的数值并返回一个代表动画当前执行进度的新的数值来控制动画的执行进度. 也就是说动画开始执行到结束的每一帧都会回调这个方法, 而这个方法的返回值决定了这帧里动画应该执行的进度. 比如说LinearInterpolator, 通过源码我们能够看到它的这个方法会把传入的参数原封不动的作为返回值返回, 所以动画执行期间的每一帧都是动画执行时间和进度相匹配的.

好了,接下来我们开始定义我们的自定义View, 因为要绘制Interpolator的曲线,所以首先要初始化一些Paint来备用
```java
public class InterpolatorPreView extends View {
    private Path path;//曲线路径
    private Paint paintLine;//上下横线画笔
    private Paint paintPath;//曲线画笔
    private float process = 0;//进度
    Interpolator interpolator;// get/set
    public InterpolatorPreView(Context context) {
        this(context, null);
    }
    public InterpolatorPreView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }
    public InterpolatorPreView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }
    private void init() {
        path = new Path();
        paintLine = new Paint();
        paintLine.setColor(Color.BLUE);
        paintLine.setStrokeWidth(2);
        paintLine.setStyle(Paint.Style.STROKE);
        paintLine.setAntiAlias(true);

        paintPath = new Paint();
        paintPath.setColor(Color.RED);
        paintPath.setStrokeWidth(2);
        paintPath.setStyle(Paint.Style.STROKE);
        paintPath.setAntiAlias(true);
    }
}
```
准备工作已经做完, 然后我们需要考虑的是如何得到曲线的绘制路径, 在完整的曲线实例图中我们应该可用看到这些曲线都是绘制在一个坐标系中的, 横坐标是时间, 纵坐标是值, 对应于我们上面对Interpolator接口的解析,横坐标就是0到1期间的这些值, 而纵坐标则是0到1期间Interpolator的getInterpolation方法的返回值. 那么我们需要做的就是根据view的宽和高以及横纵坐标的值来进行绘制, 换种说法就是把曲线每个点都放入path然后进行绘制.
我们定义一个方法initPath()用来初始化path,并放在onMeasure方法中调用
```java
    public void initPath() {
        int height = getHeight();
        int width = getWidth();
        if (height != 0) {
            path.reset();
            path.moveTo(padding, interpolator.getInterpolation(0) * (height / 2f) + height / 4f);
            for (float i = 0; i <= width; i++) {
                path.lineTo(i, interpolator.getInterpolation(i / width) * (height / 2f) + height / 4f);
            }
            invalidate();
        }
    }
```
在上面的方法中,我们通过便利view宽度的每个像素处于with中的位置来作为时间轴的坐标值,传入interpolator并返回y轴进度, 之后我们以view高度的一半来获取y轴的值, 实际上我们在view的纵向方向分别在上下空余了1/4的空间, 这是因为有一些特殊的interpolator返回的进度是超过0到1这个区间的.

剩下的就是复写onDraw方法来进行曲线的绘制
```java
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        int height = getHeight();
        int width = getWidth();
        if (height != 0) {
            canvas.drawLine(0, height / 4f, width, height / 4f, paintLine); //绘制上边界
            canvas.drawLine(0, height / 4f * 3, width, height / 4f * 3, paintLine); //绘制下边界
            if (path.isEmpty()){
                initPath();
                return;
            }
            canvas.drawPath(path, paintPath); //绘制曲线
        }
    }
```

好了interpolator的曲线的绘制已经搞定了, 接下来是添加一个能够沿着这个曲线移动的小球.
给view增加一个startAnim方法作为小球开始移动的触发方法
```java
    public void startAnim() {
        if (valueAnimator != null && valueAnimator.isRunning()) {
            valueAnimator.cancel();
        }
        process = 0;
        invalidate();
        valueAnimator = ObjectAnimator.ofFloat(0, 1);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator) {
                process = (Float) valueAnimator.getAnimatedValue();
                invalidate();
            }
        });
        valueAnimator.setStartDelay(500);
        valueAnimator.setDuration(1000);
        valueAnimator.start();
    }
```
这是一个值从0到1的ObjectAnimator, 我们把动画执行期间的value放入view的变量中, 并在ondraw方法里绘制小球
```java
    @Override
    protected void onDraw(Canvas canvas) {
        ...
        canvas.drawCircle(width* process, interpolator.getInterpolation(process) * (height / 2f) + height / 4f, 15, paintPath);
    }
```
到此为止view的定义以及完成, 如果要实现开篇的效果只需要把这个view放入布局文件指定一个interpolator然后startAnim即可.

#### 定义并测试自己的Interpolator

经过上面的设计我们已经有了一个非常好用的工具来展示和测试各种interpolator了, 那么接下来就顺便写几个interpolator看看我们的view到底效果如何.
我准备了四个interpolator的算法
**弹簧**
`Math.pow(2, -10 * input) * Math.sin((input * swing - 0.26 / 4) * Math.PI * 2.0f / 0.3) + 1`

**平方抛物线**
`input * input`

**反着的平方抛物线**
`(-1) * (input - 1) * (input - 1) + 1`

**四次方抛物线**
`input * input * input * input`

定义四个类分别实现Interpolator接口并将上列的算法放入getInterpolation方法即可
效果:
![自定义Interpolator示例](http://nightfarmer.github.io/public/static/image/InterpolatorDemo4.gif)
嗯 完工.

完整工程代码:
https://github.com/NightFarmer/InterpolatorDemo
