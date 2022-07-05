# useRef

官方介绍: 如果你想组件储存某些数据, 且这些数据的变动不会导致组件重新渲染, 就使用 useRef; 如果某个数据仅在事件处理中使用, 或者更新数据不需要从新渲染组件时, 就使用 useRef;

1. useRef 返回一个可变的 ref 对象，其 .current 属性被初始化为传入的参数（initialValue）。返回的 ref 对象在组件的整个生命周期内保持不变;
2. 本质上，useRef 就像是可以在其 .current 属性中保存一个可变值的“盒子”, 类似于在 class 中使用实例字段的方式;
3. useRef 创建的是一个普通 Javascript 对象。而 useRef() 和自建一个 {current: ...} 对象的唯一区别是，useRef 会在每次渲染时返回同一个 ref 对象; 4.当 ref 对象内容发生变化时，useRef 并不会通知, 变更 .current 属性不会引发组件重新渲染;
