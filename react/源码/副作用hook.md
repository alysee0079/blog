# 副作用hook

### useEffect/useLayoutEffect
#### 创建 Hook
在 fiber 初次构造阶段, useEffect 对应源码 mountEffect, useLayoutEffect 对应源码 mountLayoutEffect,
mountEffect 和 mountLayoutEffect 内部都直接调用 mountEffectImpl, 只是参数不同(Passive || Layout).
```javascript
function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  // 1. 创建 hook 并储存到 fiber.memoizedState, 或者拼接到当前处理的 hook 之后
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 2. 设置 workInProgress的 副作用标记
  currentlyRenderingFiber.flags |= fiberFlags;
  // 2. 创建 Effect, 挂载到 hook.memoizedState 上
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags, // hookFlags 用于创建 effect
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
  // 1. 创建 effect 对象
  const effect: Effect = {
    tag, // 副作用类型
    create, // 副作用执行方法
    destroy,
    deps, // 依赖
    next: (null: any),
  };
  // 2. 把effect对象添加到环形链表末尾
  var componentUpdateQueue = currentlyRenderingFiber$1.updateQueue;

  if (componentUpdateQueue === null) {
    // 新建 updateQueue 用于储存 effect
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    // 将 effect 联保保存到 fiber.updateQueue
    currentlyRenderingFiber$1.updateQueue = componentUpdateQueue;
     // updateQueue.lastEffect 是一个环形链表
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    var lastEffect = componentUpdateQueue.lastEffect;

    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      // 拼接 effect 到队尾, 并将 lastEffect 指向这个 effect
      var firstEffect = lastEffect.next;
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

### useRef
与其他Hook一样，对于mount与update，useRef对应两个不同dispatcher。
```javascript
function mountRef<T>(initialValue: T): {|current: T|} {
  // 获取当前useRef hook
  const hook = mountWorkInProgressHook();
  // 创建ref
  const ref = {current: initialValue};
  hook.memoizedState = ref;
  return ref;
}

function updateRef<T>(initialValue: T): {|current: T|} {
  // 获取当前useRef hook
  const hook = updateWorkInProgressHook();
  // 返回保存的数据
  return hook.memoizedState;
}
```
可见，useRef仅仅是返回一个包含current属性的对象

为了验证这个观点，我们再看下React.createRef方法的实现：
```javascript
export function createRef(): RefObject {
  // 每次都创建一个新的对象
  const refObject = {
    current: null,
  };
  return refObject;
}
```

我们知道HostComponent在commit阶段的mutation阶段执行DOM操作, 所以，对应ref的更新也是发生在mutation阶段,
再进一步，mutation阶段执行DOM操作的依据为effectTag.所以，对于HostComponent、ClassComponent如果包含ref操作，那么也会赋值相应的effectTag.
```javascript
export const Placement = /*                    */ 0b0000000000000010;
export const Update = /*                       */ 0b0000000000000100;
export const Deletion = /*                     */ 0b0000000000001000;
export const Ref = /*                          */ 0b0000000010000000;
```

所以，ref的工作流程可以分为两部分：
  - render阶段为含有ref属性的fiber添加Ref effectTag
  - commit阶段为包含Ref effectTag的fiber执行对应操作

#### render 阶段
在render阶段的beginWork与completeWork中有个同名方法markRef用于为含有ref属性的fiber增加Ref effectTag.
```javascript
// beginWork 的 markRef(mount)
function markRef(current: Fiber | null, workInProgress: Fiber) {
  const ref = workInProgress.ref;
  // current ref 不存在或者和本次不同
  if (
    (current === null && ref !== null) ||
    (current !== null && current.ref !== ref)
  ) {
    // Schedule a Ref effect(update)
    workInProgress.effectTag |= Ref;
  }
}
// completeWork 的 markRef
function markRef$1(workInProgress: Fiber) {
  workInProgress.effectTag |= Ref;
}
```
组件对应fiber被赋值Ref effectTag需要满足的条件：
  - fiber类型为HostComponent、ClassComponent、ScopeComponent
  - 对于 mount，workInProgress.ref !== null，即存在ref属性
  - 对于 update，current.ref !== workInProgress.ref，即ref属性改变

#### commit阶段

在commit阶段的mutation阶段中，对于ref属性改变的情况，需要先移除之前的ref
```javascript
function commitMutationEffects(root: FiberRoot, renderPriorityLevel) {
  while (nextEffect !== null) {

    const effectTag = nextEffect.effectTag;
    // ...

    if (effectTag & Ref) {
      const current = nextEffect.alternate;
      if (current !== null) {
        // 移除之前的ref(mount 阶段不会触发)
        commitDetachRef(current);
      }
    }
    // ...
  }
  // ...
}

function commitDetachRef(current: Fiber) {
  const currentRef = current.ref;
  if (currentRef !== null) {
    if (typeof currentRef === 'function') {
      // function类型ref，调用他，传参为null
      currentRef(null);
    } else {
      // 对象类型ref，current赋值为null
      currentRef.current = null;
    }
  }
}
```
接下来，在 mutation阶段，对于Deletion effectTag 的 fiber（对应需要删除的DOM节点），需要递归他的子树，对子孙 fiber 的 ref 执行类似 commitDetachRef 的操作.

对于Deletion effectTag的fiber，会执行commitDeletion:
在commitDeletion——unmountHostComponents——commitUnmount——ClassComponent | HostComponent类型case中调用的safelyDetachRef方法负责执行类似commitDetachRef的操作.
```javascript
function safelyDetachRef(current: Fiber) {
  const ref = current.ref;
  if (ref !== null) {
    if (typeof ref === 'function') {
      try {
        ref(null);
      } catch (refError) {
        captureCommitPhaseError(current, refError);
      }
    } else {
      ref.current = null;
    }
  }
}
```
接下来进入 ref 的赋值阶段, commitLayoutEffect 会执行 commitAttachRef（赋值ref）:
```javascript
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    // 获取ref属性对应的Component实例
    const instance = finishedWork.stateNode;
    let instanceToUse;
    switch (finishedWork.tag) {
      case HostComponent:
        instanceToUse = getPublicInstance(instance);
        break;
      default:
        instanceToUse = instance;
    }

    // 赋值ref
    if (typeof ref === 'function') {
      ref(instanceToUse);
    } else {
      ref.current = instanceToUse;
    }
  }
}
```

#### useMemo/useCallback
mount
```javascript
function mountMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  // 创建并返回当前hook
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 计算value
  const nextValue = nextCreate();
  // 将value与deps保存在hook.memoizedState
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  // 创建并返回当前hook
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 将value与deps保存在hook.memoizedState
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```
可以看到，与mountCallback这两个唯一的区别是
  - mountMemo会将回调函数(nextCreate)的执行结果作为value保存
  - mountCallback会将回调函数作为value保存

update
```javascript
function updateMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  // 返回当前hook
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;

  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      // 判断update前后value是否变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 未变化
        return prevState[0];
      }
    }
  }
  // 变化，重新计算value
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  // 返回当前hook
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;

  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      // 判断update前后value是否变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 未变化
        return prevState[0];
      }
    }
  }

  // 变化，将新的callback作为value
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```
可见，对于update，这两个hook的唯一区别也是是回调函数本身还是回调函数的执行结果作为value。
