`component.vnode`指向在父组件实例中的节点数据

`component.subTree`指向自身的节点数据

`vnode.component`指向节点对应的组件实例

##### 应用初次渲染

```
packages/runtime-core/src/apiCreateApp.ts
export function createAppAPI<HostElement>(
  render: RootRenderFunction,
  hydrate?: RootHydrateFunction
): CreateAppFunction<HostElement> {
  return function createApp(rootComponent, rootProps = null) {
    ...
    const app: App = (context.app = {
      ...
  	  mount(
        rootContainer: HostElement,
        isHydrate?: boolean,
        isSVG?: boolean
      ): any {
        if (!isMounted) {
          const vnode = createVNode(
            rootComponent as ConcreteComponent,
            rootProps
          )
          ...
          if (isHydrate && hydrate) {
            hydrate(vnode as VNode<Node, Element>, rootContainer as any)
          } else {
            render(vnode, rootContainer, isSVG)
          }
          ...
          return vnode.component!.proxy
        }
        ...
      }
      ...
    }
    ...
    return app
  }
}
```

应用从`Vue.createApp`创建应用上下文后调用`mount`进入初次渲染并开始建立层级关系与依赖追踪等

主要调用为`createVNode`创建根节点，`render`进行初次渲染

```
packages/runtime-core/src/vnode.ts
function _createVNode(
  type: VNodeTypes | ClassComponent | typeof NULL_DYNAMIC_COMPONENT,
  props: (Data & VNodeProps) | null = null,
  children: unknown = null,
  patchFlag: number = 0,
  dynamicProps: string[] | null = null,
  isBlockNode = false
): VNode {
  ...
  // encode the vnode type information into a bitmap
  const shapeFlag = isString(type)
    ? ShapeFlags.ELEMENT
    : __FEATURE_SUSPENSE__ && isSuspense(type)
      ? ShapeFlags.SUSPENSE
      : isTeleport(type)
        ? ShapeFlags.TELEPORT
        : isObject(type)
          ? ShapeFlags.STATEFUL_COMPONENT
          : isFunction(type)
            ? ShapeFlags.FUNCTIONAL_COMPONENT
            : 0
  ...
  const vnode: VNode = {
    __v_isVNode: true,
    [ReactiveFlags.SKIP]: true,
    type,
    props,
    key: props && normalizeKey(props),
    ref: props && normalizeRef(props),
    scopeId: currentScopeId,
    slotScopeIds: null,
    children: null,
    component: null,
    suspense: null,
    ssContent: null,
    ssFallback: null,
    dirs: null,
    transition: null,
    el: null,
    anchor: null,
    target: null,
    targetAnchor: null,
    staticCount: 0,
    shapeFlag,
    patchFlag,
    dynamicProps,
    dynamicChildren: null,
    appContext: null
  }
  ...
  return vnode
}
```

一般情况下对于根节点，`type`即为传入的对象或方法，对于的`shapeFlag`为`ShapeFlags.STATEFUL_COMPONENT`或`ShapeFlags.FUNCTIONAL_COMPONENT`

```
packages/runtime-core/src/renderer.ts
function baseCreateRenderer(
  options: RendererOptions,
  createHydrationFns?: typeof createHydrationFunctions
): any {
  ...
  const render: RootRenderFunction = (vnode, container, isSVG) => {
    if (vnode == null) {
      if (container._vnode) {
        unmount(container._vnode, null, null, true)
      }
    } else {
      patch(container._vnode || null, vnode, container, null, null, null, isSVG)
    }
    flushPostFlushCbs()
    container._vnode = vnode
  }
  ...
  return {
    render,
    hydrate,
    createApp: createAppAPI(render, hydrate)
  }
}
```

`vnode`在通过`render`进行初次渲染时将通过`patch`进行处理

