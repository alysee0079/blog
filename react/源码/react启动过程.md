# react 启动过程(legacy 模式)

#### 创建全局对象 {#create-global-obj}

无论 Legacy, Concurrent 或 Blocking 模式, react 在初始化时, 都会创建 3 个全局对象:

1. `ReactDOM(Blocking)Root`对象

- 属于 react-dom 包, 该对象暴露有 render,unmount 方法, 通过调用该实例的 render 方法, 可以引导 react 应用的启动.

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
   // 创建ReactDOMRoot对象, 初始化react应用环境
   1) legacyCreateRootFromDOMContainer()
      --> legacyCreateRootFromDOMContainer()
   // 更新容器
   2) unbatchedUpdates(() => {updateContainer()})
   ```

3. ```javascript
   legacyCreateRootFromDOMContainer()
   --> createLegacyRoot()
   ```
4. ```javascript
   createLegacyRoot()
   --> new ReactDOMBlockingRoot()
   --> createRootImpl()
   --> createContainer
   创建 ReactDomRoot 对象, 在其原型上有 render, unmount 等方法.
   ```
5. ```javascript
   createContainer()
   --> createFiberRoot()
   在 createFiberRoot 中:
   1) 通过 new FiberRootNode() 创建 FiberRoot 对象, 保存 fiber 构建过程中所依赖的全局状态.
   2) 通过 createHostRootFiber() 创建 RootFiber, 是 react 应用中第一个 fiber 对象.
   3) 关联 FiberRoot 和 RootFiber, FiberRoot.current = RootFiber, RootFiber.stateNode = FiberRoot.
   4) initializeUpdateQueue()
   ```

```javascript
通过以上分析,legacy 模式下调用 ReactDOM.render 有 2 个核心步骤:

创建 ReactDOMBlockingRoot 实例(在 Concurrent 模式和 Blocking 模式中详细分析该类), 初始化 react 应用环境.
调用 updateContainer 进行更新.
```

#### Concurrent 模式和 Blocking 模式
