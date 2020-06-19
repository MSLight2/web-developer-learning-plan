## Vue的响应式原理
> 入口：`core/instance/state.js --> initState() --> observe()`
```js
initData(vm)
// ...
function initData (vm: Component) {
  // ...
  // observe data
  observe(data, true /* asRootData */)
}
```
通过调用`observe()`方法把data函数里的变量定义成响应式的

**1、core/observer/index.js：observe工厂**
```js
/**
 * 为传入的值尝试创建一个Observer实例，如果观测成功则返回实例，
 * 如果这个值已经观测过，则直接返回这个值的Observer实例
 */
export function observe (value: any, asRootData: ?boolean): Observer | void {
  // 如果观测的值不是对象，或则是虚拟dom对象则不观测。
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    // 如果这个对象已经被观测过了，则返回这个观测对象，避免重复观测（value.__ob__储存的就是观测的Observer对象）
    ob = value.__ob__
  } else if (
    shouldObserve &&                                  // 一个开关，控制对象是否需要观测
    !isServerRendering() &&                           // 非服务端渲染
    (Array.isArray(value) || isPlainObject(value)) && // 观测的值是数组或则是纯对象
    Object.isExtensible(value) &&                     // 对象必须是可扩展的
    !value._isVue                                     // 对象不是Vue实例
  ) {
    // 满足以上5个条件对象才可以被观测
    // 如果没有观测过，这创建一个Observer实例
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```
调用`observe`后会为传入的对象添加一个`__ob__`属性，这个属性指向`Observer`类的实例

**1-1、Observer类**
```js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep() // 定义一个Dep实例，用于收集依赖
    this.vmCount = 0
    // 在观测的对象上定义一个'__ob__'属性，且值是当前Observer实例
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      // 判读是否为数组，是数组则对数组进行观测，判断是否有'__proto__'属性
      // 对数组的方法拦截，定义变异方法，使变异方法是具有响应式的
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      // 观测对象
      this.walk(value)
    }
  }

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

**1-2、Vued对数组的拦截（Vue变异方法的实现）**
> 原理：原型链的查找以及属性的覆盖
```js
// 思路：
// 以数组构造函数的原型创建一个对象，所以arrayMethods原型指向数组构造函数的原型，
// 即使arrayMethods拥有和数组一样的全部方法。
// 通过在arrayMethods上定义和数组构造函数相同的方法来达到方法的拦截。
// 之后导出这个对象（arrayMethods），让观测的数组的原型指向这个对象，即可到达Vue对数组的变异方法的实现

const arrayProto = Array.prototype
arrayMethods = Object.create(arrayProto)

// 要拦截的数组方法
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  // 缓存原先得方法，保证重写方法后的执行结果和原来的一致
  const original = arrayProto[method]
  // 在以数组原型为原型的对象上定义要拦截的方法
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args) // 调用原先的方法，使结果一致
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    // 对新的值进行观测
    if (inserted) ob.observeArray(inserted)
    // 通知更新
    ob.dep.notify()
    return result
  })
})
```
**1-3、Vue的defineReactive函数**
> defineReactive方法将数据对象的属性转换为访问器属性，即重写对象的`get/set`
> 
> 这个方法是Vue响应式系统的关键方法。
```js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 收集依赖的一个对象，即收集观察者对象
  const dep = new Dep()

  // 确保这个对象的值是可配置的
  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // 缓存原有个get/set方法，保证不影响原有的读写操作。
  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // shallow控制是否对对象进行深度观测；这里默认开启深度观测
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      // 调用原有的get方法返回属性值，保证正确的读取值
      const value = getter ? getter.call(obj) : val
      // 收集依赖。Dep.target指向当前观察则
      if (Dep.target) {
        // 这里通过闭包的形式引用dep实例对象，使得每个被观察的的属性都有自己的dep框，收集的都是观测自己本身的依赖
        dep.depend()
        if (childOb) {
          // 如果值是对象，且有深度观测了则一同收集依赖，以保证一个值改变后父属性或子值可以一同改变。
          childOb.dep.depend()
          if (Array.isArray(value)) {
            // 对数组进行依赖收集，因为Object.defineProperty不能像对象属性getter那样拦截数组元素访问
            // 需要手动收集依赖，这就是Object.defineProperty的缺陷。所以vue3.0使用proxy实现响应式
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      // 获取原有值与新值进行比较
      const value = getter ? getter.call(obj) : val
      // 值相等或值为NaN，则不做变更
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // 如果新值是对象，这对值进行继续观测
      childOb = !shallow && observe(newVal)
      // 通知观察者进行更新（运行依赖框中收集的观察者--所有观察者均为异步更新）
      dep.notify()
    }
  })
}