```
packages/runtime-core/src/renderer.ts
const patch: PatchFn = (
  n1,
  n2,
  container,
  anchor = null,
  parentComponent = null,
  parentSuspense = null,
  isSVG = false,
  slotScopeIds = null,
  optimized = false
) => {
  ...
  const { type, ref, shapeFlag } = n2
  switch (type) {
    case Text:
      ...
      break
    case Comment:
      ...
      break
    case Static:
      ...
      break
    case Fragment:
      ...
      break
    default:
      if (shapeFlag & ShapeFlags.ELEMENT) {
        ...
      } else if (shapeFlag & ShapeFlags.COMPONENT) {
        processComponent(
          n1,
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          slotScopeIds,
          optimized
        )
      } else if (shapeFlag & ShapeFlags.TELEPORT) {
        ...
      } else if (__FEATURE_SUSPENSE__ && shapeFlag & ShapeFlags.SUSPENSE) {
        ...
      } else if (__DEV__) {
        warn('Invalid VNode type:', type, `(${typeof type})`)
      }
  }
  ...
}
```

根节点的初次渲染将进入`processComponent`

`n1`为`null`，`n2`为`mount`时创建的`vnode`，`container`为`mount`时对应的dom节点，其他属性均为默认值

```
packages/runtime-core/src/renderer.ts
const processComponent = (
  n1: VNode | null,
  n2: VNode,
  container: RendererElement,
  anchor: RendererNode | null,
  parentComponent: ComponentInternalInstance | null,
  parentSuspense: SuspenseBoundary | null,
  isSVG: boolean,
  slotScopeIds: string[] | null,
  optimized: boolean
) => {
  n2.slotScopeIds = slotScopeIds
  if (n1 == null) {
    if (n2.shapeFlag & ShapeFlags.COMPONENT_KEPT_ALIVE) {
      ;(parentComponent!.ctx as KeepAliveContext).activate(
        n2,
        container,
        anchor,
        isSVG,
        optimized
      )
    } else {
      mountComponent(
        n2,
        container,
        anchor,
        parentComponent,
        parentSuspense,
        isSVG,
        optimized
      )
    }
  } else {
    updateComponent(n1, n2, optimized)
  }
}

const mountComponent: MountComponentFn = (
  initialVNode,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  isSVG,
  optimized
) => {
  const instance: ComponentInternalInstance = (initialVNode.component = createComponentInstance(
    initialVNode,
    parentComponent,
    parentSuspense
  ))
  
  // inject renderer internals for keepAlive
  if (isKeepAlive(initialVNode)) {
    ;(instance.ctx as KeepAliveContext).renderer = internals
  }

  // resolve props and slots for setup context
  setupComponent(instance)

  // setup() is async. This component relies on async logic to be resolved
  // before proceeding
  if (__FEATURE_SUSPENSE__ && instance.asyncDep) {
    parentSuspense && parentSuspense.registerDep(instance, setupRenderEffect)

    // Give it a placeholder if this is not hydration
    // TODO handle self-defined fallback
    if (!initialVNode.el) {
      const placeholder = (instance.subTree = createVNode(Comment))
      processCommentNode(null, placeholder, container!, anchor)
    }
    return
  }

  setupRenderEffect(
    instance,
    initialVNode,
    container,
    anchor,
    parentSuspense,
    isSVG,
    optimized
  )
  ...
}
```

因`n1`为`null`且`shapeFlag`为`ShapeFlags.STATEFUL_COMPONENT`或`ShapeFlags.FUNCTIONAL_COMPONENT`将进入`mountComponent`

然后通过`createComponentInstance`创建组件实例并通过`setupRenderEffect`进行初次渲染与建立依赖追踪

`setupComponent`用于初始化，将调用`setup`等

