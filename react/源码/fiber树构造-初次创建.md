# fiber 树构造-初次创建

> 在 React 应用首次启动时, 界面还没有渲染, 此时并不会进入对比过程, 相当于直接构造一棵全新的树.

#### ReactElement, Fiber, DOM 三者的关系

1.ReactElement 对象(type 定义在 shared 包中)

    - 所有采用 jsx 语法书写的节点, 都会被编译器转换, 最终会以 React.createElement(...)的方式, 创建出来一个与之对应的 ReactElement 对象

2.fiber 对象(type 类型的定义在 ReactInternalTypes.js 中)

    - fiber 对象是通过 ReactElement 对象进行创建的, 多个 fiber 对象构成了一棵 fiber 树, fiber 树是构造 DOM 树的数据模型, fiber 树的任何改动, 最后都体现到 DOM 树.

3.DOM 对象: 文档对象模型

    - DOM 将文档解析为一个由节点和对象（包含属性和方法的对象）组成的结构集合, 也就是常说的 DOM 树.

    - JavaScript 可以访问和操作存储在 DOM 中的内容, 也就是操作 DOM 对象, 进而触发 UI 渲染.

注意:

    - 开发人员能够控制的是 JSX, 也就是 ReactElement 对象.
    - fiber树是通过ReactElement生成的, 如果脱离了ReactElement,fiber树也无从谈起. 所以是ReactElement树(不是严格的树结构, 为了方便也称为树)驱动fiber树.
    - fiber树是DOM树的数据模型, fiber树驱动DOM树

开发人员通过编程只能控制 ReactElement 树的结构, ReactElement 树驱动 fiber 树, fiber 树再驱动 DOM 树, 最后展现到页面上. 所以 fiber 树的构造过程, 实际上就是 ReactElement 对象到 fiber 对象的转换过程.

#### fiber 构造入口

```javascript
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  const prevExecutionContext = executionContext
  executionContext |= RenderContext
  // 如果fiberRoot变动, 或者update.lane变动, 都会刷新栈帧, 丢弃上一次渲染进度
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    // 刷新栈帧, legacy模式下都会进入
    prepareFreshStack(root, lanes)
  }
  do {
    try {
      // 循环构造
      workLoopSync()
      break
    } catch (thrownValue) {
      handleError(root, thrownValue)
    }
  } while (true)
  executionContext = prevExecutionContext
  // 重置全局变量, 表明render结束
  workInProgressRoot = null
  workInProgressRootRenderLanes = NoLanes
  return workInProgressRootExitStatus
}
```

在 renderRootSync 中, 在执行 fiber 树构造前(workLoopSync)会先刷新栈帧 prepareFreshStack(参考 fiber 树构造(基础准备)).在这里创建了 HostRootFiber.alternate, 重置全局变量 workInProgress 和 workInProgressRoot 等.

#### 循环构造

```javascript
// legacy/blocking
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress)
  }
}

// concurrent
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress)
  }
}
```

可以看到 workLoopConcurrent 相比于 Sync, 会多一个停顿机制, 这个机制实现了时间切片和可中断渲染(参考 React 调度原理)

```javascript
//
function performUnitOfWork(unitOfWork: Fiber): void {
  // unitOfWork就是被传入的workInProgress
  const current = unitOfWork.alternate
  let next
  // 探寻阶段
  next = beginWork(current, unitOfWork, subtreeRenderLanes)
  unitOfWork.memoizedProps = unitOfWork.pendingProps
  if (next === null) {
    // 如果没有派生出新的节点, 则进入completeWork阶段, 传入的是当前unitOfWork
    // 回溯阶段
    completeUnitOfWork(unitOfWork)
  } else {
    workInProgress = next
  }
}
```

