# 为什么要是用 setState 更新数据?

1. 直接修改 state 无法重新渲染组件，因为 react 无法知道数据被修改了，react 不是使用 Object.defineproperty 实现数据绑定。
2. setState 是异步更新，设计为异步可以显著提升性能，通过批量更新，完成多次数据更新；同时可以避免 state 和 props 数据不同步导致出现问题。
3. setState 同步更新方式：
   将 setState 放到定时器中,

   ```
   setTimeout(() => {
     this.setState()
     console.log(this.state)
   }, 0)
   ```

   使用原生 DOM 事件触发 setState,

   ```
   document.addEventListener('click', () => {
    this.setState()
    console.log(this.state)
   })
   ```

   总结：

   1). 在组件生命周期或者 React 合成事件中，setState 是异步的；
   2). 在 setTimeOut 或者 原生 DOM 事件中，是同步的。

4. setState 对数据的合并：使用 Object.assign({}, state, newState) 组成新的对象。
5. setState 本身的合并：多个 setState 会合并为一个，如果 setState 参数是对象时，会对对象进行合并，如果是函数则会执行，并合并为最新的 state。
