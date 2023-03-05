# 副作用 hook

### useEffect/useLayoutEffect

#### 创建 Hook

在 fiber 初次构造阶段, useEffect 对应源码 mountEffect, useLayoutEffect 对应源码 mountLayoutEffect,
mountEffect 和 mountLayoutEffect 内部都直接调用 mountEffectImpl, 只是参数不同(Passive || Layout).

```javascript
function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  // 1. 创建 hook 并储存到 fiber.memoizedState, 或者拼接到当前处理的 hook 之后
  const hook = mountWorkInProgressHook()
  const nextDeps = deps === undefined ? null : deps
  // 2. 设置 workInProgress的 副作用标记
  currentlyRenderingFiber.flags |= fiberFlags
  // 2. 创建 Effect, 挂载到 hook.memoizedState 上
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags, // hookFlags 用于创建 effect
    create,
    undefined,
    nextDeps
  )
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
    next: (null: any)
  }
  // 2. 把effect对象添加到环形链表末尾
  var componentUpdateQueue = currentlyRenderingFiber$1.updateQueue

  if (componentUpdateQueue === null) {
    // 新建 updateQueue 用于储存 effect
    componentUpdateQueue = createFunctionComponentUpdateQueue()
    // 将 effect 联保保存到 fiber.updateQueue
    currentlyRenderingFiber$1.updateQueue = componentUpdateQueue
    // updateQueue.lastEffect 是一个环形链表
    componentUpdateQueue.lastEffect = effect.next = effect
  } else {
    var lastEffect = componentUpdateQueue.lastEffect

    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect
    } else {
      // 拼接 effect 到队尾, 并将 lastEffect 指向这个 effect
      var firstEffect = lastEffect.next
      lastEffect.next = effect
      effect.next = firstEffect
      componentUpdateQueue.lastEffect = effect
    }
  }
  // 3. 返回effect
  return effect
}
```

##### useEffect & useLayoutEffect

站在 fiber,hook,effect 的视角, 无需关心这个 hook 是通过 useEffect 还是 useLayoutEffect 创建的. 只需要关心内部 fiber.flags,effect.tag 的状态.

所以 useEffect 与 useLayoutEffect 的区别如下:

1.fiber.flags 不同

- 使用 useEffect 时: fiber.flags = UpdateEffect | PassiveEffect.
- 使用 useLayoutEffect 时: fiber.flags = UpdateEffect.

  2.effect.tag 不同

- 使用 useEffect 时: effect.tag = HookHasEffect | HookPassive.
- 使用 useLayoutEffect 时: effect.tag = HookHasEffect | HookLayout.

#### 处理 Effect 回调

完成 fiber 树构造后, 逻辑会进入渲染阶段. 通过 fiber 树渲染中的介绍, 在 commitRootImpl 函数中, 整个渲染过程被 3 个函数分布实现:

1.commitBeforeMutationEffects
2.commitMutationEffects
3.commitLayoutEffects

这 3 个函数会处理 fiber.flags, 也会根据情况处理 fiber.updateQueue.lastEffect

##### commitBeforeMutationEffects

第一阶段: dom 变更之前, 处理副作用队列中带有 Passive 标记的 fiber 节点.

```javascript
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    // 处理`Passive`标记
    const flags = nextEffect.flags
    if ((flags & Passive) !== NoFlags) {
      if (!rootDoesHavePassiveEffects) {
        rootDoesHavePassiveEffects = true
        scheduleCallback(NormalSchedulerPriority, () => {
          flushPassiveEffects()
          return null
        })
      }
    }
    nextEffect = nextEffect.nextEffect
  }
}
```

注意: 由于 flushPassiveEffects 被包裹在 scheduleCallback 回调中, 由调度中心来处理, 且参数是 NormalSchedulerPriority, 故这是一个异步回调(具体原理可以回顾 React 调度原理(scheduler)).

##### commitMutationEffects

第二阶段: dom 变更, 界面得到更新.

