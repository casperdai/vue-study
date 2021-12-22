```
packages/reactivity/src/effect.ts
export function effect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions = EMPTY_OBJ
): ReactiveEffect<T> {
  if (isEffect(fn)) {
    fn = fn.raw
  }
  const effect = createReactiveEffect(fn, options)
  if (!options.lazy) {
    effect()
  }
  return effect
}
```

副作用简单形容就是将`fn`与数据进行关联，数据变化时被重新调用

```
packages/reactivity/src/effect.ts
export interface ReactiveEffect<T = any> {
  (): T
  _isEffect: true
  id: number
  active: boolean
  raw: () => T
  deps: Array<Dep>
  options: ReactiveEffectOptions
  allowRecurse: boolean
}
```

`_isEffect`：ReactiveEffect实例标识

`active`：激活状态标识

`raw`：源函数

`deps`：依赖的数据

`allowRecurse`：当在源函数中触发了依赖数据变化时自身是否可再次被加入触发队列

```
packages/reactivity/src/effect.ts
export interface ReactiveEffectOptions {
  lazy?: boolean
  scheduler?: (job: ReactiveEffect) => void
  onTrack?: (event: DebuggerEvent) => void
  onTrigger?: (event: DebuggerEvent) => void
  onStop?: () => void
  allowRecurse?: boolean
}
```

`lazy`：懒执行，需自行调用后才进行依赖追踪

`scheduler`：调度副作用的调用时机，默认情况下为同步调用

`onStop`：副作用被停用时的回调

`onTrack`和`onTrigger`用于调试

##### createReactiveEffect

```
packages/reactivity/src/effect.ts
const effectStack: ReactiveEffect[] = []
let activeEffect: ReactiveEffect | undefined

function createReactiveEffect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions
): ReactiveEffect<T> {
  const effect = function reactiveEffect(): unknown {
    if (!effect.active) {
      return options.scheduler ? undefined : fn()
    }
    if (!effectStack.includes(effect)) {
      cleanup(effect)
      try {
        enableTracking()
        effectStack.push(effect)
        activeEffect = effect
        return fn()
      } finally {
        effectStack.pop()
        resetTracking()
        activeEffect = effectStack[effectStack.length - 1]
      }
    }
  } as ReactiveEffect
  effect.id = uid++
  effect.allowRecurse = !!options.allowRecurse
  effect._isEffect = true
  effect.active = true
  effect.raw = fn
  effect.deps = []
  effect.options = options
  return effect
}
```

`createReactiveEffect`将源函数进行封装，根据`active`决定是否进行依赖收集

`effectStack`维护当前调用中的副作用队列，通过`includes`避免死循环

`activeEffect`维护当前正在运行的副作用

`enableTracking`和`resetTracking`强制进入和回退依赖追踪模式

当一个副作用被触发时进行了下列操作：

1. 若已停用，无调度时运行源函数，跳出
2. 在调用中时，跳出
3. 清除现有依赖关联
4. 进入依赖追踪模式，加入`effectStack`，将`activeEffect`指向自身
5. 调用源函数，该过程中会进行依赖收集
6. 回退依赖追踪模式，从`effectStack`移出，回退`activeEffect`指向

##### track

```
packages/reactivity/src/effect.ts
const targetMap = new WeakMap<any, KeyToDepMap>()

export function track(target: object, type: TrackOpTypes, key: unknown) {
  if (!shouldTrack || activeEffect === undefined) {
    return
  }
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (!dep) {
    depsMap.set(key, (dep = new Set()))
  }
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
    if (__DEV__ && activeEffect.options.onTrack) {
      activeEffect.options.onTrack({
        effect: activeEffect,
        target,
        type,
        key
      })
    }
  }
}
```

操作响应式副本的`get`相关操作时会触发`trace`，维护`target`与`activeEffect`之间的依赖关联

`targetMap`以`target`为`key`记录收集器`depsMap`

`depsMap`保存`target`的哪些属性被外部引用，key为被引用的属性名，value为引用了属性的副作用Set容器`dep`

当允许进行依赖追踪并且`activeEffect`存在时，`dep`将记录`activeEffect`同时将自身加入`activeEffect.deps`完成依赖关联

##### trigger

```
packages/reactivity/src/effect.ts
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  const depsMap = targetMap.get(target)
  if (!depsMap) {
    // never been tracked
    return
  }

  const effects = new Set<ReactiveEffect>()
  const add = (effectsToAdd: Set<ReactiveEffect> | undefined) => {
    if (effectsToAdd) {
      effectsToAdd.forEach(effect => {
        if (effect !== activeEffect || effect.allowRecurse) {
          effects.add(effect)
        }
      })
    }
  }

  if (type === TriggerOpTypes.CLEAR) {
    // collection being cleared
    // trigger all effects for target
    depsMap.forEach(add)
  } else if (key === 'length' && isArray(target)) {
    depsMap.forEach((dep, key) => {
      if (key === 'length' || key >= (newValue as number)) {
        add(dep)
      }
    })
  } else {
    // schedule runs for SET | ADD | DELETE
    if (key !== void 0) {
      add(depsMap.get(key))
    }

    // also run for iteration key on ADD | DELETE | Map.SET
    switch (type) {
      case TriggerOpTypes.ADD:
        if (!isArray(target)) {
          add(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            add(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        } else if (isIntegerKey(key)) {
          // new index added to array -> length changes
          add(depsMap.get('length'))
        }
        break
      case TriggerOpTypes.DELETE:
        if (!isArray(target)) {
          add(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            add(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        }
        break
      case TriggerOpTypes.SET:
        if (isMap(target)) {
          add(depsMap.get(ITERATE_KEY))
        }
        break
    }
  }

  const run = (effect: ReactiveEffect) => {
    if (__DEV__ && effect.options.onTrigger) {
      effect.options.onTrigger({
        effect,
        target,
        key,
        type,
        newValue,
        oldValue,
        oldTarget
      })
    }
    if (effect.options.scheduler) {
      effect.options.scheduler(effect)
    } else {
      effect()
    }
  }

  effects.forEach(run)
}
```

操作响应式副本的`set`相关操作时会触发`trigger`，当`target`存在外部引用时根据`type`与`key`从`depsMap`获取相关副作用进行触发

副作用满足下列规则将被调用：

- `type === TriggerOpTypes.CLEAR`时，`depsMap`中所有副作用
- `target`为数组且`key === 'length'`时
  - `length`对应的副作用
  - 整型数值字符且值大于等于`newValue`的`key`对应的副作用
- 其他情况
  1. `key`为truly时所对应的副作用
  2. `type`
     - `TriggerOpTypes.ADD`：`target`不为数组时，`ITERATE_KEY`对应的副作用，`target`为Map容器时`MAP_KEY_ITERATE_KEY`对应的副作用，否则`key`为整型数值字符时`length`对应的副作用
     - `TriggerOpTypes.DELETE`：`target`不为数组时，`ITERATE_KEY`对应的副作用，`target`为Map容器时`MAP_KEY_ITERATE_KEY`对应的副作用
     - `TriggerOpTypes.SET`：`target`为Map容器时`ITERATE_KEY`对应的副作用