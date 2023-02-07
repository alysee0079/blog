# this

#### 绑定规则

1. 默认绑定
   - 独立函数绑定, 非严格模式: window, 严格模式: undefined
2. 隐式绑定
   - 通过某个对象发起函数调用, 指向发起的对象
3. 显式绑定
   - 用过 call, apply, bind 调用, 指向传递的参数
4. new 绑定
   - 在 new 构造函数时, this 指向创建的实例对象

#### 规则优先级

1. 默认规则优先级最低
2. 显示绑定优先级高于隐式绑定
3. new 绑定高于隐式绑定, new 绑定优先级高于 bind

#### 特殊情况

1. call, apply, bind 传递 null, undefined 时, this 指向 window
2. es6 箭头函数
   - 箭头函数不绑定 this, arguments
   - 箭头不能作为构造函数使用(不能和 new 使用)
   - 箭头函数的 this 根据外层作用域决定(动态)
