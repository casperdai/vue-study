

```
packages/runtime-core/src/scheduler.ts
let isFlushing = false
let isFlushPending = false

const queue: SchedulerJob[] = []
let flushIndex = 0

const pendingPreFlushCbs: SchedulerCb[] = []
let activePreFlushCbs: SchedulerCb[] | null = null
let preFlushIndex = 0

const pendingPostFlushCbs: SchedulerCb[] = []
let activePostFlushCbs: SchedulerCb[] | null = null
let postFlushIndex = 0

const resolvedPromise: Promise<any> = Promise.resolve()
let currentFlushPromise: Promise<void> | null = null

let currentPreFlushParentJob: SchedulerJob | null = null

const RECURSION_LIMIT = 100
```

完整的调度将任务分成了三个队列，分别是主队列`queue`、前置队列和后置队列（`active`标识已激活的回调即正在循环执行的，`pending`标识待激活即等待进入循环执行的）

主队列保存渲染更新任务，其他监听回调例如effect等按配置放置于前置队列或后置队列

`isFlushing`标识正在处理任务

`isFlushPending`标识已开启处理流程

`resolvedPromise`用于开启微任务，`currentFlushPromise`标识将要或正在执行的微任务

`currentPreFlushParentJob`标识触发前置队列的主任务即待执行的主任务，用于避免重复触发主任务

`RECURSION_LIMIT`用于调试模式下递归调用自身的次数限制，可定位死循环

##### queueJob

调度的核心处理，将更新任务推入队列并触发微任务进行处理

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

// #2768
// Use binary-search to find a suitable position in the queue,
// so that the queue maintains the increasing order of job's id,
// which can prevent the job from being skipped and also can avoid repeated patching.
function findInsertionIndex(job: SchedulerJob) {
  // the start index should be `flushIndex + 1`
  let start = flushIndex + 1
  let end = queue.length
  const jobId = getId(job)

  while (start < end) {
    const middle = (start + end) >>> 1
    const middleJobId = getId(queue[middle])
    middleJobId < jobId ? (start = middle + 1) : (end = middle)
  }

  return start
}
```

满足下列情况任务将被加入队列：

- 主队列为空或任务未在队列中
- 待入队列的任务不与待执行任务相同

对于检测是否已存在于队列中查询的起始索引满足下列情况：

- 任务不允许自调用时需保证未执行完数据中（正在执行的也是未执行完的）的唯一性，固起始索引为`flushIndex`
- 任务允许自调时需保证待执行数据中的唯一性
  - 未在遍历过程中时待执行数据为整个队列，固起始索引为`flushIndex`
  - 遍历过程中时待执行数据为`flushIndex`之后的数据，固起始索引为`flushIndex + 1`

因更新需保证是自上而下的，所以插入位置需比较`id`，`id`小的为先创建的

##### queueFlush

```
function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}

export function nextTick(
  this: ComponentPublicInstance | void,
  fn?: () => void
): Promise<void> {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(this ? fn.bind(this) : fn) : p
}
```

创建微任务

`nextTick`在有微任务的情况下必然在`currentFlushPromise`执行完后才会执行

##### flushJobs

```
packages/runtime-core/src/scheduler.ts
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

流程为`清空前置队列 -> 清空主队列 -> 清空后置队列`

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

因在`activePreFlushCbs`遍历时`pendingPreFlushCbs`可能会出现变化，固触发后需自调用

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

`flushPostFlushCbs`对`pendingPostFlushCbs`进行去重后按`id`排序后遍历调用

因`activePostFlushCbs`进行合并处理，固在`activePostFlushCbs`过程中再次触发`activePostFlushCbs`会将`pendingPostFlushCbs`直接去重后拼接，此时将不再进行排序按添加的先后顺序进行调用

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

export function queuePreFlushCb(cb: SchedulerCb) {
  queueCb(cb, activePreFlushCbs, pendingPreFlushCbs, preFlushIndex)
}

export function queuePostFlushCb(cb: SchedulerCbs) {
  queueCb(cb, activePostFlushCbs, pendingPostFlushCbs, postFlushIndex)
}
```

`queueCb`用于添加回调进队列并触发`flush`

`(cb as SchedulerJob).allowRecurse ? index + 1 : index`保证非可递归回调的唯一性