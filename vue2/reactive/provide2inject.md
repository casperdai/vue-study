##### provide

```
src/core/instance/inject.js
export function initProvide (vm: Component) {
  const provide = vm.$options.provide
  if (provide) {
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide
  }
}
```

作为提供者时，将在初始化后将provide挂载在_provided上

##### inject

```
src/core/instance/inject.js
export function initInjections (vm: Component) {
  const result = resolveInject(vm.$options.inject, vm)
  if (result) {
    toggleObserving(false)
    Object.keys(result).forEach(key => {
      defineReactive(vm, key, result[key])
    })
    toggleObserving(true)
  }
}

export function resolveInject (inject: any, vm: Component): ?Object {
  if (inject) {
    // inject is :any because flow is not smart enough to figure out cached
    const result = Object.create(null)
    const keys = hasSymbol
      ? Reflect.ownKeys(inject)
      : Object.keys(inject)

    for (let i = 0; i < keys.length; i++) {
      const key = keys[i]
      // #6574 in case the inject object is observed...
      if (key === '__ob__') continue
      const provideKey = inject[key].from
      let source = vm
      while (source) {
        if (source._provided && hasOwn(source._provided, provideKey)) {
          result[key] = source._provided[provideKey]
          break
        }
        source = source.$parent
      }
      if (!source) {
        if ('default' in inject[key]) {
          const provideDefault = inject[key].default
          result[key] = typeof provideDefault === 'function'
            ? provideDefault.call(vm)
            : provideDefault
        }
      }
    }
    return result
  }
}
```

inject在合并属性时已转化为{ from: key, default?: fn }格式

inject初始值是通过层层往上查询父vm._provided是否包含对应的key

因为inject是层层往上的查询所以可以进行provide的覆盖，取值由最近的提供者提供

##### 组件同时有provide和inject时，inject为什么不会取到自身的provide？

```
src/core/instance/init.js
Vue.prototype._init = function (options?: Object) {
  ...
  initInjections(vm) // resolve injections before data/props
  initState(vm)
  initProvide(vm) // // resolve provide after data/props
  ...
}
```

实例化时会优先进行inject的注入，此时自身的_provided不存在，所以不会取到自身的provide

##### inject为什么有时不是响应式的？

inject注入时是在非响应式模式下注入，则若provide对应的数据不是响应式时修改将不会触发更新

若provide提供的数据是对象，子组件若进行了修改，其他组件的inject因共享引用会造成数据污染