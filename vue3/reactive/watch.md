```
packages/runtime-core/src/apiWatch.ts
function doWatch(
  source: WatchSource | WatchSource[] | WatchEffect | object,
  cb: WatchCallback | null,
  { immediate, deep, flush, onTrack, onTrigger }: WatchOptions = EMPTY_OBJ,
  instance = currentInstance
): WatchStopHandle {
  ...
  
  let getter: () => any

  ...

  const job: SchedulerJob = () => {
    ...
  }

  ...

  const runner = effect(getter, {
    lazy: true,
    onTrack,
    onTrigger,
    scheduler
  })

  recordInstanceBoundEffect

  // initial run
  if (cb) {
    if (immediate) {
      job()
    } else {
      oldValue = runner()
    }
  } else if (flush === 'post') {
    queuePostRenderEffect(runner, instance && instance.suspense)
  } else {
    runner()
  }

  return () => {
    stop(runner)
    if (instance) {
      remove(instance.effects!, runner)
    }
  }
}
```

监听器借助`effect`实现，主要步骤就是构造`effect`的源函数然后根据配置的触发时机进行第一次的触发

`source`将决定依赖关联的建立，`cb`为依赖变更时触发的回调

```
packages/runtime-core/src/apiWatch.ts
function doWatch(
  source: WatchSource | WatchSource[] | WatchEffect | object,
  cb: WatchCallback | null,
  { immediate, deep, flush, onTrack, onTrigger }: WatchOptions = EMPTY_OBJ,
  instance = currentInstance
): WatchStopHandle {
  ...

  let getter: () => any
  let forceTrigger = false
  if (isRef(source)) {
    getter = () => (source as Ref).value
    forceTrigger = !!(source as Ref)._shallow
  } else if (isReactive(source)) {
    getter = () => source
    deep = true
  } else if (isArray(source)) {
    getter = () =>
      source.map(s => {
        if (isRef(s)) {
          return s.value
        } else if (isReactive(s)) {
          return traverse(s)
        } else if (isFunction(s)) {
          return callWithErrorHandling(s, instance, ErrorCodes.WATCH_GETTER, [
            instance && (instance.proxy as any)
          ])
        } else {
          __DEV__ && warnInvalidSource(s)
        }
      })
  } else if (isFunction(source)) {
    if (cb) {
      // getter with cb
      getter = () =>
        callWithErrorHandling(source, instance, ErrorCodes.WATCH_GETTER, [
          instance && (instance.proxy as any)
        ])
    } else {
      // no cb -> simple effect
      getter = () => {
        if (instance && instance.isUnmounted) {
          return
        }
        if (cleanup) {
          cleanup()
        }
        return callWithAsyncErrorHandling(
          source,
          instance,
          ErrorCodes.WATCH_CALLBACK,
          [onInvalidate]
        )
      }
    }
  } else {
    getter = NOOP
    __DEV__ && warnInvalidSource(source)
  }

  if (cb && deep) {
    const baseGetter = getter
    getter = () => traverse(baseGetter())
  }
  
  ...
}

```

`getter`根据`source`进行生成

- 为引用实例时，`getter`为引用值的返回，且是浅引用时设置`forceTrigger`
- 为响应式对象时，设置`deep`，若无回调`getter`为值的返回，否则`getter`为深层遍历所有属性
- 为数组时，`getter`为数据的`map`遍历
  - 数据为引用实例时，获取`value`值
  - 数据为响应式对象时，深层遍历所有属性
  - 数据为方法时，带参数`instance.proxy`的调用
- 为方法时
  - 若有回调，`getter`为带参数`instance.proxy`的调用
  - 否则`getter`为带参数`onInvalidate`的调用，`onInvalidate`用于接收清除副作用的方法`fn`，`fn`在下次`getter`被调用或`effete`停用时触发
- 其他情况`getter`为空方法

```
let oldValue = isArray(source) ? [] : INITIAL_WATCHER_VALUE
const job: SchedulerJob = () => {
  if (!runner.active) {
    return
  }
  if (cb) {
    // watch(source, cb)
    const newValue = runner()
    if (deep || forceTrigger || hasChanged(newValue, oldValue)) {
      // cleanup before running cb again
      if (cleanup) {
        cleanup()
      }
      callWithAsyncErrorHandling(cb, instance, ErrorCodes.WATCH_CALLBACK, [
        newValue,
        // pass undefined as the old value when it's changed for the first time
        oldValue === INITIAL_WATCHER_VALUE ? undefined : oldValue,
        onInvalidate
      ])
      oldValue = newValue
    }
  } else {
    // watchEffect
    runner()
  }
}

// important: mark the job as a watcher callback so that scheduler knows
// it is allowed to self-trigger (#1727)
job.allowRecurse = !!cb
```

