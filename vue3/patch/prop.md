```
packages/runtime-dom/src/patchProp.ts
export const patchProp: DOMRendererOptions['patchProp'] = (
  el,
  key,
  prevValue,
  nextValue,
  isSVG = false,
  prevChildren,
  parentComponent,
  parentSuspense,
  unmountChildren
) => {
  switch (key) {
    // special
    case 'class':
      patchClass(el, nextValue, isSVG)
      break
    case 'style':
      patchStyle(el, prevValue, nextValue)
      break
    default:
      if (isOn(key)) {
        // ignore v-model listeners
        if (!isModelListener(key)) {
          patchEvent(el, key, prevValue, nextValue, parentComponent)
        }
      } else if (shouldSetAsProp(el, key, nextValue, isSVG)) {
        patchDOMProp(
          el,
          key,
          nextValue,
          prevChildren,
          parentComponent,
          parentSuspense,
          unmountChildren
        )
      } else {
        // special case for <input v-model type="checkbox"> with
        // :true-value & :false-value
        // store value as dom properties since non-string values will be
        // stringified.
        if (key === 'true-value') {
          ;(el as any)._trueValue = nextValue
        } else if (key === 'false-value') {
          ;(el as any)._falseValue = nextValue
        }
        patchAttr(el, key, nextValue, isSVG)
      }
      break
  }
}
```

节点属性将根据`key`进行不通处理

##### patchClass

```
packages/runtime-dom/src/modules/class.ts
export function patchClass(el: Element, value: string | null, isSVG: boolean) {
  if (value == null) {
    value = ''
  }
  if (isSVG) {
    el.setAttribute('class', value)
  } else {
    // directly setting className should be faster than setAttribute in theory
    // if this is an element during a transition, take the temporary transition
    // classes into account.
    const transitionClasses = (el as ElementWithTransition)._vtc
    if (transitionClasses) {
      value = (value
        ? [value, ...transitionClasses]
        : [...transitionClasses]
      ).join(' ')
    }
    el.className = value
  }
}
```

针对`class`，若有动画时合并，直接赋值于`className`

##### patchStyle

```
packages/runtime-dom/src/modules/style.ts
export function patchStyle(el: Element, prev: Style, next: Style) {
  const style = (el as HTMLElement).style
  if (!next) {
    el.removeAttribute('style')
  } else if (isString(next)) {
    if (prev !== next) {
      const current = style.display
      style.cssText = next
      // indicates that the `display` of the element is controlled by `v-show`,
      // so we always keep the current `display` value regardless of the `style` value,
      // thus handing over control to `v-show`.
      if ('_vod' in el) {
        style.display = current
      }
    }
  } else {
    for (const key in next) {
      setStyle(style, key, next[key])
    }
    if (prev && !isString(prev)) {
      for (const key in prev) {
        if (next[key] == null) {
          setStyle(style, key, '')
        }
      }
    }
  }
}
```

针对`style`

- 若无，通过`removeAttribute`移除
- 若为字符串，直接赋值于`cssText`
- 若为对象，遍历并通过`setStyle`赋值

##### patchEvent

```
packages/runtime-dom/src/modules/events.ts
export function patchEvent(
  el: Element & { _vei?: Record<string, Invoker | undefined> },
  rawName: string,
  prevValue: EventValue | null,
  nextValue: EventValue | null,
  instance: ComponentInternalInstance | null = null
) {
  // vei = vue event invokers
  const invokers = el._vei || (el._vei = {})
  const existingInvoker = invokers[rawName]
  if (nextValue && existingInvoker) {
    // patch
    existingInvoker.value = nextValue
  } else {
    const [name, options] = parseName(rawName)
    if (nextValue) {
      // add
      const invoker = (invokers[rawName] = createInvoker(nextValue, instance))
      addEventListener(el, name, invoker, options)
    } else if (existingInvoker) {
      // remove
      removeEventListener(el, name, existingInvoker, options)
      invokers[rawName] = undefined
    }
  }
}
```

针对事件，直接通过`addEventListener`与`removeEventListener`进行处理

##### patchDOMProp

```
packages/runtime-dom/src/modules/props.ts
export function patchDOMProp(
  el: any,
  key: string,
  value: any,
  // the following args are passed only due to potential innerHTML/textContent
  // overriding existing VNodes, in which case the old tree must be properly
  // unmounted.
  prevChildren: any,
  parentComponent: any,
  parentSuspense: any,
  unmountChildren: any
) {
  if (key === 'innerHTML' || key === 'textContent') {
    if (prevChildren) {
      unmountChildren(prevChildren, parentComponent, parentSuspense)
    }
    el[key] = value == null ? '' : value
    return
  }

  if (key === 'value' && el.tagName !== 'PROGRESS') {
    // store value as _value as well since
    // non-string values will be stringified.
    el._value = value
    const newValue = value == null ? '' : value
    if (el.value !== newValue) {
      el.value = newValue
    }
    return
  }

  if (value === '' || value == null) {
    const type = typeof el[key]
    if (value === '' && type === 'boolean') {
      // e.g. <select multiple> compiles to { multiple: '' }
      el[key] = true
      return
    } else if (value == null && type === 'string') {
      // e.g. <div :id="null">
      el[key] = ''
      el.removeAttribute(key)
      return
    } else if (type === 'number') {
      // e.g. <img :width="null">
      el[key] = 0
      el.removeAttribute(key)
      return
    }
  }

  // some properties perform value validation and throw
  try {
    el[key] = value
  } catch (e) {
    if (__DEV__) {
      warn(
        `Failed setting prop "${key}" on <${el.tagName.toLowerCase()}>: ` +
          `value ${value} is invalid.`,
        e
      )
    }
  }
}
```

针对一些特殊`key`，直接操作`el`

##### patchAttr

```
packages/runtime-dom/src/modules/attrs.ts
export function patchAttr(
  el: Element,
  key: string,
  value: any,
  isSVG: boolean
) {
  if (isSVG && key.startsWith('xlink:')) {
    if (value == null) {
      el.removeAttributeNS(xlinkNS, key.slice(6, key.length))
    } else {
      el.setAttributeNS(xlinkNS, key, value)
    }
  } else {
    // note we are only checking boolean attributes that don't have a
    // corresponding dom prop of the same name here.
    const isBoolean = isSpecialBooleanAttr(key)
    if (value == null || (isBoolean && value === false)) {
      el.removeAttribute(key)
    } else {
      el.setAttribute(key, isBoolean ? '' : value)
    }
  }
}
```

未处理的直接通过`setAttribute`或`removeAttribute`处理