# react 启动过程(legacy 模式)

#### 1. 创建全局对象 {#create-global-obj}

无论 Legacy, Concurrent 或 Blocking 模式, react 在初始化时, 都会创建 3 个全局对象:

1. `ReactDOM(Blocking)Root`对象

- 属于 react-dom 包, 该对象暴露有 render, unmount 方法, 通过调用该实例的 render 方法, 可以引导 react 应用的启动. \_internalRoot 指向 fiberRoot 对象.

2. `fiberRoot` 对象

- 属于 react-reconciler 包, 作为 react-reconciler 在运行过程中的全局上下文, 保存 fiber 构建过程中所依赖的全局状态.
- 其大部分实例变量用来存储 fiber 构造循环过程的各种状态.react 应用内部, 可以根据这些实例变量的值, 控制执行逻辑.

3. `HostRootFiber` 对象

- 属于 react-reconciler 包, 这是 react 应用中的第一个 Fiber 对象, 是 Fiber 树的根节点, 节点的类型是 HostRoot.

#### 创建流程

`(--> 代表内部执行)`

1. ```javascript
   ReactDOM.render()
   ```
2. ```javascript
   legacyRenderSubtreeIntoContainer()
   // 创建 ReactDOMRoot, fiberRoot 和 rootFiber, 初始化react应用环境
   1) legacyCreateRootFromDOMContainer()
      --> legacyCreateRootFromDOMContainer()
   // 更新容器(仅在 legacy 模式下, 其他模式直接调用 updateContainer)
   2) unbatchedUpdates(() => {updateContainer()})
   ```

3. ```javascript
   legacyCreateRootFromDOMContainer()
   --> createLegacyRoot()
   ```
4. ```javascript
   createLegacyRoot()
   --> new ReactDOMBlockingRoot() 创建 ReactDomRoot 对象, 在其原型上有 render,
       unmount 等方法. ReactDomRoot._internalRoot 就是 FiberRoot, 会在 createHostRootFiber 创建.
   --> createRootImpl()
       1) createContainer 创建 fiberRoot 和 rootFiber
       2) markContainerAsRoot 标记 dom 对象, 把 dom 和 fiber 对象关联起来
   --> createContainer()
   ```

5. ```javascript
   createContainer()
   --> createFiberRoot()
   在 createFiberRoot 中:
   1) 通过 new FiberRootNode() 创建 FiberRoot 对象, 保存 fiber 构建过程中所依赖的全局状态.
   2) 通过 createHostRootFiber(tag: RootTag) 创建 RootFiber, 是 react 应用中第一个 fiber 对象,
      称为 HostRootFiber(fiber.tag = HostRoot = 3).

      注意:fiber树中所有节点的mode都会和HostRootFiber.mode一致(新建的 fiber 节点, 其 mode 来源于父节点),
      所以HostRootFiber.mode非常重要, 它决定了以后整个 fiber 树构建过程.

   3) 关联 FiberRoot 和 RootFiber, FiberRoot.current = RootFiber, RootFiber.stateNode = FiberRoot.
   4) initializeUpdateQueue(), 初始化HostRootFiber的updateQueue
   ```

```javascript
通过以上分析,legacy 模式下调用 ReactDOM.render 有 2 个核心步骤:

创建 ReactDOMBlockingRoot 实例(在 Concurrent 模式和 Blocking 模式中详细分析该类),
初始化 react 应用环境. 调用 updateContainer 进行更新.
```

##### Concurrent 模式和 Blocking 模式

1. 分别调用 `ReactDOM.createRoot` 和 `ReactDOM.createBlockingRoot` 创建 `ReactDOMRoot` 和 `ReactDOMBlockingRoot` 实例
2. 调用 `ReactDOMRoot` 和 `ReactDOMBlockingRoot` 实例的 `render` 方法

ReactDOMRoot 和 ReactDOMBlockingRoot 有相同的特性

1. 调用 `createRootImpl` 创建 `fiberRoo` t 对象, 并将其挂载到 `this._internalRoot` 上.
2. 原型上有 `render` 和 `unmount` 方法, 且内部都会调用 `updateContainer` 进行更新.

#### 2.调用更新入口

1. legacy 回到 legacyRenderSubtreeIntoContainer 函数中有:

```javascript
unbatchedUpdates(() => {
  updateContainer(children, fiberRoot, parentComponent, callback)
})
```

2. concurrent 和 blocking 在 ReactDOM(Blocking)Root 原型上有 render 方法

```javascript
ReactDOMRoot.prototype.render = ReactDOMBlockingRoot.prototype.render = function (children: ReactNodeList): void {
  const root = this._internalRoot
  // 执行更新
  updateContainer(children, root, null, null)
}
```

###### 相同点:

```javascript
3 种模式在调用更新时都会执行 updateContainer.
updateContainer 函数串联了 react-dom 与 react-reconciler,
之后的逻辑进入了 react-reconciler 包.
```

###### 不同点:

```javascript
legacy 下的更新会先调用 unbatchedUpdates,
更改执行上下文为 LegacyUnbatchedContext,
之后调用 updateContainer 进行更新.
```

```javascript
concurrent 和 blocking 不会更改执行上下文, 直接调用 updateContainer 进行更新.
```

###### 分析 updateContainer

```javascript
export function updateContainer(element: ReactNodeList, container: OpaqueRoot, parentComponent: ?React$Component<any, any>, callback: ?Function): Lane {
  const current = container.current
  // 1. 获取当前时间戳, 计算本次更新的优先级
  const eventTime = requestEventTime()
  const lane = requestUpdateLane(current)

  // 2. 设置fiber.updateQueue
  const update = createUpdate(eventTime, lane)
  update.payload = { element }
  callback = callback === undefined ? null : callback
  if (callback !== null) {
    update.callback = callback
  }
  enqueueUpdate(current, update)

  // 3. 进入reconciler运作流程中的`输入`环节
  scheduleUpdateOnFiber(current, lane, eventTime)
  return lane
}
```

updateContainer 函数位于 react-reconciler 包中, 它串联了 react-dom 与 react-reconciler. 此处暂时不深入分析 updateContainer 函数的具体功能, 需要关注其最后调用了 scheduleUpdateOnFiber.

在前文 reconciler 运作流程中, 重点分析过 scheduleUpdateOnFiber 是`输入`阶段的入口函数.

所以到此为止, 通过调用 react-dom 包的 api(如: ReactDOM.render), react 内部经过一系列运转, 完成了初始化, 并且进入了 reconciler 运作流程的第一个阶段.
