### Vue的初始化工作（options合并）
#### Vue构造函数执行this._init(options)
```
先看看Vue的基本使用
var vm = new Vue({
  el: '#app',
  data: {
    title: 'Hello Vue'
  }
})
```
当用构造函数创建`new`一个新的Vue实例时，会执行`this._init(options)`

还记得`_init()`在哪里吗？初始化原型时有看到，在在`initMixin()`方法里(`Vue.prototype._init`)，`initMixin()`在`core/instance/init.js`,

#### `new Vue`时`_init()`做了一系列的初始化工作
```
  // a uid
  vm._uid = uid++ // 生成每个实例对应的uid，不重复
  // 一个标记，用于避免在做响应式时被观测
  vm._isVue = true
  //  Vue.options
  // 这里暂时不进入_isComponent是用于内部Vue创建组件使用，不对外提供
  if (options && options._isComponent) {
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options)
  } else {
    // 实例添加$options属性
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    initProxy(vm)
  } else {
    vm._renderProxy = vm
  }
  // expose real self
  vm._self = vm
  
  // 初始化阶段
  initLifecycle(vm)
  initEvents(vm)
  initRender(vm)
  callHook(vm, 'beforeCreate')
  initInjections(vm) // resolve injections before data/props
  initState(vm)
  initProvide(vm) // resolve provide after data/props
  callHook(vm, 'created')
```
#### 实例初始化之options合并阶段：mergeOptions()
```
vm.$options = mergeOptions(
  // 解析构造函数的options Tip:需结合Vue.extend()理解，这里只是Vue.options
  resolveConstructorOptions(vm.constructor),
  options || {}, // 外部传入的option
  vm // Vue实例的引用
)
// 记得Vue.options长什么样吗？
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
// 外部传入的options：
{
  el: '#app',
  data: {
    title: 'Hello Vue'
  }
  ...
}
```
`mergeOptions()`在`core/util/options.js`
```
/*
 *在顶部例子的代码环境下
 *parent: Vue.options
 *child: options = {el: '', data: {}, ... }
 *vm?: vue实例
 */
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  // 构造函数传递函数的options属性（通过Vue.extend()创建的函数拥有options属性，以为Vue本身拥有options属性）
  if (typeof child === 'function') {
    child = child.options
  }

  // 规范化props、inject、directive，应为这三种兼容有不同的写法，Vue最终会转成一个固定规范的对象。
  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)

  // Apply extends and mixins on the child options,
  // but only if it is a raw options object that isn't
  // the result of another mergeOptions call.
  // Only merged options has the _base property.
  if (!child._base) {
    if (child.extends) {
      parent = mergeOptions(parent, child.extends, vm)
    }
    if (child.mixins) {
      for (let i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm)
      }
    }
  }

  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    // child的key已经在parent出现过，避免再次调用mergeField()
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```
#### options合并阶段之key的合并策略
```
function mergeField (key) {
  const strat = strats[key] || defaultStrat
  options[key] = strat(parent[key], child[key], vm, key)
}

// 注意stats是可以用户自定义的。通过Vue.config.optionMergeStrategies定义合并策略
const strats = config.optionMergeStrategies
```
> 如果`strats[key]`访问不到，则使用默认合并侧率

**1、默认合并策略**
```
// 很简单不解释
const defaultStrat = function (parentVal: any, childVal: any): any {
  return childVal === undefined
    ? parentVal
    : childVal
}
```
**2、el、propsData合并策略**
```
if (process.env.NODE_ENV !== 'production') {
  strats.el = strats.propsData = function (parent, child, vm, key) {
    if (!vm) {
      warn(
        `option "${key}" can only be used during instance ` +
        'creation with the `new` keyword.'
      )
    }
    return defaultStrat(parent, child)
  }
}
```
使用的是默认合并策略， 且el，和propsData只能在`new`操作符是使用。

**3、data合并策略**
```
strats.data = function (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    if (childVal && typeof childVal !== 'function') {
      process.env.NODE_ENV !== 'production' && warn(
        'The "data" option should be a function ' +
        'that returns a per-instance value in component ' +
        'definitions.',
        vm
      )

      return parentVal
    }
    return mergeDataOrFn(parentVal, childVal)
  }

  return mergeDataOrFn(parentVal, childVal, vm)
}
```
通过代码可以看出，当子组件合并时，data应该返回一个函数。最终会返回`mergeDataOrFn`函数，
而`mergeDataOrFn`永远都会返回一个函数。但在此时并不会调用。`mergeDataOrFn`返回的函数会调用`mergeData`函数，
这个函数`(mergeData)`执行结果才是`data`最后合并的数据结果