```
packages/runtime-core/src/renderer.ts
const prodEffectOptions = {
  scheduler: queueJob,
  // #1801, #2043 component render effects should allow recursive updates
  allowRecurse: true
}

const setupRenderEffect: SetupRenderEffectFn = (
  instance,
  initialVNode,
  container,
  anchor,
  parentSuspense,
  isSVG,
  optimized
) => {
  // create reactive effect for rendering
  instance.update = effect(function componentEffect() {
    if (!instance.isMounted) {
      let vnodeHook: VNodeHook | null | undefined
      const { el, props } = initialVNode
      const { bm, m, parent } = instance

      // beforeMount hook
      if (bm) {
        invokeArrayFns(bm)
      }
      // onVnodeBeforeMount
      if ((vnodeHook = props && props.onVnodeBeforeMount)) {
        invokeVNodeHook(vnodeHook, parent, initialVNode)
      }

      const subTree = (instance.subTree = renderComponentRoot(instance))
      if (el && hydrateNode) {
        ...
      } else {
        patch(
          null,
          subTree,
          container,
          anchor,
          instance,
          parentSuspense,
          isSVG
        )
        initialVNode.el = subTree.el
      }
      // mounted hook
      if (m) {
        queuePostRenderEffect(m, parentSuspense)
      }
      // onVnodeMounted
      if ((vnodeHook = props && props.onVnodeMounted)) {
        const scopedInitialVNode = initialVNode
        queuePostRenderEffect(() => {
          invokeVNodeHook(vnodeHook!, parent, scopedInitialVNode)
        }, parentSuspense)
      }
      // activated hook for keep-alive roots.
      // #1742 activated hook must be accessed after first render
      // since the hook may be injected by a child keep-alive
      const { a } = instance
      if (
        a &&
        initialVNode.shapeFlag & ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE
      ) {
        queuePostRenderEffect(a, parentSuspense)
      }
      instance.isMounted = true

      // #2458: deference mount-only object parameters to prevent memleaks
      initialVNode = container = anchor = null as any
    } else {
      ...
    }
  }, __DEV__ ? createDevEffectOptions(instance) : prodEffectOptions)
}
```

`setupRenderEffect`为实例添加了一个`update`属性，为一个`ReactiveEffect`实例

初始渲染阶段主要进行了下列操作：

1. 调用`beforeMount`钩子
2. 调用`onVnodeBeforeMount`钩子
3. 通过`renderComponentRoot`生成`subTree`，即实例对应的节点
4. 通过`patch`处理节点，获取`el`并挂载至`vnode`
5. 调用`mounted`钩子
6. 调用`onVnodeMounted`钩子
7. 调用`activated`钩子

```
packages/runtime-core/src/componentRenderUtils.ts
export function renderComponentRoot(
  instance: ComponentInternalInstance
): VNode {
  const {
    type: Component,
    vnode,
    proxy,
    withProxy,
    props,
    propsOptions: [propsOptions],
    slots,
    attrs,
    emit,
    render,
    renderCache,
    data,
    setupState,
    ctx
  } = instance

  let result
  const prev = setCurrentRenderingInstance(instance)
  try {
    let fallthroughAttrs
    if (vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
      // withProxy is a proxy with a different `has` trap only for
      // runtime-compiled render functions using `with` block.
      const proxyToUse = withProxy || proxy
      result = normalizeVNode(
        render!.call(
          proxyToUse,
          proxyToUse!,
          renderCache,
          props,
          setupState,
          data,
          ctx
        )
      )
      fallthroughAttrs = attrs
    } else {
      // functional
      const render = Component as FunctionalComponent
      result = normalizeVNode(
        render.length > 1
          ? render(
              props,
              { attrs, slots, emit }
            )
          : render(props, null as any /* we know it doesn't need it */)
      )
      fallthroughAttrs = Component.props
        ? attrs
        : getFunctionalFallthrough(attrs)
    }

    // attr merging
    // in dev mode, comments are preserved, and it's possible for a template
    // to have comments along side the root element which makes it a fragment
    let root = result
    let setRoot: ((root: VNode) => void) | undefined = undefined

    if (Component.inheritAttrs !== false && fallthroughAttrs) {
      const keys = Object.keys(fallthroughAttrs)
      const { shapeFlag } = root
      if (keys.length) {
        if (
          shapeFlag & ShapeFlags.ELEMENT ||
          shapeFlag & ShapeFlags.COMPONENT
        ) {
          if (propsOptions && keys.some(isModelListener)) {
            // If a v-model listener (onUpdate:xxx) has a corresponding declared
            // prop, it indicates this component expects to handle v-model and
            // it should not fallthrough.
            // related: #1543, #1643, #1989
            fallthroughAttrs = filterModelListeners(
              fallthroughAttrs,
              propsOptions
            )
          }
          root = cloneVNode(root, fallthroughAttrs)
        }
      }
    }

    // inherit directives
    if (vnode.dirs) {
      root.dirs = root.dirs ? root.dirs.concat(vnode.dirs) : vnode.dirs
    }
    // inherit transition data
    if (vnode.transition) {
      root.transition = vnode.transition
    }

    result = root
  } catch (err) {
    blockStack.length = 0
    handleError(err, instance, ErrorCodes.RENDER_FUNCTION)
    result = createVNode(Comment)
  }

  setCurrentRenderingInstance(prev)
  return result
}
```

