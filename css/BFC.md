# BFC 块格式化上下文

BFC 是一个独立的布局环境，其中的元素布局是不受外界的影响，并且在一个 BFC 中，块盒与行盒（行盒由一行中所有的内联元素所组成）都会垂直的沿着其父元素的边框排列。BFC 就是页面上的一个隔离的独立容器，容器里面的子元素不会影响到外面的元素。反之也如此。
因为 BFC 内部的元素和外部的元素绝对不会互相影响，因此， 当 BFC 外部存在浮动时，它不应该影响 BFC 内部 Box 的布局，BFC 会通过变窄，而不与浮动有重叠。同样的，当 BFC 内部有浮动时，为了不影响外部元素的布局，BFC 计算高度时会包括浮动的高度。避免 margin 重叠也是这样的一个道理。

#### BFC 的布局规则

内部的 Box 会在垂直方向，一个接一个地放置。
Box 垂直方向的距离由 margin 决定。属于同一个 BFC 的两个相邻 Box 的 margin 会发生重叠。
每个盒子（块盒与行盒）的 margin box 的左边，与包含块 border box 的左边相接触(对于从左往右的格式化，否则相反)。即使存在浮动也是如此。
BFC 的区域不会与 float box 重叠。
BFC 就是页面上的一个隔离的独立容器，容器里面的子元素不会影响到外面的元素。反之也如此。
计算 BFC 的高度时，浮动元素也参与计算。

#### 如何触发 BFC

根元素（<html>）
浮动元素（元素的 float 不是 none）
绝对定位元素（元素的 position 为 absolute 或 fixed）
行内块元素（元素的 display 为 inline-block）
表格单元格（元素的 display 为 table-cell，HTML 表格单元格默认为该值）
表格标题（元素的 display 为 table-caption，HTML 表格标题默认为该值）
匿名表格单元格元素（元素的 display 为 table、table-row、 table-row-group、table-header-group、table-footer-group（分别是 HTML table、row、tbody、thead、tfoot 的默认属性）或 inline-table）
overflow 计算值(Computed)不为 visible 的块元素
display 值为 flow-root 的元素
contain 值为 layout、content 或 paint 的元素
弹性元素（display 为 flex 或 inline-flex 元素的直接子元素）
网格元素（display 为 grid 或 inline-grid 元素的直接子元素）
多列容器（元素的 column-count 或 column-width (en-US) 不为 auto，包括 column-count 为 1）
column-span 为 all 的元素始终会创建一个新的 BFC，即使该元素没有包裹在一个多列容器中（标准变更，Chrome bug）

#### 解决问题

1. 同一个 BFC 下外边距会发生折叠
2. BFC 可以包含浮动的元素（清除浮动）
3. BFC 可以阻止元素被浮动元素覆盖
