```
<div ref="key"></div>

_c('div',{ref:"key"})

<div v-for="id in 10" ref="key"></div>

_l((10),function(id){return _c('div',{ref:"key",refInFor:true})})
```

解析成render函数时生成ref字段，若在v-for中还会生成refInfo字段

```
src/core/vdom/modules/ref.js
export default {
  create (_: any, vnode: VNodeWithData) {
    registerRef(vnode)
  },
  update (oldVnode: VNodeWithData, vnode: VNodeWithData) {
    if (oldVnode.data.ref !== vnode.data.ref) {
      registerRef(oldVnode, true)
      registerRef(vnode)
    }
  },
  destroy (vnode: VNodeWithData) {
    registerRef(vnode, true)
  }
}

export function registerRef (vnode: VNodeWithData, isRemoval: ?boolean) {
  const key = vnode.data.ref
  if (!isDef(key)) return

  const vm = vnode.context
  const ref = vnode.componentInstance || vnode.elm
  const refs = vm.$refs
  if (isRemoval) {
    if (Array.isArray(refs[key])) {
      remove(refs[key], ref)
    } else if (refs[key] === ref) {
      refs[key] = undefined
    }
  } else {
    if (vnode.data.refInFor) {
      if (!Array.isArray(refs[key])) {
        refs[key] = [ref]
      } else if (refs[key].indexOf(ref) < 0) {
        // $flow-disable-line
        refs[key].push(ref)
      }
    } else {
      refs[key] = ref
    }
  }
}
```

引用会在create、update、destroy三个阶段被触发，会添加至cbs各对应阶段数据中

1. 获取引用对象，若是组件获取vm实例，否则获取elm对象
2. 若为删除（引用发生变化或销毁时），在当前引用map中删除当前引用
3. 否则添加至引用map

根据refInfo决定每个引用关键字对应的为实例数组还是单一实例

```
src/core/vdom/patch.js
function createElm (
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  ...
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }
  ...
  if (isDef(tag)) {
    ...
      invokeCreateHooks(vnode, insertedVnodeQueue)
    ...
  }
  ...
}

function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
  ...
    initComponent(vnode, insertedVnodeQueue)
  ...
}

function initComponent (vnode, insertedVnodeQueue) {
  ....
    invokeCreateHooks(vnode, insertedVnodeQueue)
  ...
}

function invokeCreateHooks (vnode, insertedVnodeQueue) {
  for (let i = 0; i < cbs.create.length; ++i) {
    cbs.create[i](emptyNode, vnode)
  }
  ...
    if (isDef(i.insert)) insertedVnodeQueue.push(vnode)
  ...
}

function invokeInsertHook (vnode, queue, initial) {
  ...
    for (let i = 0; i < queue.length; ++i) {
      queue[i].data.hook.insert(queue[i])
    }
  ...
}
```

创建vnode时均会调用cbs.create，触发ref的创建

```
src/core/vdom/patch.js
function patchVnode (
  oldVnode,
  vnode,
  insertedVnodeQueue,
  ownerArray,
  index,
  removeOnly
) {
  ...
  const oldCh = oldVnode.children
  const ch = vnode.children
  if (isDef(data) && isPatchable(vnode)) {
    for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
    if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
  }
  ...
}
```

更新vnode時均会调用cbs.update和postpatch hook，触发ref的更新

```
src/core/vdom/patch.js
function invokeDestroyHook (vnode) {
  let i, j
  const data = vnode.data
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.destroy)) i(vnode)
    for (i = 0; i < cbs.destroy.length; ++i) cbs.destroy[i](vnode)
  }
  if (isDef(i = vnode.children)) {
    for (j = 0; j < vnode.children.length; ++j) {
      invokeDestroyHook(vnode.children[j])
    }
  }
}
```

销毁vnode时均会调用cbs.destroy，触发ref的销毁