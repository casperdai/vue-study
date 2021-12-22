```
types/index.d.ts
export declare class Store<S> {
  constructor(options: StoreOptions<S>);

  readonly state: S;
  readonly getters: any;

  replaceState(state: S): void;

  dispatch: Dispatch;
  commit: Commit;

  subscribe<P extends MutationPayload>(fn: (mutation: P, state: S) => any, options?: SubscribeOptions): () => void;
  subscribeAction<P extends ActionPayload>(fn: SubscribeActionOptions<P, S>, options?: SubscribeOptions): () => void;
  watch<T>(getter: (state: S, getters: any) => T, cb: (value: T, oldValue: T) => void, options?: WatchOptions): () => void;

  registerModule<T>(path: string, module: Module<T, S>, options?: ModuleOptions): void;
  registerModule<T>(path: string[], module: Module<T, S>, options?: ModuleOptions): void;

  unregisterModule(path: string): void;
  unregisterModule(path: string[]): void;

  hasModule(path: string): boolean;
  hasModule(path: string[]): boolean;

  hotUpdate(options: {
    actions?: ActionTree<S, S>;
    mutations?: MutationTree<S>;
    getters?: GetterTree<S, S>;
    modules?: ModuleTree<S>;
  }): void;
}

src/store.js
export class Store {
  constructor (options = {}) {
    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue)
    }

    const {
      plugins = [],
      strict = false
    } = options

    // store internal state
    this._committing = false
    this._actions = Object.create(null)
    this._actionSubscribers = []
    this._mutations = Object.create(null)
    this._wrappedGetters = Object.create(null)
    this._modules = new ModuleCollection(options)
    this._modulesNamespaceMap = Object.create(null)
    this._subscribers = []
    this._watcherVM = new Vue()
    this._makeLocalGettersCache = Object.create(null)

    // bind commit and dispatch to self
    const store = this
    const { dispatch, commit } = this
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }

    // strict mode
    this.strict = strict

    const state = this._modules.root.state

    // init root module.
    // this also recursively registers all sub-modules
    // and collects all module getters inside this._wrappedGetters
    installModule(this, state, [], this._modules.root)

    // initialize the store vm, which is responsible for the reactivity
    // (also registers _wrappedGetters as computed properties)
    resetStoreVM(this, state)

    // apply plugins
    plugins.forEach(plugin => plugin(this))
  }

  get state () {
    return this._vm._data.$$state
  }

  ...
}
```

初始化主要解析传入的参数，根据参数进行模块划分等处理

```
src/store.js
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm

  // bind store public getters
  store.getters = {}
  // reset local getters cache
  store._makeLocalGettersCache = Object.create(null)
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    // direct inline function use will lead to closure preserving oldVm.
    // using partial to return function with only arguments preserved in closure environment.
    computed[key] = partial(fn, store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent

  // enable strict mode for new vm
  if (store.strict) {
    enableStrictMode(store)
  }

  if (oldVm) {
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    Vue.nextTick(() => oldVm.$destroy())
  }
}
```

`store.state => store._vm._data.$$state`

`store.getters[key] => store._vm[key] => computed[key] `

vuex的动态响应原理其实是vue的响应式处理，通过vue实例来实现

```
this.commit = function boundCommit (type, payload, options) {
  return commit.call(store, type, payload, options)
}

commit (_type, _payload, _options) {
  // check object-style commit
  const {
    type,
    payload,
    options
  } = unifyObjectStyle(_type, _payload, _options)

  const mutation = { type, payload }
  const entry = this._mutations[type]
  if (!entry) {
    return
  }
  this._withCommit(() => {
    entry.forEach(function commitIterator (handler) {
      handler(payload)
    })
  })

  this._subscribers
    .slice() // shallow copy to prevent iterator invalidation if subscriber synchronously calls unsubscribe
    .forEach(sub => sub(mutation, this.state))
}
```

重定义`commit`是为了当`commit`被作为参数传递时作用域依然是`store`

`commit`调用只是根据`type`查询对应的`mutatio`然后进行调用

```
this.dispatch = function boundDispatch (type, payload) {
  return dispatch.call(store, type, payload)
}

dispatch (_type, _payload) {
  // check object-style dispatch
  const {
    type,
    payload
  } = unifyObjectStyle(_type, _payload)

  const action = { type, payload }
  const entry = this._actions[type]
  if (!entry) {
    return
  }

  try {
    this._actionSubscribers
      .slice() // shallow copy to prevent iterator invalidation if subscriber synchronously calls unsubscribe
      .filter(sub => sub.before)
      .forEach(sub => sub.before(action, this.state))
  } catch (e) { }

  const result = entry.length > 1
    ? Promise.all(entry.map(handler => handler(payload)))
    : entry[0](payload)

  return new Promise((resolve, reject) => {
    result.then(res => {
      try {
        this._actionSubscribers
          .filter(sub => sub.after)
          .forEach(sub => sub.after(action, this.state))
      } catch (e) { }
      resolve(res)
    }, error => {
      try {
        this._actionSubscribers
          .filter(sub => sub.error)
          .forEach(sub => sub.error(action, this.state, error))
      } catch (e) { }
      reject(error)
    })
  })
}
```

重定义`dispatch`是为了当`dispatch`被作为参数传递时作用域依然是`store`

`dispatch`调用只是根据`type`查询对应的`action`然后进行调用最终返回个Promise实例

##### 小结

vuex可以认为就是通过代理一个"隐藏"的vue实例的data和computed，然后提供函数来操作这个vue实例以达到单向数据流、数据共享以及动态更新