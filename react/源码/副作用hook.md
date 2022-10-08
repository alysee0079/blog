# 副作用hook

#### 创建 Hook
在fiber初次构造阶段, useEffect对应源码mountEffect, useLayoutEffect对应源码mountLayoutEffect
```javascript
function mountEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return mountEffectImpl(
    UpdateEffect | PassiveEffect, // fiberFlags
    HookPassive, // hookFlags
    create,
    deps,
  );
}

function mountLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return mountEffectImpl(
    UpdateEffect, // fiberFlags
    HookLayout, // hookFlags
    create,
    deps,
  );
}
```

可见mountEffect和mountLayoutEffect内部都直接调用mountEffectImpl, 只是参数不同.
```javascript
function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  // 1. 创建hook
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 2. 设置workInProgress的副作用标记
  currentlyRenderingFiber.flags |= fiberFlags; // fiberFlags 被标记到workInProgress
  // 2. 创建Effect, 挂载到hook.memoizedState上
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags, // hookFlags用于创建effect
    create,
    undefined,
    nextDeps,
  );
}
```

#### 创建 Effect
```javascript
//  effect 的类型:
//  HasEffect: 有副作用, 可以被触发
//  Layout: Layout, dom 突变后同步触发
//  Passive: Passive, dom 突变前异步触发

function pushEffect(tag, create, destroy, deps) {
  // 1. 创建effect对象
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    next: (null: any),
  };
  // 2. 把effect对象添加到环形链表末尾
  let componentUpdateQueue: null | FunctionComponentUpdateQueue = (currentlyRenderingFiber.updateQueue: any);
  if (componentUpdateQueue === null) {
    // 新建 workInProgress.updateQueue 用于挂载effect对象
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);
    // updateQueue.lastEffect是一个环形链表
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  // 3. 返回effect
  return effect;
}
```

##### useEffect & useLayoutEffect
站在fiber,hook,effect的视角, 无需关心这个hook是通过useEffect还是useLayoutEffect创建的. 只需要关心内部fiber.flags,effect.tag的状态.

所以useEffect与useLayoutEffect的区别如下:

1.fiber.flags不同
  - 使用useEffect时: fiber.flags = UpdateEffect | PassiveEffect.
  - 使用useLayoutEffect时: fiber.flags = UpdateEffect.

2.effect.tag不同
  - 使用useEffect时: effect.tag = HookHasEffect | HookPassive.
  - 使用useLayoutEffect时: effect.tag = HookHasEffect | HookLayout.

#### 处理 Effect 回调
完成fiber树构造后, 逻辑会进入渲染阶段. 通过fiber 树渲染中的介绍, 在commitRootImpl函数中, 整个渲染过程被 3 个函数分布实现:

1.commitBeforeMutationEffects
2.commitMutationEffects
3.commitLayoutEffects

这 3 个函数会处理fiber.flags, 也会根据情况处理fiber.updateQueue.lastEffect

##### commitBeforeMutationEffects
第一阶段: dom 变更之前, 处理副作用队列中带有Passive标记的fiber节点.
```javascript
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    // 处理`Passive`标记
    const flags = nextEffect.flags;
    if ((flags & Passive) !== NoFlags) {
      if (!rootDoesHavePassiveEffects) {
        rootDoesHavePassiveEffects = true;
        scheduleCallback(NormalSchedulerPriority, () => {
          flushPassiveEffects();
          return null;
        });
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```
注意: 由于flushPassiveEffects被包裹在scheduleCallback回调中, 由调度中心来处理, 且参数是NormalSchedulerPriority, 故这是一个异步回调(具体原理可以回顾React 调度原理(scheduler)).

