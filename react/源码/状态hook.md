# 状态 Hook

1.function 类型的 fiber 节点, 它的处理函数是 updateFunctionComponent, 其中再通过 renderWithHooks 调用 function.

2.在 function 中, 通过 Hook Api(如: useState, useEffect)创建 Hook 对象.

    - 状态 Hook 实现了状态持久化(等同于 class 组件维护 fiber.memoizedState).
    - 副作用 Hook 则实现了维护 fiber.flags,并提供副作用回调(类似于 class 组件的生命周期回调)

3.多个 Hook 对象构成一个链表结构, 并挂载到 fiber.memoizedState 之上.

4.fiber 树更新阶段, 把 current.memoizedState 链表上的所有 Hook 按照顺序克隆到 workInProgress.memoizedState 上, 实现数据的持久化.

#### kook 数据结构(每个 useState 对应一个 hook 对象)

Hook 与 update 类似，都通过链表连接。不过 Hook 是无环的单向链表。

```javascript
// hook 保存在 FunctionComponent fiber.memoizedState
// App 函数对应 fiber
const fiber = {
  // 保存该 FunctionComponent 对应的 Hooks 链表
  memoizedState: null,
  // 指向 App 函数
  stateNode: App
}

// hook 对象
const hook: Hook = {
  // 保存 hook 对应的 state
  memoizedState: null,
  // 本次更新前该 Fiber 节点的 state，Update 基于该 state 计算更新后的 state
  baseState: null,
  // 上次更新遗留下的 update(优先级不够)
  baseQueue: null,
  // 保存 update 的 queue
  queue: null,
  // 与下一个Hook连接形成单向无环链表
  next: null
}

/*
  memoizedState
  注意: hook 与 FunctionComponent fiber都存在 memoizedState 属性，不要混淆他们的概念.

  不同类型 hook 的 memoizedState 保存不同类型数据，具体如下：
  1.useState：对于const [state, updateState] = useState(initialState)，memoizedState保存state的值.
  2.useReducer：对于const [state, dispatch] = useReducer(reducer, {});，memoizedState保存state的值.
  3.useEffect：memoizedState保存包含useEffect回调函数、依赖项等的链表数据结构effect.effect链表同时会保存在fiber.updateQueue中.
  4.useRef：对于useRef(1)，memoizedState保存{current: 1}
  5.useMemo：对于useMemo(callback, [depA])，memoizedState保存[callback(), depA]
  6.useCallback：对于useCallback(callback, [depA])，memoizedState保存[callback, depA]。与useMemo的区别是，useCallback保存的是callback函数本身，而useMemo保存的是callback函数的执行结果.

  有些hook是没有memoizedState的，比如: useContext
*/

```

> 注意:
> 注意区分 update 与 hook 的所属关系：
> 每个 useState 对应一个 hook 对象。
> 调用 const [num, updateNum] = useState(0);时 updateNum（即上文介绍的 dispatchAction）产生的 update 保存在 useState 对应的 hook.queue 中。


#### update 数据结构(每一个 dispatchAction 都会产生一个 update)

```javascript
// 环状单向链表
const update = {
  lane: lane, // 更新优先级
  action: action, // 更新内容
  eagerReducer: null,
  eagerState: null,
  next: null // 下一个 update
}
// update 保存在 hook.queue.pending
```

#### 创建 Hook

在 fiber 初次构造阶段, useState 对应源码 mountState, useReducer 对应源码 mountReducer

```javascript
function mountState<S>(initialState: (() => S) | S): [S, Dispatch<BasicStateAction<S>>] {
  // 1.创建 hook 并储存到 fiber.memoizedState, 或者拼接到当前处理的 hook 之后
  const hook = mountWorkInProgressHook()

  if (typeof initialState === 'function') {
    initialState = initialState()
  }
  // 2.初始化hook的值
  // 2.1 设置 hook.memoizedState/hook.baseState
  // 2.2 设置 hook.queue
  hook.memoizedState = hook.baseState = initialState
  const queue = hook.queue = {
    // update 链表
    pending: null,
    // dispatchAction.bind()的值
    dispatch: null,
    // queue.lastRenderedReducer是内置函数
    // 上一次 render 时使用的 reducer
    lastRenderedReducer: basicStateReducer,
    // 上一次 render 时的 state
    lastRenderedState: (initialState: any)
  }
  // 2.3 设置 hook.dispatch
  const dispatch: Dispatch<BasicStateAction<S>> = (queue.dispatch = (dispatchAction.bind(null, currentlyRenderingFiber, queue): any))

  // 3. 返回[当前状态, dispatch函数]
  return [hook.memoizedState, dispatch]
}
```

