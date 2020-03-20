### Vue的响应式原理
> 入口：`core/instance/state.js --> initState() --> observe()`

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
**1-1、Observer类**
```js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    // 在观测的对象上定义一个'__ob__'属性，且值是当前Observer实例
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      // 判读是否为数组，是数组则对数组进行观测，判断是否有'__proto__'属性
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
      // 收集依赖
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      // 获取原有值与新值进行比较
      const value = getter ? getter.call(obj) : val
      // 值相等或值为NaN
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
      // 通知观察者进行更新
      dep.notify()
    }
  })
}
```
通过以上的处理，我们的观测的数据对象就是响应式的了。