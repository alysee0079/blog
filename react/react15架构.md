# react15 架构

#### Reconciler 协调器

在 React 中可以通过 this.setState、this.forceUpdate、ReactDOM.render 等 API 触发更新。
每当有更新发生时，Reconciler 会做如下工作：

1. 调用函数组件、或 class 组件的 render 方法，将返回的 JSX 转化为虚拟 DOM
2. 将虚拟 DOM 和上次更新时的虚拟 DOM 对比
3. 通过对比找出本次更新中变化的虚拟 DOM
4. 通知 Renderer 将变化的虚拟 DOM 渲染到页面上

#### Renderer（渲染器）

在每次更新发生时，Renderer 接到 Reconciler 通知，将变化的组件渲染在当前宿主环境。

#### React15 架构的缺点

在 Reconciler 中，mount 的组件会调用 mountComponent (opens new window)，update 的组件会调用 updateComponent (opens new window)。这两个方法都会递归更新子组件。

##### 递归更新的缺点

由于递归执行，所以更新一旦开始，中途就无法中断。当层级很深时，递归更新时间超过了 16ms，用户交互就会卡顿。
