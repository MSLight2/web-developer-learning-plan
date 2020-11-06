## vue-router源码阅读

### 前言
阅读源码是一种很好提高自己技术的方式，它既能帮你查漏补缺也能使你学到新的技术。虽然一直在使用vue-router，但一直没有深入去了解过。这次的就在这里做个阅读笔记，感兴趣就看看把~

阅读源码前你需要对`hash`和[`history`](https://developer.mozilla.org/zh-CN/docs/Web/API/History_API)的相关知识有一定的了解，此外如果你阅读过`vue`源码的话那就更好了，这样你对`vue-router`的源码也会有更深的了解

### 路由源码简单的思维导图
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4419e4cac2a845029509899a0d0d8944~tplv-k3u1fbpfcp-watermark.image)
### 路由源码解析
#### 路由的使用
在阅读源码前我们先来复习vue-router是如何使用的，之后我们就根据这个步骤一步步去理解源码
```js
// router.js
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)
export default new Router({
  mode: 'hash',
  routes: [
    { path: '/foo', name: 'foo', component: Foo },
    { path: '/bar',name: 'bar', component: Bar }
  ]
})

// main.js
import router from './router'

new Vue({
  router,
  render: h => h(App)
}).$mount('#app')
```
1. 路由的安装
2. 创建 router 实例，然后传 `routes` 配置
3. 创建和挂载根实例，通过 router 配置参数注入路由
4. 使用push或replace改变路由
#### 路由插件的安装(install.js)
`vue-router`是通过使用`Vue.use()`安装。所以会调用插件暴露`install`方法。`vue-router`插件的`install`方法在[https://github.com/vuejs/vue-router/blob/dev/src/install.js](https://github.com/vuejs/vue-router/blob/dev/src/install.js)。
> ps: 后面的文件路径就不一一列举了，可以自行clone到本地查看

install.js
```js
import View from './components/view'
import Link from './components/link'

export let _Vue

export function install (Vue) {
  // 确保 install 方法只调用一次，防止重复安装
  if (install.installed && _Vue === Vue) return
  install.installed = true

  _Vue = Vue

  const isDef = v => v !== undefined
  
  // 注册vue-router实例，registerRouteInstance在RouterView组件内有定义
  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

  // 在每一个Vue实列注入钩子函数，这一步很重要，
  // 在组件初始化后，beforeCreate钩子会执行，从而初始化路由
  Vue.mixin({
    beforeCreate () {
      if (isDef(this.$options.router)) {
        this._routerRoot = this 		// 根路由为自身
        this._router = this.$options.router
        this._router.init(this) 		// 初始化路由
        // 把 _route 属性定义为响应式的，这一步很关键。
        // 视图的渲染更新都是通过对这个属性的设值，从而触发视图更新；包括点击浏览器的前进和后退按钮
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })

  // 在Vue原型上添加$router属性，这样在组件实例上我们就可以以this.$router的方式访问路由实列了
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })

  // 全局注册组件
  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  // 钩子函数的合并策略
  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}
```
#### 创建路由实例(index.js)
这里我们只看它的`构造函数和init函数`，其他一些函数大多是暴露API，比如`push、addRoutes`等。就不一一介绍。
```js
export default class VueRouter {
  // ...
  constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    // 创建路由映射表和路由记录对象
    this.matcher = createMatcher(options.routes || [], this)

    let mode = options.mode || 'hash'
    // 配置一个回调函数，当不支持history模式时，回退到hash模式
    this.fallback =
      mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }

  // ...

  // 路由初始化函数，在beforeCreate钩子执行时，如果传入了 router 实例，执行这个方法
  // 只有根 Vue 实例会保存到 this.app 中
  init (app: any /* Vue component instance */) {
    // ...

    this.apps.push(app)

    // set up app destroyed handler
    // https://github.com/vuejs/vue-router/issues/2639
    app.$once('hook:destroyed', () => {
      // clean out app from this.apps array once destroyed
      const index = this.apps.indexOf(app)
      if (index > -1) this.apps.splice(index, 1)
      // ensure we still have a main app or null if no apps
      // we do not release the router so it can be reused
      if (this.app === app) this.app = this.apps[0] || null

      if (!this.app) this.history.teardown()
    })

    // main app previously initialized
    // return as we don't need to set up new history listener
    if (this.app) {
      return
    }

    this.app = app

    const history = this.history

    if (history instanceof HTML5History || history instanceof HashHistory) {
      const handleInitialScroll = routeOrError => {
        const from = history.current
        const expectScroll = this.options.scrollBehavior
        const supportsScroll = supportsPushState && expectScroll

        if (supportsScroll && 'fullPath' in routeOrError) {
          handleScroll(this, routeOrError, from, false)
        }
      }
      const setupListeners = routeOrError => {
        history.setupListeners()
        handleInitialScroll(routeOrError)
      }
      // 默认进行一次路由跳转，同时初始化监听事件
      history.transitionTo(
        history.getCurrentLocation(),
        setupListeners,
        setupListeners
      )
    }
    
    // 设置history基类中cb的值，以便在调用transitionTo后
    // 能方便对组件的 _route 属性进行赋值，触发组件渲染
    history.listen(route => {
      this.apps.forEach(app => {
        app._route = route
      })
    })
  }
  
}

// ...
```
构造函数定义了一些属性，以及调用`createMatcher`创建路由映射表和路由记录对象，这个对后面的路由匹配有很关键的作用。同时构造函数还会根据路由配置的`mode`属性初始化不同的路由对象。

在`VueRouter`类中还有其他方法这里没有一一介绍，这些方法大多是暴露出去的API，你可以在官方文档中查到。这里只讲解了一个`init`方法，这个方法在先前将插件安装时有提到，它会在beforeCreate钩子执行时调用，前提时有传入`router`实例，而我们一般在VUE的根实例中传入`router`实例，所以`this.app`中保存的都是`vue`的跟实例。

#### 字段类型声明(declarations.js)
在阅读`createMatcher、base基类`等js文件前最好线了解以下vue-router源码中一些类型的定义，这样你在看源码时才分得清`Location、RawLocation、RouteRecord、Route`等这些的区别。
```js
// ...
// new VueRouter()时传入的路由配置
declare type RouterOptions = {
  routes?: Array<RouteConfig>;
  mode?: string;
  fallback?: boolean;
  base?: string;
  linkActiveClass?: string;
  linkExactActiveClass?: string;
  parseQuery?: (query: string) => Object;
  stringifyQuery?: (query: Object) => string;
  scrollBehavior?: (
    to: Route,
    from: Route,
    savedPosition: ?Position
  ) => PositionResult | Promise<PositionResult>;
}

declare type RedirectOption = RawLocation | ((to: Route) => RawLocation)
// 单个路由的配置
declare type RouteConfig = {
  path: string;
  name?: string;
  component?: any;
  components?: Dictionary<any>;
  redirect?: RedirectOption;
  alias?: string | Array<string>;
  children?: Array<RouteConfig>;
  beforeEnter?: NavigationGuard;
  meta?: any;
  props?: boolean | Object | Function;
  caseSensitive?: boolean;
  pathToRegexpOptions?: PathToRegexpOptions;
}
// 路由记录
declare type RouteRecord = {
  path: string;
  regex: RouteRegExp;
  components: Dictionary<any>;
  instances: Dictionary<any>;
  name: ?string;
  parent: ?RouteRecord;
  redirect: ?RedirectOption;
  matchAs: ?string;
  beforeEnter: ?NavigationGuard;
  meta: any;
  props: boolean | Object | Function | Dictionary<boolean | Object | Function>;
}
// 跳转路由
declare type Location = {
  _normalized?: boolean;
  name?: string;
  path?: string;
  hash?: string;
  query?: Dictionary<string>;
  params?: Dictionary<string>;
  append?: boolean;
  replace?: boolean;
}

// 跳转的路由，比如调用push时传入的路由对象
declare type RawLocation = string | Location

// 路由对象。this.$route
declare type Route = {
  path: string;
  name: ?string;
  hash: string;
  query: Dictionary<string>;
  params: Dictionary<string>;
  fullPath: string;
  matched: Array<RouteRecord>;
  redirectedFrom?: string;
  meta?: any;
}
```

#### 创建路由映射表，路由匹配对象
create-matcher.js
```js
export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  const { pathList, pathMap, nameMap } = createRouteMap(routes)

  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }

  // 路由匹配
  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    // ...
  }

  return {
    match,
    addRoutes
  }
}
```
createMatcher主要返回了两个函数，一个用于匹配路由用的的`match`函数，一个用于动态添加路由的`addRoutes`函数。此外返回函数前还调用了`createRouteMap`方法。可以看到`addRoutes`也调用了`createRouteMap`方法。所以先来看看`createRouteMap`方法。`match`稍后了解。

create-route-map.js
```js
export function createRouteMap (
  routes: Array<RouteConfig>,
  oldPathList?: Array<string>,
  oldPathMap?: Dictionary<RouteRecord>,
  oldNameMap?: Dictionary<RouteRecord>
): {
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>
} {
  // the path list is used to control path matching priority
  const pathList: Array<string> = oldPathList || []
  // $flow-disable-line
  const pathMap: Dictionary<RouteRecord> = oldPathMap || Object.create(null)
  // $flow-disable-line
  const nameMap: Dictionary<RouteRecord> = oldNameMap || Object.create(null)

  // 每一个路由都执行addRouteRecord方法
  routes.forEach(route => {
    addRouteRecord(pathList, pathMap, nameMap, route)
  })

  // 把路径时'*'的放到数组最末尾。比如：['/foo', '/bar', '*']
  for (let i = 0, l = pathList.length; i < l; i++) {
    if (pathList[i] === '*') {
      pathList.push(pathList.splice(i, 1)[0])
      l--
      i--
    }
  }

  // ...

  return {
    pathList,
    pathMap,
    nameMap
  }
}

// 添加路由记录
function addRouteRecord (
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>,
  route: RouteConfig,
  parent?: RouteRecord,
  matchAs?: string
) {
  const { path, name } = route
  // ...
  const pathToRegexpOptions: PathToRegexpOptions =
    route.pathToRegexpOptions || {}
  // 格式化路由path
  const normalizedPath = normalizePath(path, parent, pathToRegexpOptions.strict)

  if (typeof route.caseSensitive === 'boolean') {
    pathToRegexpOptions.sensitive = route.caseSensitive
  }

  // 创建路由记录对象
  const record: RouteRecord = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    instances: {},
    name,
    parent,
    matchAs,
    redirect: route.redirect,
    beforeEnter: route.beforeEnter,
    meta: route.meta || {},
    props:
      route.props == null
        ? {}
        : route.components
          ? route.props
          : { default: route.props }
  }

  // 是否有子路由，有则递归添加路由记录对象
  if (route.children) {
    // ...
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }

  // 填充path数组和path的map对象
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }

  // 如果有别名，则为别名添加路由记录对象
  // matchAs有值的RouteRecord就是别名的RouteRecord
  if (route.alias !== undefined) {
    const aliases = Array.isArray(route.alias) ? route.alias : [route.alias]
    for (let i = 0; i < aliases.length; ++i) {
      const alias = aliases[i]
      if (process.env.NODE_ENV !== 'production' && alias === path) {
        warn(
          false,
          `Found an alias with the same value as the path: "${path}". You have to remove that alias. It will be ignored in development.`
        )
        // skip in dev to make it work
        continue
      }

      const aliasRoute = {
        path: alias,
        children: route.children
      }
      addRouteRecord(
        pathList,
        pathMap,
        nameMap,
        aliasRoute,
        parent,
        record.path || '/' // matchAs
      )
    }
  }

  // 如果路由有命名，则填充name map对象
  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    } else if (process.env.NODE_ENV !== 'production' && !matchAs) {
      warn(
        false,
        `Duplicate named routes definition: ` +
          `{ name: "${name}", path: "${record.path}" }`
      )
    }
  }
}
```
可以看到`createRouteMap`创建了3个引用类型的变量，并且返回。分别是：`pathList、pathMap、nameMap`。所以在调用`addRoutes`动态添加路由时，这个三个变量的值也会改变。同时`createRouteMap`还遍历路由配置，让每个路由都执行`addRouteRecord`方法。`addRouteRecord`方法主要是创建路由记录对象并填充到`pathList、pathMap、nameMap`这三个变量里面。

**所以`createMatcher`调用后除了返回`addRoutes和match`两个方法外，最重要的一点就是根据用户的路由配置生成对应的路由记录。**

现在来看看`match`方法：
```js
// 路由匹配
  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    // 规范化路由
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

    // 有name则用name去查找路由记录
    if (name) {
      const record = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        warn(record, `Route with name '${name}' does not exist`)
      }
      // 没有匹配的路由
      if (!record) return _createRoute(null, location)
      const paramNames = record.regex.keys
        .filter(key => !key.optional)
        .map(key => key.name)

      if (typeof location.params !== 'object') {
        location.params = {}
      }

      if (currentRoute && typeof currentRoute.params === 'object') {
        for (const key in currentRoute.params) {
          if (!(key in location.params) && paramNames.indexOf(key) > -1) {
            location.params[key] = currentRoute.params[key]
          }
        }
      }

      location.path = fillParams(record.path, location.params, `named route "${name}"`)
      return _createRoute(record, location, redirectedFrom)
    } else if (location.path) {
      // 没有name则通过path去匹配路由
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match（没有匹配的路由）
    return _createRoute(null, location)
  }
  
  function _createRoute (
    record: ?RouteRecord,
    location: Location,
    redirectedFrom?: Location
  ): Route {
    if (record && record.redirect) {
      return redirect(record, redirectedFrom || location)
    }
    if (record && record.matchAs) {
      return alias(record, location, record.matchAs)
    }
    return createRoute(record, location, redirectedFrom, router)
  }
```
`match`方法就是用来匹配路由的。`match`方法无论是否匹配到路由都会调用`_createRoute`方法，而`_createRoute`最后会调用`createRoute`方法。`createRoute`这个方法最终会返回一个不可以修改的`Route`对象。其实这个对象就是`this.$route`。源码如下：
```js
export function createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: ?Location,
  router?: VueRouter
): Route {
  const stringifyQuery = router && router.options.stringifyQuery

  let query: any = location.query || {}
  try {
    query = clone(query)
  } catch (e) {}

  const route: Route = {
    name: location.name || (record && record.name),
    meta: (record && record.meta) || {},
    path: location.path || '/',
    hash: location.hash || '',
    query,
    params: location.params || {},
    fullPath: getFullPath(location, stringifyQuery),
    matched: record ? formatMatch(record) : []
  }
  if (redirectedFrom) {
    route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery)
  }
  // 返回一个冻结的对象
  return Object.freeze(route)
}
```

#### history和路由的跳转
在前面介绍创建路由实例时，有说到创建实例时会根据用户配置创建不同history，在vue-router中主要有两种：`hash`和`history`。简单来说`hash`是带有`#`号的，`history`是不带`#`号的。通过看源码可以知道，无论是`HashHistory`还是`HTML5History`都继承`base.js`文件里的`History`。所以我们注重看`base.js`文件夹的代码

base.js
```js
// implemented by sub-classes
+go: (n: number) => void
+push: (loc: RawLocation, onComplete?: Function, onAbort?: Function) => void
+replace: (
  loc: RawLocation,
  onComplete?: Function,
  onAbort?: Function
) => void
+ensureURL: (push?: boolean) => void
+getCurrentLocation: () => string
+setupListeners: Function
```
在基类`History`中有以上这段代码，依注释说所说：这些方法都是在子类实现的，父类自是定义。

这里以`html5.js`为例。当使用`this.$router.push({ name: 'foo' })`切换路由时，此时时调用`VueRouter`类中的`push`方法，而这个方法会调用`html5.js`里的`push`方法。而这个`push`方法会调用基类的`transitionTo`方法达到切换路由的效果。所以来看看路是如何切换的。
```js
  transitionTo (
    location: RawLocation,
    onComplete?: Function,
    onAbort?: Function
  ) {
    // location保存的是传入的路由，current保存的是当前的路由对象
    let route
    // catch redirect option https://github.com/vuejs/vue-router/issues/3201
    try {
      // 匹配路由，返回路由对象。这个match就是createMatcher返回的match
      route = this.router.match(location, this.current)
    } catch (e) {
      this.errorCbs.forEach(cb => {
        cb(e)
      })
      // Exception should still be thrown
      throw e
    }
    this.confirmTransition(
      route,
      () => {
        const prev = this.current
        // 更新路由，触发渲染
        this.updateRoute(route)
        // 调用导航成功后的onComplete的回调函数
        onComplete && onComplete(route)
        this.ensureURL()
        // 调用全局的 afterEach 钩子
        this.router.afterHooks.forEach(hook => {
          hook && hook(route, prev)
        })

        // fire ready cbs once
        if (!this.ready) {
          this.ready = true
          this.readyCbs.forEach(cb => {
            cb(route)
          })
        }
      },
      err => {
        if (onAbort) {
          onAbort(err)
        }
        if (err && !this.ready) {
          this.ready = true
          // Initial redirection should still trigger the onReady onSuccess
          // https://github.com/vuejs/vue-router/issues/3225
          if (!isNavigationFailure(err, NavigationFailureType.redirected)) {
            this.readyErrorCbs.forEach(cb => {
              cb(err)
            })
          } else {
            this.readyCbs.forEach(cb => {
              cb(route)
            })
          }
        }
      }
    )
  }

  confirmTransition (route: Route, onComplete: Function, onAbort?: Function) {
    const current = this.current
    // 用于中断导航
    const abort = err => {
      // changed after adding errors with
      // https://github.com/vuejs/vue-router/pull/3047 before that change,
      // redirect and aborted navigation would produce an err == null
      if (!isNavigationFailure(err) && isError(err)) {
        if (this.errorCbs.length) {
          this.errorCbs.forEach(cb => {
            cb(err)
          })
        } else {
          warn(false, 'uncaught error during route navigation:')
          console.error(err)
        }
      }
      onAbort && onAbort(err)
    }
    const lastRouteIndex = route.matched.length - 1
    const lastCurrentIndex = current.matched.length - 1
    // 判断是否是相同的路由，相同路由则不跳转
    if (
      isSameRoute(route, current) &&
      // in the case the route map has been dynamically appended to
      lastRouteIndex === lastCurrentIndex &&
      route.matched[lastRouteIndex] === current.matched[lastCurrentIndex]
    ) {
      this.ensureURL()
      return abort(createNavigationDuplicatedError(current, route))
    }
    
    // 当前路由记录和目标路由记录比较，解析出哪些路由的组件是需要更新，哪些不需要更新
    // updated: 不需要更新的组件路由
    // deactivated: 失活组件的路由
    // activated：要更新的组件路由
    // 注意matched存放的是路由记录对象（RouteRecord）
    const { updated, deactivated, activated } = resolveQueue(
      this.current.matched,
      route.matched
    )
    
    // 把要执行的钩子函数提取到queue队列中，稍后依次执行
    const queue: Array<?NavigationGuard> = [].concat(
      // in-component leave guards
      // 提取失活组件的 beforeRouteLeave 钩子函数
      extractLeaveGuards(deactivated),
      // global before hooks
      // 全局 beforeEach 钩子函数
      this.router.beforeHooks,
      // in-component update hooks
      // 提取要更新组件的 beforeRouteUpdate 钩子函数
      extractUpdateHooks(updated),
      // in-config enter guards
      // 全局 beforeEnter 钩子函数
      activated.map(m => m.beforeEnter),
      // async components
      // 解析异步组件
      resolveAsyncComponents(activated)
    )

    this.pending = route
    // 一个迭代器函数，在runQueue时会调用
    // 这里的hook就是钩子函数，next对应runQueue中的 " () => { step(index + 1) } "函数
    // 这就是为什么官方文档说一定要调用next才会执行下一个钩子函数。
    const iterator = (hook: NavigationGuard, next) => {
      if (this.pending !== route) {
        return abort(createNavigationCancelledError(current, route))
      }
      try {
        hook(route, current, (to: any) => {
          // next(false) 中断当前的导航
          if (to === false) {
            // next(false) -> abort navigation, ensure current URL
            this.ensureURL(true)
            abort(createNavigationAbortedError(current, route))
          } else if (isError(to)) {
            // next(Error)
            this.ensureURL(true)
            abort(to)
          } else if (
            typeof to === 'string' ||
            (typeof to === 'object' &&
              (typeof to.path === 'string' || typeof to.name === 'string'))
          ) {
            // next('/') or next({ path: '/' }) -> redirect
            // next('/') 或者 next({ path: '/' }): 跳转到一个不同的地址。当前的导航被中断，然后进行一个新的导航
            abort(createNavigationRedirectedError(current, route))
            if (typeof to === 'object' && to.replace) {
              this.replace(to)
            } else {
              this.push(to)
            }
          } else {
            // confirm transition and pass on the value
            // 执行下个导航钩子
            next(to)
          }
        })
      } catch (e) {
        abort(e)
      }
    }

    // 非常经典的异步函数队列化执行的模式（同步执行异步函数）、
    runQueue(queue, iterator, () => {
      // postEnterCbs 用于存放 beforeRouteEnter 钩子调用next时传入的回调函数。
      // 比如在beforeRouteEnter要访问组件实例，但此时时拿不到实例的，所以我们一般会以
      // next(vm => { ... })这种方式访问组件实列。
      const postEnterCbs = []
      const isValid = () => this.current === route
      // wait until async components are resolved before
      // extracting in-component enter guards
      // 当异步组件加载玩，会执行这个回调函数，也就是 runQueue 中的 cb
      // 这时就可以运行激活组件内钩子函数了
      const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
      const queue = enterGuards.concat(this.router.resolveHooks)
      runQueue(queue, iterator, () => {
        // 路由跳转完成
        if (this.pending !== route) {
          return abort(createNavigationCancelledError(current, route))
        }
        this.pending = null
        onComplete(route)
        if (this.router.app) {
          // 调用postEnterCbs中的函数，此时next中的回调就可以拿到组件实例了 
          this.router.app.$nextTick(() => {
            postEnterCbs.forEach(cb => {
              cb()
            })
          })
        }
      })
    })
  }
```
再来看看对导航钩子的处理。这里组要说说对`beforeRouteEnter`钩子函数的处理。我们知道在这个函数是不能获取到组件实例的，如果要获取组件实例，需要通过回调的方式获取。所以来看看源码的实现：
```js
function extractEnterGuards (
  activated: Array<RouteRecord>,
  cbs: Array<Function>,
  isValid: () => boolean
): Array<?Function> {
  return extractGuards(
    activated,
    'beforeRouteEnter',
    (guard, _, match, key) => {
      return bindEnterGuard(guard, match, key, cbs, isValid)
    }
  )
}
```
`extractEnterGuards`的实现和`extractLeaveGuards、extractUpdateHooks`，差不多，不同的时前者调用的是`bindEnterGuard`
```js
function bindEnterGuard (
  guard: NavigationGuard,
  match: RouteRecord,
  key: string,
  cbs: Array<Function>,
  isValid: () => boolean
): NavigationGuard {
  // 返回的routeEnterGuard函数就是iterator里执行的hook函数
  // 这里的guard函数就是我们定义在组件上的beforeRouteEnter钩子函数。
  // 当iterator执行hook时，就相当于执行routeEnterGuard，会先执行我们定义的钩子函数，然后把回调函数收集到cbs中，cbs就是postEnterCbs
  // 之后在执行next，这个next就是传入hook的第三个参数。
  return function routeEnterGuard (to, from, next) {
    return guard(to, from, cb => {
      if (typeof cb === 'function') {
        cbs.push(() => {
          // #750
          // if a router-view is wrapped with an out-in transition,
          // the instance may not have been registered at this time.
          // we will need to poll for registration until current route
          // is no longer valid.
          poll(cb, match.instances, key, isValid)
        })
      }
      next(cb)
    })
  }
}
```
在根路由组件重新渲染后，遍历`postEnterCbs`执行回调，每一个回调执行的时候，其实是执行 `poll(cb, match.instances, key, isValid)` 方法，因为考虑到一些了路由组件被套 transition 組件在一些缓动模式下不一定能拿到实例，所以用一个轮询方法不断去判断，直到能获取到组件实例，再去调用 cb，并把组件实例作为参数传入，这就是我们在回调函数中能拿到组件实例的原因。

最后来看看更新路由、触发渲染调用的`updateRoute`函数
```js
updateRoute (route: Route) {
  this.current = route
  this.cb && this.cb(route)
}
```
可以看到这个函数很简单，就是改变当前路由对象的值以及调用`cb`函数。这个`cb`函数之前在`VueRouter初始化`的时候就已近赋值了，还记得不？
```js
history.listen(route => {
  this.apps.forEach(app => {
    app._route = route
  })
})
```
所以当调用`cb`函数时会执行`app._route = route`这个赋值操作，而`_route`属性又是响应式的，所以就会触发组件渲染了。

### 结语
VueRouter源码的阅读笔记就写到这里了，虽然`RouterView和RouterLink`组件的源码没完全看完，但不影响对VueRouter原理的理解，所以这里就不写出来了[狗头保命]。即使没有逐行的去理解，但整体上算明白了VueRouter的原理了。

**总的来说VueRouter的原理就是通过`beforeCreate`钩子函数为Vue根实例注入路由，然后使用`defineReactive`方法将`_route`属性变为响应式字段。然后通过监听路由变化事件（popstate，hashchange）或手动改变路由（push，replace等）更新`_route`的值，从而触发`RouterView`重新渲染。**

**参考借鉴**
- [vue-analysis](https://github.com/ustbhuangyi/vue-analysis/tree/master/docs/v2/vue-router)
- [解密vue-router: 从源码开始](https://juejin.im/post/6844903603484770311#heading-9)