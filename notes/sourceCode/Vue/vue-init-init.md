### Vue的初始化工作（进入初始化）
**initLifecycle**
```
vm.$parent = parent
vm.$root = parent ? parent.$root : vm

vm.$children = []
vm.$refs = {}

vm._watcher = null
vm._inactive = null
vm._directInactive = false
vm._isMounted = false
vm._isDestroyed = false
vm._isBeingDestroyed = false
```
定义一些与生命周期相关的属性

**initEvents**
```
vm._events = Object.create(null)
vm._hasHookEvent = false
// init parent attached events
const listeners = vm.$options._parentListeners
if (listeners) {
  updateComponentListeners(vm, listeners)
}
```
**initRender**
```
vm._vnode = null // the root of the child tree
vm._staticTrees = null // v-once cached trees
vm.$slots = resolveSlots(options._renderChildren, renderContext)
vm.$scopedSlots = emptyObject
vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
vm.$attrs
vm.$listeners
```
**callHook**

调用生命周期函数

**initState**
```
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```
对于初始化顺序：props > methods > data > computed > watch。
对数据进行观测，响应式系统开始