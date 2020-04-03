### BFC
- 什么是BFC
  - 英文（Block formatting context），即'块级格式化上下文'
- BFC的布局规则
  - 内部的Box会在垂直方向，一个接一个地放置
  - BFC的区域不会与floa盒子重叠。
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

### CSS选择器和优先级(权重)
- css有哪些选择器
  - 基础选择器
    `ID 选择器`、`类选择器`、`标签选择器`、`属性选择器`、`状态选择器`、`通配选择器`
  - 关系选择器
    - `邻近兄弟元素选择器 A + B`、`兄弟元素选择器 A ~ B`、`直接子元素选择器 A > B`、`后代元素选择器 A B`
  - 伪元素
  - 伪类
- css优先级(权重)
  1. 行内样式，指的是html文档中定义的style
  2. ID选择器
  3. 类，属性选择器和伪类选择器(伪类-> :before,:after,:active等)
  4. 元素和伪元素(::first-letter,::first-line,::before,::after,::selection,::cue,::slotted()等)
> 权重记忆口诀：一个行内样式+1000，一个id+100，一个类、属性选择器或者伪类+10，一个元素名或者伪元素+1
> 
> 权重相同：书写在后面css的会覆盖前面的；权重不同：权重高的覆盖权重低的；对于样式的`!important`: 拥有最高优先级

### CSS中哪些属性可以继承
- 字体系列属性`font`
- 文本系列属性`text-xxx`
- 元素可见性`visibility`
- 表格布局属性`caption-side、border-collapse、border-spacing、empty-cells、table-layout`
- 列表属性`list-style-type、list-style-image、list-style-position、list-style`
- 生成内容属性`quotes`
- 光标属性`cursor`
- 所有元素可以继承的属性
  - 元素可见性：`visibility、opacity`
  - 光标属性：`cursor`
- 内联元素可以继承的属性
  - 字体系列属性
  - 除`text-indent、text-align`之外的文本系列属性
- 块级元素可以继承的属性:
  - `text-indent、text-align`

### CSS可以做哪些优化工作
1. CSS压缩
2. 减少选择器的层级（即css嵌套，控制在3层以内）
3. 避免使用`ID选择器`
4. 避免使用`通配选择器`
5. 尽量不用`标签选择器`
6. 合理使用继承
7. 避免使用`@import`（会阻塞加载）
8. 少用css表达式
9. 抽取公共样式

### z-index有什么需要注意的地方
- 层叠顺序
  
  `层叠上下文background/border` < `负z-index` < `block状态水平盒子` < `float浮动盒子` < `inline/inline-box的水平盒子` < `z-index:auto或看成z-index:0,不依赖z-index的层叠上下文` < `正z-index`

  - 为什么内联元素的层叠顺序要比浮动元素和块状元素都高
    - 诸如border/background一般为装饰属性，而浮动和块状元素一般用作布局，而内联元素都是内容.因此，一定要让内容的层叠顺序相当高。
  - 层叠准则
    - 谁大谁上：当具有明显的层叠水平标示的时候，如识别的z-indx值，在同一个层叠上下文领域，层叠水平值大的那一个覆盖小的那一个。
    - 后来居上：当元素的层叠水平一致、层叠顺序相同的时候，在DOM流中处于后面的元素会覆盖前面的元素。
  - 层叠上下文的特性
    - 层叠上下文的层叠水平要比普通元素高
    - 层叠上下文可以阻断元素的混合模式
    - 层叠上下文可以嵌套，内部层叠上下文及其所有子元素均受制于外部的层叠上下文
    - 每个层叠上下文和兄弟元素独立，也就是当进行层叠变化或渲染的时候，只需要考虑后代元素
    - 每个层叠上下文是自成体系的，当元素发生层叠的时候，整个元素被认为是在父层叠上下文的层叠顺序中。
  - 层叠上下文的创建
    - position:relative/position:absolute/position:fixed的定位元素，当其z-index值不是auto的时候。
    - z-index值不为auto的flex项(父元素display:flex|inline-flex)
    - 元素的opacity值不是1
    - 元素的transform值不是none
    - 元素mix-blend-mode值不是normal
    - 元素的filter值不是none
    - 元素的isolation值是isolate
    - will-change指定的属性值为上面任意一个
    - 元素的-webkit-overflow-scrolling设为touch

