```
<div style="key:val;"></div>
_c('div',{staticStyle:{"key":"val"}})

<div :style="styleProps"></div>
_c('div',{style:styleProps})
```

解析成render函数时，根据是否为绑定属性分别存储为style和staticStyle，直接以字面量组装

```
src/platforms/web/runtime/modules/class.js
export default {
  create: updateStyle,
  update: updateStyle
}

function updateStyle (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  const data = vnode.data
  const oldData = oldVnode.data

  if (isUndef(data.staticStyle) && isUndef(data.style) &&
    isUndef(oldData.staticStyle) && isUndef(oldData.style)
  ) {
    return
  }

  let cur, name
  const el: any = vnode.elm
  const oldStaticStyle: any = oldData.staticStyle
  const oldStyleBinding: any = oldData.normalizedStyle || oldData.style || {}

  // if static style exists, stylebinding already merged into it when doing normalizeStyleData
  const oldStyle = oldStaticStyle || oldStyleBinding

  const style = normalizeStyleBinding(vnode.data.style) || {}

  // store normalized style under a different key for next diff
  // make sure to clone it if it's reactive, since the user likely wants
  // to mutate it.
  vnode.data.normalizedStyle = isDef(style.__ob__)
    ? extend({}, style)
    : style

  const newStyle = getStyle(vnode, true)

  for (name in oldStyle) {
    if (isUndef(newStyle[name])) {
      setProp(el, name, '')
    }
  }
  for (name in newStyle) {
    cur = newStyle[name]
    if (cur !== oldStyle[name]) {
      // ie9 setting to null has no effect, must use empty string
      setProp(el, name, cur == null ? '' : cur)
    }
  }
}
```

样式模块会在create、update三个阶段被触发，用于设置dom元素的style

1. 若是无style相关属性直接跳过style处理
2. 获取旧style描述
3. 获取新style描述，normalizedStyle为style合并后的数据，然后normalizedStyle覆盖合并至staticStyle
4. 若有多余的样式则进行移除
5. 若有值不一致的样式则进行设置

```
src/platforms/web/util/style.js
export function normalizeStyleBinding (bindingStyle: any): ?Object {
  if (Array.isArray(bindingStyle)) {
    return toObject(bindingStyle)
  }
  if (typeof bindingStyle === 'string') {
    return parseStyleText(bindingStyle)
  }
  return bindingStyle
}
```

- 若style为数组，遍历并合并
- 若style为字符串，解析成对象
- 其他则直接返回

因为数组是通过toObject解析，所以style为数组时内容只能为对象且同key样式后面会覆盖前面的

```
src/platforms/web/util/style.js
function normalizeStyleData (data: VNodeData): ?Object {
  const style = normalizeStyleBinding(data.style)
  // static style is pre-processed into an object during compilation
  // and is always a fresh object, so it's safe to merge into it
  return data.staticStyle
    ? extend(data.staticStyle, style)
    : style
}

export function getStyle (vnode: VNodeWithData, checkChild: boolean): Object {
  const res = {}
  let styleData

  if (checkChild) {
    let childNode = vnode
    while (childNode.componentInstance) {
      childNode = childNode.componentInstance._vnode
      if (
        childNode && childNode.data &&
        (styleData = normalizeStyleData(childNode.data))
      ) {
        extend(res, styleData)
      }
    }
  }

  if ((styleData = normalizeStyleData(vnode.data))) {
    extend(res, styleData)
  }

  let parentNode = vnode
  while ((parentNode = parentNode.parent)) {
    if (parentNode.data && (styleData = normalizeStyleData(parentNode.data))) {
      extend(res, styleData)
    }
  }
  return res
}
```

- 若有实例，与实例vnode合并style描述直至不为组件vnode
- 合并自身style描述
- 若有父vnode，与父vnode合并style描述直至无父vnode

这也是可以为组件扩展style的原因

因normalizeStyleData时若有staticStyle则将style覆盖更新至staticStyle，所以同key样式style会覆盖staticStyle

因获取style描述时是从实例vnode至组件vnode再到父vnode进行合并的，所以同key使用父vnode的样式