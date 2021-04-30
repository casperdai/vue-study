```
packages/reactivity/src/reactive.ts
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (target && (target as Target)[ReactiveFlags.IS_READONLY]) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap
  )
}

export function readonly<T extends object>(
  target: T
): DeepReadonly<UnwrapNestedRefs<T>> {
  return createReactiveObject(
    target,
    true,
    readonlyHandlers,
    readonlyCollectionHandlers,
    readonlyMap
  )
}

export function shallowReactive<T extends object>(target: T): T {
  return createReactiveObject(
    target,
    false,
    shallowReactiveHandlers,
    shallowCollectionHandlers,
    shallowReactiveMap
  )
}

export function shallowReadonly<T extends object>(
  target: T
): Readonly<{ [K in keyof T]: UnwrapNestedRefs<T[K]> }> {
  return createReactiveObject(
    target,
    true,
    shallowReadonlyHandlers,
    shallowReadonlyCollectionHandlers,
    shallowReadonlyMap
  )
}
```

响应式对象通过`createReactiveObject`创建，不同的创建方式进行有不同的行为

```
packages/reactivity/src/reactive.ts
function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>
) {
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }
  // target is already a Proxy, return it.
  // exception: calling readonly() on a reactive object
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }
  // target already has corresponding Proxy
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }
  // only a whitelist of value types can be observed.
  const targetType = getTargetType(target)
  if (targetType === TargetType.INVALID) {
    return target
  }
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
  proxyMap.set(target, proxy)
  return proxy
}
```

响应式的原理是依赖`Proxy`，实例存在`WeakMap`中

`createReactiveObject`主要进行了下列操作：

1. 非对象时原值返回
2. 对象已被代理时，若非只读化或对象已被只读化时，原值返回（保证了对象同类型代理仅可进行一次并且readonly后不可再被代理）
3. 已存在代理实例时，返回已存在的代理实例
4. 对象不可被代理时（被标记忽略、不可扩展、不可代理的类型），原值返回
5. 创建代理实例，并以`target`为key存入Map
6. 返回代理实例

可知根据`targetType === TargetType.COLLECTION`代理会分为`collectionHandlers`和`baseHandlers`，简单来说就是分容器（Set、Map）和非容器

##### mutableHandlers

```
packages/reactivity/src/baseHandlers.ts
const get = createGetter()
const set = createSetter()
export const mutableHandlers: ProxyHandler<object> = {
  get,
  set,
  deleteProperty,
  has,
  ownKeys
}
```

对于非容器，一般情况下将代理5种行为

###### get

```
packages/reactivity/src/baseHandlers.ts
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: Target, key: string | symbol, receiver: object) {
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly
    } else if (
      key === ReactiveFlags.RAW &&
      receiver ===
        (isReadonly
          ? shallow
            ? shallowReadonlyMap
            : readonlyMap
          : shallow
            ? shallowReactiveMap
            : reactiveMap
        ).get(target)
    ) {
      return target
    }

    const targetIsArray = isArray(target)

    if (!isReadonly && targetIsArray && hasOwn(arrayInstrumentations, key)) {
      return Reflect.get(arrayInstrumentations, key, receiver)
    }

    const res = Reflect.get(target, key, receiver)

    if (
      isSymbol(key)
        ? builtInSymbols.has(key as symbol)
        : isNonTrackableKeys(key)
    ) {
      return res
    }

    if (!isReadonly) {
      track(target, TrackOpTypes.GET, key)
    }

    if (shallow) {
      return res
    }

    if (isRef(res)) {
      // ref unwrapping - does not apply for Array + integer key.
      const shouldUnwrap = !targetIsArray || !isIntegerKey(key)
      return shouldUnwrap ? res.value : res
    }

    if (isObject(res)) {
      // Convert returned value into a proxy as well. we do the isObject check
      // here to avoid invalid value warning. Also need to lazy access readonly
      // and reactive here to avoid circular dependency.
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}
```

被代理后可通过`ReactiveFlags.RAW`获取源数据

被代理后`ReactiveFlags.IS_REACTIVE`与`ReactiveFlags.IS_READONLY`互为相反值，标识当前代理的行为模式

`get`行为主要进行了下列操作：

