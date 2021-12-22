```
packages/runtime-core/src/vnode.ts
let currentBlock: VNode[] | null = null

export function openBlock(disableTracking = false) {
  blockStack.push((currentBlock = disableTracking ? null : []))
}

export function closeBlock() {
  blockStack.pop()
  currentBlock = blockStack[blockStack.length - 1] || null
}

export function createBlock(
  type: VNodeTypes | ClassComponent,
  props?: Record<string, any> | null,
  children?: any,
  patchFlag?: number,
  dynamicProps?: string[]
): VNode {
  const vnode = createVNode(
    type,
    props,
    children,
    patchFlag,
    dynamicProps,
    true /* isBlock: prevent a block from tracking itself */
  )
  // save current block children on the block vnode
  vnode.dynamicChildren = currentBlock || (EMPTY_ARR as any)
  // close block
  closeBlock()
  // a block is always going to be patched, so track it as a child of its
  // parent block
  if (shouldTrack > 0 && currentBlock) {
    currentBlock.push(vnode)
  }
  return vnode
}
```

`block`用于优化`patch`，当创建后子节点若需要更新则会加入`dynamicChildren`中，未加入的节点仅进行初次渲染后期不会参与`patch`

开启后创建的`vnode`与通过`h`创建的唯一区别就是`dynamicChildren`

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
  ...
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
  } else if (!optimized) {
    ...
  }
  ...
}
```

以元素节点为例，若存在`dynamicChildren`则仅对`dynamicChildren`进行处理

```
const patchBlockChildren: PatchBlockChildrenFn = (
  oldChildren,
  newChildren,
  fallbackContainer,
  parentComponent,
  parentSuspense,
  isSVG,
  slotScopeIds
) => {
  for (let i = 0; i < newChildren.length; i++) {
    const oldVNode = oldChildren[i]
    const newVNode = newChildren[i]
    // Determine the container (parent element) for the patch.
    const container =
      // - In the case of a Fragment, we need to provide the actual parent
      // of the Fragment itself so it can move its children.
      oldVNode.type === Fragment ||
      // - In the case of different nodes, there is going to be a replacement
      // which also requires the correct parent container
      !isSameVNodeType(oldVNode, newVNode) ||
      // - In the case of a component, it could contain anything.
      oldVNode.shapeFlag & ShapeFlags.COMPONENT ||
      oldVNode.shapeFlag & ShapeFlags.TELEPORT
        ? hostParentNode(oldVNode.el!)!
        : // In other cases, the parent container is not actually used so we
          // just pass the block element here to avoid a DOM parentNode call.
          fallbackContainer
    patch(
      oldVNode,
      newVNode,
      container,
      null,
      parentComponent,
      parentSuspense,
      isSVG,
      slotScopeIds,
      true
    )
  }
}
```

遍历`dynamicChildren`通过`patch`进行处理