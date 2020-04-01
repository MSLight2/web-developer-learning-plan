### BFC
- 什么是BFC
  - 英文（Block formatting context），即'块级格式化上下文'
- BFC的布局规则
  - 内部的Box会在垂直方向，一个接一个地放置
  - BFC的区域不会与float box重叠。
  - 计算BFC的高度时，浮动元素也参与计算。
  - Box垂直方向的距离由margin决定。属于同一个BFC的两个相邻Box的margin会发生重叠。
  - BFC就是页面上的一个隔离的独立容器，容器里面的子元素不会影响到外面的元素。反之也如此。
  - 当容器有足够的剩余空间容纳 BFC 的宽度时，所有浏览器都会将 BFC 放置在浮动元素所在行的剩余空间内。 
  - 每个盒子（块盒与行盒）的margin box的左边，与包含块border box的左边相接触(对于从左往右的格式化，否则相反)。即使存在浮动也是如此。
- 如何创建BFC
  - float的值不是none
  - overflow的值不是visible
  - position的值不是static或relative
  - display的值是`inline-block`、`table-cell`、`table-caption`、`table、table-row`、 `table-row-group`、`table-header-group`、`table-footer-group`、`inline-table`、`flow-root(这个属性可以创建无副作用的BFC)`、`flex`、`inline-flex`、`grid` 或 `inline-grid`
  - column-count 或 column-width 不为 auto，包括 column-count 为 1
  - column-span 为 all 的元素始终会创建一个新的BFC，即使该元素没有包裹在一个多列容器中
- BFC的作用
  - 不和浮动元素重叠（自适用两列布局）
  - 清除元素内部浮动（即内部元素浮动，父元素高度坍塌问题）
  - 利用BFC避免margin重叠
### CSS选择器优先级
### CSS中选择器有哪些
### CSS中哪些属性可以继承
### CSS可以做哪些优化工作
### z-index有什么需要注意的地方
参考[深入理解CSS中的层叠上下文和层叠顺序](https://www.zhangxinxu.com/wordpress/2016/01/understand-css-stacking-context-order-z-index/)
### iPhoneX适配
### link和@import
### 居中
### css实现两栏布局（圣杯）原理是什么；css实现田字布局，水平居中
- 圣杯布局
  - 圣杯布局就是就是左右两边大小固定不变，中间宽度自适应。
  - 实现方式有四种
    1. float实现
    2. 绝对定位实现
    3. flex布局(justify-content: space-between)
    4. grid布局
- 田字布局
  - 一个大的容器（div）包含四个小的容器（div）
  - 实现方式
    1. float布局
    2. 绝对定位
    3. flex布局(flex-warp: warp)
    4. grid布局
### CSS规范
  - BEM
  - CSS in js
