使用`src/core/observer`下的`watcher.js`与`dep.js`进行订阅和更新（观察者模式）

- 对象劫持代理`setter`和`getter`
- 数组劫持代理`push`、pop、`shift`、`unshift`、`splice`、`sort`、`reverse`（直接操作数组无法动态响应的原因是无法劫持代理）

##### Watcher

- lazy模式：用于计算属性，更新时只是设置标识位`dirty`，被引用时才进行计算
- sync模式：立即响应
- 其他：发生变化后进入等待队列，下一个`loop`进行更新

##### Dep

收集与更新Watcher

##### 订阅与更新过程

1. 数据通过`observe`进行劫持代理
2. `Watcher`初始化时调用`get`方法（计算属性在被使用时触发）
3. `Watcher`将自身入栈（`pushTarget`）并调用创建时传入的`getter`
4. 数据`getter`被触发时，调用`Dep.depend`收集栈顶的`Watcher`
5. `Watcher`将自身出栈（`popTarget`）
7. 数据`setter`被触发时若值发生变化，通过`Dep.notify`调用收集的`Watcher`的`update`方法进行更新

```
订阅
watcher.get -> Dep.target = watcher -> prop getter -> dep.depend -> Dep.target.addDep -> watcher保存dep -> dep.addSub -> dep保存watcher -> Dep.target = null

更新
prop setter -> dep.notify -> watcher.update -> watcher.get
```

每次更新时`watcher`都会维护一个新`Dep`数组，收集后与现有数组进行对比，取消旧数组不存在于新数组的`Dep`

##### 调度

`Watcher.update`时，默认情况下将会通过`queueWatcher`将自身加入待执行队列

```
src/core/observer/scheduler.js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

通过`has`可保证一次`loop`中，同`watcher`在被执行一次前只会被添加一次，关联的多个数据同时变化时只会执行一次回调的原因

```
src/core/observer/scheduler.js
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)
}
```

`queue.sort((a, b) => a.id - b.id)`保证先创建的先执行，这是嵌套组件时父组件优先于子组件进行更新使子组件数据保持正确的原因

同时`watcher`的创建顺序可会影响调用的次数，例如`wather_a`依赖数据`a`，`wather_b`依赖数据`b`回调会修改数据`a`

针对修改`a`再修改`b`的操作，若`wather_a`先于`wather_b`创建时，`wather_a`会被调用`2`次

因`has[id] = null`先于`watcher.run()`，固`watcher`的`getter`与回调改变自身依赖的数据时可能造成循环调用

因`resetSchedulerState`先于`callActivatedHooks`和`callUpdatedHooks`，固`activated`与`updated`钩子的数据变化将在下次`loop`中响应