/**
 * Collect dependencies on array elements when the array is touched, since
 * we cannot intercept array element access like property getters.
 */
function dependArray (value: Array<any>) {
  for (let e, i = 0, l = value.length; i < l; i++) {
    e = value[i]
    e && e.__ob__ && e.__ob__.dep.depend()
    if (Array.isArray(e)) {
      dependArray(e)
    }
  }
}
```
通过`defineReactive`对对象属性的处理的处理，数据就是就具备响应式的条件了。
但是只是仅仅定义了对象属性的`get/set`，还没有观察者对数据进行观测。被观察的对象也还没收集到依赖。

从这里可以看出：定义在data里的属性在初始化后把它们变成响应式的了，也就是说在后期代码运行加入的字段都不具有响应式，这也是为什么Vue会提供`Vue.set`方法，让你手动把字段变成响应式。`Vue.set`的实现也是通过`defineReactive`方法，让你能手动把数据变成响应式的。

## 响应式属性依赖的收集以及触发
你可能会问依赖是什么？依赖是如何收集的？又在哪里收集的？有哪些收集方式？

其实依赖就是观察者对象，通过上面的代码可以知道依赖是在触发属性`getter`属性的时候收集的。每个属性在触发`getter`的时候都会收集观察它的观察者。

一般默认收集的都是**渲染函数观察者**，即在页面会渲染的时候，会根据模板里面的数据渲染页面，此时也就触发了对象属性的`getter`属性。在vue中观察者的类型有：渲染函数观察者、计算器属性观察者、用户自定义的观察者（即watch）。

所以我们先来看属性的**渲染函数观察者**

在`web/runtime/index.js`中有以下一段代码
```js {6}
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```
我们知道在使用vue是都会new一个vue实例且挂载dom。如`new Vue({...}).$mount('app')`。
这段代码在就是在我们创建Vue实例，挂载dom时会调用的一个方法，方法会调用`mountComponent`方法，并且返回`mountComponent`的返回值。

`mountComponent`在`core/instance/lifecycle.js`里
```js {13, 19}
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el

  // 省略...

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)

  // 省略...

  return vm
}
```
可以看到这里创建了一个`Watcher`实例，这个实例就是我们说的依赖，也就是观察者。`Watcher`的最后一个参数是true，表明这个观察者是渲染函数观察者。

## Dep
我们看`Watcher`之前先来瞄一瞄收集依赖的`Dep`源码。`core/observer/dep.js`
```js
let uid = 0

/**
 * A dep is an observable that can have multiple
 * directives subscribing to it.
 */
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>; // 收集依赖的框，属性所依赖的观察则都会放入这个数组中

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  // 依赖的收集：dep.depend()
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  // 通知更新
  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      // 这里的每一项都是观察者对象。即Watcher实例对象
      subs[i].update()
    }
  }
}

// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

