##### ref和shallowRef

```
packages/reactivity/src/ref.ts
export function ref(value?: unknown) {
  return createRef(value)
}

export function shallowRef(value?: unknown) {
  return createRef(value, true)
}

function createRef(rawValue: unknown, shallow = false) {
  if (isRef(rawValue)) {
    return rawValue
  }
  return new RefImpl(rawValue, shallow)
}

class RefImpl<T> {
  private _value: T

  public readonly __v_isRef = true

  constructor(private _rawValue: T, public readonly _shallow = false) {
    this._value = _shallow ? _rawValue : convert(_rawValue)
  }

  get value() {
    track(toRaw(this), TrackOpTypes.GET, 'value')
    return this._value
  }

  set value(newVal) {
    if (hasChanged(toRaw(newVal), this._rawValue)) {
      this._rawValue = newVal
      this._value = this._shallow ? newVal : convert(newVal)
      trigger(toRaw(this), TriggerOpTypes.SET, 'value', newVal)
    }
  }
}

const convert = <T extends unknown>(val: T): T =>
  isObject(val) ? reactive(val) : val
```

`ref`通过`getter`和`setter`进行`value`的操作拦截，进行依赖追踪和变更通知

非浅引用时嵌套对象的依赖追踪借助`reactive`实现

因变更判断将`newVal`进行了`toRaw`处理，所以当值为Proxy实例时容易造成意外变更通知

##### toRef和toRefs

```
packages/reactivity/src/ref.ts
export function toRef<T extends object, K extends keyof T>(
  object: T,
  key: K
): ToRef<T[K]> {
  return isRef(object[key])
    ? object[key]
    : (new ObjectRefImpl(object, key) as any)
}

export function toRefs<T extends object>(object: T): ToRefs<T> {
  if (__DEV__ && !isProxy(object)) {
    console.warn(`toRefs() expects a reactive object but received a plain one.`)
  }
  const ret: any = isArray(object) ? new Array(object.length) : {}
  for (const key in object) {
    ret[key] = toRef(object, key)
  }
  return ret
}

class ObjectRefImpl<T extends object, K extends keyof T> {
  public readonly __v_isRef = true

  constructor(private readonly _object: T, private readonly _key: K) {}

  get value() {
    return this._object[this._key]
  }

  set value(newVal) {
    this._object[this._key] = newVal
  }
}
```

`toRef`通过`getter`和`setter`进行`value`的操作拦截，建立映射`object[key]`的操作

###### `toRefs`的作用是什么呢，直接使用响应式对象不行吗？

模板为`<div>{{ val }}</div>`，`setup`中返回`{ val: reactiveProxy.val }`时，对`reactiveProxy.val`进行修改时模板将不会发生变化

因为返回的`val`是个原始类型值将无法进行依赖追踪，此时可通过`{ val: toRefs(reactiveProxy).val }`进行返回

##### customRef

```
packages/reactivity/src/ref.ts
export function customRef<T>(factory: CustomRefFactory<T>): Ref<T> {
  return new CustomRefImpl(factory) as any
}

export type CustomRefFactory<T> = (
  track: () => void,
  trigger: () => void
) => {
  get: () => T
  set: (value: T) => void
}

class CustomRefImpl<T> {
  private readonly _get: ReturnType<CustomRefFactory<T>>['get']
  private readonly _set: ReturnType<CustomRefFactory<T>>['set']

  public readonly __v_isRef = true

  constructor(factory: CustomRefFactory<T>) {
    const { get, set } = factory(
      () => track(this, TrackOpTypes.GET, 'value'),
      () => trigger(this, TriggerOpTypes.SET, 'value')
    )
    this._get = get
    this._set = set
  }

  get value() {
    return this._get()
  }

  set value(newVal) {
    this._set(newVal)
  }
}
```

`customRef`即自定义值的读取与存储同时自定义触发依赖追踪与变更通知的时机