1. `ReactiveFlags.IS_REACTIVE`、`ReactiveFlags.IS_READONLY`和`ReactiveFlags.RAW`的特殊处理，作为标识符用
2. 非只读模式下且`target`为数组并期望获取的是常用方法时，返回封装后的方法
3. 通过`Reflect.get`获取值
4. 若key为symbol且是内置symbol时或key为不可追踪的，返回值
5. 若是非只读模式，对`key`进行依赖追踪
6. 若是浅代理，返回值
7. 若值是Ref实例，根据条件返回实例或实例的值
8. 若值是对象，返回进行响应式处理后的代理实例
9. 其他情况，返回值

###### set

```
packages/reactivity/src/baseHandlers.ts
function createSetter(shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    let oldValue = (target as any)[key]
    if (!shallow) {
      value = toRaw(value)
      oldValue = toRaw(oldValue)
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    const hadKey =
      isArray(target) && isIntegerKey(key)
        ? Number(key) < target.length
        : hasOwn(target, key)
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
}
```

`set`行为主要进行了下列操作：

1. 获取当前值
2. 非浅代理时对Ref实例特殊处理，若旧值为Ref实例新值不是时，则更新Ref实例的值然后返回
3. 检测key是否已存在
4. 通过`Reflect.set`设置值
5. 非原型链操作时，若`key`存在触发 `TriggerOpTypes.ADD`，否则值有变化时触发`TriggerOpTypes.SET`
6. 返回设置结果

`target === toRaw(receiver)`保证触发事件的准确性

当`target`自身未拥有`key`时，原型链上存在且是在Proxy实例上时，会触发`Proxy.set`此时`receiver`为`target`对应的Proxy实例，当该`set`是通过`createSetter`创建时会触发意外事件

###### deleteProperty

```
packages/reactivity/src/baseHandlers.ts
function deleteProperty(target: object, key: string | symbol): boolean {
  const hadKey = hasOwn(target, key)
  const oldValue = (target as any)[key]
  const result = Reflect.deleteProperty(target, key)
  if (result && hadKey) {
    trigger(target, TriggerOpTypes.DELETE, key, undefined, oldValue)
  }
  return result
}
```

当`target`自身拥有`key`且删除成功触发`TriggerOpTypes.DELETE`

###### has

```
packages/reactivity/src/baseHandlers.ts
function has(target: object, key: string | symbol): boolean {
  const result = Reflect.has(target, key)
  if (!isSymbol(key) || !builtInSymbols.has(key)) {
    track(target, TrackOpTypes.HAS, key)
  }
  return result
}
```

使用`in`操作时，若key不为symbol且不是内置symbol时需对`key`进行依赖追踪

###### ownKeys

```
packages/reactivity/src/baseHandlers.ts
function ownKeys(target: object): (string | symbol)[] {
  track(target, TrackOpTypes.ITERATE, isArray(target) ? 'length' : ITERATE_KEY)
  return Reflect.ownKeys(target)
}
```

使用`Object.getOwnPropertyNames`或`Object.getOwnPropertySymbols`操作时，`target`为数组则对`length`否则对`ITERATE_KEY`进行依赖追踪

##### readonlyHandlers

```
packages/reactivity/src/baseHandlers.ts
const readonlyGet = createGetter(true)
export const readonlyHandlers: ProxyHandler<object> = {
  get: readonlyGet,
  set(target, key) {
    if (__DEV__) {
      console.warn(
        `Set operation on key "${String(key)}" failed: target is readonly.`,
        target
      )
    }
    return true
  },
  deleteProperty(target, key) {
    if (__DEV__) {
      console.warn(
        `Delete operation on key "${String(key)}" failed: target is readonly.`,
        target
      )
    }
    return true
  }
}
```

只读模式对`set`与`deleteProperty`行为进行屏蔽，嵌套对象将依然是只读模式

##### shallowReactiveHandlers

```
packages/reactivity/src/baseHandlers.ts
const shallowGet = createGetter(false, true)
const shallowSet = createSetter(true)
export const shallowReactiveHandlers: ProxyHandler<object> = extend(
  {},
  mutableHandlers,
  {
    get: shallowGet,
    set: shallowSet
  }
)
```

浅模式对`get`和`set`行为进行重新定义，将仅对对象的直接属性进行依赖追踪，嵌套对象将不进行响应式转换

##### shallowReadonlyHandlers

```
packages/reactivity/src/baseHandlers.ts
const shallowReadonlyGet = createGetter(true, true)
export const shallowReadonlyHandlers: ProxyHandler<object> = extend(
  {},
  readonlyHandlers,
  {
    get: shallowReadonlyGet
  }
)
```

