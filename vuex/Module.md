```
src/store.js
export class Store {
  constructor (options = {}) {
    ...
    this._modules = new ModuleCollection(options)
    ...
    installModule(this, state, [], this._modules.root)
    ...
  }
  ...
}

src/module/module-collection.js
export default class ModuleCollection {
  constructor (rawRootModule) {
    // register root module (Vuex.Store options)
    this.register([], rawRootModule, false)
  }

  get (path) {
    return path.reduce((module, key) => {
      return module.getChild(key)
    }, this.root)
  }

  getNamespace (path) {
    let module = this.root
    return path.reduce((namespace, key) => {
      module = module.getChild(key)
      return namespace + (module.namespaced ? key + '/' : '')
    }, '')
  }

  update (rawRootModule) {
    update([], this.root, rawRootModule)
  }

  register (path, rawModule, runtime = true) {
    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {
      this.root = newModule
    } else {
      const parent = this.get(path.slice(0, -1))
      parent.addChild(path[path.length - 1], newModule)
    }

    // register nested modules
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }

  unregister (path) {
    const parent = this.get(path.slice(0, -1))
    const key = path[path.length - 1]
    const child = parent.getChild(key)

    if (!child) {
      return
    }

    if (!child.runtime) {
      return
    }

    parent.removeChild(key)
  }

  isRegistered (path) {
    const parent = this.get(path.slice(0, -1))
    const key = path[path.length - 1]

    if (parent) {
      return parent.hasChild(key)
    }

    return false
  }
}
```

根据`rawRootModule`创建出`root`，然后根据`modules`递归解析组成树结构

`getNamespace`获取module的命名空间，以`namespaced`判断，以`key + /`拼接

根模块若有`namespaced`，命名空间固定为`''`

例如`{ module: { a: { namespaced: true } } }`则命名空间为`a/`

```
src/store.js
function installModule (store, rootState, path, module, hot) {
  const isRoot = !path.length
  const namespace = store._modules.getNamespace(path)

  // register in namespace map
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }

  const local = module.context = makeLocalContext(store, namespace, path)

  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```

为每个模块创建上下文`context`，然后遍历模块对应的`mutations`、`actions`和`getters`将其挂载至`store`，最后通过`installModule`处理`modules`

每个模块的`state`均会挂载至父模块的`state`上，若父模块`state`上有与模块同`key`属性，将被模块的`state`覆盖

```
src/store.js
/**
 * make localized dispatch, commit, getters and state
 * if there is no namespace, just use root ones
 */
function makeLocalContext (store, namespace, path) {
  const noNamespace = namespace === ''

  const local = {
    dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
      }

      return store.dispatch(type, payload)
    },

    commit: noNamespace ? store.commit : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
      }

      store.commit(type, payload, options)
    }
  }

  // getters and state object must be gotten lazily
  // because they will be changed by vm update
  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters
        : () => makeLocalGetters(store, namespace)
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })

  return local
}
```

重定义`dispatch`和`commit`，默认情况下会将`namespace`和`type`进行拼接

`getters`进行代理，在`store.state.getters`上通过`namespace`匹配，会获取到自身命名空间下的所有getters

例如`{ module: { a: { namespaced: true, getters: { akey: 1 } } }, { module: { b: { namespaced: true, getters: { bkey: 1 } } } } }`

a模块的`context.getters`将包含`{ akey, b/bkey }`

```
src/store.js
module.forEachMutation((mutation, key) => {
  const namespacedType = namespace + key
  registerMutation(store, namespacedType, mutation, local)
})
  
function registerMutation (store, type, handler, local) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function wrappedMutationHandler (payload) {
    handler.call(store, local.state, payload)
  })
}
```

`mutation`将通过`namespace + key`进行挂载，接收自身上下文的`state`

```
src/store.js
module.forEachAction((action, key) => {
  const type = action.root ? key : namespace + key
  const handler = action.handler || action
  registerAction(store, type, handler, local)
})

function registerAction (store, type, handler, local) {
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler (payload) {
    let res = handler.call(store, {
      dispatch: local.dispatch,
      commit: local.commit,
      getters: local.getters,
      state: local.state,
      rootGetters: store.getters,
      rootState: store.state
    }, payload)
    if (!isPromise(res)) {
      res = Promise.resolve(res)
    }
    if (store._devtoolHook) {
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error', err)
        throw err
      })
    } else {
      return res
    }
  })
}
```

`action`将通过`namespace + key`或`key`进行挂载，接收自身上下文`context`与`store.state`、`store.getters`组合的对象，返回的必定是个Promise实例

```
src/store.js
module.forEachGetter((getter, key) => {
  const namespacedType = namespace + key
  registerGetter(store, namespacedType, getter, local)
})

function registerGetter (store, type, rawGetter, local) {
  if (store._wrappedGetters[type]) {
    return
  }
  store._wrappedGetters[type] = function wrappedGetter (store) {
    return rawGetter(
      local.state, // local state
      local.getters, // local getters
      store.state, // root state
      store.getters // root getters
    )
  }
}
```

`getter`将通过`namespace + key`进行挂载，接收自身上下文的`state`和`getter`以及`store.state`和`store.getters`

因`modules`嵌套以及`namespace`可造成同key的getter，定义在后面的将丢失使用已定义的getter

##### 小结

其实模块就是个套娃的过程，state根据层级嵌套，`mutation`和`action`、`getter`根据`namespace`来决定挂载的key