`job`用于在依赖的数据变更时触发`getter`和回调`cb`，当无回调时`job`无法递归调用（自身触发自身）

当`deep`或`forceTrigger`为`true`时或`getter`的返回值发生变化时才触发回调，传入旧值、新值与`onInvalidate`

```
let scheduler: ReactiveEffectOptions['scheduler']
if (flush === 'sync') {
  scheduler = job
} else if (flush === 'post') {
  scheduler = () => queuePostRenderEffect(job, instance && instance.suspense)
} else {
  // default: 'pre'
  scheduler = () => {
    if (!instance || instance.isMounted) {
      queuePreFlushCb(job)
    } else {
      // with 'pre' option, the first call must happen before
      // the component is mounted so it is called synchronously.
      job()
    }
  }
}

const runner = effect(getter, {
  lazy: true,
  onTrack,
  onTrigger,
  scheduler
})
```

根据`getter`和`scheduler`创建懒执行的`runner`

`scheduler`根据`flush`决定调度方法的实现，用于决定`job`的执行时机

- `sync`：同步，依赖数据变更立即执行`job`
- `post`：异步，通过`queuePostRenderEffect`决定`runner`的触发（在下次`loop`中组件更新完成后执行）
- `pre`：`instance`存在且未`isMounted`时同步，否则异步，下次`loop`中组件更新前执行

```
recordInstanceBoundEffect(runner, instance)

// initial run
if (cb) {
  if (immediate) {
    job()
  } else {
    oldValue = runner()
  }
} else if (flush === 'post') {
  queuePostRenderEffect(runner, instance && instance.suspense)
} else {
  runner()
}

return () => {
  stop(runner)
  if (instance) {
    remove(instance.effects!, runner)
  }
}
```

创建`runner`后，建立与`instance`的关联，返回取消函数

初次调用分下列情况：

- 有回调时
  - 若`immediate`时，调用`job`，将会触发`getter`和`cb`
  - 否则运行`runner`获取`oldValue`并进行依赖追踪
- `flush === 'post'`时，通过`queuePostRenderEffect`决定`runner`的调用
- 其他情况立即调用`runner`

##### watchEffect和watch

```
packages/runtime-core/src/apiWatch.ts
export function watchEffect(
  effect: WatchEffect,
  options?: WatchOptionsBase
): WatchStopHandle {
  return doWatch(effect, null, options)
}

export function watch<T = any, Immediate extends Readonly<boolean> = false>(
  source: T | WatchSource<T>,
  cb: any,
  options?: WatchOptions<Immediate>
): WatchStopHandle {
  if (__DEV__ && !isFunction(cb)) {
    warn(
      `\`watch(fn, options?)\` signature has been moved to a separate API. ` +
        `Use \`watchEffect(fn, options?)\` instead. \`watch\` now only ` +
        `supports \`watch(source, cb, options?) signature.`
    )
  }
  return doWatch(source as any, cb, options)
}
```

主要区别为是否可传入回调函数

因`watchEffect`不能传回调函数，固`watchEffect`的`option`仅支持`flush`

##### instanceWatch（vm.$watch）

```
packages/runtime-core/src/apiWatch.ts
export function instanceWatch(
  this: ComponentInternalInstance,
  source: string | Function,
  cb: WatchCallback,
  options?: WatchOptions
): WatchStopHandle {
  const publicThis = this.proxy as any
  const getter = isString(source)
    ? source.includes('.')
      ? createPathGetter(publicThis, source)
      : () => publicThis[source]
    : source.bind(publicThis)
  return doWatch(getter, cb.bind(publicThis), options, this)
}

export function createPathGetter(ctx: any, path: string) {
  const segments = path.split('.')
  return () => {
    let cur = ctx
    for (let i = 0; i < segments.length && cur; i++) {
      cur = cur[segments[i]]
    }
    return cur
  }
}
```

用于`vm`实例，根据`source`为字符串时将监听`proxy`上对应数据的变化