##### commitMutationEffects
第二阶段: dom 变更, 界面得到更新.
```javascript
function commitMutationEffects(
  root: FiberRoot,
  renderPriorityLevel: ReactPriorityLevel,
) {
  while (nextEffect !== null) {
    const flags = nextEffect.flags;
    const primaryFlags = flags & (Placement | Update | Deletion | Hydrating);
    switch (primaryFlags) {
      case Update: {
        // useEffect,useLayoutEffect都会设置Update标记
        // 更新节点
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}

function commitWork(current: Fiber | null, finishedWork: Fiber): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case MemoComponent:
    case SimpleMemoComponent:
    case Block: {
      // 在突变阶段调用销毁函数, 保证所有的effect.destroy函数都会在effect.create之前执行
      commitHookEffectListUnmount(HookLayout | HookHasEffect, finishedWork);
      return;
    }
  }
}

// 依次执行: effect.destroy
function commitHookEffectListUnmount(tag: number, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & tag) === tag) {
        // 根据传入的tag过滤 effect链表.
        const destroy = effect.destroy;
        effect.destroy = undefined;
        if (destroy !== undefined) {
          destroy();
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```
调用关系: commitMutationEffects->commitWork->commitHookEffectListUnmount.
  - 注意在调用commitMutationEffects(HookLayout | HookHasEffect, finishedWork)时, 参数是HookLayout | HookHasEffect, 所以只处理由useLayoutEffect()创建的effect.
  - 根据上文的分析HookLayout | HookHasEffect是通过useLayoutEffect创建的effect. 所以commitMutationEffects函数只能处理由useLayoutEffect()创建的effect.
  - 同步调用effect.destroy().

##### commitLayoutEffects
第三阶段: dom 变更后
```javascript
function commitLayoutEffects(root: FiberRoot, committedLanes: Lanes) {
  while (nextEffect !== null) {
    const flags = nextEffect.flags;
    if (flags & (Update | Callback)) {
      // useEffect,useLayoutEffect都会设置Update标记
      const current = nextEffect.alternate;
      commitLayoutEffectOnFiber(root, current, nextEffect, committedLanes);
    }
    nextEffect = nextEffect.nextEffect;
  }
}

function commitLifeCycles(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber,
  committedLanes: Lanes,
): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
    case Block: {
      // 在此之前commitMutationEffects函数中, effect.destroy已经被调用, 所以effect.destroy永远不会影响到effect.create
      commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);

      schedulePassiveEffects(finishedWork);
      return;
    }
  }
}

function commitHookEffectListMount(tag: number, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & tag) === tag) {
        const create = effect.create;
        effect.destroy = create();
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```
1.调用关系: commitLayoutEffects->commitLayoutEffectOnFiber(commitLifeCycles)->commitHookEffectListMount.
  - 注意在调用commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork)时, 参数是HookLayout | HookHasEffect,所以只处理由useLayoutEffect()创建的effect.
  - 调用effect.create()之后, 将返回值赋值到effect.destroy.

2.为flushPassiveEffects做准备
  - commitLifeCycles中的schedulePassiveEffects(finishedWork), 其形参finishedWork实际上指代当前正在被遍历的有副作用的fiber.
  - schedulePassiveEffects比较简单, 就是把带有Passive标记的effect筛选出来(由useEffect创建), 添加到一个全局数组(pendingPassiveHookEffectsUnmount和pendingPassiveHookEffectsMount).

```javascript
function schedulePassiveEffects(finishedWork: Fiber) {
  // 1. 获取 fiber.updateQueue
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  // 2. 获取 effect环形队列
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      const { next, tag } = effect;
      // 3. 筛选出由useEffect()创建的`effect`
      if (
        (tag & HookPassive) !== NoHookEffect &&
        (tag & HookHasEffect) !== NoHookEffect
      ) {
        // 把effect添加到全局数组, 等待`flushPassiveEffects`处理
        enqueuePendingPassiveHookEffectUnmount(finishedWork, effect);
        enqueuePendingPassiveHookEffectMount(finishedWork, effect);
      }
      effect = next;
    } while (effect !== firstEffect);
  }
}

export function enqueuePendingPassiveHookEffectUnmount(
  fiber: Fiber,
  effect: HookEffect,
): void {
  // unmount effects 数组
  pendingPassiveHookEffectsUnmount.push(effect, fiber);
}

export function enqueuePendingPassiveHookEffectMount(
  fiber: Fiber,
  effect: HookEffect,
): void {
  // mount effects 数组
  pendingPassiveHookEffectsMount.push(effect, fiber);
}
```
综上 commitMutationEffects 和 commitLayoutEffects 2个函数, 带有Layout标记的effect(由useLayoutEffect创建), 已经得到了完整的回调处理(destroy和create已经被调用).

##### flushPassiveEffects
在上文commitBeforeMutationEffects阶段, 异步调用了flushPassiveEffects. 在这期间带有Passive标记的effect已经被添加到pendingPassiveHookEffectsUnmount和pendingPassiveHookEffectsMount全局数组中.

