```
<div v-my-directive:[argument].modify="value"></div>

_c('div',{directives:[{name:"my-directive",rawName:"v-my-directive:[argument].modify",value:(value),expression:"name === 10",arg:argument,modifiers:{"modify":true}}]
```

解析成render函数时生成directive存放在directives数组中，argument用中括号时为动态属性否则为字面量

使用自定义指令需以v-开头

```
src/core/vdom/modules/directives.js
export default {
  create: updateDirectives,
  update: updateDirectives,
  destroy: function unbindDirectives (vnode: VNodeWithData) {
    updateDirectives(vnode, emptyNode)
  }
}

function updateDirectives (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  if (oldVnode.data.directives || vnode.data.directives) {
    _update(oldVnode, vnode)
  }
}

function _update (oldVnode, vnode) {
  const isCreate = oldVnode === emptyNode
  const isDestroy = vnode === emptyNode
  const oldDirs = normalizeDirectives(oldVnode.data.directives, oldVnode.context)
  const newDirs = normalizeDirectives(vnode.data.directives, vnode.context)

  const dirsWithInsert = []
  const dirsWithPostpatch = []

  let key, oldDir, dir
  for (key in newDirs) {
    oldDir = oldDirs[key]
    dir = newDirs[key]
    if (!oldDir) {
      // new directive, bind
      callHook(dir, 'bind', vnode, oldVnode)
      if (dir.def && dir.def.inserted) {
        dirsWithInsert.push(dir)
      }
    } else {
      // existing directive, update
      dir.oldValue = oldDir.value
      dir.oldArg = oldDir.arg
      callHook(dir, 'update', vnode, oldVnode)
      if (dir.def && dir.def.componentUpdated) {
        dirsWithPostpatch.push(dir)
      }
    }
  }

  if (dirsWithInsert.length) {
    const callInsert = () => {
      for (let i = 0; i < dirsWithInsert.length; i++) {
        callHook(dirsWithInsert[i], 'inserted', vnode, oldVnode)
      }
    }
    if (isCreate) {
      mergeVNodeHook(vnode, 'insert', callInsert)
    } else {
      callInsert()
    }
  }

  if (dirsWithPostpatch.length) {
    mergeVNodeHook(vnode, 'postpatch', () => {
      for (let i = 0; i < dirsWithPostpatch.length; i++) {
        callHook(dirsWithPostpatch[i], 'componentUpdated', vnode, oldVnode)
      }
    })
  }

  if (!isCreate) {
    for (key in oldDirs) {
      if (!newDirs[key]) {
        // no longer present, unbind
        callHook(oldDirs[key], 'unbind', oldVnode, oldVnode, isDestroy)
      }
    }
  }
}
```

指令会在create、update、destroy三个阶段被触发，会添加至cbs各对应阶段数据中

1. 旧vnode为空节点时为创建流程，新vnode为空节点时为销毁流程
2. 对比新旧指令集合，仅存在于新vnode中的触发bind钩子存入新增指令中，新旧vnode均存在时触发update钩子存入更新指令中
3. 有新增的指令时，当为创建流程时创建一个data.hook.insert的调用，否则直接触发insert钩子（复用节点时）
4. 有更新的指令时，创建一个data.hook.postpatch的调用待节点更新patch完成后触发componentUpdated钩子
5. 有销毁的指令时，触发unbind钩子

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

创建vnode时均会调用cbs.create，触发directive的创建进而调用指令的bind钩子

当patch完成后触发data.hook.insert进而调用指令的insert钩子

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
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
  }
}
```

更新vnode時均会调用cbs.update和postpatch hook，触发directive的更新进而调用指令的update和componentUpdated钩子

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

销毁vnode时均会调用cbs.destroy，触发directive的销毁进而调用指令的unbind钩子