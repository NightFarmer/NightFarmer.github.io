---
title: RxJava常用操作符分类总汇
date: 2016-09-01 15:38:23
tags: [Android, RxJava, 干货]
category: Android
---

[上文](http://nightfarmer.github.io/2016/08/31/RxJavaClick/)提到ReactiveX在多种语言和平台上都存在着自己的实现，同时也都实现了一套操作符，在这些语言平台中有一些操作符的实现是重叠的，也有一些只存在于特定的平台实现中。每种实现都会根据当前平台的命名规范给这些操作符定义相似的命名。
本文收集了Java平台Rx的一些常用操作符，并对这些操作符进行分类并简单注释这些操作符的含义和用法。
<!-- more -->
#### 创建
`create` 提供一个回调对象，通过对回调方法中传入的subscriber进行操作实现对订阅者的操作。
`just` 传入一个或多个对象，生成一个Observable，并会依次发射这些对象
`from` 传入一个集合Array/List，生成一个Observable，并会一次发射集合中的对象
`range` 指定起始值/偏移值/调度线程，生成一个Observable，并根据指定的整数区间一次发射整型数值。
`interval` 指定首次延迟/周期/时间单位/调度线程，生成一个Observable，并会根据定时器从0开始不断发射Long类型计数对象
`timer` 指定延迟数值/时间单位/调度线程，生成一个Observable，在经过一段事件延迟后，发射一个Long类型数值(0)
`defer` 传入一个Observable构造器，生成的Observable包含这个构造器，在观察者订阅时才生成真实的Observable，多次订阅会多次调用构造器返回Observable，
`empty` 生成一个Observable，只会发射一个complete事件
`never` 生成一个Observable，什么事件都不会发射


#### 变换
这些操作符可用于对Observable发射的数据进行变换，详细解释可以看每个操作符的文档
`map` 映射，将一种类型的数据流/Observable映射为另外一种类型的数据流/Observable
`cast` 强转 传入一个class,对Observable的类型进行强转.
`buffer` 缓存/打包 按照一定规则从Observable收集一些数据到一个集合，然后把这些数据作为集合打包发射。
`flatMap` 平铺映射，从数据流的每个数据元素中映射出多个数据，并将这些数据依次发射。
`groupby` 分组，将原来的Observable分拆为Observable集合，将原始Observable发射的数据按Key分组，每一个Observable发射一组不同的数据
`to...` 将数据流中的对象转换为List/SortedList/Map/MultiMap集合对象，并打包发射
`timeInterval` 将每个数据都换为包含本次数据和离上次发射数据时间间隔的对象并发射
`timestamp` 将每个数据都转换为包含本次数据和发射数据时的时间戳的对象并发射

#### 过滤
`debounce` 传入时间间隔/时间单位/调度线程/，**如果超过这段时间没有新的数据传入**，则从数据流中获取最后一个数据进行发射，其他的数据则忽略掉。
`distinct` 去重，过滤掉重复数据项
`elementAt` 从数据流中取第几个数据项
`filter` 过滤，从数据流中只去满足条件的数据用来发射
`first/last` 传入或不传入条件，只发射第一个/最后一个或第一个/最后一个满足条件的数据。
`ignoreElements` 忽略所有数据，只保留终止通知(onError或onCompleted)
`sample` 取样，**定期**抽取一条最新的数据，发射
`throttleFirst` 传入时间，**等待**并从数据流中取新数据，**如果距离上次发射的时间间隔大于这个时间**，则发射这条数据，反之忽略
`throttleLast` 传入时间，**定期**从数据流中取最新的数据，忽略本次与上次发射数据之间的数据。效果与`sample`相似
`skip/skipLast` 传入数目或时间，跳过前面/后面的几个数据或这段时间内的数据
`take/takeLast` 传入数目或时间，只取前面/后面的几个数据或这段时间内的数据

#### 组合
组合操作符用于将多个Observable组合成一个单一的Observable
`concat` **拼接** 将多个/Array/List集合的Observable**拼接**为一个Observable，并按传入的Observable的顺序依次发射数据。
`merge` **合并** 将多个/Array/List集合的Observable**合并**为一个Observable，并按所有数据的顺序依次发射数据
`startWith` 传入一个/多个数据/Observable，在本Observable的数据发射前，首先把传入的数据流全部依次发射。
`zip` 打包 将两个或多个Observable的数据流并行打包，每次从各个Observable各取一个数据，通过合并函数合并为一个数据后，发射数据。每次打包都会对每个Observable等待最新的数据.
`combineLatest` 打包 和zip类似, 区别在于每次打包不等待，任何一个Observable发射数据都会打包一次, 其他Observable则取最近发射的数据.
`Join` 无论何时，如果一个Observable发射了一个数据项，只要在另一个Observable发射的数据项定义的时间窗口内，就将两个Observable发射的数据合并发射 
`switchOnNext` 切换 若存在一个发射Observable的Observable, switchOnNext会在发射Observable时将Observable中的数据发射出去, 若第二个Observable开始发射,则上一个被发射的Observable中的数据停止发射.


#### 错误处理
这些操作符用于从错误通知中恢复
`retry` 重试 传入重试次数或重试判断函数，当异常时按次数从头重试，或根据判断函数判断是否从头重试。
`onErrorReturn` 替换异常，当发生error时，返回根据异常返回一个正常的数据，代替error数据，并以complete终止数据流
`onErrorResumeNext` 替换异常，当发生error时，根据异常返回一个Observable，以这个Observable中的数据代理error数据发射，并以complete终止数据流
`onExceptionResumeNext` 与onErrorResumeNext相似，不过只替换Exception类型的异常，对Throwable类型的则不处理，会onError

#### 辅助

辅助操作

一组用于处理Observable的操作符
`delay` 延迟一段时间发射数据
`doOn...` 在某个事件时被回调
`observeOn` 指定观察者被通知时的调度线程
`subscribeOn`指定Observable应该在哪个调度程序上执行
`subscribe` 注册观察者的监听回调用于接收数据，并触发Observable的OnSubscribe的call方法，通知Observable一切就绪。变换/过滤操作符在subscribe方法调用前并不会被执行。
`timeout` 传入时间和单位，超过这段事件没有接收到数据，就会发送一个error。
`materialize/dematerialize` 将数据包装进Notification，并带有Kind(OnNext/OnError/OnCompleted)，合并三个类型的回调到onNext，Dematerialize是反过程
Serialize — 强制Observable按次序发射数据并且功能是有效的
`using` 创建一个Observable，传入三个函数，第一个函数生成并返回一个资源，第二个函数传入由第一个函数生成的资源并生成一个Observable作为Observable，第三个函数在Observable完成注销后调用，传入第一个函数生成的资源进行回收处理。

#### 条件和布尔操作

这些操作符可用于单个或多个数据项，也可用于Observable
`all` 依次根据函数判断所有数据项，判断所有数据是否全部满足条件，并只回调一次订阅监听。
`amb` 传入多个Observable，他们是竞争关系，哪个Observable首先发射数据，则只发射这个Observable的数据，忽略其他的Observable的所有数据。
`contains` 传入一个数据，判断数据流中的数据是否包含本数据，并只回调一次订阅监听。
`defaultIfEmpty` 传入一个数据，如果数据流为空的时候发射本数据。
`sequenceEqual` 传入两个Observable和一个判断函数，判断两个数据队列是否是相同的序列，并只回调一次订阅监听。
`skipWhile` 传入一个判断函数，从数据流中的第一个数据开始依次判断，如果满足条件则跳过，如果不满足则开始依次发射后续的所有数据
`takeWhile` 传入一个判断函数，如果从数据流中获取到的数据满足判断条件，则发射此数据，如果不满足则忽略以后所有数据。
`skipUtil` 传入一个Observable，在这个Observable发射数据之前原始Observable的所有数据都被忽略，直到传入的Observable发出第一个数据，则原始Observable开始发射数据
`takUtil` 传入一个Observable，在这个Observable发射数据之前原始Observable的所有数据都被正常发射，直到传入的Observable发出第一个数据，则原始Observable忽略后续的所有数据

####  算术和聚合操作

这些操作符可用于整个数据序列
`count` 计算Observable发射的数据个数，然后发射这个数字。
`reduce` 传入计算函数，根据初始值和每次计算的结果对每个数据进行计算，并将计算结果传入下个数据的计算参数，将最终结果发射。
`collect` 和reduce类似，不过不是计算而是对象的聚合，传入两个函数，第一个是结构体的初始化并返回，第二个是聚合函数，用于聚合的结构体和每个数据传入并将数据聚合到结构体中，最终将聚合完成的结构体发射给订阅者
`scan` 和reduce一样
`window` 和buffer类似, 不同之处在于返回的聚合对象是Observable

#### 连接操作

一些有精确可控的订阅行为的特殊Observable

`share`share保证引用的是同一个资源，而不是subscriber在不同的时间点订阅，会收到准确的相同的数据，是publish和refcount调用立案的包装，等价于.publish().refcount()。
`publish` 将一个普通的Observable转换为可连接的，可连接的Observable在调用.connect之前接受不到任何数据
`refCount` 使一个可连接的Observable表现得像一个普通的Observable
`connect` 指示一个可连接的Observable开始发射数据给订阅者
`replay` 确保所有的观察者收到同样的数据序列，即使他们在Observable开始发射数据之后才订阅


#### 其他
`toBlocking` 阻塞Observable的操作符。可用与单元测试中，在subscribe之前调用，使方法阻塞等待便于观察测试过程。
操作符决策树

#### 几种主要的需求

直接创建一个Observable（创建操作）
组合多个Observable（组合操作）
对Observable发射的数据执行变换操作（变换操作）
从Observable发射的数据中取特定的值（过滤操作）
转发Observable的部分值（条件/布尔/过滤操作）
对Observable发射的数据序列求值（算术/聚合操作）