参考[深入理解CSS中的层叠上下文和层叠顺序](https://www.zhangxinxu.com/wordpress/2016/01/understand-css-stacking-context-order-z-index/)

### 居中
- 水平居中
  - 行内元素在块级父元素内：text-align: center
  - 块级元素在块级父元素内：
    - 多个块级元素：display: inline-block; 父元素设置：text-align: center，或justify-content: center;
    - 单个块级元素：margin: 0 auto; 或justify-content: center;
- 垂直居中
  - 行内元素：
    - 单行：padding-bottom和padding-top相等、line-height居中、flex
    - 多行：padding-bottom和padding-top相等、父元素table-cell，后使用vertical-align居中、flex
    - 以上适用父级容器拥有确定高度的元素，如果上述方法都不起作用则：在垂直居中的元素上添加伪元素，设置伪元素的高等于父级容器的高，然后为文本添加 vertical-align: middle;
  - 块元素：flex、设置高度和父元素一致、position: absolute和margin负值、position: absolute和transform:translateY(-50%)
- 水平垂直居中
  - flex
  - position: absolute和margin负值
  - position: absolute和transform:translate(-50%, -50%)（如果会模糊则父级元素设置transform-style: preserve-3d;）
### iPhoneX适配
- `safa area`内：`viewport-fit:contain`
- viewport-fit=cover之后，Web中会新增四个常量
  - `viewport-fit=cover` + `padding: constant(safe-area-inset-top) constant(safe-area-inset-right) constant(safe-area-inset-bottom) constant(safe-area-inset-left);`
- 媒体查询
  ```css
  // iphoneX
  @media only screen and (device-width: 375px) and (device-height: 812px) and (-webkit-device-pixel-ratio: 3){
  }
  //iphone Xs Max
  @media only screen and (device-width: 414px) and (device-height: 896px) and (-webkit-device-pixel-ratio:3) {
  }
  //iphone XR
  @media only screen and (device-width: 414px) and (device-height: 896px) and (-webkit-device-pixel-ratio:2) {
  }
  ```
### link和@import
- 两种方式都是可以加载css文件，但link还可以用于加载其他文件，而不仅仅是css文件
- link引用的CSS会同时被加载，而@import引用的CSS会等到页面全部被下载完再加载（link可以并行加载,@import不行）。会引发闪烁问题
- @import需要使用在IE5以上的浏览器（现在基本也看做没兼容问题了），link无兼容性问题
- link可是用js控制dom添加，@import不行
- link方式的样式权重高于@import的权重

> css不会注册DOM树的解析，但会阻塞DOM树的渲染
> css加载会阻塞后面js语句的执行

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
  - 命名规范
    1. BEM
      - 什么是BEM：BEM是css的一种命名规范；即：块（block）、元素（element）、修饰符（modifier）
      - 命名约定的模式如下：
        ```css
        .block{}
        .block__element{}
        .block--modifier{}
        ```
        - .block 代表了更高级别的抽象或组件。
        - .block__element 代表.block的后代，.block里的元素，用于形成一个完整的.block的整体
        - .block--modifier代表.block的不同状态或不同版本
    2. OOCSS：面向对象的css
       - 原则1：独立的结构和样式；原则2：独立的容器和内容
       - 创建OOCSS
         - 创建一个组件库
         - 独立的容器和内容，并且避免样式来依赖位置
         - 独立的结构和样式
         - 使用类名为扩展基本对象
         - 坚持以语义类来命名类名
       - OOCSS缺点
         - 样式(CSS)和结构(HTML)藕合太紧
         - 创建了数千行CSS，但有可能这些CSS永远不会被使用
         - OOCSS适合真正的大型网站开发，小项目不易看见成效
         - 需巧妙运用，运用不好会使CSS更难维护
         - 需要写说明文档，便于维护
       - OOCSS优点
         - 减少CSS代码
         - 具有清洁的HTML标记，有语义的类名，逻辑性强的层次关系
         - 语义标记，有助于SEO
         - 更好的页面优化，更快的加载时间（因为有很多组件重用）
         - 可扩展的标记和CSS样式，有更多的组件可以放到库中，而不影响其他的组件著作权归作者所有。
         - 能轻松构造新的页面布局，或制作新的页面风格
        
        参考：[OOCSS——概念篇](https://www.w3cplus.com/css/oocss-concept)
    3. SMACSS：[smacss.com/](http://smacss.com/)
       - SMACSS定义了五种样式类型
         - Base (基本)
         - Layout（布局）
         - Module (模块)
         - State (状态) 
         - Theme (皮肤)
    4. AMCSS：AMCSS是Attribute Modules for CSS的缩写，表示借助HTML属性来进行CSS相关开发
       - 属性值得灵活性，相当于通过属性值来增加命名空间，减少全局名称空间，以便更好的将css模块化 
  - CSS in js: 用 JavaScript 在写 CSS
    - 解决的一些痛点
      - 全局污染
      - 命名混乱
      - 难以复用
      - 维护性差
      - 冗余代码
      - 兼容问题
      - 组件化，方便开发维护和测试
      - JavaScript 和 CSS 可以方便的共享变量和方法
    - 不足之处
      - 没有统一的业界标准
      - 代码可读性差 
      - 学习成本高
      - 运行时消耗更高
    - 热门库：[styled-components](https://github.com/styled-components/styled-components)
  - CSS Modules：所有的类名和动画名称默认都有各自的作用域的CSS文件。
    - CSS Modules不是一个官方的规范，也不是浏览器的一种机制，它是一种构建步骤中对CSS类名和选择器限定作用域的一种方式。
    - CSS Modules集中在同一个地方，只应用于该组件
    - 能结局传统css的很多问题，一些css的书写规范不再是必须的了，当然懂得要更多，学习成本也就更高
    - 和webpack、react配合使用更加；vue-cli3.0也支持。