```javascript
function mountReducer<S, I, A>(reducer: (S, A) => S, initialArg: I, init?: I => S): [S, Dispatch<A>] {
  // 1.创建 hook 并储存到 fiber.memoizedState, 或者拼接到当前处理的 hook 之后
  const hook = mountWorkInProgressHook()
  let initialState
  if (init !== undefined) {
    initialState = init(initialArg)
  } else {
    initialState = ((initialArg: any): S)
  }
  // 2. 初始化hook的属性
  // 2.1 设置 hook.memoizedState/hook.baseState
  hook.memoizedState = hook.baseState = initialState
  // 2.2 设置 hook.queue
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    // queue.lastRenderedReducer是由外传入
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any)
  })
  // 2.3 设置 hook.dispatch
  const dispatch: Dispatch<A> = (queue.dispatch = (dispatchAction.bind(null, currentlyRenderingFiber, queue): any))

  // 3. 返回[当前状态, dispatch函数]
  return [hook.memoizedState, dispatch]
}
```

mountState 和 mountReducer 逻辑简单: 主要负责创建 hook, 初始化 hook 的属性, 最后返回[当前状态, dispatch 函数].

唯一的不同点是 hook.queue.lastRenderedReducer:

- mountState 使用的是内置的 basicStateReducer
  ```javascript
  function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
    return typeof action === 'function' ? action(state) : action
  }
  ```
- mountReducer 使用的是外部传入自定义 reducer

可见, useState 就是对 useReducer 的基本封装, 内置了一个特殊的 reducer. 创建 hook 之后返回值[hook.memoizedState, dispatch]中的 dispatch 实际上会调用 reducer 函数.

#### 状态初始化

在 useState(initialState)函数内部, 设置 hook.memoizedState = hook.baseState = initialState;, 初始状态被同时保存到了 hook.baseState,hook.memoizedState 中.

- hook.memoizedState: 当前状态
- hook.baseState: 基础状态, 作为合并 hook.baseQueue 的初始值(下文介绍).

最后返回[hook.memoizedState, dispatch], 所以在 function 中使用的是 hook.memoizedState.

#### 状态更新

通过 dispatch 函数进行更新, dispatch 实际就是 dispatchAction:

```javascript
function dispatchAction<S, A>(fiber: Fiber, queue: UpdateQueue<S, A>, action: A) {
  // 1. 创建update对象
  const eventTime = requestEventTime()
  const lane = requestUpdateLane(fiber) // Legacy模式返回SyncLane
  const update: Update<S, A> = {
    lane,
    action,
    eagerReducer: null,
    eagerState: null,
    next: (null: any)
  }

  // 2. 将update对象添加到hook.queue.pending队列
  const pending = queue.pending
  if (pending === null) {
    // 首个update, 创建一个环形链表
    update.next = update
  } else {
    update.next = pending.next
    pending.next = update
  }
  queue.pending = update

  const alternate = fiber.alternate
  if (fiber === currentlyRenderingFiber || (alternate !== null && alternate === currentlyRenderingFiber)) {
    // 渲染时更新, 做好全局标记
    // didScheduleRenderPhaseUpdate: 是否是 render 阶段触发的更新
    didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true
  } else {
   // 性能优化
   /*
      下面这个if判断, 能保证当前创建的update, 是`queue.pending`中第一个`update`.
      发起更新之后fiber.lanes会被改动(可以回顾`fiber 树构造(对比更新)`章节),
      如果`fiber.lanes && alternate.lanes`没有被改动, 自然就是首个update.
   */
   /*
      fiber.lanes保存fiber上存在的update的优先级.

      fiber.lanes === NoLanes意味着fiber上不存在update.

      我们已经知道，通过update计算state发生在声明阶段，这是因为该hook上可能存在多个不同优先级的update，最终state的值由多个update共同决定.

      但是当fiber上不存在update，则调用阶段创建的update为该hook上第一个update，在声明阶段计算state时也只依赖于该update，完全不需要进入声明阶段再计算state.

      这样做的好处是：如果计算出的state与该hook之前保存的state一致，那么完全不需要开启一次调度(不会执行更新 hook 逻辑), 即使计算出的state与该hook之前保存的state不一致，

      在声明阶段也可以直接使用调用阶段已经计算出的state.
   */
    if (fiber.lanes === NoLanes && (alternate === null || alternate.lanes === NoLanes)) {
      // The queue is currently empty, which means we can eagerly compute the
      // next state before entering the render phase. If the new state is the
      // same as the current state, we may be able to bail out entirely.
      var lastRenderedReducer = queue.lastRenderedReducer

      if (lastRenderedReducer !== null) {
        var prevDispatcher

        {
          prevDispatcher = ReactCurrentDispatcher$1.current
          ReactCurrentDispatcher$1.current = InvalidNestedHooksDispatcherOnUpdateInDEV
        }

        try {
          var currentState = queue.lastRenderedState
          var eagerState = lastRenderedReducer(currentState, action) // Stash the eagerly computed state, and the reducer used to compute
          // it, on the update object. If the reducer hasn't changed by the
          // time we enter the render phase, then the eager state can be used
          // without calling the reducer again.

          update.eagerReducer = lastRenderedReducer
          update.eagerState = eagerState

          if (objectIs(eagerState, currentState)) {
            // Fast path. We can bail out without scheduling React to re-render.
            // It's still possible that we'll need to rebase this update later,
            // if the component re-renders for a different reason and by that
            // time the reducer has changed.
            // 快速通道, eagerState与currentState相同, 无需调度更新
            // 注: update已经被添加到了queue.pending, 并没有丢弃. 之后需要更新的时候, 此update还是会起作用
            return
          }
        } catch (error) {
          // Suppress the error. It will throw again in the render phase.
        } finally {
          {
            ReactCurrentDispatcher$1.current = prevDispatcher
          }
        }
      }
    }
    // 3. 发起调度更新, 进入`reconciler 运作流程`中的输入阶段.
    scheduleUpdateOnFiber(fiber, lane, eventTime)
  }
}
```

