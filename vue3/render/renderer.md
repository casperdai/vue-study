```
packages/runtime-core/src/renderer.ts
export function createRenderer<
  HostNode = RendererNode,
  HostElement = RendererElement
>(options: RendererOptions<HostNode, HostElement>) {
  return baseCreateRenderer<HostNode, HostElement>(options)
}

function baseCreateRenderer(
  options: RendererOptions,
  createHydrationFns?: typeof createHydrationFunctions
): any {
  ...
}
```

通过`baseCreateRenderer`构造渲染器

```
function baseCreateRenderer(
  options: RendererOptions,
  createHydrationFns?: typeof createHydrationFunctions
): any {
  ...
  const {
    insert: hostInsert,
    remove: hostRemove,
    patchProp: hostPatchProp,
    forcePatchProp: hostForcePatchProp,
    createElement: hostCreateElement,
    createText: hostCreateText,
    createComment: hostCreateComment,
    setText: hostSetText,
    setElementText: hostSetElementText,
    parentNode: hostParentNode,
    nextSibling: hostNextSibling,
    setScopeId: hostSetScopeId = NOOP,
    cloneNode: hostCloneNode,
    insertStaticContent: hostInsertStaticContent
  } = options
  ...
  // Note: functions inside this closure should use `const xxx = () => {}`
  // style in order to prevent being inlined by minifiers.
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
  }
  ...
  const unmount: UnmountFn = (
    vnode,
    parentComponent,
    parentSuspense,
    doRemove = false,
    optimized = false
  ) => {
    ...
  }
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

渲染器提供3个函数进行调用，分别对应浏览器渲染，SSR与创建应用上下文

`patch`提供初次渲染与更新的能力，`unmount`提供卸载

`n1`为当前已渲染的`vnode`

`n2`为将要渲染的`vnode`

`container`为`vnode`挂载至的dom节点

`anchor`为挂载时用于定位的节点，通过该属性获取插入的位置

`parentComponent`为父组件实例

##### 操作

```
packages/runtime-dom/src/nodeOps.ts
packages/runtime-dom/src/patchProp.ts
```

`baseCreateRenderer`的`options`由上述文件的方法合并而成

提供了DOM操作与`props`的`patch`操作

##### patch

```
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
  // patching & not same type, unmount old tree
  if (n1 && !isSameVNodeType(n1, n2)) {
    anchor = getNextHostNode(n1)
    unmount(n1, parentComponent, parentSuspense, true)
    n1 = null
  }

  if (n2.patchFlag === PatchFlags.BAIL) {
    optimized = false
    n2.dynamicChildren = null
  }

  const { type, ref, shapeFlag } = n2
  switch (type) {
    case Text:
      processText(n1, n2, container, anchor)
      break
    case Comment:
      processCommentNode(n1, n2, container, anchor)
      break
    case Static:
      if (n1 == null) {
        mountStaticNode(n2, container, anchor, isSVG)
      } else if (__DEV__) {
        patchStaticNode(n1, n2, container, isSVG)
      }
      break
    case Fragment:
      processFragment(
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
      break
    default:
      if (shapeFlag & ShapeFlags.ELEMENT) {
        processElement(
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
        ;(type as typeof TeleportImpl).process(
          n1 as TeleportVNode,
          n2 as TeleportVNode,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          slotScopeIds,
          optimized,
          internals
        )
      } else if (__FEATURE_SUSPENSE__ && shapeFlag & ShapeFlags.SUSPENSE) {
        ;(type as typeof SuspenseImpl).process(
          n1,
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          slotScopeIds,
          optimized,
          internals
        )
      } else if (__DEV__) {
        warn('Invalid VNode type:', type, `(${typeof type})`)
      }
  }

  // set ref
  if (ref != null && parentComponent) {
    setRef(ref, n1 && n1.ref, parentSuspense, n2)
  }
}
```

`patch`根据`vnode`的`type`、`shapeFlag`调用相应的处理函数

当新旧节点不是相似节点时将先卸载旧节点，相似即`type`与`key`均相等

##### unmount

```
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
    if (__FEATURE_SUSPENSE__ && shapeFlag & ShapeFlags.SUSPENSE) {
      vnode.suspense!.unmount(parentSuspense, doRemove)
      return
    }

    if (shouldInvokeDirs) {
      invokeDirectiveHook(vnode, null, parentComponent, 'beforeUnmount')
    }

    if (shapeFlag & ShapeFlags.TELEPORT) {
      ;(vnode.type as typeof TeleportImpl).remove(
        vnode,
        parentComponent,
        parentSuspense,
        optimized,
        internals,
        doRemove
      )
    } else if (
      dynamicChildren &&
      // #1153: fast path should not be taken for non-stable (v-for) fragments
      (type !== Fragment ||
        (patchFlag > 0 && patchFlag & PatchFlags.STABLE_FRAGMENT))
    ) {
      // fast path for block nodes: only need to unmount dynamic children.
      unmountChildren(
        dynamicChildren,
        parentComponent,
        parentSuspense,
        false,
        true
      )
    } else if (
      (type === Fragment &&
        (patchFlag & PatchFlags.KEYED_FRAGMENT ||
          patchFlag & PatchFlags.UNKEYED_FRAGMENT)) ||
      (!optimized && shapeFlag & ShapeFlags.ARRAY_CHILDREN)
    ) {
      unmountChildren(children as VNode[], parentComponent, parentSuspense)
    }

    if (doRemove) {
      remove(vnode)
    }
  }

  if ((vnodeHook = props && props.onVnodeUnmounted) || shouldInvokeDirs) {
    queuePostRenderEffect(() => {
      vnodeHook && invokeVNodeHook(vnodeHook, parentComponent, vnode)
      shouldInvokeDirs &&
        invokeDirectiveHook(vnode, null, parentComponent, 'unmounted')
    }, parentSuspense)
  }
}
```

主要进行了下列操作：

1. 释放`ref`引用
2. 若是`ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE`，进行`deactivate`处理然后跳出
3. 触发`onVnodeBeforeUnmount`钩子
4. 若是组件，调用`unmountComponent`
5. 若不是组件
   - 若是`ShapeFlags.SUSPENSE`，进行`vnode.suspense!.unmount`处理然后跳出
   - 触发指令`beforeUnmount`钩子
   - 若是`ShapeFlags.TELEPORT`则进行`vnode.type.remove`处理，否则`unmountChildren`
   - 若需移除dom，通过`remove`移除
6. 触发`onVnodeUnmounted`钩子与指令的`unmounted`钩子

##### mountChildren

```
const mountChildren: MountChildrenFn = (
  children,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  isSVG,
  optimized,
  slotScopeIds,
  start = 0
) => {
  for (let i = start; i < children.length; i++) {
    const child = (children[i] = optimized
      ? cloneIfMounted(children[i] as VNode)
      : normalizeVNode(children[i]))
    patch(
      null,
      child,
      container,
      anchor,
      parentComponent,
      parentSuspense,
      isSVG,
      optimized,
      slotScopeIds
    )
  }
}
```

`mountChildren`只是遍历子节点，然后使用`patch`进行处理

##### unmountChildren

```
const unmountChildren: UnmountChildrenFn = (
  children,
  parentComponent,
  parentSuspense,
  doRemove = false,
  optimized = false,
  start = 0
) => {
  for (let i = start; i < children.length; i++) {
    unmount(children[i], parentComponent, parentSuspense, doRemove, optimized)
  }
}
```

`unmountChildren`只是遍历子节点，然后使用`unmount`进行处理

##### patchProps

```
const patchProps = (
  el: RendererElement,
  vnode: VNode,
  oldProps: Data,
  newProps: Data,
  parentComponent: ComponentInternalInstance | null,
  parentSuspense: SuspenseBoundary | null,
  isSVG: boolean
) => {
  if (oldProps !== newProps) {
    for (const key in newProps) {
      // empty string is not valid prop
      if (isReservedProp(key)) continue
      const next = newProps[key]
      const prev = oldProps[key]
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
          vnode.children as VNode[],
          parentComponent,
          parentSuspense,
          unmountChildren
        )
      }
    }
    if (oldProps !== EMPTY_OBJ) {
      for (const key in oldProps) {
        if (!isReservedProp(key) && !(key in newProps)) {
          hostPatchProp(
            el,
            key,
            oldProps[key],
            null,
            isSVG,
            vnode.children as VNode[],
            parentComponent,
            parentSuspense,
            unmountChildren
          )
        }
      }
    }
  }
}
```

对比新旧`props`，通过`hostPatchProp`进行处理

用于更新`class`、`style`、事件以及特性属性

##### setScopeId

```
const setScopeId = (
  el: RendererElement,
  vnode: VNode,
  scopeId: string | null,
  slotScopeIds: string[] | null,
  parentComponent: ComponentInternalInstance | null
) => {
  if (scopeId) {
    hostSetScopeId(el, scopeId)
  }
  if (slotScopeIds) {
    for (let i = 0; i < slotScopeIds.length; i++) {
      hostSetScopeId(el, slotScopeIds[i])
    }
  }
  if (parentComponent) {
    let subTree = parentComponent.subTree
    if (
      __DEV__ &&
      subTree.patchFlag > 0 &&
      subTree.patchFlag & PatchFlags.DEV_ROOT_FRAGMENT
    ) {
      subTree =
        filterSingleRoot(subTree.children as VNodeArrayChildren) || subTree
    }
    if (vnode === subTree) {
      const parentVNode = parentComponent.vnode
      setScopeId(
        el,
        parentVNode,
        parentVNode.scopeId,
        parentVNode.slotScopeIds,
        parentComponent.parent
      )
    }
  }
}
```

添加scopeId特性属性

##### patchChildren

```
const patchChildren: PatchChildrenFn = (
  n1,
  n2,
  container,
  anchor,
  parentComponent,
  parentSuspense,
  isSVG,
  slotScopeIds,
  optimized = false
) => {
  const c1 = n1 && n1.children
  const prevShapeFlag = n1 ? n1.shapeFlag : 0
  const c2 = n2.children

  const { patchFlag, shapeFlag } = n2
  // fast path
  if (patchFlag > 0) {
    if (patchFlag & PatchFlags.KEYED_FRAGMENT) {
      // this could be either fully-keyed or mixed (some keyed some not)
      // presence of patchFlag means children are guaranteed to be arrays
      patchKeyedChildren(
        c1 as VNode[],
        c2 as VNodeArrayChildren,
        container,
        anchor,
        parentComponent,
        parentSuspense,
        isSVG,
        slotScopeIds,
        optimized
      )
      return
    } else if (patchFlag & PatchFlags.UNKEYED_FRAGMENT) {
      // unkeyed
      patchUnkeyedChildren(
        c1 as VNode[],
        c2 as VNodeArrayChildren,
        container,
        anchor,
        parentComponent,
        parentSuspense,
        isSVG,
        slotScopeIds,
        optimized
      )
      return
    }
  }

  // children has 3 possibilities: text, array or no children.
  if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
    // text children fast path
    if (prevShapeFlag & ShapeFlags.ARRAY_CHILDREN) {
      unmountChildren(c1 as VNode[], parentComponent, parentSuspense)
    }
    if (c2 !== c1) {
      hostSetElementText(container, c2 as string)
    }
  } else {
    if (prevShapeFlag & ShapeFlags.ARRAY_CHILDREN) {
      // prev children was array
      if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
        // two arrays, cannot assume anything, do full diff
        patchKeyedChildren(
          c1 as VNode[],
          c2 as VNodeArrayChildren,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          slotScopeIds,
          optimized
        )
      } else {
        // no new children, just unmount old
        unmountChildren(c1 as VNode[], parentComponent, parentSuspense, true)
      }
    } else {
      // prev children was text OR null
      // new children is array OR null
      if (prevShapeFlag & ShapeFlags.TEXT_CHILDREN) {
        hostSetElementText(container, '')
      }
      // mount new if array
      if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
        mountChildren(
          c2 as VNodeArrayChildren,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG,
          slotScopeIds,
          optimized
        )
      }
    }
  }
}
```

更新子节点

- `v-for`通过`patchKeyedChildren`与`patchUnkeyedChildren`处理
- 子节点为文本时，若当前有子节点则进行卸载，更新文本
- 原子节点为数组时
  - 新子节点为数组时，通过`patchKeyedChildren`处理
  - 否则卸载原子节点
- 其他情况，若原子节点为 文本时则清空文本，新子节点为数组时通过`mountChildren`挂载子节点

##### patchUnkeyedChildren

```
const patchUnkeyedChildren = (
  c1: VNode[],
  c2: VNodeArrayChildren,
  container: RendererElement,
  anchor: RendererNode | null,
  parentComponent: ComponentInternalInstance | null,
  parentSuspense: SuspenseBoundary | null,
  isSVG: boolean,
  slotScopeIds: string[] | null,
  optimized: boolean
) => {
  c1 = c1 || EMPTY_ARR
  c2 = c2 || EMPTY_ARR
  const oldLength = c1.length
  const newLength = c2.length
  const commonLength = Math.min(oldLength, newLength)
  let i
  for (i = 0; i < commonLength; i++) {
    const nextChild = (c2[i] = optimized
      ? cloneIfMounted(c2[i] as VNode)
      : normalizeVNode(c2[i]))
    patch(
      c1[i],
      nextChild,
      container,
      null,
      parentComponent,
      parentSuspense,
      isSVG,
      slotScopeIds,
      optimized
    )
  }
  if (oldLength > newLength) {
    // remove old
    unmountChildren(
      c1,
      parentComponent,
      parentSuspense,
      true,
      false,
      commonLength
    )
  } else {
    // mount new
    mountChildren(
      c2,
      container,
      anchor,
      parentComponent,
      parentSuspense,
      isSVG,
      slotScopeIds,
      optimized,
      commonLength
    )
  }
}
```

当忽略`key`的子节点更新时分两步：

1. 通过`patch`逐个比对，当其中一个集合遍历完后结束
2. 若旧集合还有未遍历的节点则全部卸载，若新集合还有未遍历的节点则全部挂载

##### patchKeyedChildren

```
const patchKeyedChildren = (
  c1: VNode[],
  c2: VNodeArrayChildren,
  container: RendererElement,
  parentAnchor: RendererNode | null,
  parentComponent: ComponentInternalInstance | null,
  parentSuspense: SuspenseBoundary | null,
  isSVG: boolean,
  slotScopeIds: string[] | null,
  optimized: boolean
) => {
  let i = 0
  const l2 = c2.length
  let e1 = c1.length - 1 // prev ending index
  let e2 = l2 - 1 // next ending index

  ...
}
```

关注`key`的子节点比对

```
while (i <= e1 && i <= e2) {
  const n1 = c1[i]
  const n2 = (c2[i] = optimized
    ? cloneIfMounted(c2[i] as VNode)
    : normalizeVNode(c2[i]))
  if (isSameVNodeType(n1, n2)) {
    patch(
      n1,
      n2,
      container,
      null,
      parentComponent,
      parentSuspense,
      isSVG,
      slotScopeIds,
      optimized
    )
  } else {
    break
  }
  i++
}
```

第一步从前往后找出相似子节点进行`patch`

```
while (i <= e1 && i <= e2) {
  const n1 = c1[e1]
  const n2 = (c2[e2] = optimized
    ? cloneIfMounted(c2[e2] as VNode)
    : normalizeVNode(c2[e2]))
  if (isSameVNodeType(n1, n2)) {
    patch(
      n1,
      n2,
      container,
      null,
      parentComponent,
      parentSuspense,
      isSVG,
      slotScopeIds,
      optimized
    )
  } else {
    break
  }
  e1--
  e2--
}
```

第二步从后往前找出相似子节点进行`patch`

```
if (i > e1) {
  if (i <= e2) {
    const nextPos = e2 + 1
    const anchor = nextPos < l2 ? (c2[nextPos] as VNode).el : parentAnchor
    while (i <= e2) {
      patch(
        null,
        (c2[i] = optimized
          ? cloneIfMounted(c2[i] as VNode)
          : normalizeVNode(c2[i])),
        container,
        anchor,
        parentComponent,
        parentSuspense,
        isSVG,
        slotScopeIds,
        optimized
      )
      i++
    }
  }
} else if (i > e2) {
  while (i <= e1) {
    unmount(c1[i], parentComponent, parentSuspense, true)
    i++
  }
} else {
  // 第四步
  ...
}
```

第三步

- 若仅新集合还有未遍历数据时，遍历挂载剩余节点
- 若仅旧集合还有未遍历数据时，遍历卸载剩余节点
- 否则进入第四步

```
const s1 = i // prev starting index
const s2 = i // next starting index

