# 状态 Hook

1.function类型的fiber节点, 它的处理函数是updateFunctionComponent, 其中再通过renderWithHooks调用function.

2.在function中, 通过Hook Api(如: useState, useEffect)创建Hook对象.
  - 状态Hook实现了状态持久化(等同于class组件维护fiber.memoizedState).
  - 副作用Hook则实现了维护fiber.flags,并提供副作用回调(类似于class组件的生命周期回调)

3.多个Hook对象构成一个链表结构, 并挂载到fiber.memoizedState之上.

4.fiber树更新阶段, 把current.memoizedState链表上的所有Hook按照顺序克隆到workInProgress.memoizedState上, 实现数据的持久化.

#### 创建 Hook

在fiber初次构造阶段, useState对应源码mountState, useReducer对应源码mountReducer
```javascript
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  // 1. 创建hook
  const hook = mountWorkInProgressHook();
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  // 2. 初始化hook的属性
  // 2.1 设置 hook.memoizedState/hook.baseState
  // 2.2 设置 hook.queue
  hook.memoizedState = hook.baseState = initialState;
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    // queue.lastRenderedReducer是内置函数
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  });
  // 2.3 设置 hook.dispatch
  const dispatch: Dispatch<
    BasicStateAction<S>,
  > = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));

  // 3. 返回[当前状态, dispatch函数]
  return [hook.memoizedState, dispatch];
}
```
```javascript
function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 1. 创建hook
  const hook = mountWorkInProgressHook();
  let initialState;
  if (init !== undefined) {
    initialState = init(initialArg);
  } else {
    initialState = ((initialArg: any): S);
  }
  // 2. 初始化hook的属性
  // 2.1 设置 hook.memoizedState/hook.baseState
  hook.memoizedState = hook.baseState = initialState;
  // 2.2 设置 hook.queue
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    // queue.lastRenderedReducer是由外传入
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any),
  });
  // 2.3 设置 hook.dispatch
  const dispatch: Dispatch<A> = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));

  // 3. 返回[当前状态, dispatch函数]
  return [hook.memoizedState, dispatch];
}
```
mountState和mountReducer逻辑简单: 主要负责创建hook, 初始化hook的属性, 最后返回[当前状态, dispatch函数].

唯一的不同点是hook.queue.lastRenderedReducer:

  - mountState使用的是内置的basicStateReducer
    ```javascript
    function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
      return typeof action === 'function' ? action(state) : action;
    }
    ```
  - mountReducer使用的是外部传入自定义reducer

可见, useState就是对useReducer的基本封装, 内置了一个特殊的reducer. 创建hook之后返回值[hook.memoizedState, dispatch]中的dispatch实际上会调用reducer函数.

#### 状态初始化
在useState(initialState)函数内部, 设置hook.memoizedState = hook.baseState = initialState;, 初始状态被同时保存到了hook.baseState,hook.memoizedState中.

  - hook.memoizedState: 当前状态
  - hook.baseState: 基础状态, 作为合并hook.baseQueue的初始值(下文介绍).

最后返回[hook.memoizedState, dispatch], 所以在function中使用的是hook.memoizedState.

#### 状态更新
通过dispatch函数进行更新, dispatch实际就是dispatchAction:
```javascript
function dispatchAction<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  // 1. 创建update对象
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(fiber); // Legacy模式返回SyncLane
  const update: Update<S, A> = {
    lane,
    action,
    eagerReducer: null,
    eagerState: null,
    next: (null: any),
  };

  // 2. 将update对象添加到hook.queue.pending队列
  const pending = queue.pending;
  if (pending === null) {
    // 首个update, 创建一个环形链表
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;

  const alternate = fiber.alternate;
  if (
    fiber === currentlyRenderingFiber ||
    (alternate !== null && alternate === currentlyRenderingFiber)
  ) {
    // 渲染时更新, 做好全局标记
    didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true;
  } else {
    // 3. 发起调度更新, 进入`reconciler 运作流程`中的输入阶段.
    scheduleUpdateOnFiber(fiber, lane, eventTime);
  }
}
```
逻辑十分清晰:

1.创建update对象, 其中update.lane代表优先级(可回顾fiber 树构造(基础准备)中的update优先级).

2.将update对象添加到hook.queue.pending环形链表.
  - 环形链表的特征: 为了方便添加新元素和快速拿到队首元素(都是O(1)), 所以pending指针指向了链表中最后一个元素.
  - 链表的使用方式可以参考React 算法之链表操作

3.发起调度更新: 调用scheduleUpdateOnFiber, 进入reconciler 运作流程中的输入阶段.


在fiber树构造(对比更新)过程中, 再次调用function, 这时useState对应的函数是updateState, 实际调用updateReducer.
```javascript
function updateState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any));
}
```
```javascript
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 1. 获取workInProgressHook对象
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  queue.lastRenderedReducer = reducer;
  const current: Hook = (currentHook: any);
  let baseQueue = current.baseQueue;

  // 2. 链表拼接: 将 hook.queue.pending 拼接到 current.baseQueue
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    if (baseQueue !== null) {
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }
  // 3. 状态计算
  if (baseQueue !== null) {
    const first = baseQueue.next;
    let newState = current.baseState;

    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;

    do {
      const updateLane = update.lane;
      // 3.1 优先级提取update
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 优先级不够: 加入到baseQueue中, 等待下一次render
        const clone: Update<S, A> = {
          lane: updateLane,
          action: update.action,
          eagerReducer: update.eagerReducer,
          eagerState: update.eagerState,
          next: (null: any),
        };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane,
        );
        markSkippedUpdateLanes(updateLane);
      } else {
        // 优先级足够: 状态合并
        if (newBaseQueueLast !== null) {
          // 更新baseQueue
          const clone: Update<S, A> = {
            lane: NoLane,
            action: update.action,
            eagerReducer: update.eagerReducer,
            eagerState: update.eagerState,
            next: (null: any),
          };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        if (update.eagerReducer === reducer) {
          // 性能优化: 如果存在 update.eagerReducer, 直接使用update.eagerState.避免重复调用reducer
          newState = ((update.eagerState: any): S);
        } else {
          const action = update.action;
          // 调用reducer获取最新状态
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    // 3.2. 更新属性
    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }
    // 把计算之后的结果更新到workInProgressHook上
    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;
    queue.lastRenderedState = newState;
  }

  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```
updateReducer函数, 代码相对较长, 但是逻辑分明:

1.调用updateWorkInProgressHook获取workInProgressHook对象.

2.链表拼接: 将 hook.queue.pending 拼接到 current.baseQueue.

3.状态计算
  - update优先级不够: 加入到 baseQueue 中, 等待下一次 render
  - update优先级足够: 状态合并
  - 更新属性

