```
<div class="classname"></div>
_c('div',{staticClass:"classname"})

<div :class="classProps"></div>
_c('div',{class:classProps})
```

解析成render函数时，根据是否为绑定属性分别存储为class和staticClass，直接以字面量组装

```
src/platforms/web/runtime/modules/class.js
export default {
  create: updateClass,
  update: updateClass
}

function updateClass (oldVnode: any, vnode: any) {
  const el = vnode.elm
  const data: VNodeData = vnode.data
  const oldData: VNodeData = oldVnode.data
  if (
    isUndef(data.staticClass) &&
    isUndef(data.class) && (
      isUndef(oldData) || (
        isUndef(oldData.staticClass) &&
        isUndef(oldData.class)
      )
    )
  ) {
    return
  }

  let cls = genClassForVnode(vnode)

  // handle transition classes
  const transitionClass = el._transitionClasses
  if (isDef(transitionClass)) {
    cls = concat(cls, stringifyClass(transitionClass))
  }

  // set the class
  if (cls !== el._prevClass) {
    el.setAttribute('class', cls)
    el._prevClass = cls
  }
}
```

类名模块会在create、update三个阶段被触发，用于设置dom元素的class

1. 若是无class相关属性直接跳过class处理
2. 获取class
3. 若与当前class不一致时更新class

```
src/platforms/web/util/class.js
export function genClassForVnode (vnode: VNodeWithData): string {
  let data = vnode.data
  let parentNode = vnode
  let childNode = vnode
  while (isDef(childNode.componentInstance)) {
    childNode = childNode.componentInstance._vnode
    if (childNode && childNode.data) {
      data = mergeClassData(childNode.data, data)
    }
  }
  while (isDef(parentNode = parentNode.parent)) {
    if (parentNode && parentNode.data) {
      data = mergeClassData(data, parentNode.data)
    }
  }
  return renderClass(data.staticClass, data.class)
}
```

- 若有实例，与实例vnode合并class描述直至不为组件vnode
- 若有父vnode，与父vnode合并class描述直至无父vnode

这也是可以为组件扩展class的原因

```
src/platforms/web/util/class.js
function mergeClassData (child: VNodeData, parent: VNodeData): {
  staticClass: string,
  class: any
} {
  return {
    staticClass: concat(child.staticClass, parent.staticClass),
    class: isDef(child.class)
      ? [child.class, parent.class]
      : parent.class
  }
}
```

合并时，staticClass直接拼接，class组装成嵌套数组（每个层级均为子节点class描述在前）

```
src/platforms/web/util/class.js
export function renderClass (
  staticClass: ?string,
  dynamicClass: any
): string {
  if (isDef(staticClass) || isDef(dynamicClass)) {
    return concat(staticClass, stringifyClass(dynamicClass))
  }
  /* istanbul ignore next */
  return ''
}

export function concat (a: ?string, b: ?string): string {
  return a ? b ? (a + ' ' + b) : a : (b || '')
}

export function stringifyClass (value: any): string {
  if (Array.isArray(value)) {
    return stringifyArray(value)
  }
  if (isObject(value)) {
    return stringifyObject(value)
  }
  if (typeof value === 'string') {
    return value
  }
  /* istanbul ignore next */
  return ''
}

function stringifyArray (value: Array<any>): string {
  let res = ''
  let stringified
  for (let i = 0, l = value.length; i < l; i++) {
    if (isDef(stringified = stringifyClass(value[i])) && stringified !== '') {
      if (res) res += ' '
      res += stringified
    }
  }
  return res
}

function stringifyObject (value: Object): string {
  let res = ''
  for (const key in value) {
    if (value[key]) {
      if (res) res += ' '
      res += key
    }
  }
  return res
}
```

解析时将staticClass与遍历拼接后的class进行拼接

遍历class时，若为数组递归遍历，若为对象且当val为truly拼接key，若为字符串直接拼接

因是字符串拼接未校验重复性，所以会有重复class name的情况