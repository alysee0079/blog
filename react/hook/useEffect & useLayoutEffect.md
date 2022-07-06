# useEffect & useLayoutEffect

#### 副作用
1. 一个函数在执行过程中产生了外部可观察的变化，则这个函数是有副作用(Side Effect)的;
2. 通俗点就是函数内部做了和运算返回值无关的事，比如修改外部作用域/全局变量、修改传入的参数、发送请求、console.log、手动修改 DOM 都属于副作用;

#### 函数作为 useEffect 依赖的情况
1.通常在 effect 内部去声明它所需要的函数, 这样就能容易的看出那个 effect 依赖了组件作用域中的哪些值;
2.只有当函数（以及它所调用的函数）不引用 props、state 以及由它们衍生而来的值时，你才能放心地把它们从依赖列表中省略;
3.如果一个函数没有使用组件内的任何值，你应该把它提到组件外面去定义，然后就可以自由地在effects中使用;
4.万不得已的情况下，你可以把函数加入 effect 的依赖但把它的定义包裹进 useCallback Hook。这就确保了它不随渲染而改变，除非它自身的依赖发生了改变;
5.只有变化时，需要重新执行 useEffect 的变量，才要放到 deps 中。而不是 useEffect 用到的变量都放到 deps 中;

#### useEffect effect 执行时机
1.传给 useEffect 的函数会在浏览器完成布局与绘制之后(渲染后异步执行)，在一个延迟事件中被调用;
2.绝大多数操作不应阻塞浏览器对屏幕的更新;
3.即使在 useEffect 被推迟到浏览器绘制之后的情况下，它也能保证在任何新的渲染前启动。React 在开始新的更新前，总会先刷新之前的渲染的 effect;

#### useEffect 清除 effect 时机
1.在上一次的effect会在重新渲染后, 下一次的 effect 执行之前;
2.effect的清除并不会读取“最新”的props。它只能读取到定义它的那次渲染中的props值;

#### useLayoutEffect effect 执行时机
1.useLayoutEffect 与 componentDidMount、componentDidUpdate 的调用阶段是一样的(同步执行);
2.在浏览器把内容真正渲染到界面之前执行;
3.useLayoutEffect 会阻塞浏览器渲染;
4.将修改 DOM 的操作放到 useLayoutEffect里, 可以减少回流、重绘;

#### effect 是如何读取到最新的count 状态值？
1.React会记住你提供的 effect 函数，并且会在每次更改作用于DOM并让浏览器绘制屏幕后去调用它;
2.所以虽然我们说的是一个 effect（这里指更新document的title），但其实每次渲染都是一个不同的函数 — 并且每个effect函数“看到”的props和state都来自于它属于的那次特定渲染;
3.组件内的每一个函数（包括事件处理函数，effects，定时器或者API调用等等）会捕获定义它们的那次渲染中的props和state;