## Watcher
接下来看Watcher观察者对象。`core/observer/watcher.js`
```js
let uid = 0

/**
 * A watcher parses an expression, collects dependencies,
 * and fires callback when the expression value changes.
 * This is used for both the $watch() api and directives.
 */
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  deep: boolean;
  user: boolean;
  lazy: boolean;
  sync: boolean;
  dirty: boolean;
  active: boolean;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  before: ?Function;
  getter: Function;
  value: any;

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    // 只有渲染函数观察者vue实例的_watcher才有值
    if (isRenderWatcher) {
      vm._watcher = this
    }
    // vue实例收集所有的观察者
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    // 这里的lazy只有是计算器属性观察者的时候才会是true，一般情况都会是false，会调用this.get()
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   * 调用get方法，触发数据属性的getter，收集依赖
   */
  get () {
    pushTarget(this) // 使Dep.target指向当前观察者实例
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        // 深度观测（用于用户定义的watch，当配置deep为true时，深度观测），其实先原理就是触发对象属性的getter使其可以收集到依赖
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

  /**
   * Add a dependency to this directive.
   * dep收集依赖，且dep和依赖建立联系。互相关联
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) { // 如果dep.id已经存在newDepIds里了，则不进行依赖的收集。防止依赖的重复收集。
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      // 这里才是Dep真正的收集依赖的地方
      // 至于为什么还有个对depIds判断呢？因为newDepIds是用于‘一次求值’时防止依赖的重复收集，
      // 而depIds是用于‘多次求值’时防止依赖收集。‘一次求值’和‘多次求值’什么意思呢？
      // 一次求值：可以理解为在一次渲染的时候多次调用了getter,
      // 多次求值：可以理解为在多次渲染是调用getter
      // 那为什么不用newDepIds处理呢，因为每次收集完依赖后，newDepIds都活别清空。可以看上面的get()方法里调用了this.cleanupDeps()
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      // 移除废弃的观察者
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    // newDepIds的值赋给depIds，同时清楚newDepIds。newDeps同理
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
   // 但数据变化时，通知观察者更新。此方法会被调用。
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      // 异步更新队列（vue中更新使用的是异步更新）
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  //  异步更新队列调度时watch调用更新，这里才是真正的更新
  run () {
    // 当观察者对象处于激活转态时才执行（默认状态是激活的）
    if (this.active) {
      // 调用get函数，触发视图更新
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            // 用户自定义的watch，调用用户定义的方法，传入当前vue实例、新值和旧值作为参数
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }

  /**
   * Evaluate the value of the watcher.
   * This only gets called for lazy watchers.
   */
  //  手动触发属性的get，用于惰性求值的watcher，例如：计算器属性观察者
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }

  /**
   * Depend on all deps collected by this watcher.
   */
  // 通过一个watcher让和它关联的dep收集另一个watcher。让另一个watcher和这些dep关联。
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }

  /**
   * Remove self from all dependencies' subscriber list.
   */
  // 当一个观察者是非激活时，或者已经无用了，就和组件对象已经它关联的属性对象解除关联
  // 这个函数用于取消属性的观察者，即移除观察。
  teardown () {
    if (this.active) {
      // remove self from vm's watcher list
      // this is a somewhat expensive operation so we skip it
      // if the vm is being destroyed.
      if (!this.vm._isBeingDestroyed) {
        // 由于这个操作开销较大所以仅在组件没有销毁时才执行
        remove(this.vm._watchers, this)
      }
      let i = this.deps.length
      while (i--) {
        // 当一个属性收集到观察者依赖时，观察者也会收集Dep对象实例，这是个双向的过程。且一个观察者可以同时观察多个属性，所以解除关联时，要将观察者从这些属性的Dep实例对象中移除
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
}
```

## 观察者的异步更新
我们知道在数据变化时会触发属性的set时会有以下调用：
- dep.notify() --> watcher.update() --> queueWatcher() --> watcher.run()

其中`watcher.run()`是真正的视图更新，而queueWatcher是什么呢，其实就是异步更新队列。

为什么要使用异步更新呢？其实很简单，如果是同步更新的的话，一个属性值变化就要重新渲染一次，两个属性变化就要渲染两次，如果一次变更多个属性值那就要渲染多次，并且如果一个属性值同时变更多次那么也会多次渲染，这样的话在复杂的业务场景下会有严重的性能问题。而异步更新能解决这个问题。那就是把所有要更新的观察者放入一个队列，如果队列中已存在要更新的观察者对象则不会入队，等所有的属性的数据变更完后，再一次性执行队列中所有观察者对象的更新方法。就可以达到优化的目的

