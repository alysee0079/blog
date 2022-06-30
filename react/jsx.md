# jsx

JSX是一种JavaScript的语法扩展，其格式比较像是模版语言，但事实上完全是在JavaScript内部实现的;

#### 老版本的 React 中，为什么写 jsx 的文件要默认引入 React?
因为 jsx 在被 babel 编译后，写的 jsx 会变成上述 React.createElement 形式，所以需要引入 React，防止找不到 React 引起报错;