浅只读模式对只读模式的`get`行为进行重新定义，将仅对对象的直接属性的`set`与`deleteProperty`行为进行屏蔽，嵌套对象将不进行响应式转换

##### 容器行为

```
packages/reactivity/src/collectionHandlers.ts
export const mutableCollectionHandlers: ProxyHandler<CollectionTypes> = {
  get: createInstrumentationGetter(false, false)
}

export const shallowCollectionHandlers: ProxyHandler<CollectionTypes> = {
  get: createInstrumentationGetter(false, true)
}

export const readonlyCollectionHandlers: ProxyHandler<CollectionTypes> = {
  get: createInstrumentationGetter(true, false)
}

export const shallowReadonlyCollectionHandlers: ProxyHandler<
  CollectionTypes
> = {
  get: createInstrumentationGetter(true, true)
}
```

对于容器，只代理了`get`行为

```
packages/reactivity/src/collectionHandlers.ts
function createInstrumentationGetter(isReadonly: boolean, shallow: boolean) {
  const instrumentations = shallow
    ? isReadonly
      ? shallowReadonlyInstrumentations
      : shallowInstrumentations
    : isReadonly
      ? readonlyInstrumentations
      : mutableInstrumentations

  return (
    target: CollectionTypes,
    key: string | symbol,
    receiver: CollectionTypes
  ) => {
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly
    } else if (key === ReactiveFlags.RAW) {
      return target
    }

    return Reflect.get(
      hasOwn(instrumentations, key) && key in target
        ? instrumentations
        : target,
      key,
      receiver
    )
  }
}
```

若期望获取的是容器的常用方法或属性时（存在于`instrumentations`中）返回封装后的方法否则通过`Reflect.get`获取

方法包括：`get`、`has`、`add`、`set`、`delete`、`clear`、`forEach`、`keys`、`values`和`entries`

属性包含：`size`

###### get

```
packages/reactivity/src/collectionHandlers.ts
function get(
  target: MapTypes,
  key: unknown,
  isReadonly = false,
  isShallow = false
) {
  // #1772: readonly(reactive(Map)) should return readonly + reactive version
  // of the value
  target = (target as any)[ReactiveFlags.RAW]
  const rawTarget = toRaw(target)
  const rawKey = toRaw(key)
  if (key !== rawKey) {
    !isReadonly && track(rawTarget, TrackOpTypes.GET, key)
  }
  !isReadonly && track(rawTarget, TrackOpTypes.GET, rawKey)
  const { has } = getProto(rawTarget)
  const wrap = isShallow ? toShallow : isReadonly ? toReadonly : toReactive
  if (has.call(rawTarget, key)) {
    return wrap(target.get(key))
  } else if (has.call(rawTarget, rawKey)) {
    return wrap(target.get(rawKey))
  }
}
```

`get`行为主要进行了下列操作：

1. 获取源容器
2. 获取源`key`，`key`为Proxy实例时需要
3. 非只读模式时，若`key !== rawKey`对`key`进行依赖追踪
4. 非只读模式时对`rawKey`进行依赖追踪
5. 若`key`存在，则通过原生`get`获取值并进行封装

###### has

```
packages/reactivity/src/collectionHandlers.ts
function has(this: CollectionTypes, key: unknown, isReadonly = false): boolean {
  const target = (this as any)[ReactiveFlags.RAW]
  const rawTarget = toRaw(target)
  const rawKey = toRaw(key)
  if (key !== rawKey) {
    !isReadonly && track(rawTarget, TrackOpTypes.HAS, key)
  }
  !isReadonly && track(rawTarget, TrackOpTypes.HAS, rawKey)
  return key === rawKey
    ? target.has(key)
    : target.has(key) || target.has(rawKey)
}
```

`has`行为主要进行了下列操作：

1. 获取源容器
2. 获取源`key`，`key`为Proxy实例时需要
3. 非只读模式时，若`key !== rawKey`对`key`进行依赖追踪
4. 非只读模式时对`rawKey`进行依赖追踪
5. 通过原生`has`返回结果

###### add

```
packages/reactivity/src/collectionHandlers.ts
function add(this: SetTypes, value: unknown) {
  value = toRaw(value)
  const target = toRaw(this)
  const proto = getProto(target)
  const hadKey = proto.has.call(target, value)
  if (!hadKey) {
    target.add(value)
    trigger(target, TriggerOpTypes.ADD, value, value)
  }
  return this
}
```

