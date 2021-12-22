```
const processElement = (
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
  isSVG = isSVG || (n2.type as string) === 'svg'
  if (n1 == null) {
    mountElement(
      n2,
      container,
      anchor,
      parentComponent,
      parentSuspense,
      isSVG,
      slotScopeIds,
      optimized
    )
  } else {
    patchElement(
      n1,
      n2,
      parentComponent,
      parentSuspense,
      isSVG,
      slotScopeIds,
      optimized
    )
  }
}
```

元素节点通过`mountElement`进行挂载，通过`patchElement`进行更新

##### mountElement

```
const mountElement = (
  vnode: VNode,
  container: RendererElement,
  anchor: RendererNode | null,
  parentComponent: ComponentInternalInstance | null,
  parentSuspense: SuspenseBoundary | null,
  isSVG: boolean,
  slotScopeIds: string[] | null,
  optimized: boolean
) => {
  let el: RendererElement
  let vnodeHook: VNodeHook | undefined | null
  const { type, props, shapeFlag, transition, patchFlag, dirs } = vnode
  if (
    !__DEV__ &&
    vnode.el &&
    hostCloneNode !== undefined &&
    patchFlag === PatchFlags.HOISTED
  ) {
    // If a vnode has non-null el, it means it's being reused.
    // Only static vnodes can be reused, so its mounted DOM nodes should be
    // exactly the same, and we can simply do a clone here.
    // only do this in production since cloned trees cannot be HMR updated.
    el = vnode.el = hostCloneNode(vnode.el)
  } else {
    ...
  }
  if (dirs) {
    invokeDirectiveHook(vnode, null, parentComponent, 'beforeMount')
  }
  // #1583 For inside suspense + suspense not resolved case, enter hook should call when suspense resolved
  // #1689 For inside suspense + suspense resolved case, just call it
  const needCallTransitionHooks =
    (!parentSuspense || (parentSuspense && !parentSuspense.pendingBranch)) &&
    transition &&
    !transition.persisted
  if (needCallTransitionHooks) {
    transition!.beforeEnter(el)
  }
  hostInsert(el, container, anchor)
  if (
    (vnodeHook = props && props.onVnodeMounted) ||
    needCallTransitionHooks ||
    dirs
  ) {
    queuePostRenderEffect(() => {
      vnodeHook && invokeVNodeHook(vnodeHook, parentComponent, vnode)
      needCallTransitionHooks && transition!.enter(el)
      dirs && invokeDirectiveHook(vnode, null, parentComponent, 'mounted')
    }, parentSuspense)
  }
}
```

挂载主要进行了下列 操作：

1. 创建元素节点`el`
   - 若进行了静态提升，直接复制
   - 否则需进行完整创建
2. 触发指令的`created`钩子
3. 通过`hostPatchProp`为`el`添加特性属性、监听等
4. 触发`onVnodeBeforeMount`钩子
5. 添加`scopeId`
6. 触发指令的`beforeMount`钩子
7. 将`el`插入父节点
8. 触发`onVnodeMounted`钩子与指令的`mounted`钩子

```
el = vnode.el = hostCreateElement(
  vnode.type as string,
  isSVG,
  props && props.is,
  props
)

// mount children first, since some props may rely on child content
// being already rendered, e.g. `<select value>`
if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
  hostSetElementText(el, vnode.children as string)
} else if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
  mountChildren(
    vnode.children as VNodeArrayChildren,
    el,
    null,
    parentComponent,
    parentSuspense,
    isSVG && type !== 'foreignObject',
    slotScopeIds,
    optimized || !!vnode.dynamicChildren
  )
}

if (dirs) {
  invokeDirectiveHook(vnode, null, parentComponent, 'created')
}
// props
if (props) {
  for (const key in props) {
    if (!isReservedProp(key)) {
      hostPatchProp(
        el,
        key,
        null,
        props[key],
        isSVG,
        vnode.children as VNode[],
        parentComponent,
        parentSuspense,
        unmountChildren
      )
    }
  }
  if ((vnodeHook = props.onVnodeBeforeMount)) {
    invokeVNodeHook(vnodeHook, parentComponent, vnode)
  }
}
// scopeId
setScopeId(el, vnode, vnode.scopeId, slotScopeIds, parentComponent)
```

完整创建主要进行了下列操作：

1. `hostCreateElement`创建元素节点
2. 处理子节点
   - 若子节点为文本，通过`hostSetElementText`处理
   - 若子节点为数组，通过`mountChildren`处理
3. 触发指令的`created`钩子
4. 通过`hostPatchProp`处理`props`并触发`onVnodeBeforeMount`钩子
5. 设置`scopeId`

##### patchElement

