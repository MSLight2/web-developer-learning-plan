### Vue的初始化工作（Vue原型和全局API）
#### Vue构造函数
```
  function Vue (options) {
    if (process.env.NODE_ENV !== 'production' &&
      !(this instanceof Vue)
    ) {
      warn('Vue is a constructor and should be called with the `new` keyword')
    }
    this._init(options)
  }
```
#### Vue原型的初始化（Vue.prototype.xxx）
- 由initMixin() 添加的原型
  - Vue.prototype._init() function (options?: Object) {}
- 由stateMixin() 添加的原型
  - Vue.prototype.$data 只读，代理this._data
  - Vue.prototype.$data 只读，代理this._props
  - Vue.prototype.$set = set
  - Vue.prototype.$delete = del
  - Vue.prototype.$watch = function(...)
- 由eventsMixin() 添加的原型
  - Vue.prototype.$on = function(...)
  - Vue.prototype.$once = function(...)
  - Vue.prototype.$off = function(...)
  - Vue.prototype.$emit = function(...)
- 由lifecycleMixin() 添加的原型
  - Vue.prototype._update = function(...)
  - Vue.prototype.$forceUpdate = function(...)
  - Vue.prototype.$destroy = function(...)
- 由renderMixin() 添加的原型
  - installRenderHelpers(Vue.prototype)
    - Vue.prototype._o = markOnce
    - Vue.prototype._n = toNumber
    - Vue.prototype._s = toString
    - Vue.prototype._l = renderList
    - Vue.prototype._t = renderSlot
    - Vue.prototype._q = looseEqual
    - Vue.prototype._i = looseIndexOf
    - Vue.prototype._m = renderStatic
    - Vue.prototype._f = resolveFilter
    - Vue.prototype._k = checkKeyCodes
    - Vue.prototype._b = bindObjectProps
    - Vue.prototype._v = createTextVNode
    - Vue.prototype._e = createEmptyVNode
    - Vue.prototype._u = resolveScopedSlots
    - Vue.prototype._g = bindObjectListeners
    - Vue.prototype._d = bindDynamicKeys
    - Vue.prototype._p = prependModifier
  - Vue.prototype.$nextTick = function(...)
  - Vue.prototype._render = function(...)
- initProps
  - Vue.prototype._props: Vue.options.props[key]
- code/index.js
  - Vue.prototype.$isServer
  - Vue.prototype.$ssrContext
  - Vue.prototype.$ssrContext
- wen/runtime/index.js
  - Vue.prototype.__patch__ = inBrowser ? patch : noop
  - Vue.prototype.$mount = function(...)
- web/entry-runtime-with-compiler.js
  - 重写Vue.prototype.$mount
#### Vue全局API（Vue.xxx） --> initGlobalAPI()
- initGlobalAPI()
  - Vue.config =
    {

      optionMergeStrategies: Object.create(null), // merge策略配置

      silent: false, // 是否禁止警告

      productionTip: process.env.NODE_ENV !== 'production', // 启动时显示生产模式提示消息

      devtools: process.env.NODE_ENV !== 'production', // devtools是否开启

      performance: false, // 是否录制信息

      errorHandler: null, // 观察错误的处理程序

      warnHandler: null, // 观察警告的处理程序

      ignoredElements: [], // 忽略某些自定义元素

      keyCodes: Object.create(null), // v-on自定义按键key的别名

      isReservedTag: no, // 检查组件标签是否是保留标签。

      isReservedAttr: no, // 检查是否是保留属性

      isUnknownElement: no, // 检查tag是否是未知元素,依赖于平台
      
      getTagNamespace: noop, // 获取元素命名空间
      
      parsePlatformTagName: identity, // 解析平台的特别标签，转化为真实标签

      mustUseProp: no, // 检查是否必须使用属性绑定属性,依赖于平台

      async: true, // 异步执行更新，用于Vue的视图测试工具，如果设置为false，则将显著降低性能

      _lifecycleHooks: LIFECYCLE_HOOKS // 因遗留原因暴露
    }
  - Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }
  - Vue.set = set
  - Vue.delete = del
  - Vue.nextTick = nextTick
  - Vue.observable // 2.6 explicit observable API
  - Vue.options = Object.create(null)
    ```
    Vue.options的初始化
    ASSET_TYPES = [
      'component',
      'directive',
      'filter'
    ]
    ASSET_TYPES.forEach(type => {
      Vue.options[type + 's'] = Object.create(null)
    })
    extend(Vue.options.components, builtInComponents)
    ---->
    Vue.options = {
      components: {
        KeepAlive
      },
      directives: {},
      filters: {},
      _base = Vue
    }
    ```
  - initUse --> Vue.use = function(...)
  - initMixin --> Vue.mixin = function(...)
  - initExtend
    - Vue.extend = function(...)
    - Vue.cid = 0
  - initAssetRegisters 生命周期初始化
  ```
  LIFECYCLE_HOOKS = [
    'beforeCreate',
    'created',
    'beforeMount',
    'mounted',
    'beforeUpdate',
    'updated',
    'beforeDestroy',
    'destroyed',
    'activated',
    'deactivated',
    'errorCaptured',
    'serverPrefetch'
  ]
  Vue.beforeCreate = function(...)
  Vue.created = function(...)
  ...
  ```
- Vue.version = ' __VERSION __ '
- wen/runtime/index.js
  ```
  // 添加平台相关的配置
  Vue.config.mustUseProp = mustUseProp
  Vue.config.isReservedTag = isReservedTag
  Vue.config.isReservedAttr = isReservedAttr
  Vue.config.getTagNamespace = getTagNamespace
  Vue.config.isUnknownElement = isUnknownElement

  // 添加平台运行相关的 directives & components
  extend(Vue.options.directives, platformDirectives)
  extend(Vue.options.components, platformComponents)
  经以上代码合并之后Vue.options长这样
  Vue.options = {
    components: {
      KeepAlive，
      Transition: ,
      TransitionGroup
    },
    directives: {
      model
      show
    },
    filters: {},
    _base = Vue
  }
  ```
- web/entry-runtime-with-compiler.js
  - Vue.compile = compileToFunctions // 添加Vue编译器

》PS: Vue 2.6.10