逻辑十分清晰:

1.创建 update 对象, 其中 update.lane 代表优先级(可回顾 fiber 树构造(基础准备)中的 update 优先级).

2.将 update 对象添加到 hook.queue.pending 环形链表.

- 环形链表的特征: 为了方便添加新元素和快速拿到队首元素(都是 O(1)), 所以 pending 指针指向了链表中最后一个元素.
- 链表的使用方式可以参考 React 算法之链表操作

  3.发起调度更新: 调用 scheduleUpdateOnFiber, 进入 reconciler 运作流程中的输入阶段.

在 fiber 树构造(对比更新)过程中, 再次调用 function, 这时 useState 对应的函数是 updateState, 实际调用 updateReducer.

```javascript
function updateState<S>(initialState: (() => S) | S): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any))
}
```

```javascript
function updateReducer<S, I, A>(reducer: (S, A) => S, initialArg: I, init?: I => S): [S, Dispatch<A>] {
  // 1.创建 hook 并储存到 fiber.memoizedState, 或者拼接到当前处理的 hook 之后
  const hook = updateWorkInProgressHook()
  const queue = hook.queue
  queue.lastRenderedReducer = reducer
  const current: Hook = (currentHook: any)
  let baseQueue = current.baseQueue

  // 2. 链表拼接: 将 hook.queue.pending 拼接到 current.baseQueue
  const pendingQueue = queue.pending
  if (pendingQueue !== null) {
    if (baseQueue !== null) {
      const baseFirst = baseQueue.next
      const pendingFirst = pendingQueue.next
      baseQueue.next = pendingFirst
      pendingQueue.next = baseFirst
    }
    current.baseQueue = baseQueue = pendingQueue
    queue.pending = null
  }
  // 3. 状态计算
  if (baseQueue !== null) {
    const first = baseQueue.next
    let newState = current.baseState

    let newBaseState = null
    let newBaseQueueFirst = null
    let newBaseQueueLast = null
    let update = first

    do {
      const updateLane = update.lane
      // 3.1 优先级提取update
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 优先级不够: 加入到baseQueue中, 等待下一次render
        const clone: Update<S, A> = {
          lane: updateLane,
          action: update.action,
          eagerReducer: update.eagerReducer,
          eagerState: update.eagerState,
          next: (null: any)
        }
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone
          newBaseState = newState
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone
        }
        currentlyRenderingFiber.lanes = mergeLanes(currentlyRenderingFiber.lanes, updateLane)
        markSkippedUpdateLanes(updateLane)
      } else {
        // 优先级足够: 状态合并
        if (newBaseQueueLast !== null) {
          // 更新baseQueue
          const clone: Update<S, A> = {
            lane: NoLane,
            action: update.action,
            eagerReducer: update.eagerReducer,
            eagerState: update.eagerState,
            next: (null: any)
          }
          newBaseQueueLast = newBaseQueueLast.next = clone
        }
        if (update.eagerReducer === reducer) {
          // 性能优化: 如果存在 update.eagerReducer, 直接使用update.eagerState.避免重复调用reducer
          newState = ((update.eagerState: any): S)
        } else {
          const action = update.action
          // 调用reducer获取最新状态
          newState = reducer(newState, action)
        }
      }
      update = update.next
    } while (update !== null && update !== first)

    // 3.2. 更新属性
    if (newBaseQueueLast === null) {
      newBaseState = newState
    } else {
      newBaseQueueLast.next = (newBaseQueueFirst: any)
    }
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate()
    }
    // 把计算之后的结果更新到workInProgressHook上
    hook.memoizedState = newState
    hook.baseState = newBaseState
    hook.baseQueue = newBaseQueueLast
    queue.lastRenderedState = newState
  }

  const dispatch: Dispatch<A> = (queue.dispatch: any)
  return [hook.memoizedState, dispatch]
}
```