`renderComponentRoot`主要进行了下列操作：

1. 记录当前实例
2. 获取节点
3. 挂载未使用的属性
4. 继承指令与动画
5. 还原当前实例

##### 组件初次渲染

与应用大致相同，仅`patch`时机是通过`mountChildren`或高阶组件时直接作为`subTree`进行触发

##### 父组件触发组件更新

```
packages/runtime-core/src/renderer.ts
const updateComponent = (n1: VNode, n2: VNode, optimized: boolean) => {
  const instance = (n2.component = n1.component)!
  if (shouldUpdateComponent(n1, n2, optimized)) {
    if (
      __FEATURE_SUSPENSE__ &&
      instance.asyncDep &&
      !instance.asyncResolved
    ) {
      updateComponentPreRender(instance, n2, optimized)
      return
    } else {
      // normal update
      instance.next = n2
      // in case the child component is also queued, remove it to avoid
      // double updating the same child component in the same flush.
      invalidateJob(instance.update)
      // instance.update is the reactive effect runner.
      instance.update()
    }
  } else {
    // no update needed. just copy over properties
    n2.component = n1.component
    n2.el = n1.el
    instance.vnode = n2
  }
}
```

- 节点数据有变化
  - 记录新`vnode`
  - 触发更新
- 否则直接传递数据并更新`vnode`

##### 组件更新

```
packages/runtime-core/src/renderer.ts
const setupRenderEffect: SetupRenderEffectFn = (
  instance,
  initialVNode,
  container,
  anchor,
  parentSuspense,
  isSVG,
  optimized
) => {
  // create reactive effect for rendering
  instance.update = effect(function componentEffect() {
    if (!instance.isMounted) {
      ...
    } else {
      // updateComponent
      // This is triggered by mutation of component's own state (next: null)
      // OR parent calling processComponent (next: VNode)
      let { next, bu, u, parent, vnode } = instance
      let originNext = next
      let vnodeHook: VNodeHook | null | undefined

      if (next) {
        next.el = vnode.el
        updateComponentPreRender(instance, next, optimized)
      } else {
        next = vnode
      }

      // beforeUpdate hook
      if (bu) {
        invokeArrayFns(bu)
      }
      // onVnodeBeforeUpdate
      if ((vnodeHook = next.props && next.props.onVnodeBeforeUpdate)) {
        invokeVNodeHook(vnodeHook, parent, next, vnode)
      }

      // render
      const nextTree = renderComponentRoot(instance)
      const prevTree = instance.subTree
      instance.subTree = nextTree
      patch(
        prevTree,
        nextTree,
        // parent may have changed if it's in a teleport
        hostParentNode(prevTree.el!)!,
        // anchor may have changed if it's in a fragment
        getNextHostNode(prevTree),
        instance,
        parentSuspense,
        isSVG
      )
      next.el = nextTree.el
      if (originNext === null) {
        // self-triggered update. In case of HOC, update parent component
        // vnode el. HOC is indicated by parent instance's subTree pointing
        // to child component's vnode
        updateHOCHostEl(instance, nextTree.el)
      }
      // updated hook
      if (u) {
        queuePostRenderEffect(u, parentSuspense)
      }
      // onVnodeUpdated
      if ((vnodeHook = next.props && next.props.onVnodeUpdated)) {
        queuePostRenderEffect(() => {
          invokeVNodeHook(vnodeHook!, parent, next!, vnode)
        }, parentSuspense)
      }
    }
  }, __DEV__ ? createDevEffectOptions(instance) : prodEffectOptions)
}
```

初次渲染时通过进行了依赖收集，数据变化时会触发`instance.onUpdate`，此时`isMounted`为`true`

更新阶段主要进行了下列操作：

