# useState

一个 state 必须不能通过其它 state/props 直接计算出来，否则就不用定义 state;

#### 为什么顺序调用对 React Hooks 很重要?

1. hook 内部使用链表保存数据, 存取数据都使用索引;