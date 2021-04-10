```
src/compiler/optimizer.js
export function optimize (root: ?ASTElement, options: CompilerOptions) {
  if (!root) return
  isStaticKey = genStaticKeysCached(options.staticKeys || '')
  isPlatformReservedTag = options.isReservedTag || no
  // first pass: mark all non-static nodes.
  markStatic(root)
  // second pass: mark static roots.
  markStaticRoots(root, false)
}
```

optimizer主要进行静态渲染标记，可避免重复生成vnode

##### isStaticKey

```
src/compiler/optimizer.js
const genStaticKeysCached = cached(genStaticKeys)

function genStaticKeys (keys: string): Function {
  return makeMap(
    'type,tag,attrsList,attrsMap,plain,parent,children,attrs,start,end,rawAttrsMap' +
    (keys ? ',' + keys : '')
  )
}

src/platforms/web/compiler/options.js
export const baseOptions: CompilerOptions = {
  ...
  staticKeys: genStaticKeys(modules)
}

src/shared/util.js
export function genStaticKeys (modules: Array<ModuleOptions>): string {
  return modules.reduce((keys, m) => {
    return keys.concat(m.staticKeys || [])
  }, []).join(',')
}
```

isStaticKey用于校验静态属性或默认属性，可以看出先从各模块取出静态属性的key然后与默认属性拼接后进行缓存

##### isPlatformReservedTag

```
src/platforms/web/compiler/options.js
export const baseOptions: CompilerOptions = {
  ...
  isReservedTag
  ...
}

src/platforms/web/util/element.js
export const isReservedTag = (tag: string): ?boolean => {
  return isHTMLTag(tag) || isSVG(tag)
}
```

该函数用于检测是否为保留标签即html标签或svg

##### markStatic

```
src/compiler/optimizer.js
function markStatic (node: ASTNode) {
  node.static = isStatic(node)
  if (node.type === 1) {
    // do not make component slot content static. this avoids
    // 1. components not able to mutate slot nodes
    // 2. static slot content fails for hot-reloading
    if (
      !isPlatformReservedTag(node.tag) &&
      node.tag !== 'slot' &&
      node.attrsMap['inline-template'] == null
    ) {
      return
    }
    for (let i = 0, l = node.children.length; i < l; i++) {
      const child = node.children[i]
      markStatic(child)
      if (!child.static) {
        node.static = false
      }
    }
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        const block = node.ifConditions[i].block
        markStatic(block)
        if (!block.static) {
          node.static = false
        }
      }
    }
  }
}
```

该函数用于标记节点是否可进行静态渲染

若是标签节点，除非是非slot且不是内联模板时，否则必须所有子节点均满足静态渲染的情况自身才能进行静态渲染

```
src/compiler/optimizer.js
function isStatic (node: ASTNode): boolean {
  if (node.type === 2) { // expression
    return false
  }
  if (node.type === 3) { // text
    return true
  }
  return !!(node.pre || (
    !node.hasBindings && // no dynamic bindings
    !node.if && !node.for && // not v-if or v-for or v-else
    !isBuiltInTag(node.tag) && // not a built-in
    isPlatformReservedTag(node.tag) && // not a component
    !isDirectChildOfTemplateFor(node) &&
    Object.keys(node).every(isStaticKey)
  ))
}
```

根据下列情况进行初始static标识：

- 若为含义声明式插值的文本节点必定不能进行静态渲染
- 若为纯文本可进行静态渲染
- 有v-pre指令可进行静态渲染
- 无绑定属性、无v-if和v-for指令、非内置标签（slot、component）、非保留标签、非template标签或有v-for指令标签的直接子节点且仅包含静态属性或默认属性时可进行静态渲染

##### markStaticRoots

```
src/compiler/optimizer.js
function markStaticRoots (node: ASTNode, isInFor: boolean) {
  if (node.type === 1) {
    if (node.static || node.once) {
      node.staticInFor = isInFor
    }
    // For a node to qualify as a static root, it should have children that
    // are not just static text. Otherwise the cost of hoisting out will
    // outweigh the benefits and it's better off to just always render it fresh.
    if (node.static && node.children.length && !(
      node.children.length === 1 &&
      node.children[0].type === 3
    )) {
      node.staticRoot = true
      return
    } else {
      node.staticRoot = false
    }
    if (node.children) {
      for (let i = 0, l = node.children.length; i < l; i++) {
        markStaticRoots(node.children[i], isInFor || !!node.for)
      }
    }
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        markStaticRoots(node.ifConditions[i].block, isInFor)
      }
    }
  }
}
```

该函数用于标记节点是否应静态渲染，仅处理标签节点，主要进行了下列操作：

- 节点可进行静态渲染或有v-once指令且处于v-for指令范围则设置staticInFor为true
- 节点可进行静态渲染、有子节点并且子节点不是唯一的和不是纯文本节点则可进行静态渲染则设置staticRoot为true，否则继续遍历子节点和条件数组