1. 若是`组件vnode`有变化则进行数据更新
2. 调用`beforeUpdate`钩子
3. 调用`onVnodeBeforeUpdate`钩子
4. 通过`renderComponentRoot`生成`subTree`
5. 通过`patch`处理新旧节点，获取`el`并挂载至`vnode`
6. 若是高阶组件，更新父节点
7. 调用`updated`钩子
8. 调用`onVnodeUpdated`钩子

```
packages/runtime-core/src/renderer.ts
const updateComponentPreRender = (
  instance: ComponentInternalInstance,
  nextVNode: VNode,
  optimized: boolean
) => {
  nextVNode.component = instance
  const prevProps = instance.vnode.props
  instance.vnode = nextVNode
  instance.next = null
  updateProps(instance, nextVNode.props, prevProps, optimized)
  updateSlots(instance, nextVNode.children, optimized)

  pauseTracking()
  // props update may have triggered pre-flush watchers.
  // flush them before the render update.
  flushPreFlushCbs(undefined, instance.update)
  resetTracking()
}
```

更新属性并触发清空前置队列的操作

##### 卸载

```
packages/runtime-core/src/renderer.ts
const unmount: UnmountFn = (
  vnode,
  parentComponent,
  parentSuspense,
  doRemove = false,
  optimized = false
) => {
  const {
    type,
    props,
    ref,
    children,
    dynamicChildren,
    shapeFlag,
    patchFlag,
    dirs
  } = vnode
  // unset ref
  if (ref != null) {
    setRef(ref, null, parentSuspense, null)
  }

  if (shapeFlag & ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE) {
    ;(parentComponent!.ctx as KeepAliveContext).deactivate(vnode)
    return
  }

  const shouldInvokeDirs = shapeFlag & ShapeFlags.ELEMENT && dirs

  let vnodeHook: VNodeHook | undefined | null
  if ((vnodeHook = props && props.onVnodeBeforeUnmount)) {
    invokeVNodeHook(vnodeHook, parentComponent, vnode)
  }

  if (shapeFlag & ShapeFlags.COMPONENT) {
    unmountComponent(vnode.component!, parentSuspense, doRemove)
  } else {
    ...
  }

  if ((vnodeHook = props && props.onVnodeUnmounted) || shouldInvokeDirs) {
    queuePostRenderEffect(() => {
      vnodeHook && invokeVNodeHook(vnodeHook, parentComponent, vnode)
      shouldInvokeDirs &&
        invokeDirectiveHook(vnode, null, parentComponent, 'unmounted')
    }, parentSuspense)
  }
}

const unmountComponent = (
  instance: ComponentInternalInstance,
  parentSuspense: SuspenseBoundary | null,
  doRemove?: boolean
) => {
  const { bum, effects, update, subTree, um } = instance
  // beforeUnmount hook
  if (bum) {
    invokeArrayFns(bum)
  }
  if (effects) {
    for (let i = 0; i < effects.length; i++) {
      stop(effects[i])
    }
  }
  // update may be null if a component is unmounted before its async
  // setup has resolved.
  if (update) {
    stop(update)
    unmount(subTree, instance, parentSuspense, doRemove)
  }
  // unmounted hook
  if (um) {
    queuePostRenderEffect(um, parentSuspense)
  }
  queuePostRenderEffect(() => {
    instance.isUnmounted = true
  }, parentSuspense)

  // A component with async dep inside a pending suspense is unmounted before
  // its async dep resolves. This should remove the dep from the suspense, and
  // cause the suspense to resolve immediately if that was the last dep.
  if (
    __FEATURE_SUSPENSE__ &&
    parentSuspense &&
    parentSuspense.pendingBranch &&
    !parentSuspense.isUnmounted &&
    instance.asyncDep &&
    !instance.asyncResolved &&
    instance.suspenseId === parentSuspense.pendingId
  ) {
    parentSuspense.deps--
    if (parentSuspense.deps === 0) {
      parentSuspense.resolve()
    }
  }
}
```

卸载主要进行了下列操作：

1. 卸载`ref`
2. 若是keepalive，触发`deactivate`并跳出
3. 调用`onVnodeBeforeUnmount`钩子
4. 调用`beforeUnmount`钩子
5. 销毁effect
6. 卸载节点
7. 调用`unmounted`钩子
8. 调用`onVnodeUnmounted`钩子与指令的`unmounted`钩子