updateReducer 函数, 代码相对较长, 但是逻辑分明:

1.调用 updateWorkInProgressHook 获取 workInProgressHook 对象.
```javascript
function updateWorkInProgressHook() {
  // This function is used both for updates and for re-renders triggered by a
  // render phase update. It assumes there is either a current hook we can
  // clone, or a work-in-progress hook from a previous render pass that we can
  // use as a base. When we reach the end of the base list, we must switch to
  // the dispatcher used for mounts.

  // 1. 移动 currentHook 指针

  // 下一个要处理的 current hook
  var nextCurrentHook;

  // currentHook: current.memoizedState
  // currentlyRenderingFiber$1: 当前正在构造的 fiber, 等同于 workInProgress
  if (currentHook === null) {
    var current = currentlyRenderingFiber$1.alternate;

    if (current !== null) {
      // 如果 current.memoizedState 没有则从 workInProgress.alternate 获取 memoizedState 并储存到下次要处理的 hook
      nextCurrentHook = current.memoizedState;
    } else {
      nextCurrentHook = null;
    }
  } else {
    // 如果上次 currentHook 存在, 获取它的下一个 hook(指针移动)
    nextCurrentHook = currentHook.next;
  }
 
  // 2. 移动 workInProgressHook 指针
  var nextWorkInProgressHook;

  // workInProgressHook: workInProgress.memoizedState
  if (workInProgressHook === null) {
    nextWorkInProgressHook = currentlyRenderingFiber$1.memoizedState;
  } else {
    // workInProgressHook 指针移动
    nextWorkInProgressHook = workInProgressHook.next; // next = null
  }

  if (nextWorkInProgressHook !== null) {
    // There's already a work-in-progress. Reuse it.
    workInProgressHook = nextWorkInProgressHook;
    nextWorkInProgressHook = workInProgressHook.next;
    currentHook = nextCurrentHook;
  } else {
    // renderWithHooks 阶段会重置 workInProgress.memoizedState = null, 所以初始化更新时会复用 current 的 memoizedState
    // 将 current.memoizedState 按照顺序克隆到 workInProgress.memoizedState 中
    // Clone from the current hook.
    if (!(nextCurrentHook !== null)) {
      {
        throw Error( "Rendered more hooks than during the previous render." );
      }
    }

    // 更新当前处理的 current hook
    currentHook = nextCurrentHook;
    // 3. 克隆currentHook作为新的workInProgressHook.
    var newHook = {
      memoizedState: currentHook.memoizedState,
      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,
      next: null // 初始化为 null, 保证后续更新都要从 current 克隆
    };

    if (workInProgressHook === null) {
      // This is the first hook in the list.
      currentlyRenderingFiber$1.memoizedState = workInProgressHook = newHook;
    } else {
      // Append to the end of the list.
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }

  return workInProgressHook;
}
```

2.链表拼接: 将 hook.queue.pending 拼接到 current.baseQueue.

3.状态计算

- update 优先级不够: 加入到 baseQueue 中, 等待下一次 render
- update 优先级足够: 状态合并
- 更新属性