```javascript
function commitMutationEffects(root: FiberRoot, renderPriorityLevel: ReactPriorityLevel) {
  while (nextEffect !== null) {
    const flags = nextEffect.flags
    const primaryFlags = flags & (Placement | Update | Deletion | Hydrating)
    switch (primaryFlags) {
      case Update: {
        // useEffect,useLayoutEffect都会设置Update标记
        // 更新节点
        const current = nextEffect.alternate
        commitWork(current, nextEffect)
        break
      }
    }
    nextEffect = nextEffect.nextEffect
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
      commitHookEffectListUnmount(HookLayout | HookHasEffect, finishedWork)
      return
    }
  }
}

// 依次执行: effect.destroy
function commitHookEffectListUnmount(tag: number, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any)
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next
    let effect = firstEffect
    do {
      if ((effect.tag & tag) === tag) {
        // 根据传入的tag过滤 effect链表.
        const destroy = effect.destroy
        effect.destroy = undefined
        if (destroy !== undefined) {
          destroy()
        }
      }
      effect = effect.next
    } while (effect !== firstEffect)
  }
}
```

调用关系: commitMutationEffects->commitWork->commitHookEffectListUnmount.

- 注意在调用 commitMutationEffects(HookLayout | HookHasEffect, finishedWork)时, 参数是 HookLayout | HookHasEffect, 所以只处理由 useLayoutEffect()创建的 effect.
- 根据上文的分析 HookLayout | HookHasEffect 是通过 useLayoutEffect 创建的 effect. 所以 commitMutationEffects 函数只能处理由 useLayoutEffect()创建的 effect.
- 同步调用 effect.destroy().

##### commitLayoutEffects

第三阶段: dom 变更后

```javascript
function commitLayoutEffects(root: FiberRoot, committedLanes: Lanes) {
  while (nextEffect !== null) {
    const flags = nextEffect.flags
    if (flags & (Update | Callback)) {
      // useEffect,useLayoutEffect都会设置Update标记
      const current = nextEffect.alternate
      commitLayoutEffectOnFiber(root, current, nextEffect, committedLanes)
    }
    nextEffect = nextEffect.nextEffect
  }
}

function commitLifeCycles(finishedRoot: FiberRoot, current: Fiber | null, finishedWork: Fiber, committedLanes: Lanes): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
    case Block: {
      // 在此之前commitMutationEffects函数中, effect.destroy已经被调用, 所以effect.destroy永远不会影响到effect.create
      commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork)

      schedulePassiveEffects(finishedWork)
      return
    }
  }
}

function commitHookEffectListMount(tag: number, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any)
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next
    let effect = firstEffect
    do {
      if ((effect.tag & tag) === tag) {
        const create = effect.create
        effect.destroy = create()
      }
      effect = effect.next
    } while (effect !== firstEffect)
  }
}
```

1.调用关系: commitLayoutEffects->commitLayoutEffectOnFiber(commitLifeCycles)->commitHookEffectListMount.

- 注意在调用 commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork)时, 参数是 HookLayout | HookHasEffect,所以只处理由 useLayoutEffect()创建的 effect.
- 调用 effect.create()之后, 将返回值赋值到 effect.destroy.

  2.为 flushPassiveEffects 做准备

- commitLifeCycles 中的 schedulePassiveEffects(finishedWork), 其形参 finishedWork 实际上指代当前正在被遍历的有副作用的 fiber.
- schedulePassiveEffects 比较简单, 就是把带有 Passive 标记的 effect 筛选出来(由 useEffect 创建), 添加到一个全局数组(pendingPassiveHookEffectsUnmount 和 pendingPassiveHookEffectsMount).

```javascript
function schedulePassiveEffects(finishedWork: Fiber) {
  // 1. 获取 fiber.updateQueue
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any)
  // 2. 获取 effect环形队列
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next
    let effect = firstEffect
    do {
      const { next, tag } = effect
      // 3. 筛选出由useEffect()创建的`effect`
      if ((tag & HookPassive) !== NoHookEffect && (tag & HookHasEffect) !== NoHookEffect) {
        // 把effect添加到全局数组, 等待`flushPassiveEffects`处理
        enqueuePendingPassiveHookEffectUnmount(finishedWork, effect)
        enqueuePendingPassiveHookEffectMount(finishedWork, effect)
      }
      effect = next
    } while (effect !== firstEffect)
  }
}

export function enqueuePendingPassiveHookEffectUnmount(fiber: Fiber, effect: HookEffect): void {
  // unmount effects 数组
  pendingPassiveHookEffectsUnmount.push(effect, fiber)
}

export function enqueuePendingPassiveHookEffectMount(fiber: Fiber, effect: HookEffect): void {
  // mount effects 数组
  pendingPassiveHookEffectsMount.push(effect, fiber)
}
```