const keyToNewIndexMap: Map<string | number, number> = new Map()
for (i = s2; i <= e2; i++) {
  const nextChild = (c2[i] = optimized
    ? cloneIfMounted(c2[i] as VNode)
    : normalizeVNode(c2[i]))
  if (nextChild.key != null) {
    keyToNewIndexMap.set(nextChild.key, i)
  }
}
```

第四步首先记录新集合剩余节点中有`key`的节点的索引

```
let j
let patched = 0
const toBePatched = e2 - s2 + 1
let moved = false
let maxNewIndexSoFar = 0
const newIndexToOldIndexMap = new Array(toBePatched)
for (i = 0; i < toBePatched; i++) newIndexToOldIndexMap[i] = 0

for (i = s1; i <= e1; i++) {
  const prevChild = c1[i]
  if (patched >= toBePatched) {
    // all new children have been patched so this can only be a removal
    unmount(prevChild, parentComponent, parentSuspense, true)
    continue
  }
  let newIndex
  if (prevChild.key != null) {
    newIndex = keyToNewIndexMap.get(prevChild.key)
  } else {
    // key-less node, try to locate a key-less node of the same type
    for (j = s2; j <= e2; j++) {
      if (
        newIndexToOldIndexMap[j - s2] === 0 &&
        isSameVNodeType(prevChild, c2[j] as VNode)
      ) {
        newIndex = j
        break
      }
    }
  }
  if (newIndex === undefined) {
    unmount(prevChild, parentComponent, parentSuspense, true)
  } else {
    newIndexToOldIndexMap[newIndex - s2] = i + 1
    if (newIndex >= maxNewIndexSoFar) {
      maxNewIndexSoFar = newIndex
    } else {
      moved = true
    }
    patch(
      prevChild,
      c2[newIndex] as VNode,
      container,
      null,
      parentComponent,
      parentSuspense,
      isSVG,
      slotScopeIds,
      optimized
    )
    patched++
  }
}
```

`patched`记录已处理的节点数

`toBePatched`标识新集合中待处理的节点数

`newIndexToOldIndexMap`记录新集合中的节点在旧集合是否有匹配的项，`0`标识无匹配否则保持相对索引

`maxNewIndexSoFar`与`moved`用于判断是否需进行节点移动

遍历旧集合剩余节点

- 若已处理比待处理节点数大时说明旧集合的未处理节点均为多余节点可直接卸载
- 查询在新集合中的索引
  - 若旧节点有`key`时，从`keyToNewIndexMap`中查询
  - 否则从新集合中遍历查询相似节点
- 进行对比更新
  - 若未查询到则直接卸载旧节点
  - 否则更新数据并进行`patch`

遍历过程中`maxNewIndexSoFar`记录在新集合中查询到的节点的索引最大值，当相对位置未发生变化时`maxNewIndexSoFar`因是增长的将不会更新`moved`，否则将需要移动节点

```
const increasingNewIndexSequence = moved
  ? getSequence(newIndexToOldIndexMap)
  : EMPTY_ARR
