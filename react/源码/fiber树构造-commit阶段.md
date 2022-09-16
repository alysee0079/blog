# fiber树构造 commit 阶段

#### commitRoot
```javascript
function commitRoot(root) {
  const renderPriorityLevel = getCurrentPriorityLevel();
  runWithPriority(
    ImmediateSchedulerPriority,
    commitRootImpl.bind(null, root, renderPriorityLevel),
  );
  return null;
}
```

在commitRoot中同时使用到了渲染优先级和调度优先级, 有关优先级的讨论, 在前文已经做出了说明(参考React 中的优先级管理和fiber 树构造(基础准备)#优先级). 最后的实现是通过commitRootImpl函数:
```javascript
function commitRootImpl(root, renderPriorityLevel) {
  // ============ 渲染前: 准备 ============

  const finishedWork = root.finishedWork;
  const lanes = root.finishedLanes;

  // 清空FiberRoot对象上的属性
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  root.callbackNode = null;

  if (root === workInProgressRoot) {
    // 重置全局变量
    workInProgressRoot = null;
    workInProgress = null;
    workInProgressRootRenderLanes = NoLanes;
  }

  // 再次更新副作用队列
  let firstEffect;
  if (finishedWork.flags > PerformedWork) {
    // 默认情况下fiber节点的副作用队列是不包括自身的
    // 如果根节点有副作用, 则将根节点添加到副作用队列的末尾
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork;
      firstEffect = finishedWork.firstEffect;
    } else {
      firstEffect = finishedWork;
    }
  } else {
    firstEffect = finishedWork.firstEffect;
  }

  // ============ 渲染 ============
  let firstEffect = finishedWork.firstEffect;
  if (firstEffect !== null) {
    const prevExecutionContext = executionContext;
    executionContext |= CommitContext;
    // 阶段1: dom突变之前
    nextEffect = firstEffect;
    do {
      commitBeforeMutationEffects();
    } while (nextEffect !== null);

    // 阶段2: dom突变, 界面发生改变
    nextEffect = firstEffect;
    do {
      commitMutationEffects(root, renderPriorityLevel);
    } while (nextEffect !== null);
    // 恢复界面状态
    resetAfterCommit(root.containerInfo);
    // 切换current指针
    root.current = finishedWork;

    // 阶段3: layout阶段, 调用生命周期componentDidUpdate和回调函数等
    nextEffect = firstEffect;
    do {
      commitLayoutEffects(root, lanes);
    } while (nextEffect !== null);
    nextEffect = null;
    executionContext = prevExecutionContext;
  }

  // ============ 渲染后: 重置与清理 ============
  if (rootDoesHavePassiveEffects) {
    // 有被动作用(使用useEffect), 保存一些全局变量
  } else {
    // 分解副作用队列链表, 辅助垃圾回收
    // 如果有被动作用(使用useEffect), 会把分解操作放在flushPassiveEffects函数中
    nextEffect = firstEffect;
    while (nextEffect !== null) {
      const nextNextEffect = nextEffect.nextEffect;
      nextEffect.nextEffect = null;
      if (nextEffect.flags & Deletion) {
        detachFiberAfterEffects(nextEffect);
      }
      nextEffect = nextNextEffect;
    }
  }
  // 重置一些全局变量(省略这部分代码)...
  // 下面代码用于检测是否有新的更新任务
  // 比如在componentDidMount函数中, 再次调用setState()

  // 1. 检测常规(异步)任务, 如果有则会发起异步调度(调度中心`scheduler`只能异步调用)
  ensureRootIsScheduled(root, now());
  // 2. 检测同步任务, 如果有则主动调用flushSyncCallbackQueue(无需再次等待scheduler调度), 再次进入fiber树构造循环
  flushSyncCallbackQueue();

  return null;
}
```

#### 渲染前

为接下来正式渲染, 做一些准备工作. 主要包括:

1.设置全局状态(如: 更新fiberRoot上的属性)

2.重置全局变量(如: workInProgressRoot, workInProgress等)

3.再次更新副作用队列: 只针对根节点fiberRoot.finishedWork
    - 默认情况下根节点的副作用队列是不包括自身的, 如果根节点有副作用, 则将根节点添加到副作用队列的末尾
    - 注意只是延长了副作用队列, 但是fiberRoot.lastEffect指针并没有改变. 比如首次构造时, 根节点拥有Snapshot标记:

![effectList](../../images/36.png)


#### 渲染中

> commitRootImpl函数中, 渲染阶段的主要逻辑是处理副作用队列, 将最新的 DOM 节点(已经在内存中, 只是还没渲染)渲染到界面上.

整个渲染过程被分为 3 个函数分布实现:

1.commitBeforeMutationEffects
    - dom 变更之前, 处理副作用队列中带有Snapshot,Passive标记的fiber节点.

2.commitMutationEffects
    - dom 变更, 界面得到更新. 处理副作用队列中带有Placement, Update, Deletion, Hydrating标记的fiber节点.

3.commitLayoutEffects
    - dom 变更后, 处理副作用队列中带有Update | Callback标记的fiber节点.



通过上述源码分析, 可以把commitRootImpl的职责概括为 2 个方面:

1.处理副作用队列. (步骤 1,2,3 都会处理, 只是处理节点的标识fiber.flags不同).

2.调用渲染器, 输出最终结果. (在步骤 2: commitMutationEffects中执行).

所以commitRootImpl是处理fiberRoot.finishedWork这棵即将被渲染的fiber树, 理论上无需关心这棵fiber树是如何产生的(可以是首次构造产生, 也可以是对比更新产生). 为了清晰简便, 在下文的所有图示都使用初次创建的fiber树结构来进行演示.


这 3 个函数处理的对象是副作用队列和DOM对象.

所以无论fiber树结构有多么复杂, 到了commitRoot阶段, 实际起作用的只有 2 个节点:
    - 副作用队列所在节点: 根节点, 即HostRootFiber节点.
    - DOM对象所在节点: 从上至下首个HostComponent类型的fiber节点, 此节点 fiber.stateNode实际上指向最新的 DOM 树.


下图为了清晰, 省略了一些无关引用, 只留下commitRoot阶段实际会用到的fiber节点:

![effectList](../../images/37.png)


#### commitBeforeMutationEffects

第一阶段: dom 变更之前, 处理副作用队列中带有 Snapshot, Passive 标记的 fiber 节点.
```javascript
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    const current = nextEffect.alternate;
    const flags = nextEffect.flags;
    // 处理`Snapshot`标记
    if ((flags & Snapshot) !== NoFlags) {
      commitBeforeMutationEffectOnFiber(current, nextEffect);
    }
    // 处理`Passive`标记
    if ((flags & Passive) !== NoFlags) {
      // Passive标记只在使用了hook, useEffect会出现. 所以此处是针对hook对象的处理
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






