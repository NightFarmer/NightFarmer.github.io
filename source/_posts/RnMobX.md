---
title: 在ReactNative中使用MobX框架管理State实现ListView的局部更新
date: 2017-02-14 09:11:30
tags: [Android, ReactNative]
category: ReactNative
---

在ReactNative中UI的刷新大多数情况依赖于state的变更，通过调用组件的setState方法来更新state以达到通知组件重新渲染UI的目的。当然这种做法是官方提供的标准解决方案，在进行简单UI设计时足以满足大多数需求。
但是当遇到结构复杂并存在数据交互的界面设计时，手动管理state这种做法则会把代码逻辑变得非常混乱，组件内不但要负责UI的渲染，还要兼顾state的变更以及不同组件间数据的传递和同步。当项目遇到这种时，引入一个状态管理框架则显得尤为重要。
本篇则着重以一个List的数据展示及更新的例子来展开，记录如何在ReactNative中使用`MobX`框架来代替手动管理state更新Ui，以及如何实现最小粒度的界面渲染。
下面两图分别是不做任何处理的全局更新列表数据和使用`MobX`框架局部更新列表数据的运行效果，可以看到在使用`MobX`框架在更新行数据时只会重新渲染当前行的组件，而全局更新则会将ListView中的所有ItemView全部渲染一次，同时在实际操作中也会感到细微的卡顿。
![image](http://nightfarmer.github.io/public/static/image/rnListRefresh.gif) ![image](http://nightfarmer.github.io/public/static/image/mobxListRefresh.gif)
<!-- more -->
# 介绍

MobX是一个经久考验的库,使得状态管理简单而且透明、可伸缩的应用功能反应性编程(TFRP)。MobX背后的哲学很简单:

任何可以由应用程序状态,应该是自动派生的。

包括用户界面、数据序列化、服务器通信,等等

![image](https://raw.githubusercontent.com/mobxjs/mobx/master/docs/flow.png)

React和MobX在一起是一个强大的组合，React呈现应用程序状态通过提供机制,把它翻译成可渲染的树组件，React使用MobX提供的机制来存储和更新应用程序。

React和MobX提供非常优和独特的在应用程序开发中常见问题的解决方案。React提供了机制优化渲染UI使用虚拟DOM,减少高代价的DOM突变的数量。MobX提供机制优化同步应用程序状态和React组件通过使用活性虚拟依赖状态图,只有当严格需要更新,永远不会过期。

核心概念

1. Observable state
可用于订阅的状态
2. Computed values
可用于订阅的计算结果
3. Reactions
触发事件并通知订阅者
4. Actions
用于粒度更新state
 
# 在ReactNative中配置环境

安装依赖
```
npm install mobx --save
npm install mobx-react --save
```
或
```
npm i mob mobx-react --save
```

安装babel插件，以支持ES7的decorator特性
```
npm i babel-plugin-transform-decorators-legacy babel-preset-react-native-stage-0 --save-dev
```
创建或编辑babel配置文件
.babelrc：
```
{
 'presets': ['react-native'],
 'plugins': ['transform-decorators-legacy']
}
```

# 在ReactNative中使用

下面以一个代办事件的列表为例，来展示如何使用`MobX`
```
import {observable, computed, action} from 'mobx'
import {observer} from 'mobx-react/native'
```
## 使用订阅对象代替state
```JavaScript
//待办事项行数据
class TodoListItem {
    @observable
    title;

    @observable
    finished = false;

    @action
    toggleFinish() {
        this.finished = !this.finished;
    }
}
```
`@observable`是一个装饰器(`Decorator`)，标注了`@observable`的类属性将作为一个可订阅的字段存在，任何修该字段的行为都会发送一个事件给所有订阅了这个字段的订阅者。
`@action`装饰器会把装饰的方法作为一个原子性操作，在方法完成后才会将最终执行结果与所关联的订阅字段进行比较，然后针对发生变更的订阅字段进行事件推送。
```JavaScript
//待办事项列表数据
class TodoListHolder {
    @observable
    dataList = [];

    @computed
    get taskLeft() {
        return this.dataList.filter((it) => !it.finished).length
    }
}
```
`@computed`标记的get方法会把方法的返回值作为订阅对象，方法中使用到的`@observable`字段变更时都会造成此方法被重新执行，但是只有在方法返回值变更时，才会对订阅此方法的订阅者进行通知。
## 将组件作为订阅者来进行注册，并绑定订阅对象的属性来渲染UI
接下来定义三个组件，分别代表父容器，列表组件，和数量统计组件

TodoList.js
```JavaScript
class TodoList extends Component {
    todoList = new TodoListHolder();

    constructor(props) {
        super(props);
        for (let i = 1; i < 30; i++) {
            let listItem = new TodoListItem();
            listItem.title = `待办事项${i}`;
            this.todoList.dataList.push(listItem)
        }
    }

    render() {
        return <View {...this.props}>
            <TodoListView todoListHolder={this.todoList}/>
            <TodoSumView todoListHolder={this.todoList}/>
        </View>
    }
}
```
创建一个`TodoListHolder`对象作为组件的成员变量用于数据的订阅，初始化一些测试数据，并将可订阅的数据对象传入列表组件和统计组件

TodoListView.js
```JavaScript
//列表展示组件
@observer
class TodoListView extends Component {
    ds = new ListView.DataSource({
        rowHasChanged: (r1, r2) => {
            return r1 !== r2
        }
    });

    render() {
        return <ListView style={{flex:1}}
                         enableEmptySections={true}
                         dataSource={this.ds.cloneWithRows(this.props.todoListHolder.dataList.slice())}
                         renderRow={this.renderRow}
        />
    }
    renderRow = (rowData) => {
        return <TodoListItemView rowData={rowData}/>
    }
}
```
```JavaScript
//列表行组件
@observer
class TodoListItemView extends Component {
    renderCount = 0;
    render() {
        this.renderCount++;
        return <TouchableOpacity style={{height:50,flexDirection:'row'}}
                                 onPress={()=>this.props.rowData.toggleFinish()}>
            <Text style={{flex: 1}}> {this.props.rowData.title} {this.props.rowData.finished && '√'}</Text>
            <Text> {this.renderCount}次渲染</Text>
        </TouchableOpacity>
    }
}

export default TodoListView
```
使用`@observer`来标注列表组件可以接收订阅通知，并在render方法中使用订阅对象的数据。
同样ListView的ItemVie也使用`@observer`来进行标注，并在render方法中使用订阅对象的数据渲染UI。
**在UI的触发事件回调函数中调用订阅对象的`@action`方法来变更订阅对象的数据，之后`MobX`框架则会根据变更的数据来通知使用了这些数据的observer组件重新调用render方法来更新UI**。

TodoSumView.js
```JavaScript
//统计组件
@observer
class TodoSumView extends Component {
    render() {
        return <View style={{height:30,backgroundColor:'#e4d3ff'}}>
            <Text>待办事项总数量：{this.props.todoListHolder.taskLeft}</Text>
        </View>
    }
}

export default TodoSumView
```
统计组件同样使用`@observer`来进行标注订阅。使用订阅对象的`@computed`值来绑定UI，当`@computed`值发生变更的时候组件作为订阅者接收通知并重新渲染UI

## 最小粒度的拆分组件达到小粒度的更新UI
如上述代码中所示，列表组件和统计组件的分别定义，以及列表组件ItemView的单独定义，都是以最小粒度地拆分组件为目的的设计。这种设计的好处一是遵从了ReactNative组件化的设计思想，二是使用MobX框架将这些组件单独作为订阅者进行注册，共享了父容器的state并支持了小粒度的局部UI更新。
可以这样说，在使用MobX的情况下，组件拆分的粒度决定了UI更新的粒度。

MobX的github地址：

https://github.com/mobxjs/mobx

https://github.com/mobxjs/mobx-react