j = increasingNewIndexSequence.length - 1
// looping backwards so that we can use last patched node as anchor
for (i = toBePatched - 1; i >= 0; i--) {
  const nextIndex = s2 + i
  const nextChild = c2[nextIndex] as VNode
  const anchor =
    nextIndex + 1 < l2 ? (c2[nextIndex + 1] as VNode).el : parentAnchor
  if (newIndexToOldIndexMap[i] === 0) {
    // mount new
    patch(
      null,
      nextChild,
      container,
      anchor,
      parentComponent,
      parentSuspense,
      isSVG,
      slotScopeIds,
      optimized
    )
  } else if (moved) {
    // move if:
    // There is no stable subsequence (e.g. a reverse)
    // OR current node is not among the stable sequence
    if (j < 0 || i !== increasingNewIndexSequence[j]) {
      move(nextChild, container, anchor, MoveType.REORDER)
    } else {
      j--
    }
  }
}
```

最后处理新集合中未处理节点和移动节点

- 若发生移动则获取最长上升子序列索引
- 倒序遍历新集合剩余节点，因为正序且需要位移时无法获取定位元素
  - 未处理的直接挂载
  - 若需发生位移且未在最长上升子序列中则进行移动

```
function getSequence(arr: number[]): number[] {
  const p = arr.slice()
  const result = [0]
  let i, j, u, v, c
  const len = arr.length
  for (i = 0; i < len; i++) {
    const arrI = arr[i]
    if (arrI !== 0) {
      j = result[result.length - 1]
      if (arr[j] < arrI) {
        p[i] = j
        result.push(i)
        continue
      }
      u = 0
      v = result.length - 1
      while (u < v) {
        c = ((u + v) / 2) | 0
        if (arr[result[c]] < arrI) {
          u = c + 1
        } else {
          v = c
        }
      }
      if (arrI < arr[result[u]]) {
        if (u > 0) {
          p[i] = result[u - 1]
        }
        result[u] = i
      }
    }
  }
  u = result.length
  v = result[u - 1]
  while (u-- > 0) {
    result[u] = v
    v = p[v]
  }
  return result
}
```

`arr`为值集合

`p`为`arr`的副本，会被替换成序列索引

`result`为最长子序列的索引集合，默认最长子序列仅包含第一个数据

`arrI`为当前遍历的值

算法思路如下：

- 过滤`arrI`为`0`的数据处理，因为该数据无可复用的节点
- `arrI`比`result`最后一个索引对应的值大时直接入序列，将`p`对应的值改为在序列中前一个数据在`arr`中的索引
- 否则`arrI`可在`result`中查询出插入位置
  - 通过二分查找查询出插入位置
  - 插入位置为`0`时，将`p`对应的值改为在序列中前一个数据在`arr`中的索引
  - 替换`result`对应位置存储的索引
- 修正，因为插入时未保证数据为同一子序列固需从后往前进行矫正
  - `result`的长度为最长子序列的长度且最后一个必为序列的最后一个值
  - `p`保存的为前一个数据的索引

##### move

```
const move: MoveFn = (
  vnode,
  container,
  anchor,
  moveType,
  parentSuspense = null
) => {
  const { el, type, transition, children, shapeFlag } = vnode
  if (shapeFlag & ShapeFlags.COMPONENT) {
    move(vnode.component!.subTree, container, anchor, moveType)
    return
  }

  if (__FEATURE_SUSPENSE__ && shapeFlag & ShapeFlags.SUSPENSE) {
    vnode.suspense!.move(container, anchor, moveType)
    return
  }

  if (shapeFlag & ShapeFlags.TELEPORT) {
    ;(type as typeof TeleportImpl).move(vnode, container, anchor, internals)
    return
  }

  if (type === Fragment) {
    hostInsert(el!, container, anchor)
    for (let i = 0; i < (children as VNode[]).length; i++) {
      move((children as VNode[])[i], container, anchor, moveType)
    }
    hostInsert(vnode.anchor!, container, anchor)
    return
  }

  if (type === Static) {
    moveStaticNode(vnode, container, anchor)
    return
  }

  // single nodes
  const needTransition =
    moveType !== MoveType.REORDER &&
    shapeFlag & ShapeFlags.ELEMENT &&
    transition
  if (needTransition) {
    if (moveType === MoveType.ENTER) {
      transition!.beforeEnter(el!)
      hostInsert(el!, container, anchor)
      queuePostRenderEffect(() => transition!.enter(el!), parentSuspense)
    } else {
      const { leave, delayLeave, afterLeave } = transition!
      const remove = () => hostInsert(el!, container, anchor)
      const performLeave = () => {
        leave(el!, () => {
          remove()
          afterLeave && afterLeave()
        })
      }
      if (delayLeave) {
        delayLeave(el!, remove, performLeave)
      } else {
        performLeave()
      }
    }
  } else {
    hostInsert(el!, container, anchor)
  }
}
```

移动`vnode`主要进行了下列操作：

1. 若是组件节点，转为移动`subTree`
2. 若是`Suspense`，移交移动权限
3. 若是`Teleport`，移交移动权限
4. 若是`Fragment`，转为移动子节点
5. 若是`Static`，通过`moveStaticNode`处理
6. 若需要动画，进行动画操作
7. 最终通过`hostInsert`进行移动

##### remove

```
const remove: RemoveFn = vnode => {
  const { type, el, anchor, transition } = vnode
  if (type === Fragment) {
    removeFragment(el!, anchor!)
    return
  }

  if (type === Static) {
    removeStaticNode(vnode)
    return
  }

  const performRemove = () => {
    hostRemove(el!)
    if (transition && !transition.persisted && transition.afterLeave) {
      transition.afterLeave()
    }
  }

  if (
    vnode.shapeFlag & ShapeFlags.ELEMENT &&
    transition &&
    !transition.persisted
  ) {
    const { leave, delayLeave } = transition
    const performLeave = () => leave(el!, performRemove)
    if (delayLeave) {
      delayLeave(vnode.el!, performRemove, performLeave)
    } else {
      performLeave()
    }
  } else {
    performRemove()
  }
}
```

移除`vnode`主要进行了下列操作：

1. 若是`Fragment`，通过`removeFragment`处理
2. 若是`Static`，通过`removeStaticNode`处理
3. 若需要动画，进行动画操作
4. 最终通过`hostRemove`进行移除