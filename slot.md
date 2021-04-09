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
}
```

$slots保存了未定义作用域的插槽的节点数据

$scopedSlots保存了插槽节点数据的构造函数

```
<component><div></div></component>
_c('component',[_c('div')])

<component><template><div></div></template></component>
_c('component',[[_c('div')]],2)

<component><template #default><div></div></template></component>
_c('component',{scopedSlots:_u([{key:"default",fn:function(){return [_c('div')]},proxy:true}])})

<component><template #slotKey><div></div></template></component>
_c('component',{scopedSlots:_u([{key:"slotKey",fn:function(){return [_c('div')]},proxy:true}])})

<component><template #slotKey="scopedProps"><div></div></template></component>
_c('component',{scopedSlots:_u([{key:"slotKey",fn:function(scopedProps){return [_c('div')]}}])})

<component><template v-slot:[slotKey]><div></div></template></component>
_c('component',{scopedSlots:_u([{key:slotKey,fn:function(){return [_c('div')]},proxy:true}],null,true)})

<component><div slot="slotKey"></div></component>
_c('component',[_c('div',{attrs:{"slot":"slotKey"},slot:"slotKey"})])

<slot :prop="prop" v-bind="props"><div></div></slot>
_t("default",[_c('div')],{"prop":prop},props)
```

废弃的slot语法以及新语法的具名插槽不会解析成构造函数存储在data.scopedSlots中，需进行slots解析

```
_u = resolveScopedSlots

src/core/instance/render-helpers/resolve-scoped-slots.js
export function resolveScopedSlots (
  fns: ScopedSlotsData, // see flow/vnode
  res?: Object,
  // the following are added in 2.6
  hasDynamicKeys?: boolean,
  contentHashKey?: number
): { [key: string]: Function, $stable: boolean } {
  res = res || { $stable: !hasDynamicKeys }
  for (let i = 0; i < fns.length; i++) {
    const slot = fns[i]
    if (Array.isArray(slot)) {
      resolveScopedSlots(slot, res, hasDynamicKeys)
    } else if (slot) {
      // marker for reverse proxying v-slot without scope on this.$slots
      if (slot.proxy) {
        slot.fn.proxy = true
      }
      res[slot.key] = slot.fn
    }
  }
  if (contentHashKey) {
    (res: any).$key = contentHashKey
  }
  return res
}
```

构造scopedSlots，根据是否有动态变更的情况来决定是否为稳定的scopedSlots

动态变更情况包括组件有v-for，动态slot，slot有v-if和v-for等

```
_t = renderSlot

src/core/instance/render-helpers/render-slot.js
export function renderSlot (
  name: string,
  fallback: ?Array<VNode>,
  props: ?Object,
  bindObject: ?Object
): ?Array<VNode> {
  const scopedSlotFn = this.$scopedSlots[name]
  let nodes
  if (scopedSlotFn) { // scoped slot
    props = props || {}
    if (bindObject) {
      props = extend(extend({}, bindObject), props)
    }
    nodes = scopedSlotFn(props) || fallback
  } else {
    nodes = this.$slots[name] || fallback
  }

  const target = props && props.slot
  if (target) {
    return this.$createElement('template', { slot: target }, nodes)
  } else {
    return nodes
  }
}
```

根据key从$scopedSlots中获得构造函数传入prop获取子节点或从$slots直接获取子节点

未定义slot时使用后备内容

```
src/core/instance/render-helpers/resolve-slots.js
export function resolveSlots (
  children: ?Array<VNode>,
  context: ?Component
): { [key: string]: Array<VNode> } {
  if (!children || !children.length) {
    return {}
  }
  const slots = {}
  for (let i = 0, l = children.length; i < l; i++) {
    const child = children[i]
    const data = child.data
    // remove slot attribute if the node is resolved as a Vue slot node
    if (data && data.attrs && data.attrs.slot) {
      delete data.attrs.slot
    }
    // named slots should only be respected if the vnode was rendered in the
    // same context.
    if ((child.context === context || child.fnContext === context) &&
      data && data.slot != null
    ) {
      const name = data.slot
      const slot = (slots[name] || (slots[name] = []))
      if (child.tag === 'template') {
        slot.push.apply(slot, child.children || [])
      } else {
        slot.push(child)
      }
    } else {
      (slots.default || (slots.default = [])).push(child)
    }
  }
  // ignore slots that contains only whitespace
  for (const name in slots) {
    if (slots[name].every(isWhitespace)) {
      delete slots[name]
    }
  }
  return slots
}

function isWhitespace (node: VNode): boolean {
  return (node.isComment && !node.asyncFactory) || node.text === ' '
}
```

slots解析，遍历_renderChildren（子节点数组）将数据分配至对应slot

存在data.slot则存入对应slot中，否则存入默认slot中

```
src/core/instance/init.js
export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode
  ...
  const vnodeComponentOptions = parentVnode.componentOptions
  ...
  opts._renderChildren = vnodeComponentOptions.children
  ...
}
```

_renderChildren来源于内部组件解析，对应组件下不属于具名插槽（v-slot）的节点数据

```
src/core/instance/render.js
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
}
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

将父组件节点的scopedSlots中存在的插槽与$slots中额外插槽封装成构造函数后合并

当_parentVnode.data.scopedSlots为稳定的且已存在scopedSlots并且不存在已解析的slots时可复用之前scopedSlots

```
src/core/vdom/helpers/normalize-scoped-slots.js
function normalizeScopedSlot(normalSlots, key, fn) {
  const normalized = function () {
    let res = arguments.length ? fn.apply(null, arguments) : fn({})
    res = res && typeof res === 'object' && !Array.isArray(res)
      ? [res] // single vnode
      : normalizeChildren(res)
    return res && (
      res.length === 0 ||
      (res.length === 1 && res[0].isComment) // #9658
    ) ? undefined
      : res
  }
  // this is a slot using the new v-slot syntax without scope. although it is
  // compiled as a scoped slot, render fn users would expect it to be present
  // on this.$slots because the usage is semantically a normal slot.
  if (fn.proxy) {
    Object.defineProperty(normalSlots, key, {
      get: normalized,
      enumerable: true,
      configurable: true
    })
  }
  return normalized
}

function proxyNormalSlot(slots, key) {
  return () => slots[key]
}
```

具名插槽描述包含proxy（无作用域）时，将挂载至$slots上

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
  const newScopedSlots = parentVnode.data.scopedSlots
  const oldScopedSlots = vm.$scopedSlots
  const hasDynamicScopedSlot = !!(
    (newScopedSlots && !newScopedSlots.$stable) ||
    (oldScopedSlots !== emptyObject && !oldScopedSlots.$stable) ||
    (newScopedSlots && vm.$scopedSlots.$key !== newScopedSlots.$key)
  )
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

若存在子节点或scopedSlots不是稳定的时重新进行slots解析更新$slots并触发强制更新然后在render中更新$scopedSlots

##### vnode中的描述

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

vnode.componentOptions.children中放置需进行slots解析的节点数据

vnode.data.scopedSlots中放置具名插槽的构造函数