综上 commitMutationEffects 和 commitLayoutEffects 2 个函数, 带有 Layout 标记的 effect(由 useLayoutEffect 创建), 已经得到了完整的回调处理(destroy 和 create 已经被调用).

##### flushPassiveEffects

在上文 commitBeforeMutationEffects 阶段, 异步调用了 flushPassiveEffects. 在这期间带有 Passive 标记的 effect 已经被添加到 pendingPassiveHookEffectsUnmount 和 pendingPassiveHookEffectsMount 全局数组中.

接下来 flushPassiveEffects 就可以脱离 fiber 节点, 直接访问 effects

```javascript
export function flushPassiveEffects(): boolean {
  // Returns whether passive effects were flushed.
  if (pendingPassiveEffectsRenderPriority !== NoSchedulerPriority) {
    const priorityLevel = pendingPassiveEffectsRenderPriority > NormalSchedulerPriority ? NormalSchedulerPriority : pendingPassiveEffectsRenderPriority
    pendingPassiveEffectsRenderPriority = NoSchedulerPriority
    // `runWithPriority`设置Schedule中的调度优先级, 如果在flushPassiveEffectsImpl中处理effect时又发起了新的更新, 那么新的update.lane将会受到这个priorityLevel影响.
    return runWithPriority(priorityLevel, flushPassiveEffectsImpl)
  }
  return false
}

function flushPassiveEffectsImpl() {
  if (rootWithPendingPassiveEffects === null) {
    return false
  }
  rootWithPendingPassiveEffects = null
  pendingPassiveEffectsLanes = NoLanes

  // 1. 执行 effect.destroy()
  const unmountEffects = pendingPassiveHookEffectsUnmount
  pendingPassiveHookEffectsUnmount = []
  for (let i = 0; i < unmountEffects.length; i += 2) {
    const effect = ((unmountEffects[i]: any): HookEffect)
    const fiber = ((unmountEffects[i + 1]: any): Fiber)
    const destroy = effect.destroy
    effect.destroy = undefined
    if (typeof destroy === 'function') {
      destroy()
    }
  }

  // 2. 执行新 effect.create(), 重新赋值到 effect.destroy
  const mountEffects = pendingPassiveHookEffectsMount
  pendingPassiveHookEffectsMount = []
  for (let i = 0; i < mountEffects.length; i += 2) {
    const effect = ((mountEffects[i]: any): HookEffect)
    const fiber = ((mountEffects[i + 1]: any): Fiber)
    effect.destroy = create()
  }
}
```

其核心逻辑:

1. 遍历 pendingPassiveHookEffectsUnmount 中的所有 effect, 调用 effect.destroy().

- 同时清空 pendingPassiveHookEffectsUnmount

2. 遍历 pendingPassiveHookEffectsMount 中的所有 effect, 调用 effect.create(), 并更新 effect.destroy.

- 同时清空 pendingPassiveHookEffectsMount

所以, 带有 Passive 标记的 effect, 在 flushPassiveEffects 函数中得到了完整的回调处理.

#### 更新 Hook

假设在初次调用之后, 发起更新, 会再次执行 function, 这时 function 只使用的 useEffect, useLayoutEffect 等 api 也会再次执行.

在更新过程中 useEffect 对应源码 updateEffect, useLayoutEffect 对应源码 updateLayoutEffect.它们内部都会调用 updateEffectImpl, 与初次创建时一样, 只是参数不同.

##### 更新 Effect

```javascript
function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
  // 1. 获取当前hook
  const hook = updateWorkInProgressHook()
  const nextDeps = deps === undefined ? null : deps
  let destroy = undefined
  // 2. 分析依赖
  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState
    // 获取上一个 effect.destroy
    destroy = prevEffect.destroy
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps
      // 比较依赖是否变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 2.1 如果依赖不变, 新建effect(tag不含HookHasEffect)
        pushEffect(hookFlags, create, destroy, nextDeps)
        return
      }
    }
  }
  // 2.2 如果依赖改变, 更改fiber.flag, 新建effect
  currentlyRenderingFiber.flags |= fiberFlags

  hook.memoizedState = pushEffect(HookHasEffect | hookFlags, create, destroy, nextDeps)
}
```

