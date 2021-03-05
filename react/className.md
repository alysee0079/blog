# 为什么 React 要使用 className 来代替 HTML 中的 class？

---
1. React 一开始的理念是想与浏览器的 DOM API 保持一直而不是 HTML，因为这样会和元素的创建更为接近。
2. ES5 之前，在对象中不能使用保留字，以下代码在 IE8 中将会抛出错误：
    ```
    const element = {
      attributes: {
        class: "hello"
      }
    } 
    ```
    在现代浏览器环境中，如果尝试为其分配一个变量时解构时就会抛出错误：
    ```
    const { class } = { class: 'foo' } // Uncaught SyntaxError: Unexpected token }
    ```
