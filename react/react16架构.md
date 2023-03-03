# react16 架构

React16 架构可以分为三层：

1. Scheduler（调度器）—— 调度任务的优先级，高优任务优先进入 Reconciler
2. Reconciler（协调器）—— 负责找出变化的组件
3. Renderer（渲染器）—— 负责将变化的组件渲染到页面上

#### Reconciler（协调器）

更新工作从递归变成了可以中断的循环过程。每次循环都会调用 shouldYield 判断当前是否有剩余时间。

```javascript
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}
```

在 React16 中，Reconciler 与 Renderer 不再是交替工作。当 Scheduler 将任务交给 Reconciler 后，Reconciler 会为变化的虚拟 DOM 打上代表增/删/更新的标记，类似这样：

```javascript
export const Placement = /*             */ 0b0000000000010;
export const Update = /*                */ 0b0000000000100;
export const PlacementAndUpdate = /*    */ 0b0000000000110;
export const Deletion = /*              */ 0b0000000001000;
```

整个 Scheduler 与 Reconciler 的工作都在内存中进行。只有当所有组件都完成 Reconciler 的工作，才会统一交给 Renderer。

1. 接受输入(scheduleUpdateOnFiber), 将 fiber 树生成逻辑封装到一个回调函数中(涉及 fiber 树形结构, fiber.updateQueue 队列, 调和算法等),
2. 把此回调函数(performSyncWorkOnRoot 或 performConcurrentWorkOnRoot)送入 scheduler 进行调度
3. scheduler 会控制回调函数执行的时机, 回调函数执行完成后得到全新的 fiber 树
4. 再调用渲染器(如 react-dom, react-native 等)将 fiber 树形结构最终反映到界面上

#### Scheduler（调度器）

既然我们以浏览器是否有剩余时间作为任务中断的标准，那么我们需要一种机制，当浏览器有剩余时间时通知我们。

其实部分浏览器已经实现了这个 API，这就是 requestIdleCallback (opens new window)。但是由于以下因素，React 放弃使用：

- 浏览器兼容性
- 触发频率不稳定，受很多因素影响。比如当我们的浏览器切换 tab 后，之前 tab 注册的 requestIdleCallback 触发的频率会变得很低

基于以上原因，React 实现了功能更完备的 requestIdleCallbackpolyfill，这就是 Scheduler。除了在空闲时触发回调的功能外，Scheduler 还提供了多种调度优先级供任务设置。

1. 核心任务就是执行回调(回调函数由 react-reconciler 提供)
2. 通过控制回调函数的执行时机, 来达到任务分片的目的, 实现可中断渲染(concurrent 模式下才有此特性)

#### Renderer（渲染器）

Renderer 根据 Reconciler 为虚拟 DOM 打的标记，同步执行对应的 DOM 操作。
在 React16 架构中整个更新流程为：
![](../../images/41.png)
其中红框中的步骤随时可能由于以下原因被中断：

- 有其他更高优任务需要先更新
- 当前帧没有剩余时间

由于红框中的工作都在内存中进行，不会更新页面上的 DOM，所以即使反复中断，用户也不会看见更新不完全的 DOM（即上一节演示的情况）。
