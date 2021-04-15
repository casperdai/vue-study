```
src/components/view.js
export default {
  name: 'RouterView',
  functional: true,
  props: {
    name: {
      type: String,
      default: 'default'
    }
  },
  render (_, { props, children, parent, data }) {
    // used by devtools to display a router-view badge
    data.routerView = true

    // directly use parent context's createElement() function
    // so that components rendered by router-view can resolve named slots
    const h = parent.$createElement
    const name = props.name
    const route = parent.$route
    const cache = parent._routerViewCache || (parent._routerViewCache = {})

    // determine current view depth, also check to see if the tree
    // has been toggled inactive but kept-alive.
    let depth = 0
    let inactive = false
    while (parent && parent._routerRoot !== parent) {
      const vnodeData = parent.$vnode ? parent.$vnode.data : {}
      if (vnodeData.routerView) {
        depth++
      }
      if (vnodeData.keepAlive && parent._directInactive && parent._inactive) {
        inactive = true
      }
      parent = parent.$parent
    }
    data.routerViewDepth = depth

    // render previous view if the tree is inactive and kept-alive
    if (inactive) {
      const cachedData = cache[name]
      const cachedComponent = cachedData && cachedData.component
      if (cachedComponent) {
        // #2301
        // pass props
        if (cachedData.configProps) {
          fillPropsinData(cachedComponent, data, cachedData.route, cachedData.configProps)
        }
        return h(cachedComponent, data, children)
      } else {
        // render previous empty view
        return h()
      }
    }

    const matched = route.matched[depth]
    const component = matched && matched.components[name]

    // render empty node if no matched route or no config component
    if (!matched || !component) {
      cache[name] = null
      return h()
    }

    // cache component
    cache[name] = { component }

    // attach instance registration hook
    // this will be called in the instance's injected lifecycle hooks
    data.registerRouteInstance = (vm, val) => {
      // val could be undefined for unregistration
      const current = matched.instances[name]
      if (
        (val && current !== vm) ||
        (!val && current === vm)
      ) {
        matched.instances[name] = val
      }
    }

    // also register instance in prepatch hook
    // in case the same component instance is reused across different routes
    ;(data.hook || (data.hook = {})).prepatch = (_, vnode) => {
      matched.instances[name] = vnode.componentInstance
    }

    // register instance in init hook
    // in case kept-alive component be actived when routes changed
    data.hook.init = (vnode) => {
      if (vnode.data.keepAlive &&
        vnode.componentInstance &&
        vnode.componentInstance !== matched.instances[name]
      ) {
        matched.instances[name] = vnode.componentInstance
      }

      // if the route transition has already been confirmed then we weren't
      // able to call the cbs during confirmation as the component was not
      // registered yet, so we call it here.
      handleRouteEntered(route)
    }

    const configProps = matched.props && matched.props[name]
    // save route and configProps in cache
    if (configProps) {
      extend(cache[name], {
        route,
        configProps
      })
      fillPropsinData(component, data, route, configProps)
    }

    return h(component, data, children)
  }
}
```

`router-view`根据`$route`选择对应的component创建vnode

```
// determine current view depth, also check to see if the tree
// has been toggled inactive but kept-alive.
let depth = 0
let inactive = false
while (parent && parent._routerRoot !== parent) {
  const vnodeData = parent.$vnode ? parent.$vnode.data : {}
  if (vnodeData.routerView) {
    depth++
  }
  if (vnodeData.keepAlive && parent._directInactive && parent._inactive) {
    inactive = true
  }
  parent = parent.$parent
}
data.routerViewDepth = depth
```

从自身往上查找，查询自身是从路由根节点开始的第几层，用于在嵌套路由中查询自身匹配的RouteRecord

```
// render previous view if the tree is inactive and kept-alive
if (inactive) {
  const cachedData = cache[name]
  const cachedComponent = cachedData && cachedData.component
  if (cachedComponent) {
    // #2301
    // pass props
    if (cachedData.configProps) {
      fillPropsinData(cachedComponent, data, cachedData.route, cachedData.configProps)
    }
    return h(cachedComponent, data, children)
  } else {
    // render previous empty view
    return h()
  }
}
```

`inactive`标识是否处于`keep-alive`中并且渲染过且当前是不活跃状态

若是则直接使用缓存的数据生成vnode

```
const matched = route.matched[depth]
const component = matched && matched.components[name]

// render empty node if no matched route or no config component
if (!matched || !component) {
  cache[name] = null
  return h()
}

// cache component
cache[name] = { component }
```

查询自身匹配的RouteRecord`matched`和对应的component

```
// attach instance registration hook
// this will be called in the instance's injected lifecycle hooks
data.registerRouteInstance = (vm, val) => {
  // val could be undefined for unregistration
  const current = matched.instances[name]
  if (
    (val && current !== vm) ||
    (!val && current === vm)
  ) {
    matched.instances[name] = val
  }
}
```

记录`matched`对应的组件实例，在`beforeCreate`和`destroyed`阶段被调用

- `val && current !== vm`保证不同组件实例才能修改缓存（切换不同路由时）
- `!val && current === vm`保证同组件实例才能修改缓存（未匹配到路由时）

```
// also register instance in prepatch hook
// in case the same component instance is reused across different routes
;(data.hook || (data.hook = {})).prepatch = function (_, vnode) {
  matched.instances[name] = vnode.componentInstance;
};
```

假如从`{ path: '/path1', component: Comp }`切换到`{ path: '/path2', component: Comp }`时将只会进行更新操作而不会触发`registerRouteInstance`，这时`path2`对应的RouteRecord的`matched.instances`丢失，所以需在`prepatch`时进行更新

```
// register instance in init hook
// in case kept-alive component be actived when routes changed
data.hook.init = (vnode) => {
  if (vnode.data.keepAlive &&
    vnode.componentInstance &&
    vnode.componentInstance !== matched.instances[name]
  ) {
    matched.instances[name] = vnode.componentInstance
  }

  // if the route transition has already been confirmed then we weren't
  // able to call the cbs during confirmation as the component was not
  // registered yet, so we call it here.
  handleRouteEntered(route)
}
```

假如在`keep-alive`中从`path1`切换到`{ path: '/path3', component: Comp2 }`后再切换到`path2`时，会走创建流程但是因`path1`和`path2`对应同一组件固会复用组件实例不会重新创建则`registerRouteInstance`将不会触发，不走更新流程则`prepatch`不会触发，则`path2`对应的RouteRecord的`matched.instances`丢失，所以需要`hook.init`进行对比更新

```
const configProps = matched.props && matched.props[name]
// save route and configProps in cache
if (configProps) {
  extend(cache[name], {
    route,
    configProps
  })
  fillPropsinData(component, data, route, configProps)
}
```

最后根据`props`的配置设置`data.props`和`data.attrs`