`add`行为主要进行了下列操作：

1. 获取源容器
2. 检测`key`是否存在，若`key`不存在通过原生`add`添加并触发`TriggerOpTypes.ADD`

###### set

```
packages/reactivity/src/collectionHandlers.ts
function set(this: MapTypes, key: unknown, value: unknown) {
  value = toRaw(value)
  const target = toRaw(this)
  const { has, get } = getProto(target)

  let hadKey = has.call(target, key)
  if (!hadKey) {
    key = toRaw(key)
    hadKey = has.call(target, key)
  } else if (__DEV__) {
    checkIdentityKeys(target, has, key)
  }

  const oldValue = get.call(target, key)
  target.set(key, value)
  if (!hadKey) {
    trigger(target, TriggerOpTypes.ADD, key, value)
  } else if (hasChanged(value, oldValue)) {
    trigger(target, TriggerOpTypes.SET, key, value, oldValue)
  }
  return this
}
```

`set`行为主要进行了下列操作：

1. 获取源`value`，`value`为Proxy实例时需要
2. 获取源容器
3. 检测`key`是否存在
4. 获取旧值并通过原生`set`设置新值
5. `key`不存在时，触发`TriggerOpTypes.ADD`，否则值有变化时触发`TriggerOpTypes.SET`

###### deleteEntry

```
packages/reactivity/src/collectionHandlers.ts
function deleteEntry(this: CollectionTypes, key: unknown) {
  const target = toRaw(this)
  const { has, get } = getProto(target)
  let hadKey = has.call(target, key)
  if (!hadKey) {
    key = toRaw(key)
    hadKey = has.call(target, key)
  } else if (__DEV__) {
    checkIdentityKeys(target, has, key)
  }

  const oldValue = get ? get.call(target, key) : undefined
  // forward the operation before queueing reactions
  const result = target.delete(key)
  if (hadKey) {
    trigger(target, TriggerOpTypes.DELETE, key, undefined, oldValue)
  }
  return result
}
```

`delete行为主要进行了下列操作：

1. 获取源容器
2. 检测`key`是否存在
3. 调用原生`delete`
4. `key`存在时，触发`TriggerOpTypes.DELETE`

###### clear

```
packages/reactivity/src/collectionHandlers.ts
function clear(this: IterableCollections) {
  const target = toRaw(this)
  const hadItems = target.size !== 0
  const oldTarget = __DEV__
    ? isMap(target)
      ? new Map(target)
      : new Set(target)
    : undefined
  // forward the operation before queueing reactions
  const result = target.clear()
  if (hadItems) {
    trigger(target, TriggerOpTypes.CLEAR, undefined, undefined, oldTarget)
  }
  return result
}
```

`clear`行为主要进行了下列操作：

1. 获取源容器
2. 调用原生`clear`
3. 若清空前有数据则触发`TriggerOpTypes.CLEAR`

###### forEach

```
packages/reactivity/src/collectionHandlers.ts
function createForEach(isReadonly: boolean, isShallow: boolean) {
  return function forEach(
    this: IterableCollections,
    callback: Function,
    thisArg?: unknown
  ) {
    const observed = this as any
    const target = observed[ReactiveFlags.RAW]
    const rawTarget = toRaw(target)
    const wrap = isShallow ? toShallow : isReadonly ? toReadonly : toReactive
    !isReadonly && track(rawTarget, TrackOpTypes.ITERATE, ITERATE_KEY)
    return target.forEach((value: unknown, key: unknown) => {
      // important: make sure the callback is
      // 1. invoked with the reactive map as `this` and 3rd arg
      // 2. the value received should be a corresponding reactive/readonly.
      return callback.call(thisArg, wrap(value), wrap(key), observed)
    })
  }
}
```

`forEach`行为主要进行了下列操作：

1. 获取源容器
2. 非只读模式时对`ITERATE_KEY`进行依赖追踪
3. 原生`forEach`调用，将`value`和`key`进行封装后传入回调

###### keys、values和entries

```
packages/reactivity/src/collectionHandlers.ts
function createIterableMethod(
  method: string | symbol,
  isReadonly: boolean,
  isShallow: boolean
) {
  return function(
    this: IterableCollections,
    ...args: unknown[]
  ): Iterable & Iterator {
    const target = (this as any)[ReactiveFlags.RAW]
    const rawTarget = toRaw(target)
    const targetIsMap = isMap(rawTarget)
    const isPair =
      method === 'entries' || (method === Symbol.iterator && targetIsMap)
    const isKeyOnly = method === 'keys' && targetIsMap
    const innerIterator = target[method](...args)
    const wrap = isShallow ? toShallow : isReadonly ? toReadonly : toReactive
    !isReadonly &&
      track(
        rawTarget,
        TrackOpTypes.ITERATE,
        isKeyOnly ? MAP_KEY_ITERATE_KEY : ITERATE_KEY
      )
    // return a wrapped iterator which returns observed versions of the
    // values emitted from the real iterator
    return {
      // iterator protocol
      next() {
        const { value, done } = innerIterator.next()
        return done
          ? { value, done }
          : {
              value: isPair ? [wrap(value[0]), wrap(value[1])] : wrap(value),
              done
            }
      },
      // iterable protocol
      [Symbol.iterator]() {
        return this
      }
    }
  }
}
```

主要进行了下列操作：

1. 获取源容器
2. 获取原生迭代器
3. 返回迭代器，将值进行封装

###### size

```
packages/reactivity/src/collectionHandlers.ts
function size(target: IterableCollections, isReadonly = false) {
  target = (target as any)[ReactiveFlags.RAW]
  !isReadonly && track(toRaw(target), TrackOpTypes.ITERATE, ITERATE_KEY)
  return Reflect.get(target, 'size', target)
}
```

`size`的获取主要进行了下列操作：

1. 获取源容器
2. 获取原生迭代器
3. 非只读模式时对`ITERATE_KEY`进行依赖追踪
4. 通过`Reflect.get`返回结果

###### instrumentations

```
packages/reactivity/src/collectionHandlers.ts
const mutableInstrumentations: Record<string, Function> = {
  get(this: MapTypes, key: unknown) {
    return get(this, key)
  },
  get size() {
    return size((this as unknown) as IterableCollections)
  },
  has,
  add,
  set,
  delete: deleteEntry,
  clear,
  forEach: createForEach(false, false)
}

