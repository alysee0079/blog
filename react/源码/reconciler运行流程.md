# reconciler 运行流程

#### 输入(scheduleUpdateOnFiber)

```javascript
在ReactFiberWorkLoop.js中, 承接输入的函数只有scheduleUpdateOnFiber源码地址.
在react-reconciler对外暴露的 api 函数中, 只要涉及到需要改变 fiber 的操作(无论是首次渲染或后续更新操作),
最后都会间接调用scheduleUpdateOnFiber, 所以scheduleUpdateOnFiber函数是输入链路中的必经之路.
```

```javascript
// 唯一接收输入信号的函数
export function scheduleUpdateOnFiber(fiber: Fiber, lane: Lane, eventTime: number) {
  // ... 省略部分无关代码
  const root = markUpdateLaneFromFiberToRoot(fiber, lane)
  if (lane === SyncLane) {
    // 是否非批处理, 是否未渲染或者提交
    if ((executionContext & LegacyUnbatchedContext) !== NoContext && (executionContext & (RenderContext | CommitContext)) === NoContext) {
      // legacy 或 blocking模式, 直接进行 fiber 构造
      performSyncWorkOnRoot(root)
    } else {
      // 后续的更新
      // 进入第2阶段, 注册调度任务, 经过`Scheduler`包的调度, 间接进行`fiber构造`
      ensureRootIsScheduled(root, eventTime)
    }
  } else {
    // concurrent模式
    // 无论是否初次更新, 都正常进入第2阶段, 注册调度任务, 经过`Scheduler`包的调度, 间接进行`fiber构造`
    ensureRootIsScheduled(root, eventTime)
  }
}
```

逻辑进入到 scheduleUpdateOnFiber 之后, 后面有 2 种可能:

1. 不经过调度, 直接进行 fiber 构造.
2. 注册调度任务, 经过 Scheduler 包的调度, 间接进行 fiber 构造.

#### 注册调度任务(ensureRootIsScheduled)

```javascript
1. 前半部分: 判断是否需要注册新的调度(如果无需新的调度, 会退出函数)
2. 后半部分: 注册调度任务
   - performSyncWorkOnRoot 或 performConcurrentWorkOnRoot 被封装到了任务回调
     scheduleSyncCallback 或 scheduleCallback中, 根据 unstable_scheduleCallback 创建任务

   - 等待调度中心执行任务, 任务运行其实就是执行 performSyncWorkOnRoot 或 performConcurrentWorkOnRoot
```

#### 执行任务回调

任务回调, 实际上就是执行 performSyncWorkOnRoot(legacy) 或 performConcurrentWorkOnRoot

```javascript
// ... 省略部分无关代码
function performSyncWorkOnRoot(root) {
  let lanes
  let exitStatus

  lanes = getNextLanes(root, NoLanes)
  // 1. fiber树构造
  exitStatus = renderRootSync(root, lanes)

  // 2. 异常处理: 有可能fiber构造过程中出现异常
  if (root.tag !== LegacyRoot && exitStatus === RootErrored) {
    // ...
  }

  // 3. 输出: 渲染fiber树
  const finishedWork: Fiber = (root.current.alternate: any)
  root.finishedWork = finishedWork
  root.finishedLanes = lanes
  commitRoot(root)

  // 退出前再次检测, 是否还有其他更新, 是否需要发起新调度
  ensureRootIsScheduled(root, now())
  return null
}
```

```javascript
// ... 省略部分无关代码
function performConcurrentWorkOnRoot(root) {

  const originalCallbackNode = root.callbackNode;

  // 1. 刷新pending状态的effects, 有可能某些effect会取消本次任务
  const didFlushPassiveEffects = flushPassiveEffects();
  if (didFlushPassiveEffects) {
    if (root.callbackNode !== originalCallbackNode) {
      // 任务被取消, 退出调用
      return null;
    } else {
      // Current task was not canceled. Continue.
    }
  }
  // 2. 获取本次渲染的优先级
  let lanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  // 3. 构造fiber树
  let exitStatus = renderRootConcurrent(root, lanes);

  if (
    includesSomeLane(
      workInProgressRootIncludedLanes,
      workInProgressRootUpdatedLanes,
    )
  ) {
    // 如果在render过程中产生了新的update, 且新update的优先级与最初render的优先级有交集
    // 那么最初render无效, 丢弃最初render的结果, 等待下一次调度
    prepareFreshStack(root, NoLanes);
  } else if (exitStatus !== RootIncomplete) {
    // 4. 异常处理: 有可能fiber构造过程中出现异常
    if (exitStatus === RootErrored) {
      // ...
    }.
    const finishedWork: Fiber = (root.current.alternate: any);
    root.finishedWork = finishedWork;
    root.finishedLanes = lanes;
    // 5. 输出: 渲染fiber树
    finishConcurrentRender(root, exitStatus, lanes);
  }

  // 退出前再次检测, 是否还有其他更新, 是否需要发起新调度
  ensureRootIsScheduled(root, now());
  if (root.callbackNode === originalCallbackNode) {
    // 渲染被阻断, 返回一个新的performConcurrentWorkOnRoot函数, 等待下一次调用
    return performConcurrentWorkOnRoot.bind(null, root);
  }
  return null;
}
```

performConcurrentWorkOnRoot 的逻辑与 performSyncWorkOnRoot 的不同之处在于, 对于可中断渲染的支持:

1. 调用 performConcurrentWorkOnRoot 函数时, 首先检查是否处于 render 过程中, 是否需要恢复上一次渲染.
2. 如果本次渲染被中断, 最后返回一个新的 performConcurrentWorkOnRoot 函数, 等待下一次调用.

#### 输出

```javascript
// ... 省略部分无关代码
function commitRootImpl(root, renderPriorityLevel) {
  // 设置局部变量
  const finishedWork = root.finishedWork
  const lanes = root.finishedLanes

  // 清空FiberRoot对象上的属性
  root.finishedWork = null
  root.finishedLanes = NoLanes
  root.callbackNode = null

  // 提交阶段
  let firstEffect = finishedWork.firstEffect
  if (firstEffect !== null) {
    const prevExecutionContext = executionContext
    executionContext |= CommitContext
    // 阶段1: dom突变之前
    nextEffect = firstEffect
    do {
      commitBeforeMutationEffects()
    } while (nextEffect !== null)

    // 阶段2: dom突变, 界面发生改变
    nextEffect = firstEffect
    do {
      commitMutationEffects(root, renderPriorityLevel)
    } while (nextEffect !== null)
    root.current = finishedWork

    // 阶段3: layout阶段, 调用生命周期componentDidUpdate和回调函数等
    nextEffect = firstEffect
    do {
      commitLayoutEffects(root, lanes)
    } while (nextEffect !== null)
    nextEffect = null
    executionContext = prevExecutionContext
  }
  ensureRootIsScheduled(root, now())
  return null
}
```

在输出阶段,commitRoot 的实现逻辑是在 commitRootImpl 函数中, 其主要逻辑是处理副作用队列, 将最新的 fiber 树结构反映到 DOM 上.

核心逻辑分为 3 个步骤:

1. commitBeforeMutationEffects
   dom 变更之前, 主要处理副作用队列中带有 Snapshot,Passive 标记的 fiber 节点.
2. commitMutationEffects
   dom 变更, 界面得到更新. 主要处理副作用队列中带有 Placement, Update, Deletion, Hydrating 标记的 fiber 节点.
3. commitLayoutEffects
   dom 变更后, 主要处理副作用队列中带有 Update | Callback 标记的 fiber 节点.
