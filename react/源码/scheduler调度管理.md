# scheduler 调度管理

#### 任务队列管理

在 Scheduler.js 中, 维护了一个 taskQueue, 任务队列管理就是围绕这个 taskQueue 展开.

```javascript
// Tasks are stored on a min heap
var taskQueue = []
var timerQueue = []
```

注意:

- taskQueue 是一个小顶堆数组, 关于堆排序的详细解释, 可以查看 React 算法之堆排序.
- 源码中除了 taskQueue 队列之外还有一个 timerQueue 队列. 这个队列是预留给延时任务使用的, 在 react@17.0.2 版本里面, 从源码中的引用来看, 算一个保留功能, 没有用到.

#### 创建任务

```javascript
// 省略部分无关代码
function unstable_scheduleCallback(priorityLevel, callback, options) {
  // 1. 获取当前时间
  var currentTime = getCurrentTime()
  var startTime
  if (typeof options === 'object' && options !== null) {
    // 从函数调用关系来看, 在v17.0.2中,所有调用 unstable_scheduleCallback 都未传入options
    // 所以省略延时任务相关的代码
  } else {
    startTime = currentTime
  }
  // 2. 根据传入的优先级, 设置任务的过期时间 expirationTime
  var timeout
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT
      break
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT
      break
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT
      break
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT
      break
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT
      break
  }
  var expirationTime = startTime + timeout
  // 3. 创建新任务
  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1
  }
  if (startTime > currentTime) {
    // 省略无关代码 v17.0.2中不会使用
  } else {
    newTask.sortIndex = expirationTime
    // 4. 加入任务队列
    push(taskQueue, newTask)
    // 5. 请求调度
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true
      requestHostCallback(flushWork)
    }
  }
  return newTask
}
```

逻辑很清晰(在注释中已标明), 重点分析 task 对象的各个属性:

```javascript
var newTask = {
  id: taskIdCounter++, // id: 一个自增编号
  callback, // callback: 传入的回调函数
  priorityLevel, // priorityLevel: 优先级等级
  startTime, // startTime: 创建task时的当前时间
  expirationTime, // expirationTime: task的过期时间, 优先级越高 expirationTime = startTime + timeout 越小
  sortIndex: -1
}
newTask.sortIndex = expirationTime // sortIndex: 排序索引, 全等于过期时间. 保证过期时间越小, 越紧急的任务排在最前面
```

#### 消费任务

创建任务之后, 最后请求调度 requestHostCallback(flushWork)(创建任务源码中的第 5 步), flushWork 函数作为参数被传入调度中心内核等待回调. requestHostCallback 函数在上文调度内核中已经介绍过了, 在调度中心中, 只需下一个事件循环就会执行回调, 最终执行 flushWork.

```javascript
function flushWork(hasTimeRemaining, initialTime) {
  // 1. 做好全局标记, 表示现在已经进入调度阶段
  isHostCallbackScheduled = false
  isPerformingWork = true
  const previousPriorityLevel = currentPriorityLevel
  try {
    // 2. 循环消费队列
    return workLoop(hasTimeRemaining, initialTime)
  } finally {
    // 3. 还原全局标记
    currentTask = null
    currentPriorityLevel = previousPriorityLevel
    isPerformingWork = false
  }
}
```

flushWork 中调用了 workLoop. 队列消费的主要逻辑是在 workLoop 函数中, 这就是 React 工作循环一文中提到的任务调度循环.

```javascript
// 省略部分无关代码
function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime // 保存当前时间, 用于判断任务是否过期
  currentTask = peek(taskQueue) // 获取队列中的第一个任务
  while (currentTask !== null) {
    if (currentTask.expirationTime > currentTime && (!hasTimeRemaining || shouldYieldToHost())) {
      // 虽然currentTask没有过期, 但是执行时间超过了限制(毕竟只有5ms, shouldYieldToHost()返回true). 停止继续执行, 让出主线程
      break
    }
    const callback = currentTask.callback
    if (typeof callback === 'function') {
      currentTask.callback = null
      currentPriorityLevel = currentTask.priorityLevel
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime
      // 执行回调
      const continuationCallback = callback(didUserCallbackTimeout)
      currentTime = getCurrentTime()
      // 回调完成, 判断是否还有连续(派生)回调
      if (typeof continuationCallback === 'function') {
        // 产生了连续回调(如fiber树太大, 出现了中断渲染), 保留currentTask
        currentTask.callback = continuationCallback
      } else {
        // 把currentTask移出队列
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue)
        }
      }
    } else {
      // 如果任务被取消(这时currentTask.callback = null), 将其移出队列
      pop(taskQueue)
    }
    // 更新currentTask
    currentTask = peek(taskQueue)
  }
  if (currentTask !== null) {
    return true // 如果task队列没有清空, 返回true. 等待调度中心下一次回调
  } else {
    return false // task队列已经清空, 返回false.
  }
}
```

workLoop 就是一个大循环, 虽然代码也不多, 但是非常精髓, 在此处实现了时间切片(time slicing)和 fiber 树的可中断渲染. 这 2 大特性的实现, 都集中于这个 while 循环.

每一次 while 循环的退出就是一个时间切片, 深入分析 while 循环的退出条件:

1. 队列被完全清空: 这种情况就是很正常的情况, 一气呵成, 没有遇到任何阻碍.
2. 执行超时: 在消费 taskQueue 时, 在执行 task.callback 之前, 都会检测是否超时, 所以超时检测是以 task 为单位.
   - 如果某个 task.callback 执行时间太长(如: fiber 树很大, 或逻辑很重)也会造成超时
   * 所以在执行 task.callback 过程中, 也需要一种机制检测是否超时, 如果超时了就立刻暂停 task.callback 的执行.

#### 时间切片原理

消费任务队列的过程中, 可以消费 1~n 个 task, 甚至清空整个 queue. 但是在每一次具体执行 task.callback 之前都要进行超时检测, 如果超时可以立即退出循环并等待下一次调用.

#### 可中断渲染原理

在时间切片的基础之上, 如果单个 task.callback 执行时间就很长(假设 200ms). 就需要 task.callback 自己能够检测是否超时, 所以在 fiber 树构造过程中, 每构造完成一个单元, 都会检测一次超时(源码链接), 如遇超时就退出 fiber 树构造循环, 并返回一个新的回调函数(就是此处的 continuationCallback)并等待下一次回调继续未完成的 fiber 树构造.
