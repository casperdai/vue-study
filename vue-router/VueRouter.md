```
types/router.d.ts
export declare class VueRouter {
  constructor(options?: RouterOptions)

  app: Vue
  options: RouterOptions
  mode: RouterMode
  currentRoute: Route

  beforeEach(guard: NavigationGuard): Function
  beforeResolve(guard: NavigationGuard): Function
  afterEach(hook: (to: Route, from: Route) => any): Function
  push(location: RawLocation): Promise<Route>
  replace(location: RawLocation): Promise<Route>
  push(
    location: RawLocation,
    onComplete?: Function,
    onAbort?: ErrorHandler
  ): void
  replace(
    location: RawLocation,
    onComplete?: Function,
    onAbort?: ErrorHandler
  ): void
  go(n: number): void
  back(): void
  forward(): void
  getMatchedComponents(to?: RawLocation | Route): Component[]
  onReady(cb: Function, errorCb?: ErrorHandler): void
  onError(cb: ErrorHandler): void
  addRoutes(routes: RouteConfig[]): void

  addRoute(parent: string, route: RouteConfig): void
  addRoute(route: RouteConfig): void
  getRoutes(): RouteRecordPublic[]

  resolve(
    to: RawLocation,
    current?: Route,
    append?: boolean
  ): {
    location: Location
    route: Route
    href: string
    // backwards compat
    normalizedTo: Location
    resolved: Route
  }

  static install: PluginFunction<never>
  static version: string

  static isNavigationFailure: (
    error: any,
    type?: number
  ) => error is NavigationFailure
  static NavigationFailureType: {
    [k in keyof typeof NavigationFailureType]: NavigationFailureType
  }

  static START_LOCATION: Route
}

src/index.js
export default class VueRouter {
  static install: () => void
  static version: string
  static isNavigationFailure: Function
  static NavigationFailureType: any
  static START_LOCATION: Route

  app: any
  apps: Array<any>
  ready: boolean
  readyCbs: Array<Function>
  options: RouterOptions
  mode: string
  history: HashHistory | HTML5History | AbstractHistory
  matcher: Matcher
  fallback: boolean
  beforeHooks: Array<?NavigationGuard>
  resolveHooks: Array<?NavigationGuard>
  afterHooks: Array<?AfterNavigationHook>

  constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    this.matcher = createMatcher(options.routes || [], this)

    let mode = options.mode || 'hash'
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

  ...

}
```

构造VueRouter时，主要通过`createMatcher`获取匹配器以及根据`mode`生成`history`对象

```
src/create-matcher.js
export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  
  ...
  
  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }

  function addRoute (parentOrRoute, route) {
    ...
  }

  function getRoutes () {
    return pathList.map(path => pathMap[path])
  }

  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    ...
  }
  
  ...
  
  return {
    match,
    addRoute,
    getRoutes,
    addRoutes
  }
}
```

匹配器根据`routes`通过`createRouteMap`生成路由相关映射，然后返回`match`、`addRoute`、`getRoutes`和`addRoutes`四个方法与VueRouter实例方法对应

##### createRouteMap

```
src/create-route-map.js
export function createRouteMap (
  routes: Array<RouteConfig>,
  oldPathList?: Array<string>,
  oldPathMap?: Dictionary<RouteRecord>,
  oldNameMap?: Dictionary<RouteRecord>,
  parentRoute?: RouteRecord
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

  routes.forEach(route => {
    addRouteRecord(pathList, pathMap, nameMap, route, parentRoute)
  })

  // ensure wildcard routes are always at the end
  for (let i = 0, l = pathList.length; i < l; i++) {
    if (pathList[i] === '*') {
      pathList.push(pathList.splice(i, 1)[0])
      l--
      i--
    }
  }

  return {
    pathList,
    pathMap,
    nameMap
  }
}
```

`addRouteRecord`根据routes生成`pathList`、`pathMap`和`nameMap`

pathList保存了所有RouteRecord实例的path

pathMap保存了RouteRecord实例与`path`的映射关系

nameMap保存了RouteRecord实例与`name`的映射关系

```
types/router.d.ts
export interface RouteRecord {
  path: string
  regex: RegExp
  components: Dictionary<Component>
  instances: Dictionary<Vue>
  name?: string
  parent?: RouteRecord
  redirect?: RedirectOption
  matchAs?: string
  meta: any
  beforeEnter?: (
    route: Route,
    redirect: (location: RawLocation) => void,
    next: () => void
  ) => any
  props:
    | boolean
    | Object
    | RoutePropsFunction
    | Dictionary<boolean | Object | RoutePropsFunction>
}

src/create-route-map.js
function addRouteRecord (
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>,
  route: RouteConfig,
  parent?: RouteRecord,
  matchAs?: string
) {
  const { path, name } = route

  const pathToRegexpOptions: PathToRegexpOptions =
    route.pathToRegexpOptions || {}
  const normalizedPath = normalizePath(path, parent, pathToRegexpOptions.strict)

  if (typeof route.caseSensitive === 'boolean') {
    pathToRegexpOptions.sensitive = route.caseSensitive
  }

  const record: RouteRecord = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    alias: route.alias
      ? typeof route.alias === 'string'
        ? [route.alias]
        : route.alias
      : [],
    instances: {},
    enteredCbs: {},
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

  if (route.children) {
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }

  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }

  if (route.alias !== undefined) {
    const aliases = Array.isArray(route.alias) ? route.alias : [route.alias]
    for (let i = 0; i < aliases.length; ++i) {
      const alias = aliases[i]

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

  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    }
  }
}
```

通过`addRouteRecord`生成RouteRecord实例，然后对`children`进行递归调用，最终可构成完整的路由映射

通过[path-to-regexp](https://www.npmjs.com/package/path-to-regexp)生成`regex`，这是通过路径匹配需使用的字段

保存时有如下规则：

- `normalizedPath`是完整路径，即当为子路由时会与父路由`path`进行拼接
- 同`path`路由，`pathMap`中后定义的丢弃
- 同`name`路由，`nameMap`中后定义的丢弃
- 有`alias`的会根据alias创建额外的RouteRecord实例

##### init

```
export default class VueRouter {

  ...
  
  init (app: any /* Vue component instance */) {
    process.env.NODE_ENV !== 'production' &&
      assert(
        install.installed,
        `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
          `before creating root instance.`
      )

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
      history.transitionTo(
        history.getCurrentLocation(),
        setupListeners,
        setupListeners
      )
    }

    history.listen(route => {
      this.apps.forEach(app => {
        app._route = route
      })
    })
  }
  
  ...
  
}
```

`init`主要是添加根实例关联与非`abstract` 模式触发初次跳转、监听路由变化