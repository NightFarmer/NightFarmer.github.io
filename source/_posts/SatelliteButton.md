---
title: 设计一个通用的卫星菜单
date: 2016-08-04 17:04:48
tags: [Android, 自定义控件, 动画]
category: Android
---

卫星菜单算是一个非常久远的控件效果了，到现在也存在了各种各样的开源的此类控件，至于为什么要重复造轮子，是因为翻阅了一些卫星菜单的实现源码发现这些代码水平参差不齐，无论是从整体设计或是逻辑实现或是代码简洁度以及扩展性上都没有找到比较满意的。
鉴于这个效果的实现并不复杂，还有昨天刚好做了个轮子([上篇](http://nightfarmer.github.io/2016/08/03/make-a-wheel/))，所以打算做一个可复用/可灵活配置/具有较好封装/可以当做普通View来使用的卫星菜单控件。
老规矩先上效果图：
![不靠边360度示例](http://nightfarmer.github.io/public/static/image/satellite1.gif) ![靠角90度示例](http://nightfarmer.github.io/public/static/image/satellite2.gif) ![靠边180度示例](http://nightfarmer.github.io/public/static/image/satellite3.gif)
<!-- more -->

卫星菜单的核心实现就是如何按照圆弧对菜单的item进行布局，刚好[上篇](http://nightfarmer.github.io/2016/08/03/make-a-wheel/)的轮子的实现中也遇到了这样的需求，这里就采用相同的逻辑来处理。
但是本篇的控件和上篇的轮子还是有区别的，因为我们要做的是*通用*的卫星控件，什么是通用的呢？通用包括两反面。
一是指这个控件可以放置在界面的任何位置，并由控件自身识别并做出卫星类型的判断，比如在中心就是360度的展开，在侧边就是180度的展开。
通用的第二个含义是指，在调整卫星的动画时间/卫星半径/item样式等部分的时候是不用修改控件本身的，也就是说这个控件需要做出良好的抽象封装来满足这个需求。
由第一个通用我们需要考虑到这个控件的绘制轨迹不止是一个圆弧，而且可能是半圆或者四分之一圆。

首先我们先把自定义控件的框架写好，定义一个类继承自View，同时定义一个抽象类Adapter做为卫星item以及button的提供者。
view类
```java
public class SatelliteView extends RelativeLayout {
    private SatelliteAdapter adapter;
    private int Radius = 400;//卫星菜单布局半径

    public void setAdapter(SatelliteAdapter adapter) {
        this.adapter = adapter;
        init();
    }

    public SatelliteView(Context context) {
        this(context, null);
    }

    public SatelliteView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public SatelliteView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    public void setRadius(int radius) {
        Radius = radius;
    }
}
```
item和button的提供者adapter
```java
public abstract class SatelliteAdapter {
    public abstract View createMenuItem(ViewGroup parent, int index);
    public abstract View createToggleItem(ViewGroup parent);

    public abstract int getCount();
}
```

在自定义的view中我们定义了一个init方法，并在赋予adapter的时候来调用，因为在控件初始化的时候其实是没有任何用户数据的，
所以将控件的初始化时机放在adapter指定之后来执行。
```java
    private void init() {
        RelativeLayout.LayoutParams layoutParams1 = (LayoutParams) getLayoutParams();
        toggleButton = adapter.createToggleItem(this);
        toggleButton.setId(R.id.satelliteBtn);
        addView(toggleButton);
        toggleButton.setLayoutParams(layoutParams1);
        for (int i = 0; i < adapter.getCount(); i++) {
            View menuItem = adapter.createMenuItem(this, i);
            menuItem.setAlpha(0);
            addView(menuItem, 0);
            RelativeLayout.LayoutParams layoutParams2 = (LayoutParams) menuItem.getLayoutParams();
            layoutParams2.addRule(RelativeLayout.ALIGN_LEFT, R.id.satelliteBtn);
            layoutParams2.addRule(RelativeLayout.ALIGN_TOP, R.id.satelliteBtn);
            menuItem.setLayoutParams(layoutParams2);
            menuItemList.add(menuItem);
        }

        setLayoutParams(new RelativeLayout.LayoutParams(RelativeLayout.LayoutParams.MATCH_PARENT, RelativeLayout.LayoutParams.MATCH_PARENT));

        toggleButton.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View view) {
                toggleShow();
            }
        });
    }

```
在init中调用了adapter的createToggleItem方法创建了一个点击按钮实例加入到组件容器中，并把组件容器的LayoutParams赋给了这个按钮，同时把组件容器设为填充填充父容器，这样我们的组件就可以在使用的过程中灵活指定位置，在实际初始化时则会把用户指定的位置通知给toggleButton，而我们的组件则是被设定为了填充父容器。
这样做是一个比较大胆的尝试，目前并没有发现有何不可。这样做给了我们的组件有了非常大的灵活空间来让用户指定我们的组件位置，不过同时有了一些局限性，我们的组件只能使用在Relativelayout中。

在toggleButton的点击事件中我们定义了一个toggleShow()方法，这个方法控制卫星菜单的打开与折叠。
```java
    public void toggleShow() {
        if (animRunning) return;
        partToggle(360, 0);
    }

    private void partToggle(int angleMax, int startAngle) {
        int count = adapter.getCount();
        AnimatorSet animatorSetTotal = new AnimatorSet();
        ArrayList<Animator> animators = new ArrayList<>();
        for (int i = 0; i < count; i++) {
            int angle;
            if (angleMax == 360) {
                angle = angleMax / count;
            } else {
                angle = angleMax / (count - 1);
            }
            float x = (float) (Math.cos((angle * i + startAngle) / 180f * Math.PI) * Radius);
            float y = (float) (Math.sin((angle * i + startAngle) / 180f * Math.PI) * Radius);

            View view = menuItemList.get(i);
            AnimatorSet animatorSet = new AnimatorSet();
            int x_trans = toggleButton.getMeasuredWidth() / 2 - view.getMeasuredWidth() / 2;
            int y_trans = toggleButton.getMeasuredHeight() / 2 - view.getMeasuredHeight() / 2;
            if (show) {
                ObjectAnimator translateX = ObjectAnimator.ofFloat(view, "translationX", x + x_trans, 0 + x_trans);
                ObjectAnimator translateY = ObjectAnimator.ofFloat(view, "translationY", y + y_trans, 0 + y_trans);
                ObjectAnimator alpha = ObjectAnimator.ofFloat(view, "alpha", 1, 0);
                animatorSet.playTogether(translateX, translateY, alpha);
                animatorSet.setStartDelay(50 * (count - 1 - i));
                animators.add(animatorSet);
            } else {
                ObjectAnimator translateX = ObjectAnimator.ofFloat(view, "translationX", 0 + x_trans, x + x_trans);
                ObjectAnimator translateY = ObjectAnimator.ofFloat(view, "translationY", 0 + y_trans, y + y_trans);
                ObjectAnimator alpha = ObjectAnimator.ofFloat(view, "alpha", 0, 1);
                animatorSet.playTogether(translateX, translateY, alpha);
                animatorSet.setStartDelay(50 * i);
                animators.add(animatorSet);
            }
        }
        animRunning = true;
        animatorSetTotal.playTogether(animators);
        animatorSetTotal.setDuration(300);
        if (show) {
            animatorSetTotal.setInterpolator(new AnticipateInterpolator());
        } else {
            animatorSetTotal.setInterpolator(new OvershootInterpolator());
        }
        animatorSetTotal.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                animRunning = false;
            }
        });
        animatorSetTotal.start();
        show = !show;
    }
```

上面这个方法就是卫星菜单的展开和收缩的动画控制了，根据角度的计算(详见上篇)获得不同角度展开下的平分角度和卫星的xy坐标，并根据toggle和卫星的宽高来计算出需要的动画偏移，并新增alpha动画，之后设定合适的Interpolator，执行动画。

至此卫星控件以及可以进行360度的展开和收缩了，我们仍需继续设计来让控件能够识别出自身的位置，并做出展开角度的判断。
在toggleShow方法中，我新增八个判断条件，分别识别靠近四个角，靠近四个边的情况。

```java
    public void toggleShow() {
        if (animRunning) return;

        int left = toggleButton.getLeft();
        int top = toggleButton.getTop();
        int right = getMeasuredWidth() - (left + toggleButton.getWidth());
        int bottom = getMeasuredHeight() - (top + toggleButton.getHeight());

        if (left < Radius && top < Radius) {
            partToggle(90, 0);
            return;
        }
        if (right < Radius && top < Radius) {
            partToggle(90, 90);
            return;
        }
        if (right < Radius && bottom < Radius) {
            partToggle(90, 180);
            return;
        }
        if (left < Radius && bottom < Radius) {
            partToggle(90, 270);
            return;
        }
        if (top < Radius) {
            partToggle(180, 0);
            return;
        }
        if (right < Radius) {
            partToggle(180, 90);
            return;
        }
        if (bottom < Radius) {
            partToggle(180, 180);
            return;
        }
        if (left < Radius) {
            partToggle(180, 270);
            return;
        }

        partToggle(360, 0);
    }
```
根据展开的半径和控件侧边距离父容器的距离进行对比，如果距离过小，则判定为依靠，并根据判定进行相应的角度和展开设定。

代码地址：https://github.com/NightFarmer/SatelliteButton











