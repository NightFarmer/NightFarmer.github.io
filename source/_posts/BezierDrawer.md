---
title: 做一个二阶Bezier曲线的绘图工具
date: 2016-08-23 10:51:11
tags: [Android, Bezier曲线, 工具]
category: Android
---

在开发一些特效或者控件的时候常常会用到一些数学函数，除了三角数学函数外，使用最多的就是Bezier了，也就是我们常说的贝塞尔曲线。
Android开发中常常会使用Bezier曲线绘制一些图形，或者使用Bezier曲线生成一些动画的运动轨迹，亦或者使用贝塞尔曲线制作动画的差值器。
总之，这是一个作用非常大的东西，它不像三角数学函数那样有固定的曲线特性，而是根据参数的变更而呈现出不同的曲线，可以这样说，Bezier曲线可以根据用户的需求调整自身的曲线形式，可以满足绝大部分用户对曲线的需求，如果不能，那就再加一阶。
本篇的初衷是因为贝塞尔函数看起来是非常抽象的，使用时则需要不断调整参数来尝试出合适的结果，而本工具则是根据调整出的曲线逆向显示出函数的参数，废话不多说，上图：
![Bezier曲线绘制效果](http://nightfarmer.github.io/public/static/image/BezierDrawer.gif)
<!-- more -->

前言：本篇不描述Bezier算法的实现，只基于AndroidSDK记录二阶Bezier曲线函数的API使用。

Android为我们提供了二阶和三阶Bezier函数的API，更高阶的Bezier函数可以使用[github](https://github.com/7heaven/Bezier)上的扩展实现
二阶和三阶的AndroidAPI分别是：
```java
//二阶，四个参数分别代表锚点坐标和终点坐标
path.quadTo(anchorPoint.x, anchorPoint.y, endPoint.x, endPoint.y);
//三阶，四个参数分别代表两个锚点坐标和终点坐标
path.cubicTo(x1, y1, x2, y2, x3, y3);
```
上面两个是Path类提供的方法，而我们如果需要绘制的话则可以使用canvas的drawPath方法进行使用。

为了实现开篇的效果，我们首先需要定义一个View，并初始化一些Paint。
```java
public class BezierDrawer extends View {
    private Paint paint;//曲线绘制画笔
    private Paint lineEndPaint;//曲线末端圆角画笔
    private Paint pointPaint;//曲线起止点及锚点画笔
    private Paint linePaint;//辅助线画笔
    //构造方法略...构造方法中调用init方法

    private void init() {
        paint = new Paint();
        paint.setColor(Color.BLUE);
        paint.setStrokeWidth(20);
        paint.setStyle(Paint.Style.STROKE);
        paint.setAntiAlias(true);

        lineEndPaint = new Paint();
        lineEndPaint.setColor(Color.BLUE);
        lineEndPaint.setStyle(Paint.Style.FILL);
        lineEndPaint.setAntiAlias(true);

        pointPaint = new Paint();
        pointPaint.setStrokeWidth(8);
        pointPaint.setColor(Color.DKGRAY);
        pointPaint.setStyle(Paint.Style.STROKE);
        pointPaint.setAntiAlias(true);

        linePaint = new Paint();
        linePaint.setStrokeWidth(2);
        linePaint.setColor(Color.DKGRAY);
        linePaint.setStyle(Paint.Style.STROKE);
        linePaint.setAntiAlias(true);
    }
```

在创建过画笔之后我们需要知道绘制二阶Bezier曲线需要的三个点，我们初始化一个List&lt;Point&gt;来存贮这些点，并在控件被首次测量的时候初始化这些位置。
```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int width = getMeasuredWidth();
        int height = getMeasuredHeight();
        if (pointList.size() == 0) {
            pointList.add(new Point(width / 5, height / 2));
            pointList.add(new Point(width / 2, height / 4));
            pointList.add(new Point(width / 5 * 4, height / 2));
            if (onChangeListener != null) onChangeListener.onchange(pointList);
        }
    }
```

这样我们有了一些固定的点，那么接下来我们在ondraw方法中讲这些点使用Bezier的API把曲线画出来
```java
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Path path = new Path();
        Point startPoint = pointList.get(0);//开始点
        Point anchorPoint = pointList.get(1);//锚点
        Point endPoint = pointList.get(2);//结束点
        //绘制曲线
        path.moveTo(startPoint.x, startPoint.y);
        path.quadTo(anchorPoint.x, anchorPoint.y, endPoint.x, endPoint.y);
        canvas.drawPath(path, paint);
        //绘制曲线两端的圆角
        canvas.drawCircle(startPoint.x, startPoint.y, paint.getStrokeWidth() / 2, lineEndPaint);
        canvas.drawCircle(endPoint.x, endPoint.y, paint.getStrokeWidth() / 2, lineEndPaint);
        //绘制辅助线
        canvas.drawLine(startPoint.x, startPoint.y, anchorPoint.x, anchorPoint.y, linePaint);
        canvas.drawLine(anchorPoint.x, anchorPoint.y, endPoint.x, endPoint.y, linePaint);
        //绘制起止点及锚点
        for (Point point : pointList) {
            canvas.drawCircle(point.x, point.y, 20, pointPaint);
        }
    }
```

这时候一个最简单的二阶Bezier曲线以及被我们画出来了，那么接下来要做的就是让这个曲线可以根据我们的触摸滑动操作来进行变化，复写onTouchEvent方法
```java
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                Point point = new Point((int) event.getX(), (int) event.getY());
                touchedPoint = getNearestPoint(point);
                break;
            case MotionEvent.ACTION_MOVE:
                if (touchedPoint != -1) {
                    Point pointMove = new Point((int) event.getX(), (int) event.getY());
                    pointList.set(touchedPoint, pointMove);//如果有一个点处于被用户操作中，则根据用户的拖拽操作的坐标更新这个点
                    invalidate();
                    if (onChangeListener != null) onChangeListener.onchange(pointList);//定义一个回调接口，回调点的信息
                }
                break;

            case MotionEvent.ACTION_UP:
                touchedPoint = -1;
                break;
            default:

        }
        return true;
    }

    int touchDist = 50;//触发距离，若用户触摸的点距离Bezier的参数点的xy坐标距离都小于这个数值，则认定为用户在触摸这个点
    //找出距离触摸点满足触发距离的点的index
    private int getNearestPoint(Point point) {
        for (int i = 0; i < pointList.size(); i++) {
            Point p = pointList.get(i);
            if (Math.abs(p.x - point.x) < touchDist && Math.abs(p.y - point.y) < touchDist) {
                return i;
            }
        }
        return -1;
    }
```

补上监听相关的代码
```java
    interface OnChangeListener {
        void onchange(List<Point> pointList);
    }
    private OnChangeListener onChangeListener;
    public void setOnChangeListener(OnChangeListener onChangeListener) {
        this.onChangeListener = onChangeListener;
    }
```

完工

完整的工程代码及示例见github：https://github.com/NightFarmer/BezierDemo