```
const patchElement = (
  n1: VNode,
  n2: VNode,
  parentComponent: ComponentInternalInstance | null,
  parentSuspense: SuspenseBoundary | null,
  isSVG: boolean,
  slotScopeIds: string[] | null,
  optimized: boolean
) => {
  const el = (n2.el = n1.el!)
  let { patchFlag, dynamicChildren, dirs } = n2
  // #1426 take the old vnode's patch flag into account since user may clone a
  // compiler-generated vnode, which de-opts to FULL_PROPS
  patchFlag |= n1.patchFlag & PatchFlags.FULL_PROPS
  const oldProps = n1.props || EMPTY_OBJ
  const newProps = n2.props || EMPTY_OBJ
  let vnodeHook: VNodeHook | undefined | null

  if ((vnodeHook = newProps.onVnodeBeforeUpdate)) {
    invokeVNodeHook(vnodeHook, parentComponent, n2, n1)
  }
  if (dirs) {
    invokeDirectiveHook(n2, n1, parentComponent, 'beforeUpdate')
  }

  ...

  if ((vnodeHook = newProps.onVnodeUpdated) || dirs) {
    queuePostRenderEffect(() => {
      vnodeHook && invokeVNodeHook(vnodeHook, parentComponent, n2, n1)
      dirs && invokeDirectiveHook(n2, n1, parentComponent, 'updated')
    }, parentSuspense)
  }
}
```

更新主要进行了下列操作：

1. 触发`onVnodeBeforeUpdate`钩子与指令的`beforeUpdate`钩子
2. 更新操作
4. 触发`onVnodeUpdated`钩子与指令的`updated`钩子

更新操作主要分为两个部分：

- 属性更新
- 子节点更新

```
if (patchFlag > 0) {
  // the presence of a patchFlag means this element's render code was
  // generated by the compiler and can take the fast path.
  // in this path old node and new node are guaranteed to have the same shape
  // (i.e. at the exact same position in the source template)
  if (patchFlag & PatchFlags.FULL_PROPS) {
    // element props contain dynamic keys, full diff needed
    patchProps(
      el,
      n2,
      oldProps,
      newProps,
      parentComponent,
      parentSuspense,
      isSVG
    )
  } else {
    // class
    // this flag is matched when the element has dynamic class bindings.
    if (patchFlag & PatchFlags.CLASS) {
      if (oldProps.class !== newProps.class) {
        hostPatchProp(el, 'class', null, newProps.class, isSVG)
      }
    }

    // style
    // this flag is matched when the element has dynamic style bindings
    if (patchFlag & PatchFlags.STYLE) {
      hostPatchProp(el, 'style', oldProps.style, newProps.style, isSVG)
    }

    // props
    // This flag is matched when the element has dynamic prop/attr bindings
    // other than class and style. The keys of dynamic prop/attrs are saved for
    // faster iteration.
    // Note dynamic keys like :[foo]="bar" will cause this optimization to
    // bail out and go through a full diff because we need to unset the old key
    if (patchFlag & PatchFlags.PROPS) {
      // if the flag is present then dynamicProps must be non-null
      const propsToUpdate = n2.dynamicProps!
      for (let i = 0; i < propsToUpdate.length; i++) {
        const key = propsToUpdate[i]
        const prev = oldProps[key]
        const next = newProps[key]
        if (
          next !== prev ||
          (hostForcePatchProp && hostForcePatchProp(el, key))
        ) {
          hostPatchProp(
            el,
            key,
            prev,
            next,
            isSVG,
            n1.children as VNode[],
            parentComponent,
            parentSuspense,
            unmountChildren
          )
        }
      }
    }
  }

  // text
  // This flag is matched when the element has only dynamic text children.
  if (patchFlag & PatchFlags.TEXT) {
    if (n1.children !== n2.children) {
      hostSetElementText(el, n2.children as string)
    }
  }
} else if (!optimized && dynamicChildren == null) {
  // unoptimized, full diff
  patchProps(
    el,
    n2,
    oldProps,
    newProps,
    parentComponent,
    parentSuspense,
    isSVG
  )
}
```

- 属性更新时若`patchFlag`大于`0`，则根据`patchFlag`选择性更新

- 否则通过`patchProps`进行全属性更新

```
const areChildrenSVG = isSVG && n2.type !== 'foreignObject'
if (dynamicChildren) {
  patchBlockChildren(
    n1.dynamicChildren!,
    dynamicChildren,
    el,
    parentComponent,
    parentSuspense,
    areChildrenSVG,
    slotScopeIds
  )
  if (__DEV__ && parentComponent && parentComponent.type.__hmrId) {
    traverseStaticChildren(n1, n2)
  }
} else if (!optimized) {
  // full diff
  patchChildren(
    n1,
    n2,
    el,
    null,
    parentComponent,
    parentSuspense,
    areChildrenSVG,
    slotScopeIds,
    false
  )
}
```

- 存在`dynamicChildren`则通过`patchBlockChildren`处理
- 否则通过`patchChildren`处理