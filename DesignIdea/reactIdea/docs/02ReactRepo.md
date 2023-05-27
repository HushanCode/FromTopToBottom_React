# React仓库

# 一、顶层目录

经过之前的学习，我们已经知道`React16`的架构分为三层：

- Scheduler（调度器）—— 调度任务的优先级，高优任务优先进入**Reconciler**
- Reconciler（协调器）—— 负责找出变化的组件
- Renderer（渲染器）—— 负责将变化的组件渲染到页面上

除去配置文件和隐藏文件夹，根目录的文件夹包括三个：

```
根目录
├── fixtures        # 包含一些给贡献者准备的小型 React 测试项目
├── packages        # 包含元数据（比如 package.json）和 React 仓库中所有 package 的源码（子目录 src）
├── scripts         # 各种工具链的脚本，比如git、jest、eslint等
```

# 二、packages目录

目录下的文件夹非常多，我们来看下：

## 1.[react](https://github.com/facebook/react/tree/master/packages/react)文件夹

React的核心，包含所有全局 React API，如：

- React.createElement
- React.Component
- React.Children

这些 API 是全平台通用的，它不包含`ReactDOM`、`ReactNative`等平台特定的代码。在 NPM 上作为[单独的一个包](https://www.npmjs.com/package/react)发布。

## 2.[scheduler](https://github.com/facebook/react/tree/master/packages/scheduler)文件夹

Scheduler（调度器）的实现。

## 3.[shared](https://github.com/facebook/react/tree/master/packages/shared)文件夹

源码中其他模块公用的**方法**和**全局变量**，比如在[shared/ReactSymbols.js](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/shared/ReactSymbols.js)中保存`React`不同组件类型的定义。

```js
// ...
export let REACT_ELEMENT_TYPE = 0xeac7;
export let REACT_PORTAL_TYPE = 0xeaca;
export let REACT_FRAGMENT_TYPE = 0xeacb;
// ...
```

## 4.Renderer相关的文件夹

如下几个文件夹为对应的**Renderer**

```react
- react-art
- react-dom                 # 注意这同时是DOM和SSR（服务端渲染）的入口
- react-native-renderer
- react-noop-renderer       # 用于debug fiber（后面会介绍fiber）
- react-test-renderer
```

## 5.试验性包的文件夹

`React`将自己流程中的一部分抽离出来，形成可以独立使用的包，由于他们是试验性质的，所以不被建议在生产环境使用。包括如下文件夹：

```
- react-server        # 创建自定义SSR流
- react-client        # 创建自定义的流
- react-fetch         # 用于数据请求
- react-interactions  # 用于测试交互相关的内部特性，比如React的事件模型
- react-reconciler    # Reconciler的实现，你可以用他构建自己的Renderer
```

## 6.辅助包的文件夹

`React`将一些辅助功能形成单独的包。包括如下文件夹：

```
- react-is       # 用于测试组件是否是某类型
- react-client   # 创建自定义的流
- react-fetch    # 用于数据请求
- react-refresh  # “热重载”的React官方实现
```

## 7.[ react-reconciler](https://github.com/facebook/react/tree/master/packages/react-reconciler)文件夹

我们需要重点关注**react-reconciler**，在接下来源码学习中 80%的代码量都来自这个包。

虽然他是一个实验性的包，内部的很多功能在正式版本中还未开放。但是他一边对接**Scheduler**，一边对接不同平台的**Renderer**，构成了整个 React16 的架构体系。

# 三、调试源码

## 1.基本

了解了源码的文件目录，这一节我们看看如何调试源码。

即使版本号相同（当前最新版为`17.0.0 RC`），但是`facebook/react`项目`master`分支的代码和我们使用`create-react-app`创建的项目`node_modules`下的`react`项目代码还是有些区别。

因为`React`的新代码都是直接提交到`master`分支，而`create-react-app`内的`react`使用的是稳定版的包。

为了始终使用最新版`React`教学，我们调试源码遵循以下步骤：

1. 从`facebook/react`项目`master`分支拉取最新源码
2. 基于最新源码构建`react`、`scheduler`、`react-dom`三个包
3. 通过`create-react-app`创建测试项目，并使用步骤2创建的包作为项目依赖的包

## 2.拉取源码

拉取`facebook/react`代码

```
# 拉取代码
git clone https://github.com/facebook/react.git

# 如果拉取速度很慢，可以考虑如下2个方案：

# 1. 使用cnpm代理
git clone https://github.com.cnpmjs.org/facebook/react

# 2. 使用码云的镜像（一天会与react同步一次）
git clone https://gitee.com/mirrors/react.git
```

安装依赖

```
# 切入到react源码所在文件夹
cd react

# 安装依赖
yarn
```

打包`react`、`scheduler`、`react-dom`三个包为dev环境可以使用的`cjs`包。

我们的步骤只包含具体做法，对每一步更详细的介绍可以参考`React`文档[源码贡献章节](https://zh-hans.reactjs.org/docs/how-to-contribute.html#development-workflow)

```sh
# 执行打包命令
yarn build react/index,react/jsx,react-dom/index,scheduler --type=NODE
```

现在源码目录`build/node_modules`下会生成最新代码的包。我们为`react`、`react-dom`创建`yarn link`。

> 通过`yarn link`可以改变项目中依赖包的目录指向

```sh
cd build/node_modules/react
# 申明react指向
yarn link
cd build/node_modules/react-dom
# 申明react-dom指向
yarn link
```

## 3.创建项目

接下来我们通过`create-react-app`在其他地方创建新项目。这里我们随意起名，比如“a-react-demo”。

```sh
npx create-react-app a-react-demo
```

在新项目中，将`react`与`react-dom`2个包指向`facebook/react`下我们刚才生成的包。

```sh
# 将项目内的react react-dom指向之前申明的包
yarn link react react-dom
```

现在试试在`react/build/node_modules/react-dom/cjs/react-dom.development.js`中随意打印些东西。

在`a-react-demo`项目下执行`yarn start`。现在浏览器控制台已经可以打印出我们输入的东西了。

通过以上方法，我们的运行时代码就和`React`最新代码一致了。



