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
// 注册调度任务, 经过`Scheduler`包的调度, 间接进行`fiber构造`
function ensureRootIsScheduled(root, currentTime) {
  // 前半部分: 判断是否需要注册新的调度
  var existingCallbackNode = root.callbackNode; // Check if any lanes are being starved by other work. If so, mark them as
  // expired so we know to work on those next.
  var nextLanes = getNextLanes(root, root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes); // This returns the priority level computed during the `getNextLanes` call.

  // 获取本次更新的优先级
  var newCallbackPriority = returnNextLanesPriority();

  if (nextLanes === NoLanes) {
    // Special case: There's nothing to work on.
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
      root.callbackNode = null;
      root.callbackPriority = NoLanePriority;
    }

    return;
  } // Check if there's an existing task. We may be able to reuse it.

  // 节流/防抖
  if (existingCallbackNode !== null) {
    var existingCallbackPriority = root.callbackPriority;
    console.log('existingCallbackPriority', existingCallbackPriority)
    console.log('newCallbackPriority', newCallbackPriority)

    // 节流: 新旧更新的优先级相同, 如连续多次执行setState), 则无需注册新task(继续沿用上一个优先级相同的task), 直接退出调用.
    if (existingCallbackPriority === newCallbackPriority) {
      // The priority hasn't changed. We can reuse the existing task. Exit.
      return;
    } // The priority changed. Cancel the existing callback. We'll schedule a new
    // one below.

    // 防抖: 新旧更新的优先级不同), 则取消旧task, 重新注册新task.
    cancelCallback(existingCallbackNode);
  } // Schedule a new callback.

  // 后半部分: 注册调度任务
  var newCallbackNode;
  if (newCallbackPriority === SyncLanePriority) {
    // Special case: Sync React callbacks are scheduled on a special
    // internal queue
    // 注册任务(performSyncWorkOnRoot)(同步)
    newCallbackNode = scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
  } else if (newCallbackPriority === SyncBatchedLanePriority) {
    // 注册任务(performSyncWorkOnRoot)(异步)
    newCallbackNode = scheduleCallback(ImmediatePriority$1, performSyncWorkOnRoot.bind(null, root));
  } else {
    var schedulerPriorityLevel = lanePriorityToSchedulerPriority(newCallbackPriority);
    // 注册任务(performConcurrentWorkOnRoot)(异步)
    newCallbackNode = scheduleCallback(schedulerPriorityLevel, performConcurrentWorkOnRoot.bind(null, root));
  }

  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```
1. 前半部分: 判断是否需要注册新的调度(如果无需新的调度, 会退出函数)
2. 后半部分: 注册调度任务
  - performSyncWorkOnRoot 或 performConcurrentWorkOnRoot 被封装到了任务回调
    scheduleSyncCallback 或 scheduleCallback中, 根据 unstable_scheduleCallback 创建任务
  - 等待调度中心执行任务, 任务运行其实就是执行 performSyncWorkOnRoot 或 performConcurrentWorkOnRoot
#### 注册调度任务(scheduleCallback)
```javascript
// 用于以某个优先级异步注册调度任务
function scheduleCallback(reactPriorityLevel, callback, options) {
  var priorityLevel = reactPriorityToSchedulerPriority(reactPriorityLevel);
  // unstable_scheduleCallback === unstable_scheduleCallback (SchedulerPostTask.js)
  return Scheduler_scheduleCallback(priorityLevel, callback, options);
}
```
#### 创建调度任务(unstable_scheduleCallback)
```javascript
// 创建调度任务(区分是否过期)
// callback: performSyncWorkOnRoot/performConcurrentWorkOnRoot
function unstable_scheduleCallback(priorityLevel, callback, options) {
  // 当前时间
  var currentTime = exports.unstable_now();
  // 任务开始时间
  var startTime;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      // 开始时间 = 当前时间 + 延迟时间
      startTime = currentTime + delay;
    } else {
      // 开始时间 = 当前时间
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }
  var timeout;
  // 任务优先级, 优先级越高, timeout 越小(立即执行为负数)
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }
  // 任务过期时间 = 开始时间 + 超时时间
  var expirationTime = startTime + timeout;
  // 创建新任务
  var newTask = {
    id: taskIdCounter++,
    callback: callback, // performSyncWorkOnRoot/performConcurrentWorkOnRoot
    priorityLevel: priorityLevel,
    startTime: startTime,
    expirationTime: expirationTime,
    sortIndex: -1
  };
  if (startTime > currentTime) {
    // This is a delayed task.
    newTask.sortIndex = startTime;
    // 如果任务开始时间大于当前时间, 说明该任务还未过期(延时), 将其放到未就绪列表
    push(timerQueue, newTask);
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // 没有有过期的任务, 同时当前任务是离过期时间最接近的任务
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      } // Schedule a timeout.
      // 定时检测未过期任务是否过期, 是否需要加入到过期任务队列
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    newTask.sortIndex = expirationTime;
    // 任务已经过期, 将其放到过期任务列表
    push(taskQueue, newTask);
    // wait until the next time we yield.
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      // 执行过期任务队列
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```
#### 执行过期任务队列(requestHostCallback)
```javascript
// 执行过期任务
var performWorkUntilDeadline = function () {
  if (scheduledHostCallback !== null) {
    var currentTime = exports.unstable_now(); // Yield after `yieldInterval` ms, regardless of where we are in the vsync
    // cycle. This means there's always time remaining at the beginning of
    // the message event.

    // 过期时间 = 当前时间 + 时间切片周期(默认是5ms)
    deadline = currentTime + yieldInterval;
    var hasTimeRemaining = true;

    try {
      // scheduledHostCallback: flushWork
      var hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);

      if (!hasMoreWork) {
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      } else {
        // If there's more work, schedule the next message event at the end
        // of the preceding one.
        port.postMessage(null);
      }
    } catch (error) {
      // If a scheduler task throws, exit the current browser task so the
      // error can be observed.
      port.postMessage(null);
      throw error;
    }
  } else {
    isMessageLoopRunning = false;
  } // Yielding to the browser will give it a chance to paint, so we can
};

var channel = new MessageChannel();
var port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;

// 请求及时回调(处理过期任务)
// callback: flushWork
requestHostCallback = function (callback) {
  // 1. 保存callback, 会在 performWorkUntilDeadline 中执行
  scheduledHostCallback = callback;

  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    // 2. 通过 MessageChannel 触发任务回调(宏任务)
    port.postMessage(null);
  }
}
```
#### 执行任务回调(performWorkUntilDeadline => scheduledHostCallback => flushWork => workLoop)
workLoop 取出已过期的任务 currentTask: performSyncWorkOnRoot(legacy) 或 performConcurrentWorkOnRoot

```javascript
// ... 省略部分无关代码
function performSyncWorkOnRoot(root) {
  let lanes
  let exitStatus

  lanes = getNextLanes(root, NoLanes)
  // 1. fiber树构造(经过beginWork, completeWork)
  // 这个过程单独在 fiber 流程梳理
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

#### 输出阶段(commitRootImpl)

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