updateEffectImpl 与 mountEffectImpl 逻辑有所不同: - 如果 useEffect/useLayoutEffect 的依赖不变, 新建的 effect 对象不带 HasEffect 标记.

##### 处理 Effect 回调

新的 hook 以及新的 effect 创建完成之后, 余下逻辑与初次渲染完全一致. 处理 Effect 回调时也会根据 effect.tag 进行判断: 只有 effect.tag 包含 HookHasEffect 时才会调用 effect.destroy 和 effect.create()

#### 组件销毁

当 function 组件被销毁时, fiber 节点必然会被打上 Deletion 标记, 即 fiber.flags |= Deletion. 带有 Deletion 标记的 fiber 在 commitMutationEffects 被处理:

```javascript
function commitMutationEffects(root: FiberRoot, renderPriorityLevel: ReactPriorityLevel) {
  while (nextEffect !== null) {
    const primaryFlags = flags & (Placement | Update | Deletion | Hydrating)
    switch (primaryFlags) {
      case Deletion: {
        commitDeletion(root, nextEffect, renderPriorityLevel)
        break
      }
    }
  }
}
```

在 commitDeletion 函数之后, 继续调用 unmountHostComponents->commitUnmount, 在 commitUnmount 中, 执行 effect.destroy(), 结束整个闭环.

### useRef

与其他 Hook 一样，对于 mount 与 update，useRef 对应两个不同 dispatcher。

```javascript
function mountRef<T>(initialValue: T): {| current: T |} {
  // 获取当前useRef hook
  const hook = mountWorkInProgressHook()
  // 创建ref
  const ref = { current: initialValue }
  hook.memoizedState = ref
  return ref
}

function updateRef<T>(initialValue: T): {| current: T |} {
  // 获取当前useRef hook
  const hook = updateWorkInProgressHook()
  // 返回保存的数据
  return hook.memoizedState
}
```

可见，useRef 仅仅是返回一个包含 current 属性的对象

为了验证这个观点，我们再看下 React.createRef 方法的实现：

```javascript
export function createRef(): RefObject {
  // 每次都创建一个新的对象
  const refObject = {
    current: null
  }
  return refObject
}
```

我们知道 HostComponent 在 commit 阶段的 mutation 阶段执行 DOM 操作, 所以，对应 ref 的更新也是发生在 mutation 阶段,
再进一步，mutation 阶段执行 DOM 操作的依据为 effectTag.所以，对于 HostComponent、ClassComponent 如果包含 ref 操作，那么也会赋值相应的 effectTag.

```javascript
export const Placement = /*                    */ 0b0000000000000010
export const Update = /*                       */ 0b0000000000000100
export const Deletion = /*                     */ 0b0000000000001000
export const Ref = /*                          */ 0b0000000010000000
```

所以，ref 的工作流程可以分为两部分：

- render 阶段为含有 ref 属性的 fiber 添加 Ref effectTag
- commit 阶段为包含 Ref effectTag 的 fiber 执行对应操作

#### render 阶段

在 render 阶段的 beginWork 与 completeWork 中有个同名方法 markRef 用于为含有 ref 属性的 fiber 增加 Ref effectTag.

```javascript
// beginWork 的 markRef(mount)
function markRef(current: Fiber | null, workInProgress: Fiber) {
  const ref = workInProgress.ref
  // current ref 不存在或者和本次不同
  if ((current === null && ref !== null) || (current !== null && current.ref !== ref)) {
    // Schedule a Ref effect(update)
    workInProgress.effectTag |= Ref
  }
}
// completeWork 的 markRef
function markRef$1(workInProgress: Fiber) {
  workInProgress.effectTag |= Ref
}
```

组件对应 fiber 被赋值 Ref effectTag 需要满足的条件：

- fiber 类型为 HostComponent、ClassComponent、ScopeComponent
- 对于 mount，workInProgress.ref !== null，即存在 ref 属性
- 对于 update，current.ref !== workInProgress.ref，即 ref 属性改变

