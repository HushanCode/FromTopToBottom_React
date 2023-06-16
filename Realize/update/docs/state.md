# 状态更新

# 一、几个关键节点

## 1.render阶段的开始

我们在[render阶段流程概览一节](https://react.iamkasong.com/process/reconciler.html)讲到，

`render阶段`开始于`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`方法的调用。这取决于本次更新是同步更新还是异步更新。

## 2.commit阶段的开始

我们在[commit阶段流程概览一节](https://react.iamkasong.com/renderer/prepare.html)讲到，

`commit阶段`开始于`commitRoot`方法的调用。其中`rootFiber`会作为传参。

我们已经知道，`render阶段`完成后会进入`commit阶段`。让我们继续补全从`触发状态更新`到`render阶段`的路径。

```sh
触发状态更新（根据场景调用不同方法）
    |
    |
    v

    ？

    |
    |
    v
render阶段（`performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`）

    |
    |
    v

commit阶段（`commitRoot`）
```

## 3.创建Update对象

在`React`中，有如下方法可以触发状态更新（排除`SSR`相关）：

- ReactDOM.render
- this.setState
- this.forceUpdate
- useState
- useReducer

这些方法调用的场景各不相同，他们是如何接入同一套**状态更新机制**呢？

答案是：每次`状态更新`都会创建一个保存**更新状态相关内容**的对象，我们叫他`Update`。在`render阶段`的`beginWork`中会根据`Update`计算新的`state`。

我们会在下一节详细讲解`Update`

## 4.从fiber到root

现在`触发状态更新的fiber`上已经包含`Update`对象。

我们知道，`render阶段`是从`rootFiber`开始向下遍历。那么如何从`触发状态更新的fiber`得到`rootFiber`呢？

答案是：调用`markUpdateLaneFromFiberToRoot`方法。

> 你可以从[这里](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L636)看到`markUpdateLaneFromFiberToRoot`的源码

该方法做的工作可以概括为：从`触发状态更新的fiber`一直向上遍历到`rootFiber`，并返回`rootFiber`。

由于不同更新优先级不尽相同，所以过程中还会更新遍历到的`fiber`的优先级。这对于我们当前属于超纲内容。

## 5.调度更新

现在我们拥有一个`rootFiber`，该`rootFiber`对应的`Fiber树`中某个`Fiber节点`包含一个`Update`。

接下来通知`Scheduler`根据**更新**的优先级，决定以**同步**还是**异步**的方式调度本次更新。

这里调用的方法是`ensureRootIsScheduled`。

以下是`ensureRootIsScheduled`最核心的一段代码：

```js
if (newCallbackPriority === SyncLanePriority) {
  // 任务已经过期，需要同步执行render阶段
  newCallbackNode = scheduleSyncCallback(
    performSyncWorkOnRoot.bind(null, root)
  );
} else {
  // 根据任务优先级异步执行render阶段
  var schedulerPriorityLevel = lanePriorityToSchedulerPriority(
    newCallbackPriority
  );
  newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root)
  );
}
```

> 你可以从[这里](https://github.com/facebook/react/blob/b6df4417c79c11cfb44f965fab55b573882b1d54/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L602)看到`ensureRootIsScheduled`的源码

其中，`scheduleCallback`和`scheduleSyncCallback`会调用`Scheduler`提供的调度方法根据`优先级`调度回调函数执行。

可以看到，这里调度的回调函数为：

```js
performSyncWorkOnRoot.bind(null, root);
performConcurrentWorkOnRoot.bind(null, root);
```

即`render阶段`的入口函数。

至此，`状态更新`就和我们所熟知的`render阶段`连接上了。

# 二、状态更新调用路径

```sh
触发状态更新（根据场景调用不同方法）

    |
    |
    v

创建Update对象（接下来三节详解）

    |
    |
    v

从fiber到root（`markUpdateLaneFromFiberToRoot`）

    |
    |
    v

调度更新（`ensureRootIsScheduled`）

    |
    |
    v

render阶段（`performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`）

    |
    |
    v

commit阶段（`commitRoot`）
```

# 三、更新心智模型

## 1.同步更新的React

我们可以将`更新机制`类比`代码版本控制`。

在没有`代码版本控制`前，我们在代码中逐步叠加功能。一切看起来井然有序，直到我们遇到了一个紧急线上bug（红色节点）。

![流程1](https://cdn.jsdelivr.net/gh/HushanCode/DrawingBed/img/git1.png)

为了修复这个bug，我们需要首先将之前的代码提交。

在`React`中，所有通过`ReactDOM.render`创建的应用（其他创建应用的方式参考[ReactDOM.render一节](https://react.iamkasong.com/state/reactdom.html#react的其他入口函数)）都是通过类似的方式`更新状态`。

即没有`优先级`概念，`高优更新`（红色节点）需要排在其他`更新`后面执行。

## 2.并发更新的React

当有了`代码版本控制`，有紧急线上bug需要修复时，我们暂存当前分支的修改，在`master分支`修复bug并紧急上线。

![流程2](https://cdn.jsdelivr.net/gh/HushanCode/DrawingBed/img/git2.png)

bug修复上线后通过`git rebase`命令和`开发分支`连接上。`开发分支`基于`修复bug的版本`继续开发。

![流程3](https://cdn.jsdelivr.net/gh/HushanCode/DrawingBed/img/git3.png)

在`React`中，通过`ReactDOM.createBlockingRoot`和`ReactDOM.createRoot`创建的应用会采用`并发`的方式`更新状态`。

`高优更新`（红色节点）中断正在进行中的`低优更新`（蓝色节点），先完成`render - commit流程`。

待`高优更新`完成后，`低优更新`基于`高优更新`的结果`重新更新`。

接下来两节我们会从源码角度讲解这套`并发更新`是如何实现的。

# 四、更新

## 1.Update的分类

我们先来了解`Update`的结构。

首先，我们将可以触发更新的方法所隶属的组件分类：

- ReactDOM.render —— HostRoot
- this.setState —— ClassComponent
- this.forceUpdate —— ClassComponent
- useState —— FunctionComponent
- useReducer —— FunctionComponent

可以看到，一共三种组件（`HostRoot` | `ClassComponent` | `FunctionComponent`）可以触发更新。

由于不同类型组件工作方式不同，所以存在两种不同结构的`Update`，其中`ClassComponent`与`HostRoot`共用一套`Update`结构，`FunctionComponent`单独使用一种`Update`结构。

虽然他们的结构不同，但是他们工作机制与工作流程大体相同。在本节我们介绍前一种`Update`，`FunctionComponent`对应的`Update`在`Hooks`章节介绍。

## 2.Update的结构

`ClassComponent`与`HostRoot`（即`rootFiber.tag`对应类型）共用同一种`Update结构`。

对应的结构如下：

```js
const update: Update<*> = {
  eventTime,
  lane,
  suspenseConfig,
  tag: UpdateState,
  payload: null,
  callback: null,

  next: null,
};
```

> `Update`由`createUpdate`方法返回，你可以从[这里](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactUpdateQueue.old.js#L189)看到`createUpdate`的源码

字段意义如下：

- eventTime：任务时间，通过`performance.now()`获取的毫秒数。由于该字段在未来会重构，当前我们不需要理解他。
- lane：优先级相关字段。当前还不需要掌握他，只需要知道不同`Update`优先级可能是不同的。

> 你可以将`lane`类比`心智模型`中`需求的紧急程度`。

- suspenseConfig：`Suspense`相关，暂不关注。
- tag：更新的类型，包括`UpdateState` | `ReplaceState` | `ForceUpdate` | `CaptureUpdate`。
- payload：更新挂载的数据，不同类型组件挂载的数据不同。对于`ClassComponent`，`payload`为`this.setState`的第一个传参。对于`HostRoot`，`payload`为`ReactDOM.render`的第一个传参。
- callback：更新的回调函数。即在[commit 阶段的 layout 子阶段一节](https://react.iamkasong.com/renderer/layout.html#commitlayouteffectonfiber)中提到的`回调函数`。
- next：与其他`Update`连接形成链表。

## 3.Update与Fiber的联系

我们发现，`Update`存在一个连接其他`Update`形成链表的字段`next`。联系`React`中另一种以链表形式组成的结构`Fiber`，他们之间有什么关联么？

答案是肯定的。

从[双缓存机制一节](https://react.iamkasong.com/process/doubleBuffer.html)我们知道，`Fiber节点`组成`Fiber树`，页面中最多同时存在两棵`Fiber树`：

- 代表当前页面状态的`current Fiber树`
- 代表正在`render阶段`的`workInProgress Fiber树`

类似`Fiber节点`组成`Fiber树`，`Fiber节点`上的多个`Update`会组成链表并被包含在`fiber.updateQueue`中。

> 什么情况下一个Fiber节点会存在多个Update？

你可能疑惑为什么一个`Fiber节点`会存在多个`Update`。这其实是很常见的情况。

在这里介绍一种最简单的情况：

```js
onClick() {
  this.setState({
    a: 1
  })

  this.setState({
    b: 2
  })
}
```

在一个`ClassComponent`中触发`this.onClick`方法，方法内部调用了两次`this.setState`。这会在该`fiber`中产生两个`Update`。

`Fiber节点`最多同时存在两个`updateQueue`：

- `current fiber`保存的`updateQueue`即`current updateQueue`
- `workInProgress fiber`保存的`updateQueue`即`workInProgress updateQueue`

在`commit阶段`完成页面渲染后，`workInProgress Fiber树`变为`current Fiber树`，`workInProgress Fiber树`内`Fiber节点`的`updateQueue`就变成`current updateQueue`

## 4.updateQueue

`updateQueue`有三种类型，其中针对`HostComponent`的类型我们在[completeWork一节](https://react.iamkasong.com/process/completeWork.html#update时)介绍过。

剩下两种类型和`Update`的两种类型对应。

`ClassComponent`与`HostRoot`使用的`UpdateQueue`结构如下：

```js
const queue: UpdateQueue<State> = {
    baseState: fiber.memoizedState,
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: {
      pending: null,
    },
    effects: null,
  };
```

> `UpdateQueue`由`initializeUpdateQueue`方法返回，你可以从[这里 (opens new window)](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactUpdateQueue.new.js#L157)看到`initializeUpdateQueue`的源码

字段说明如下：

- baseState：本次更新前该`Fiber节点`的`state`，`Update`基于该`state`计算更新后的`state`。

> 你可以将`baseState`类比`心智模型`中的`master分支`。

- `firstBaseUpdate`与`lastBaseUpdate`：本次更新前该`Fiber节点`已保存的`Update`。以链表形式存在，链表头为`firstBaseUpdate`，链表尾为`lastBaseUpdate`。之所以在更新产生前该`Fiber节点`内就存在`Update`，是由于某些`Update`优先级较低所以在上次`render阶段`由`Update`计算`state`时被跳过。

> 你可以将`baseUpdate`类比`心智模型`中执行`git rebase`基于的`commit`（节点D）。

- `shared.pending`：触发更新时，产生的`Update`会保存在`shared.pending`中形成单向环状链表。当由`Update`计算`state`时这个环会被剪开并连接在`lastBaseUpdate`后面。

> 你可以将`shared.pending`类比`心智模型`中本次需要提交的`commit`（节点ABC）。

- effects：数组。保存`update.callback !== null`的`Update`。

## 5.例子

`updateQueue`相关代码逻辑涉及到大量链表操作，比较难懂。在此我们举例对`updateQueue`的工作流程讲解下。

假设有一个`fiber`刚经历`commit阶段`完成渲染。

该`fiber`上有两个由于优先级过低所以在上次的`render阶段`并没有处理的`Update`。他们会成为下次更新的`baseUpdate`。

我们称其为`u1`和`u2`，其中`u1.next === u2`。

```js
fiber.updateQueue.firstBaseUpdate === u1;
fiber.updateQueue.lastBaseUpdate === u2;
u1.next === u2;
```

我们用`-->`表示链表的指向：

```js
fiber.updateQueue.baseUpdate: u1 --> u2
```

现在我们在`fiber`上触发两次状态更新，这会先后产生两个新的`Update`，我们称为`u3`和`u4`。

每个 `update` 都会通过 `enqueueUpdate` 方法插入到 `updateQueue` 队列上

当插入`u3`后：

```js
fiber.updateQueue.shared.pending === u3;
u3.next === u3;
```

`shared.pending`的环状链表，用图表示为：

```js
fiber.updateQueue.shared.pending:   u3 ─────┐ 
                                     ^      |                                    
                                     └──────┘
```

接着插入`u4`之后：

```js
fiber.updateQueue.shared.pending === u4;
u4.next === u3;
u3.next === u4;
```

`shared.pending`是环状链表，用图表示为：

```js
fiber.updateQueue.shared.pending:   u4 ──> u3
                                     ^      |                                    
                                     └──────┘
```

`shared.pending` 会保证始终指向最后一个插入的`update`，你可以在[这里 (opens new window)](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactUpdateQueue.new.js#L208)看到`enqueueUpdate`的源码

更新调度完成后进入`render阶段`。

此时`shared.pending`的环被剪开并连接在`updateQueue.lastBaseUpdate`后面：

```js
fiber.updateQueue.baseUpdate: u1 --> u2 --> u3 --> u4
```

接下来遍历`updateQueue.baseUpdate`链表，以`fiber.updateQueue.baseState`为`初始state`，依次与遍历到的每个`Update`计算并产生新的`state`（该操作类比`Array.prototype.reduce`）。

在遍历时如果有优先级低的`Update`会被跳过。

当遍历完成后获得的`state`，就是该`Fiber节点`在本次更新的`state`（源码中叫做`memoizedState`）。

> `render阶段`的`Update操作`由`processUpdateQueue`完成，你可以从[这里 (opens new window)](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactUpdateQueue.new.js#L405)看到`processUpdateQueue`的源码

`state`的变化在`render阶段`产生与上次更新不同的`JSX`对象，通过`Diff算法`产生`effectTag`，在`commit阶段`渲染在页面上。

渲染完成后`workInProgress Fiber树`变为`current Fiber树`，整个更新流程结束。

# 五、深入理解优先级

在[React理念一节](https://react.iamkasong.com/preparation/idea.html#理解-响应自然)我们聊到`React`将人机交互研究的结果整合到真实的`UI`中。具体到`React`运行上这是什么意思呢？

`状态更新`由`用户交互`产生，用户心里对`交互`执行顺序有个预期。`React`根据`人机交互研究的结果`中用户对`交互`的预期顺序为`交互`产生的`状态更新`赋予不同优先级。

具体如下：

- 生命周期方法：同步执行。
- 受控的用户输入：比如输入框内输入文字，同步执行。
- 交互事件：比如动画，高优先级执行。
- 其他：比如数据请求，低优先级执行。

## 1.如何调度优先级

我们在[新的React结构一节](https://react.iamkasong.com/preparation/newConstructure.html)讲到，`React`通过`Scheduler`调度任务。

具体到代码，每当需要调度任务时，`React`会调用`Scheduler`提供的方法`runWithPriority`。

该方法接收一个`优先级`常量与一个`回调函数`作为参数。`回调函数`会以`优先级`高低为顺序排列在一个`定时器`中并在合适的时间触发。

对于更新来讲，传递的`回调函数`一般为[状态更新流程概览一节](https://react.iamkasong.com/state/prepare.html#render阶段的开始)讲到的`render阶段的入口函数`。

> 你可以在[==unstable_runWithPriority== 这里](https://github.com/facebook/react/blob/970fa122d8188bafa600e9b5214833487fbf1092/packages/scheduler/src/Scheduler.js#L217)看到`runWithPriority`方法的定义。在[这里](https://github.com/facebook/react/blob/970fa122d8188bafa600e9b5214833487fbf1092/packages/scheduler/src/SchedulerPriorities.js)看到`Scheduler`对优先级常量的定义。

## 2.例子

优先级最终会反映到`update.lane`变量上。当前我们只需要知道这个变量能够区分`Update`的优先级。

接下来我们通过一个例子结合上一节介绍的`Update`相关字段讲解优先级如何决定更新的顺序。

> 该例子来自[React Core Team Andrew向网友讲解Update工作流程的推文](https://twitter.com/acdlite/status/978412930973687808)

![优先级如何决定更新的顺序](https://react.iamkasong.com/img/update-process.png)

在这个例子中，有两个`Update`。我们将“关闭黑夜模式”产生的`Update`称为`u1`，输入字母“I”产生的`Update`称为`u2`。

其中`u1`先触发并进入`render阶段`。其优先级较低，执行时间较长。此时：

```js
fiber.updateQueue = {
  baseState: {
    blackTheme: true,
    text: 'H'
  },
  firstBaseUpdate: null,
  lastBaseUpdate: null
  shared: {
    pending: u1
  },
  effects: null
};
```

在`u1`完成`render阶段`前用户通过键盘输入字母“I”，产生了`u2`。`u2`属于**受控的用户输入**，优先级高于`u1`，于是中断`u1`产生的`render阶段`。

此时：

```js
fiber.updateQueue.shared.pending === u2 ----> u1
                                     ^        |
                                     |________|
// 即
u2.next === u1;
u1.next === u2;
```

其中`u2`优先级高于`u1`。

接下来进入`u2`产生的`render阶段`。

在`processUpdateQueue`方法中，`shared.pending`环状链表会被剪开并拼接在`baseUpdate`后面。

需要明确一点，`shared.pending`指向最后一个`pending`的`update`，所以实际执行时`update`的顺序为：

```js
u1 -- u2
```

接下来遍历`baseUpdate`，处理优先级合适的`Update`（这一次处理的是更高优的`u2`）。

由于`u2`不是`baseUpdate`中的第一个`update`，在其之前的`u1`由于优先级不够被跳过。

`update`之间可能有依赖关系，所以被跳过的`update`及其后面所有`update`会成为下次更新的`baseUpdate`。（即`u1 -- u2`）。

最终`u2`完成`render - commit阶段`。

此时：

```js
fiber.updateQueue = {
  baseState: {
    blackTheme: true,
    text: 'HI'
  },
  firstBaseUpdate: u1,
  lastBaseUpdate: u2
  shared: {
    pending: null
  },
  effects: null
};
```

在`commit`阶段结尾会再调度一次更新。在该次更新中会基于`baseState`中`firstBaseUpdate`保存的`u1`，开启一次新的`render阶段`。

最终两次`Update`都完成后的结果如下：

```js
fiber.updateQueue = {
  baseState: {
    blackTheme: false,
    text: 'HI'
  },
  firstBaseUpdate: null,
  lastBaseUpdate: null
  shared: {
    pending: null
  },
  effects: null
};
```

我们可以看见，`u2`对应的更新执行了两次，相应的`render阶段`的生命周期勾子`componentWillXXX`也会触发两次。这也是为什么这些勾子会被标记为`unsafe_`。