> 为什么要处理成一个函数呢？应为通过函数能返回一个处理数据对象的一个副本，可以避免组件间相互影响

**4、生命周期的合并策略**
```
LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})

function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  const res = childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
  return res
    ? dedupeHooks(res)
    : res
}
function dedupeHooks (hooks) {
  const res = []
  for (let i = 0; i < hooks.length; i++) {
    if (res.indexOf(hooks[i]) === -1) {
      res.push(hooks[i])
    }
  }
  return res
}
```
可以看到生命周期方法最后会被合并成一个数组，通过`dedupeHooks`的处理，
当执行生命周期函数时，父组件的生命周期方法的执行会优先与子组件的生命周期方法
```
mounted: [
  function () {
    console.log('parent mounted~')
  },
  function () {
    console.log('child mounted~')
  }
]
```

**5、ASSET_TYPES合并策略**
`ASSET_TYPES`即：`component、directive、filter`
```
function mergeAssets (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): Object {
  const res = Object.create(parentVal || null)
  if (childVal) {
    process.env.NODE_ENV !== 'production' && assertObjectType(key, childVal, vm)
    return extend(res, childVal)
  } else {
    return res
  }
}

ASSET_TYPES.forEach(function (type) {
  strats[type + 's'] = mergeAssets
})
```
可以看到合并前会以`parentVal`为原型创建一个`res`对象，之后再根据`childVal`是否有值进行合并。
同过`Object.create`为原型创建对象，可以让我们不用显式地注册组件就能够使用一些组件（例如：内置组件）的原因
```
// 以component为例
components: {
  // 原型
  __proto__: {
    KeepAlive,
    Transition,
    TransitionGroup
  },
  // 子组件的组件
  childCompoent,
}
```
**6、watch的合并策略**
```
strats.watch = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  // work around Firefox's Object.prototype.watch...
  if (parentVal === nativeWatch) parentVal = undefined
  if (childVal === nativeWatch) childVal = undefined
  /* istanbul ignore if */
  if (!childVal) return Object.create(parentVal || null)
  if (process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  // 当父组件没有watch时，直接返回子组件的，此时返回的watch对象里的可以是一个函数，而不是数组
  if (!parentVal) return childVal
  // 当父子组件都有watch时
  const ret = {}
  extend(ret, parentVal)
  for (const key in childVal) {
    let parent = ret[key]
    const child = childVal[key]
    if (parent && !Array.isArray(parent)) {
      parent = [parent]
    }
    ret[key] = parent
      ? parent.concat(child)
      : Array.isArray(child) ? child : [child]
  }
  return ret
}
```
应为`Firefox`存在原生的`watch`说以要做处理。可以看到最终合并的watch对象里的key值，有可能是一个函数数组，也可能是一个函数。列入：
```
watch: {
  test: function () {...}
  test2: [
    function () {...},
    function () {...}
  ]
}
```
**7、props、methods、inject、computed合并策略**
```
strats.props =
strats.methods =
strats.inject =
strats.computed = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  if (childVal && process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal
  const ret = Object.create(null)
  extend(ret, parentVal)
  if (childVal) extend(ret, childVal)
  return ret
}
```
通过代码可以看出，合并很简单，当父组件没有时，则返回子组件的；当父子间有时，会混合到一个空对象`ret`中；当父子组件都有时，子组件会把父组件覆盖掉。
**8、provide合并策略**
```
strats.provide = mergeDataOrFn
```
`provide`合并策略就一句，可以看出可`data`合并策略相同

> 最后以上没有处理的key的合并策略都使用默认合并策略，即`defaultStrat`

**9、mixins、extends实现**
```
// 在mergeOptions下有段代码
// 记得_base在哪里吗？在initGlobalAPI初始化全局时 Vue.options._base = Vue
if (!child._base) {
  if (child.extends) {
    parent = mergeOptions(parent, child.extends, vm)
  }
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm)
    }
  }
}
```
现在明白了options的合并策略，是不是一看即懂了o(*￣︶￣*)o
