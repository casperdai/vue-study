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

引用模块会在create、update、destroy三个阶段被触发，用于获取ref实例

1. 获取引用对象，若是组件获取vm实例，否则获取elm对象
2. 若为删除（引用发生变化或销毁时），在当前引用map中删除当前引用
3. 否则添加至引用map

根据refInfo决定每个引用关键字对应的为实例数组还是单一实例