接下来flushPassiveEffects就可以脱离fiber节点, 直接访问effects
```javascript
export function flushPassiveEffects(): boolean {
  // Returns whether passive effects were flushed.
  if (pendingPassiveEffectsRenderPriority !== NoSchedulerPriority) {
    const priorityLevel =
      pendingPassiveEffectsRenderPriority > NormalSchedulerPriority
        ? NormalSchedulerPriority
        : pendingPassiveEffectsRenderPriority;
    pendingPassiveEffectsRenderPriority = NoSchedulerPriority;
    // `runWithPriority`设置Schedule中的调度优先级, 如果在flushPassiveEffectsImpl中处理effect时又发起了新的更新, 那么新的update.lane将会受到这个priorityLevel影响.
    return runWithPriority(priorityLevel, flushPassiveEffectsImpl);
  }
  return false;
}

function flushPassiveEffectsImpl() {
  if (rootWithPendingPassiveEffects === null) {
    return false;
  }
  rootWithPendingPassiveEffects = null;
  pendingPassiveEffectsLanes = NoLanes;

  // 1. 执行 effect.destroy()
  const unmountEffects = pendingPassiveHookEffectsUnmount;
  pendingPassiveHookEffectsUnmount = [];
  for (let i = 0; i < unmountEffects.length; i += 2) {
    const effect = ((unmountEffects[i]: any): HookEffect);
    const fiber = ((unmountEffects[i + 1]: any): Fiber);
    const destroy = effect.destroy;
    effect.destroy = undefined;
    if (typeof destroy === 'function') {
      destroy();
    }
  }

  // 2. 执行新 effect.create(), 重新赋值到 effect.destroy
  const mountEffects = pendingPassiveHookEffectsMount;
  pendingPassiveHookEffectsMount = [];
  for (let i = 0; i < mountEffects.length; i += 2) {
    const effect = ((mountEffects[i]: any): HookEffect);
    const fiber = ((mountEffects[i + 1]: any): Fiber);
    effect.destroy = create();
  }
}
```

其核心逻辑:

1.遍历pendingPassiveHookEffectsUnmount中的所有effect, 调用effect.destroy().
  - 同时清空pendingPassiveHookEffectsUnmount

2.遍历pendingPassiveHookEffectsMount中的所有effect, 调用effect.create(), 并更新effect.destroy.
  - 同时清空pendingPassiveHookEffectsMount

所以, 带有Passive标记的effect, 在flushPassiveEffects函数中得到了完整的回调处理.

#### 更新 Hook
假设在初次调用之后, 发起更新, 会再次执行function, 这时function只使用的useEffect, useLayoutEffect等api也会再次执行.

在更新过程中useEffect对应源码updateEffect, useLayoutEffect对应源码updateLayoutEffect.它们内部都会调用updateEffectImpl, 与初次创建时一样, 只是参数不同.

##### 更新 Effect
```javascript
function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
  // 1. 获取当前hook
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;
  // 2. 分析依赖
  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    // 继续使用先前effect.destroy
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      // 比较依赖是否变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 2.1 如果依赖不变, 新建effect(tag不含HookHasEffect)
        pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }
  // 2.2 如果依赖改变, 更改fiber.flag, 新建effect
  currentlyRenderingFiber.flags |= fiberFlags;

  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    destroy,
    nextDeps,
  );
}
```
updateEffectImpl与mountEffectImpl逻辑有所不同: - 如果useEffect/useLayoutEffect的依赖不变, 新建的effect对象不带HasEffect标记.

##### 处理 Effect 回调
新的hook以及新的effect创建完成之后, 余下逻辑与初次渲染完全一致. 处理 Effect 回调时也会根据effect.tag进行判断: 只有effect.tag包含HookHasEffect时才会调用effect.destroy和effect.create()

#### 组件销毁
当function组件被销毁时, fiber节点必然会被打上Deletion标记, 即fiber.flags |= Deletion. 带有Deletion标记的fiber在commitMutationEffects被处理:
```javascript
function commitMutationEffects(
  root: FiberRoot,
  renderPriorityLevel: ReactPriorityLevel,
) {
  while (nextEffect !== null) {
    const primaryFlags = flags & (Placement | Update | Deletion | Hydrating);
    switch (primaryFlags) {
      case Deletion: {
        commitDeletion(root, nextEffect, renderPriorityLevel);
        break;
      }
    }
  }
}
```
在commitDeletion函数之后, 继续调用unmountHostComponents->commitUnmount, 在commitUnmount中, 执行effect.destroy(), 结束整个闭环.

