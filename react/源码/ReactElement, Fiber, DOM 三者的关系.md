# ReactElement, Fiber, DOM 三者的关系

1. ReactElement 对象(type 定义在 shared 包中)

   - 所有采用 jsx 语法书写的节点, 都会被编译器转换, 最终会以 React.createElement(...)的方式, 创建出来一个与之对应的 ReactElement 对象

2. fiber 对象(type 类型的定义在 ReactInternalTypes.js 中)

   - fiber 对象是通过 ReactElement 对象进行创建的, 多个 fiber 对象构成了一棵 fiber 树, fiber 树是构造 DOM 树的数据模型, fiber 树的任何改动, 最后都体现到 DOM 树.

3. DOM 对象: 文档对象模型

   - DOM 将文档解析为一个由节点和对象（包含属性和方法的对象）组成的结构集合, 也就是常说的 DOM 树.
   - JavaScript 可以访问和操作存储在 DOM 中的内容, 也就是操作 DOM 对象, 进而触发 UI 渲染.