可以明显的看出, 整个 fiber 树构造是一个深度优先遍历(可参考 React 算法之深度优先遍历), 其中有 2 个重要的变量 workInProgress 和 current(可参考前文 fiber 树构造(基础准备)中介绍的双缓冲技术):

    - workInProgress和current都视为指针
    - workInProgress指向当前正在构造的fiber节点
    - current = workInProgress.alternate(即fiber.alternate), 指向当前页面正在使用的fiber节点. 初次构造时, 页面还未渲染, 此时current = null.

在深度优先遍历中, 每个 fiber 节点都会经历 2 个阶段:

1.探寻阶段 beginWork

2.回溯阶段 completeWork

这 2 个阶段共同完成了每一个 fiber 节点的创建, 所有 fiber 节点则构成了 fiber 树.

#### 探寻阶段 beginWork

beginWork(current, unitOfWork, subtreeRenderLanes)(源码地址)针对所有的 Fiber 类型, 其中的每一个 case 处理一种 Fiber 类型. updateXXX 函数(如: updateHostRoot, updateClassComponent 等)的主要逻辑:

1.根据 ReactElement 对象创建所有的 fiber 节点, 最终构造出 fiber 树形结构(设置 return 和 sibling 指针)

2.设置 fiber.flags(二进制形式变量, 用来标记 fiber 节点 的增,删,改状态, 等待 completeWork 阶段处理)

3.设置 fiber.stateNode 局部状态(如 Class 类型节点: fiber.stateNode=new Class())

updateXXX 函数(如: updateHostRoot, updateClassComponent 等)虽然 case 较多, 但是主要逻辑可以概括为 3 个步骤:

1.根据 fiber.pendingProps, fiber.updateQueue 等输入数据状态, 计算 fiber.memoizedState 作为输出状态

2.获取下级 ReactElement 对象

    1.class 类型的 fiber 节点

    - 构建 React.Component 实例
    - 把新实例挂载到 fiber.stateNode 上
    - 执行 render 之前的生命周期函数
    - 执行 render 方法, 获取下级 reactElement
    - 根据实际情况, 设置 fiber.flags

    2.function 类型的 fiber 节点

    - 执行 function, 获取下级 reactElement
    - 根据实际情况, 设置fiber.flags

    3.HostComponent 类型(如: div, span, button 等)的 fiber 节点

    - pendingProps.children作为下级reactElement
    - 如果下级节点是文本节点,则设置下级节点为 null. 准备进入completeUnitOfWork阶段
    - 根据实际情况, 设置fiber.flags

3.根据 ReactElement 对象, 调用 reconcileChildren 生成 Fiber 子节点(只生成次级子节点)

- 根据实际情况, 设置 fiber.flags

不同的 updateXXX 函数处理的 fiber 节点类型不同, 总的目的是为了向下生成子节点. 在这个过程中把一些需要持久化的数据挂载到 fiber 节点上(如 fiber.stateNode,fiber.memoizedState 等); 把 fiber 节点的特殊操作设置到 fiber.flags(如:节点 ref,class 组件的生命周期,function 组件的 hook,节点删除等).

#### 回溯阶段 completeWork

completeUnitOfWork(unitOfWork)(源码地址), 处理 beginWork 阶段已经创建出来的 fiber 节点, 核心逻辑:

1.调用 completeWork

    - 给 fiber 节点(tag=HostComponent, HostText)创建 DOM 实例, 设置 fiber.stateNode 局部状态(如 tag=HostComponent, HostText 节点: fiber.stateNode 指向这个 DOM 实例).
    - 为 DOM 节点设置属性, 绑定事件(这里先说明有这个步骤, 详细的事件处理流程, 在合成事件原理中详细说明).
    - 设置 fiber.flags 标记

2.把当前 fiber 对象的副作用队列(firstEffect 和 lastEffect)添加到父节点的副作用队列之后, 更新父节点的 firstEffect 和 lastEffect 指针.

3.识别 beginWork 阶段设置的 fiber.flags, 判断当前 fiber 是否有副作用(增,删,改), 如果有, 需要将当前 fiber 加入到父节点的 effects 队列, 等待 commit 阶段处理.