const shallowInstrumentations: Record<string, Function> = {
  get(this: MapTypes, key: unknown) {
    return get(this, key, false, true)
  },
  get size() {
    return size((this as unknown) as IterableCollections)
  },
  has,
  add,
  set,
  delete: deleteEntry,
  clear,
  forEach: createForEach(false, true)
}

const readonlyInstrumentations: Record<string, Function> = {
  get(this: MapTypes, key: unknown) {
    return get(this, key, true)
  },
  get size() {
    return size((this as unknown) as IterableCollections, true)
  },
  has(this: MapTypes, key: unknown) {
    return has.call(this, key, true)
  },
  add: createReadonlyMethod(TriggerOpTypes.ADD),
  set: createReadonlyMethod(TriggerOpTypes.SET),
  delete: createReadonlyMethod(TriggerOpTypes.DELETE),
  clear: createReadonlyMethod(TriggerOpTypes.CLEAR),
  forEach: createForEach(true, false)
}

const shallowReadonlyInstrumentations: Record<string, Function> = {
  get(this: MapTypes, key: unknown) {
    return get(this, key, true, true)
  },
  get size() {
    return size((this as unknown) as IterableCollections, true)
  },
  has(this: MapTypes, key: unknown) {
    return has.call(this, key, true)
  },
  add: createReadonlyMethod(TriggerOpTypes.ADD),
  set: createReadonlyMethod(TriggerOpTypes.SET),
  delete: createReadonlyMethod(TriggerOpTypes.DELETE),
  clear: createReadonlyMethod(TriggerOpTypes.CLEAR),
  forEach: createForEach(true, true)
}
```

`mutableInstrumentations`：可进行依赖追踪，数据项会通过`reactive`处理

`shallowInstrumentations`：可进行依赖追踪，数据项不会进行处理

`readonlyInstrumentations`：仅可进行`get`相关操作且不会进行依赖追踪，数据项会通过`readonly`处理

`shallowReadonlyInstrumentations`：仅可进行`get`相关操作且不会进行依赖追踪，数据项不会进行处理

##### 小结

响应式依赖`Proxy`，通过`createReactiveObject`与`target`的类型创建不同代理行为的Proxy实例存于对应的`WeakMap`中

在操作Proxy实例过程中，`get`相关的操作中通过`trace`进行依赖追踪，在`set`相关的操作中通过`trigger`进行变更通知

readyonly封装后将不可再度封装，至多可进行两次封装（readonly(reactive(obj))）