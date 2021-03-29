```
<div v-show="val"></div>

_c('div',{directives:[{name:"show",rawName:"v-show",value:(val),expression:"val"}]})
```

解析成render函数时生成一个directive描述

```
src/platforms/web/runtime/directives/show.js
export default {
  bind (el: any, { value }: VNodeDirective, vnode: VNodeWithData) {
    vnode = locateNode(vnode)
    const transition = vnode.data && vnode.data.transition
    const originalDisplay = el.__vOriginalDisplay =
      el.style.display === 'none' ? '' : el.style.display
    if (value && transition) {
      vnode.data.show = true
      enter(vnode, () => {
        el.style.display = originalDisplay
      })
    } else {
      el.style.display = value ? originalDisplay : 'none'
    }
  },

  update (el: any, { value, oldValue }: VNodeDirective, vnode: VNodeWithData) {
    /* istanbul ignore if */
    if (!value === !oldValue) return
    vnode = locateNode(vnode)
    const transition = vnode.data && vnode.data.transition
    if (transition) {
      vnode.data.show = true
      if (value) {
        enter(vnode, () => {
          el.style.display = el.__vOriginalDisplay
        })
      } else {
        leave(vnode, () => {
          el.style.display = 'none'
        })
      }
    } else {
      el.style.display = value ? el.__vOriginalDisplay : 'none'
    }
  },

  unbind (
    el: any,
    binding: VNodeDirective,
    vnode: VNodeWithData,
    oldVnode: VNodeWithData,
    isDestroy: boolean
  ) {
    if (!isDestroy) {
      el.style.display = el.__vOriginalDisplay
    }
  }
}
```

bind钩子进行了下列操作：

1. 若为组件vnode，从自身开始查找第一个有transition的vnode或非组件vnode
2. 若有动画时，执行入场动画（会直接结束入场动画）然后修改display
3. 若无动画时，直接修改display

update钩子进行了下列操作：

1. 若值未发生变化则跳过处理
2. 若有动画时，value为truly则执行入场动画否则执行出场动画然后修改display
3. 若无动画时，直接修改display

unbind钩子进行了下列操作：

若不是销毁阶段则将display设置为源显示值

##### bug

若vnode使用了v-show且vnode.elm发生了复用且有设置style的情况，vnode.elm.style.display会异常