以下就是`queueWatcher`的全部代码
```js
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  // has默认是一个空对象，通过控制has防止相同观察者重复入队
  if (has[id] == null) {
    has[id] = true
    // 一个标识，标识当前队列是否正在执行更新
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      // 处理在更新时，观察者入队的情况。
      // 比如计算属性观察者，由于计算属性在实现方式上与普通响应式属性有所不同，当渲染函数里有计算器属性时，当触发计算属性的 `get` 时，get函数是用户定义的，有可能里面有set操作，所以此时可能会有观察者入队的行为，这个时候需要特殊处理
      // 找到队列中小于入队watcher.id的位置，然后插入，保证观察者的执行顺序。
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    // 无论调用多少次queueWatcher，里面的代码只会执行一次
    // 你可能会想为什么只执行一次，因为里面的代码是异步的，一次执行就够了（Event Loop）
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

记下来看队列（`flushSchedulerQueue`）是怎么执行的
```js
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  // 为什么要排序，如注释所说：
  // 1. 父组件的更新要优先于子组件，因为父组件总是比子组件要优先创建
  // 2. 一个组件用户定义的watch要优先于渲染函数的watcher，因为用户定义的watcher在render watcher前（用户定义的watch在初始化时已经创建了。可以看initWatch方法）
  // 3. 如果组件在父组件watcher运行时已经销毁了，那么这个组件的watcher也能被跳过，销毁。
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```
## 计算属性实现原理
```js
// 源码
export function initState (vm: Component) {
  // ...
  if (opts.computed) initComputed(vm, opts.computed)
  // ...
}

function initComputed (vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
     // ...
    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      // ...
    }
  }
}

export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  // 省略其它代码..
  sharedPropertyDefinition.get = shouldCache
    ? createComputedGetter(key)
    : createGetterInvoker(userDef)
  sharedPropertyDefinition.set = noop
  // 省略其它代码..
  Object.defineProperty(target, key, sharedPropertyDefinition)
}

function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```
通过以上源码可以得知：在vue实例初始化时，`initState`通过调用`initComputed`为定义的每个计算属性创建一个观察者实例对象，并把观察者对象存入Vue实例的`_computedWatchers`属性中。我们把这个观察者称为：**计算器属性观察者**。之后再通过`defineComputed`方法为计算属性添加`get/set`属性。

通过`createComputedGetter`可以看到，在计算属性获取值时，即`get`时，会触发计算属性的`get`方法。`get`方法会取出它自身的观察者对象，在调用`watcher.evaluate()`和`watcher.depend()`，最后返回值。

#### watcher.evaluate()的调用
在计算属性获取值时，**计算器属性观察者**会调用这个方法后，此时`Dep.target`就是当前**计算器属性观察者**实例对象，所以**计算属性所依赖的响应式字段**会收集**计算器属性观察者**依赖；**计算器属性观察者**也会和它所观测的响应式字段相关联。
#### watcher.depend()的调用
渲染时，计算器属性观察者会调用`depend`这个方法时，`Dep.target`必然是**渲染函数观察者**，所以会使计算器属性所依赖的响应式字段收集**渲染函数观察者**依赖。此时计算器属性所依赖的响应式字段的`Dep`中就收集了“计算器属性观察者”和“渲染函数观察者”。又因为计算器属性观察者是惰性求值的，因此当计算属性依赖的值发生变化时，会通知观察者更新；但计算器属性观察者是不会触发更新的，触发更新的是渲染函数观察者。所以会导致重新渲染，最终更新视图。

> 综上所述：计算器属性的实现就是为计算属性字段添加**计算器属性观察者**，之后通过计算器属性观察者让它依赖的响应式字段收集渲染函数观察者，从而实现计算属性的缓存。计算器属性观察者就是在这中间起到一个桥梁的作用。

## Vue.watch/set/delete实现原理