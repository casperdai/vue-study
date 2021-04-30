```
packages/runtime-core/src/scheduler.ts
let isFlushing = false
let isFlushPending = false
const resolvedPromise: Promise<any> = Promise.resolve()
let currentFlushPromise: Promise<void> | null = null

export function nextTick(
  this: ComponentPublicInstance | void,
  fn?: () => void
): Promise<void> {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(this ? fn.bind(this) : fn) : p
}

function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}
```

`loop`借助`Promise`实现，`queueFlush`触发`loop`

##### flushJobs

```
packages/runtime-core/src/scheduler.ts
const queue: SchedulerJob[] = []
let flushIndex = 0

function flushJobs(seen?: CountMap) {
  isFlushPending = false
  isFlushing = true

  flushPreFlushCbs(seen)

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child so its render effect will have smaller
  //    priority number)
  // 2. If a component is unmounted during a parent component's update,
  //    its update can be skipped.
  queue.sort((a, b) => getId(a) - getId(b))

  try {
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      if (job) {
        callWithErrorHandling(job, null, ErrorCodes.SCHEDULER)
      }
    }
  } finally {
    flushIndex = 0
    queue.length = 0

    flushPostFlushCbs(seen)

    isFlushing = false
    currentFlushPromise = null
    // some postFlushCb queued jobs!
    // keep flushing until it drains.
    if (queue.length || pendingPostFlushCbs.length) {
      flushJobs(seen)
    }
  }
}
```

`flushPreFlushCbs`、`queue`和`flushPostFlushCbs`共`3`个队列

`queue`主要用于更新组件，`SchedulerJob`根据触发时机在组件更新前或后触发

##### queueJob

```
packages/runtime-core/src/scheduler.ts
export function queueJob(job: SchedulerJob) {
  // the dedupe search uses the startIndex argument of Array.includes()
  // by default the search index includes the current job that is being run
  // so it cannot recursively trigger itself again.
  // if the job is a watch() callback, the search will start with a +1 index to
  // allow it recursively trigger itself - it is the user's responsibility to
  // ensure it doesn't end up in an infinite loop.
  if (
    (!queue.length ||
      !queue.includes(
        job,
        isFlushing && job.allowRecurse ? flushIndex + 1 : flushIndex
      )) &&
    job !== currentPreFlushParentJob
  ) {
    const pos = findInsertionIndex(job)
    if (pos > -1) {
      queue.splice(pos, 0, job)
    } else {
      queue.push(job)
    }
    queueFlush()
  }
}
```

`isFlushing && job.allowRecurse ? flushIndex + 1 : flushIndex`保证在`queue`在未进入遍历处理前`job`是唯一的

`job !== currentPreFlushParentJob`保证自身触发的`flushPreFlushCbs`不会再触发自身

##### queueCb

```
packages/runtime-core/src/scheduler.ts
function queueCb(
  cb: SchedulerCbs,
  activeQueue: SchedulerCb[] | null,
  pendingQueue: SchedulerCb[],
  index: number
) {
  if (!isArray(cb)) {
    if (
      !activeQueue ||
      !activeQueue.includes(
        cb,
        (cb as SchedulerJob).allowRecurse ? index + 1 : index
      )
    ) {
      pendingQueue.push(cb)
    }
  } else {
    // if cb is an array, it is a component lifecycle hook which can only be
    // triggered by a job, which is already deduped in the main queue, so
    // we can skip duplicate check here to improve perf
    pendingQueue.push(...cb)
  }
  queueFlush()
}

const pendingPreFlushCbs: SchedulerCb[] = []
let activePreFlushCbs: SchedulerCb[] | null = null
let preFlushIndex = 0

export function queuePreFlushCb(cb: SchedulerCb) {
  queueCb(cb, activePreFlushCbs, pendingPreFlushCbs, preFlushIndex)
}

const pendingPostFlushCbs: SchedulerCb[] = []
let activePostFlushCbs: SchedulerCb[] | null = null
let postFlushIndex = 0

export function queuePostFlushCb(cb: SchedulerCbs) {
  queueCb(cb, activePostFlushCbs, pendingPostFlushCbs, postFlushIndex)
}
```

`queueCb`用于添加回调进队列并触发`loop`，根据一定策略保证迭代中的队列数据的准确性

`(cb as SchedulerJob).allowRecurse ? index + 1 : index`保证非可递归触发的回调的唯一性

##### flushPreFlushCbs

```
packages/runtime-core/src/scheduler.ts
export function flushPreFlushCbs(
  seen?: CountMap,
  parentJob: SchedulerJob | null = null
) {
  if (pendingPreFlushCbs.length) {
    currentPreFlushParentJob = parentJob
    activePreFlushCbs = [...new Set(pendingPreFlushCbs)]
    pendingPreFlushCbs.length = 0
    if (__DEV__) {
      seen = seen || new Map()
    }
    for (
      preFlushIndex = 0;
      preFlushIndex < activePreFlushCbs.length;
      preFlushIndex++
    ) {
      if (__DEV__) {
        checkRecursiveUpdates(seen!, activePreFlushCbs[preFlushIndex])
      }
      activePreFlushCbs[preFlushIndex]()
    }
    activePreFlushCbs = null
    preFlushIndex = 0
    currentPreFlushParentJob = null
    // recursively flush until it drains
    flushPreFlushCbs(seen, parentJob)
  }
}
```

`flushPreFlushCbs`对`pendingPreFlushCbs`进行去重后按添加顺序进行调用

因按添加顺序进行调用，固数据变化的顺序等将影响`effect`触发次数

因在`activePreFlushCbs`遍历触发中`pendingPreFlushCbs`可能会出现变化，固触发后需自调用

因`activePreFlushCbs`无合并处理，固在`flushPreFlushCbs`过程中再次触发`flushPreFlushCbs`会中断前次的遍历

##### flushPostFlushCbs

```
packages/runtime-core/src/scheduler.ts
export function flushPostFlushCbs(seen?: CountMap) {
  if (pendingPostFlushCbs.length) {
    const deduped = [...new Set(pendingPostFlushCbs)]
    pendingPostFlushCbs.length = 0

    // #1947 already has active queue, nested flushPostFlushCbs call
    if (activePostFlushCbs) {
      activePostFlushCbs.push(...deduped)
      return
    }

    activePostFlushCbs = deduped
    if (__DEV__) {
      seen = seen || new Map()
    }

    activePostFlushCbs.sort((a, b) => getId(a) - getId(b))

    for (
      postFlushIndex = 0;
      postFlushIndex < activePostFlushCbs.length;
      postFlushIndex++
    ) {
      if (__DEV__) {
        checkRecursiveUpdates(seen!, activePostFlushCbs[postFlushIndex])
      }
      activePostFlushCbs[postFlushIndex]()
    }
    activePostFlushCbs = null
    postFlushIndex = 0
  }
}
```

`flushPostFlushCbs`对`flushPostFlushCbs`进行去重后按`id`排序后遍历调用

因`activePostFlushCbs`进行合并处理，固在`activePostFlushCbs`过程中再次触发`activePostFlushCbs`会将`pendingPostFlushCbs`直接去重后拼接，此时将不再进行排序按添加的先后顺序进行调用