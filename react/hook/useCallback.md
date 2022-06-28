# useCallback

1.useCallback 可以记住函数，避免函数重复生成，这样函数在传递给子组件时，可以避免子组件重复渲染，提高性能;
2.useCallback 是要和 shouldComponentUpdate/React.memo 配套使用;