##### 初始化

```
src/core/instance/render.js
export function initRender (vm: Component) {
  ...
  const options = vm.$options
  const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
  const renderContext = parentVnode && parentVnode.context
  vm.$slots = resolveSlots(options._renderChildren, renderContext)
  vm.$scopedSlots = emptyObject
  ...
```

进行数据初始化

$slots最多保存了默认插槽的数据

$scopedSlots保存的各个具名slot的构造函数

```
src/core/instance/init.js
export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  ...
}
```

_renderChildren来源于内部组件解析，对应组件下的默认slot数据

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
  ...
```

render时通过normalizeScopedSlots生成$scorpedSlots并对$slots进行更新

```
src/core/vdom/helpers/normalize-scoped-slots.js
export function normalizeScopedSlots (
  slots: { [key: string]: Function } | void,
  normalSlots: { [key: string]: Array<VNode> },
  prevSlots?: { [key: string]: Function } | void
): any {
  let res
  const hasNormalSlots = Object.keys(normalSlots).length > 0
  const isStable = slots ? !!slots.$stable : !hasNormalSlots
  const key = slots && slots.$key
  if (!slots) {
    res = {}
  } else if (slots._normalized) {
    // fast path 1: child component re-render only, parent did not change
    return slots._normalized
  } else if (
    isStable &&
    prevSlots &&
    prevSlots !== emptyObject &&
    key === prevSlots.$key &&
    !hasNormalSlots &&
    !prevSlots.$hasNormal
  ) {
    // fast path 2: stable scoped slots w/ no normal slots to proxy,
    // only need to normalize once
    return prevSlots
  } else {
    res = {}
    for (const key in slots) {
      if (slots[key] && key[0] !== '$') {
        res[key] = normalizeScopedSlot(normalSlots, key, slots[key])
      }
    }
  }
  // expose normal slots on scopedSlots
  for (const key in normalSlots) {
    if (!(key in res)) {
      res[key] = proxyNormalSlot(normalSlots, key)
    }
  }
  // avoriaz seems to mock a non-extensible $scopedSlots object
  // and when that is passed down this would cause an error
  if (slots && Object.isExtensible(slots)) {
    (slots: any)._normalized = res
  }
  def(res, '$stable', isStable)
  def(res, '$key', key)
  def(res, '$hasNormal', hasNormalSlots)
  return res
}
```

将父组件节点的scopedSlots中存在的具名插槽，通过构造函数生成vnode数组后加入$slots

将父组件节点的scopedSlots中存在的具名插槽与$slots中默认插槽封装成构造函数后合并

##### 更新

```
src/core/instance/lifecycle.js
export function updateChildComponent (
  vm: Component,
  propsData: ?Object,
  listeners: ?Object,
  parentVnode: MountedComponentVNode,
  renderChildren: ?Array<VNode>
) {
  ...

  // Any static slot children from the parent may have changed during parent's
  // update. Dynamic scoped slots may also have changed. In such cases, a forced
  // update is necessary to ensure correctness.
  const needsForceUpdate = !!(
    renderChildren ||               // has new static slots
    vm.$options._renderChildren ||  // has old static slots
    hasDynamicScopedSlot
  )

  ...
  
  vm.$options._renderChildren = renderChildren

  ...

  // resolve slots + force update if has children
  if (needsForceUpdate) {
    vm.$slots = resolveSlots(renderChildren, parentVnode.context)
    vm.$forceUpdate()
  }

  ...
}
```

若默认插槽数据存在则更新$slots，生成新的默认插槽数据，然后强制更新，将会在_render中去继续更新slot相关内容

##### 详情

```
父组件
<Child><div>default slot</div><template #second :name="name">second slot defaults</template></Child>

_c('child',{scopedSlots:_u([{key:"second",fn:function({ name }){return [_v("second slot")]},proxy:true}])},[_c('div',[_v("default slot")])])

子组件
<div><slot/><slot name="second"/></div>

_c('div',[_t("default"),_t("second",[_v("second slot defaults")],{"name":name})],2)
```

父组件vnode生成时将子组件vnode的默认插槽和具名插槽分开放置

```
{
  ...
  componentOptions: {
  	children: [ VNode ]
  	...
  },
  data: {
  	scopedSlots: { second: fn }
  	...
  }
  ...
}
```

vnode.componentOptions.children中放置的默认插槽的子节点

vnode.data.scopedSlots中放置的具名插槽的构造函数

当构造Vue实例与刷新时进行合并操作

当Vue实例调用render时将从vm.$scopedSlots中查找各插槽对应的构造函数，获取子节点数据

默认数据和作用域仅仅是在调用插槽的构造函数时传入了参数