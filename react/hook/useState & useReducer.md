# useState
一个 state 必须不能通过其它 state/props 直接计算出来，否则就不用定义 state;

#### 为什么顺序调用对 React Hooks 很重要?
1. hook 内部使用链表保存数据, 存取数据都使用索引;
2. state 中永远不要保存可以通过计算得到的值;


# useReducer
1.处理逻辑较复杂且包含多个子值，或者下一个 state 依赖于之前的 state 等.
2.并且，使用 useReducer 还能给那些会触发深更新的组件做性能优化，因为你可以向子组件传递 dispatch 而不是回调函数;