# 为什么要是用 setState 更新数据?

1. 直接修改 state 无法重新渲染组件，因为 react 无法知道数据被修改了，react 不是使用 Object.defineproperty 实现数据绑定。
2. setState 是异步更新，设计为异步可以显著提升性能，通过批量更新，完成多次数据更新；同时可以避免 state 和 props 数据不同步导致出现问题。
