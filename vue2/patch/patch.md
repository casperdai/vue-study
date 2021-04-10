##### 生成vnode


```
Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    if (_parentVnode) {
      vm.$scopedSlots = normalizeScopedSlots(
        _parentVnode.data.scopedSlots,
        vm.$slots,
        vm.$scopedSlots
      )
    }

    // set parent vnode. this allows render functions to have access
    // to the data on the placeholder node.
    vm.$vnode = _parentVnode
    // render self
    let vnode
    try {
      // There's no need to maintain a stack because all render fns are called
      // separately from one another. Nested component's render fns are called
      // when parent component is patched.
      currentRenderingInstance = vm
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      // return error render result,
      // or previous vnode to prevent render error causing blank component
      vnode = vm._vnode
    } finally {
      currentRenderingInstance = null
    }
    // if the returned array contains only a single node, allow it
    if (Array.isArray(vnode) && vnode.length === 1) {
      vnode = vnode[0]
    }
    // return empty vnode in case the render function errored out
    if (!(vnode instanceof VNode)) {
      vnode = createEmptyVNode()
    }
    // set parent
    vnode.parent = _parentVnode
    return vnode
  }
```

定义CompA为\<div>\<CompB/>\</div>，CompB为\<div>\</div>，则CompA在render时children中会有一个组件vnode（E），CompB在render时\_parentVnode将指向E

##### 更新vnode


```
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }
```

_vnode代表当前正在使用的vnode

##### 部分变量说明

- vm.$vnode为父vnode
- vm.$options._parentVnode为父vnode
- vm.$options.parent为父vm
- vm._node为当前vnode
- vnode.parent为父vnode

##### 组件创建过程


```
src/core/vdom/patch.js
new Vue -> patch -> createElm -> createChildren -> createComponent -> hook.init -> hook.insert -> mounted
```

父子节点生命周期为：

父beforeCreate -> 父created -> 父beforeMount -> 子beforeCreate(hook.init) -> 子created -> 子beforeMount -> 子mounted(hook.insert) -> 父mounted

##### 组件更新流程

```
src/core/vdom/patch.js
patch -> patchVnode -> hook.prepatch -> updateChildren -> updated
```

