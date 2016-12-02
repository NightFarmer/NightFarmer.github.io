---
title: Android7.0新增工具DiffUtil详解
date: 2016-12-02 14:40:37
tags: [Android, RecyclerView]
category: Android
---

自从Google提供了RecyclerView之后，这个控件在Android列表和表格的开发中越来越多的替代了ListView和GridView，而且RecyclerView额外的提供了一些旧控件所没有的行为，比如局部刷新、列表表格更新动画、元素拖拽等等，极大程度的简化和方便了集合数据的展示UI的开发。
而在使用RecyclerView进行一些局部刷新时，往往会手动记录集合元素的增、删、变更、移动等事件，并分别调用不同的方法来通知adapter来对控件进行更新。而本篇介绍的工具类则对这些操作进行了完美的封装，让我们不再需要手动计算并记录元素的变更，而是交由DiffUtil来计算新旧数据集合的差异并自动通知adapter来通过不同的更新方法来进行UI。
![排序](http://nightfarmer.github.io/public/static/image/diffutil1.gif)  ![局部刷新](http://nightfarmer.github.io/public/static/image/diffutil2.gif)
<!-- more -->

#### 基本用法
在已经使用了RecyclerView的项目中使用DiffUtil非常简单
```java
DiffResult diffResult = DiffUtil.calculateDiff(new MyCallBack(oldList, newList));
adapter.dataList = newList;
diffResult.dispatchUpdatesTo(adapter);
```
比较重要的方法调用只有上面三行代码，那么这三行代码究竟做了什么呢？
1、使用DiffUtil计算新的dataList和旧的dataList的区别，这个区别包含了数据的增加和移除，元素位置的移动，以及元素数据的变更。
2、将新的数据列表赋值给adapter
3、使用DiffUtil计算的结果来通知adapter所有的数据变更，并自动调用所需要的UI刷新方法。

过程1中使用到了一个MyCallBack类，这个类是我们提供给DiffUtil来计算数据变更的依据，继承了DiffUtil.Callback:
```java
public class MyCallBack extends DiffUtil.Callback {
    List<Person> newList;
    List<Person> oldList;

    public MyCallBack(List<Person> newList, List<Person> oldList) {
        this.newList = newList;
        this.oldList = oldList;
    }
    //获取旧数据源的数据集合大小
    @Override
    public int getOldListSize() {
        return oldList.size();
    }
    //获取新数据源的数据集合大小
    @Override
    public int getNewListSize() {
        return newList.size();
    }
    //用来匹配新旧数据源中的数据，若匹配到了则旧数据源对应的View可以更新或移动，没有匹配到的旧数据则需要移除
    @Override
    public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
        return oldList.get(oldItemPosition).getId() == newList.get(newItemPosition).getId();
    }
    //通过上一个方法匹配到的数据会调用此方法，用来判断旧数据源中的数据是否变更
    @Override
    public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
        return oldList.get(oldItemPosition).getName().equals(newList.get(newItemPosition).getName());
    }
}
```
这个类给DiffUtil提供了用来比较新旧元素依据，DiffUtil会根据callback和自己的算法来计算所有数据的变更，并将所有的变更记录返回。


过程3中dispatchUpdatesTo()方法的调用其实只是对RecyclerView方法的一个包装，通过源码我们可以看到：
```java
        public void dispatchUpdatesTo(final RecyclerView.Adapter adapter) {
            dispatchUpdatesTo(new ListUpdateCallback() {
                @Override
                public void onInserted(int position, int count) {
                    adapter.notifyItemRangeInserted(position, count);
                }

                @Override
                public void onRemoved(int position, int count) {
                    adapter.notifyItemRangeRemoved(position, count);
                }

                @Override
                public void onMoved(int fromPosition, int toPosition) {
                    adapter.notifyItemMoved(fromPosition, toPosition);
                }

                @Override
                public void onChanged(int position, int count, Object payload) {
                    adapter.notifyItemRangeChanged(position, count, payload);
                }
            });
        }
```
所以说DiffUtil不只可以用于RecyclerView的刷新，当我们有两个新旧数据源而展示UI使用的是其他的控件且需要计算新数据源相对于旧数据源的变更方法时，都可以使用DiffUtil来进行计算，并使用dispatchUpdatesTo方法通过传入的ListUpdateCallback来接收处理结果。

#### Item的局部更新
DiffUtil给我们提供了RecyclerView的局部更新的封装方法，使我们在更新数据源时RecyclerView不用更新全部的ItemView，这样极大程度的节省了刷新UI的开销。
**同时 DiffUtil提供了更细粒度的更新，对ItemView中的局部数据进行更新，而不是更新ItemView中的所有子View**
我们假设一个场景，现在有一个通讯录列表，每一行都有联系人的头像和名称以及联系方式，而我们更新了一些联系人的手机号，头像和名称并没有改变，这时候如果使用上面的方法，那么所有数据变动的联系人对应的ItemView都会被重新绑定一次数据，也就说都要重新设置一次头像，相应的我们会看到这些ItemView都会有一次更新动画的执行。
而使用下面的这个方法，则会让对应的ItemView只更新联系方式对应的TextView，联系人头像对应的ImageView则不会进行任何变动。
```java
public class MyCallBack extends DiffUtil.Callback {
    ....
    //获取新数据源的数据集合大小
    @Override
    public int getNewListSize() {
        return newList.size();
    }
    //用来匹配新旧数据源中的数据，若匹配到了则旧数据源对应的View可以更新或移动，没有匹配到的旧数据则需要移除
    @Override
    public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
        return oldList.get(oldItemPosition).getId() == newList.get(newItemPosition).getId();
    }
    //通过上一个方法匹配到的数据会调用此方法，用来判断旧数据源中的数据是否变更
    @Override
    public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
        return oldList.get(oldItemPosition).getName().equals(newList.get(newItemPosition).getName());
    }
    //通过上一个方法判断到需要更新数据的元素会调用此方法，返回null则表示更新所有数据，返回非空数据则可以返回任何可以代表局部更新数据也可以返回Bundle。
    @Nullable
    @Override
    public Object getChangePayload(int oldItemPosition, int newItemPosition) {
        if (!oldList.get(oldItemPosition).imgUrl.equals(newList.get(newItemPosition).imgUrl)) {
            return null;
        }
        return "ImgNoChange";
    }
}
```
当CallBack使用了getChangePayload时，Adapter中同样也需要复写一个方法来接收这个方法返回的局部更新标识
```java
public class MyAdapter extends RecyclerView.Adapter {

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position, List payloads) {
        if (payloads.isEmpty()) {
            onBindViewHolder(holder, position);//对ItemView重新绑定全部数据
        } else {
            //todo 只更新联系人对应的TextView
        }
    }

    ...
}
```
这个方法需要注意，这里重写的三个参数的onBindViewHolder，这个方法默认情况是不做任何处理直接调用两参的onBindViewHolder的。
在getChangePayload方法中返回null时，onBindViewHolder的第三个参数会返回一个空的List，这时候需要调用两参的onBindViewHolder来进行全部更新
在getChangePayload方法中返回非空对象时，onBindViewHolder的第三个参数会把这个对象放在list的第一个元素中传入，接下来可以根据这个对象来解析如何对item进行更新，这里只传入了一个字符串代表不更新图片，所以只对ItemView中的TextView的值进行设置即可，而不用调用两参方法来bind整个ItemView了。