```
<div v-my-directive:[argument].modify="value"></div>

_c('div',{directives:[{name:"my-directive",rawName:"v-my-directive:[argument].modify",value:(value),expression:"name === 10",arg:argument,modifiers:{"modify":true}}]
```

解析成render函数时生成directive存放在directives数组中，argument用中括号时为动态属性否则为字面量

指令需以v-开头

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

指令模块会在create、update、destroy三个阶段被触发

1. 旧vnode为空节点时为创建流程，新vnode为空节点时为销毁流程
2. 获取新旧指令的描述
3. 对比新旧指令集合，仅存在于新vnode中的触发指令bind钩子并存入新增指令集合中，新旧vnode均存在时触发指令update钩子并存入更新指令集合中
4. 有新增的指令时，当为创建流程时创建一个data.hook.insert的调用，否则直接触发指令inserted钩子（复用节点时）
5. 有更新的指令时，创建一个data.hook.postpatch的调用待节点patch完成后触发指令componentUpdated钩子
6. 不是创建流程时，若旧指令集合中有不存在于新指令集合的指令时，触发指令unbind钩子

```
src/core/vdom/patch.js
function invokeInsertHook (vnode, queue, initial) {
  ...
    for (let i = 0; i < queue.length; ++i) {
      queue[i].data.hook.insert(queue[i])
    }
  ...
}
```

当patch完成后会触发data.hook.insert

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
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
  }
}
```

更新vnode時均会触发data.hook.postpatch
