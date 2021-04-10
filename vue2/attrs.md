```
<div prop="propVal"></div>
_c('div',{attrs:{"prop":"propVal"}})

<div :prop="propVal"></div>
_c('div',{attrs:{"prop":propVal}})

<div :prop-key.camel="propVal"></div>
_c('div',{attrs:{"propKey":propVal}})
不添加时为prop-key

<div :prop.sync="propVal"></div>
_c('div',{attrs:{"prop":propVal},on:{"update:prop":function($event){propVal=$event}}})

<div v-bind.sync="props"></div
_c('div',_b({},'div',props,false,true))
```

解析成render函数时，根据修饰符进行特性名修改或添加额外处理

```
_b = bindObjectProps

src/core/instance/render-helpers/bind-object-props.js
export function bindObjectProps (
  data: any,
  tag: string,
  value: any,
  asProp: boolean,
  isSync?: boolean
): VNodeData {
  if (value) {
    if (!isObject(value)) {
      process.env.NODE_ENV !== 'production' && warn(
        'v-bind without argument expects an Object or Array value',
        this
      )
    } else {
      if (Array.isArray(value)) {
        value = toObject(value)
      }
      let hash
      for (const key in value) {
        if (
          key === 'class' ||
          key === 'style' ||
          isReservedAttribute(key)
        ) {
          hash = data
        } else {
          const type = data.attrs && data.attrs.type
          hash = asProp || config.mustUseProp(tag, type, key)
            ? data.domProps || (data.domProps = {})
            : data.attrs || (data.attrs = {})
        }
        const camelizedKey = camelize(key)
        const hyphenatedKey = hyphenate(key)
        if (!(camelizedKey in hash) && !(hyphenatedKey in hash)) {
          hash[key] = value[key]

          if (isSync) {
            const on = data.on || (data.on = {})
            on[`update:${key}`] = function ($event) {
              value[key] = $event
            }
          }
        }
      }
    }
  }
  return data
}
```

属性描述对象中满足下列条件的属性合并至data.attrs中

- 不是保留字段
- 未添加prop修饰符
- vnode不是元素vnode或不是元素的特殊特性
- data.attrs未定义同名属性

因为数组是通过toObject解析，所以数组时内容只能为对象且同key属性后面会覆盖前面的

```
src/platforms/web/runtime/modules/attrs.js
export default {
  create: updateAttrs,
  update: updateAttrs
}

function updateAttrs (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  const opts = vnode.componentOptions
  if (isDef(opts) && opts.Ctor.options.inheritAttrs === false) {
    return
  }
  if (isUndef(oldVnode.data.attrs) && isUndef(vnode.data.attrs)) {
    return
  }
  let key, cur, old
  const elm = vnode.elm
  const oldAttrs = oldVnode.data.attrs || {}
  let attrs: any = vnode.data.attrs || {}
  // clone observed objects, as the user probably wants to mutate it
  if (isDef(attrs.__ob__)) {
    attrs = vnode.data.attrs = extend({}, attrs)
  }

  for (key in attrs) {
    cur = attrs[key]
    old = oldAttrs[key]
    if (old !== cur) {
      setAttr(elm, key, cur)
    }
  }
  // #4391: in IE9, setting type can reset value for input[type=radio]
  // #6666: IE/Edge forces progress value down to 1 before setting a max
  /* istanbul ignore if */
  if ((isIE || isEdge) && attrs.value !== oldAttrs.value) {
    setAttr(elm, 'value', attrs.value)
  }
  for (key in oldAttrs) {
    if (isUndef(attrs[key])) {
      if (isXlink(key)) {
        elm.removeAttributeNS(xlinkNS, getXlinkProp(key))
      } else if (!isEnumeratedAttr(key)) {
        elm.removeAttribute(key)
      }
    }
  }
}
```

属性模块会在create、update两个阶段被触发，用于将未被props引用的atrrs设置至dom元素或从dom元素上移除

1. 若是组件vnode且组件定义inheritAttrs为false时或无attrs时跳过attrs处理
2. 若有值不一致的特性则进行设置
3. 若有多余的特性则进行移除