#### commit 阶段

在 commit 阶段的 mutation 阶段中，对于 ref 属性改变的情况，需要先移除之前的 ref

```javascript
function commitMutationEffects(root: FiberRoot, renderPriorityLevel) {
  while (nextEffect !== null) {
    const effectTag = nextEffect.effectTag
    // ...

    if (effectTag & Ref) {
      const current = nextEffect.alternate
      if (current !== null) {
        // 移除之前的ref(mount 阶段不会触发)
        commitDetachRef(current)
      }
    }
    // ...
  }
  // ...
}

function commitDetachRef(current: Fiber) {
  const currentRef = current.ref
  if (currentRef !== null) {
    if (typeof currentRef === 'function') {
      // function类型ref，调用他，传参为null
      currentRef(null)
    } else {
      // 对象类型ref，current赋值为null
      currentRef.current = null
    }
  }
}
```

接下来，在 mutation 阶段，对于 Deletion effectTag 的 fiber（对应需要删除的 DOM 节点），需要递归他的子树，对子孙 fiber 的 ref 执行类似 commitDetachRef 的操作.

对于 Deletion effectTag 的 fiber，会执行 commitDeletion:
在 commitDeletion——unmountHostComponents——commitUnmount——ClassComponent | HostComponent 类型 case 中调用的 safelyDetachRef 方法负责执行类似 commitDetachRef 的操作.

```javascript
function safelyDetachRef(current: Fiber) {
  const ref = current.ref
  if (ref !== null) {
    if (typeof ref === 'function') {
      try {
        ref(null)
      } catch (refError) {
        captureCommitPhaseError(current, refError)
      }
    } else {
      ref.current = null
    }
  }
}
```

接下来进入 ref 的赋值阶段, commitLayoutEffect 会执行 commitAttachRef（赋值 ref）:

```javascript
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref
  if (ref !== null) {
    // 获取ref属性对应的Component实例
    const instance = finishedWork.stateNode
    let instanceToUse
    switch (finishedWork.tag) {
      case HostComponent:
        instanceToUse = getPublicInstance(instance)
        break
      default:
        instanceToUse = instance
    }

    // 赋值ref
    if (typeof ref === 'function') {
      ref(instanceToUse)
    } else {
      ref.current = instanceToUse
    }
  }
}
```

#### useMemo/useCallback

mount

```javascript
function mountMemo<T>(nextCreate: () => T, deps: Array<mixed> | void | null): T {
  // 创建并返回当前hook
  const hook = mountWorkInProgressHook()
  const nextDeps = deps === undefined ? null : deps
  // 计算value
  const nextValue = nextCreate()
  // 将value与deps保存在hook.memoizedState
  hook.memoizedState = [nextValue, nextDeps]
  return nextValue
}

function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  // 创建并返回当前hook
  const hook = mountWorkInProgressHook()
  const nextDeps = deps === undefined ? null : deps
  // 将value与deps保存在hook.memoizedState
  hook.memoizedState = [callback, nextDeps]
  return callback
}
```

可以看到，与 mountCallback 这两个唯一的区别是

- mountMemo 会将回调函数(nextCreate)的执行结果作为 value 保存
- mountCallback 会将回调函数作为 value 保存

update

```javascript
function updateMemo<T>(nextCreate: () => T, deps: Array<mixed> | void | null): T {
  // 返回当前hook
  const hook = updateWorkInProgressHook()
  const nextDeps = deps === undefined ? null : deps
  const prevState = hook.memoizedState

  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1]
      // 判断update前后value是否变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 未变化
        return prevState[0]
      }
    }
  }
  // 变化，重新计算value
  const nextValue = nextCreate()
  hook.memoizedState = [nextValue, nextDeps]
  return nextValue
}

function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  // 返回当前hook
  const hook = updateWorkInProgressHook()
  const nextDeps = deps === undefined ? null : deps
  const prevState = hook.memoizedState

  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1]
      // 判断update前后value是否变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 未变化
        return prevState[0]
      }
    }
  }

  // 变化，将新的callback作为value
  hook.memoizedState = [callback, nextDeps]
  return callback
}
```

可见，对于 update，这两个 hook 的唯一区别也是是回调函数本身还是回调函数的执行结果作为 value。
