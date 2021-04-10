```
<component @click="click" @click.native="clickNative"></component>
_c('component',{on:{"click":click},nativeOn:{"click":function($event){return clickNative($event)}}})

<div @click.stop.prevent="click"></div>
_c('div',{on:{"click":function($event){$event.stopPropagation();$event.preventDefault();return click($event)}}})

<div @click.self="click"></div>
_c('div',{on:{"click":function($event){if($event.target !== $event.currentTarget)return null;return click($event)}}})

<div @click.left="click"></div>
_c('div',{on:{"click":function($event){if(!$event.type.indexOf('key')&&_k($event.keyCode,"left",37,$event.key,["Left","ArrowLeft"]))return null;if('button' in $event && $event.button !== 0)return null;return click($event)}}})

<div @click.13="click"></div>
_c('div',{on:{"click":function($event){if(!$event.type.indexOf('key')&&$event.keyCode!==13)return null;return click($event)}}})

<div @click.enter="click"></div>
_c('div',{on:{"click":function($event){if(!$event.type.indexOf('key')&&_k($event.keyCode,"enter",13,$event.key,"Enter"))return null;return click($event)}}})

<div @click.someKey="click"></div>
_c('div',{on:{"click":function($event){if(!$event.type.indexOf('key')&&_k($event.keyCode,"someKey",undefined,$event.key,undefined))return null;return click($event)}}})

<div @click.once="click"></div>
_c('div',{on:{"~click":function($event){return click($event)}}})

<div @click.capture="click"></div>
_c('div',{on:{"!click":function($event){return click($event)}}})

<div @click.passive="click"></div>
_c('div',{on:{"&click":function($event){return click($event)}}})

<div v-on="events"></div>
_c('div',_g({},events))

<div @click="click" v-on="events"></div>
_c('div',_g({on:{"click":click}},events))
```

解析成render函数时，根据修饰符进行事件名修改或添加额外处理

```
_g = bindObjectListeners

src/core/instance/render-helpers/bind-object-listeners.js
export function bindObjectListeners (data: any, value: any): VNodeData {
  if (value) {
    if (!isPlainObject(value)) {
      process.env.NODE_ENV !== 'production' && warn(
        'v-on without argument expects an Object value',
        this
      )
    } else {
      const on = data.on = data.on ? extend({}, data.on) : {}
      for (const key in value) {
        const existing = on[key]
        const ours = value[key]
        on[key] = existing ? [].concat(existing, ours) : ours
      }
    }
  }
  return data
}
```

事件描述对象与data.on合并，同event的回调函数合并成回调函数数组

```
src/platforms/web/runtime/modules/events.js
export default {
  create: updateDOMListeners,
  update: updateDOMListeners
}

function updateDOMListeners (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  if (isUndef(oldVnode.data.on) && isUndef(vnode.data.on)) {
    return
  }
  const on = vnode.data.on || {}
  const oldOn = oldVnode.data.on || {}
  target = vnode.elm
  normalizeEvents(on)
  updateListeners(on, oldOn, add, remove, createOnceHandler, vnode.context)
  target = undefined
}

function add (
  name: string,
  handler: Function,
  capture: boolean,
  passive: boolean
) {
  ...
  target.addEventListener(
    name,
    handler,
    supportsPassive
      ? { capture, passive }
      : capture
  )
}

function remove (
  name: string,
  handler: Function,
  capture: boolean,
  _target?: HTMLElement
) {
  (_target || target).removeEventListener(
    name,
    handler._wrapper || handler,
    capture
  )
}
```

事件模块会在create、update两个阶段被触发，用于添加dom事件

1. 处理部分特殊事件
2. 添加与移除dom事件

```
src/core/vdom/helpers/update-listeners.js
const normalizeEvent = cached((name: string): {
  name: string,
  once: boolean,
  capture: boolean,
  passive: boolean,
  handler?: Function,
  params?: Array<any>
} => {
  const passive = name.charAt(0) === '&'
  name = passive ? name.slice(1) : name
  const once = name.charAt(0) === '~' // Prefixed last, checked first
  name = once ? name.slice(1) : name
  const capture = name.charAt(0) === '!'
  name = capture ? name.slice(1) : name
  return {
    name,
    once,
    capture,
    passive
  }
})

export function updateListeners (
  on: Object,
  oldOn: Object,
  add: Function,
  remove: Function,
  createOnceHandler: Function,
  vm: Component
) {
  let name, def, cur, old, event
  for (name in on) {
    def = cur = on[name]
    old = oldOn[name]
    event = normalizeEvent(name)
    /* istanbul ignore if */
    if (__WEEX__ && isPlainObject(def)) {
      cur = def.handler
      event.params = def.params
    }
    if (isUndef(cur)) {
      process.env.NODE_ENV !== 'production' && warn(
        `Invalid handler for event "${event.name}": got ` + String(cur),
        vm
      )
    } else if (isUndef(old)) {
      if (isUndef(cur.fns)) {
        cur = on[name] = createFnInvoker(cur, vm)
      }
      if (isTrue(event.once)) {
        cur = on[name] = createOnceHandler(event.name, cur, event.capture)
      }
      add(event.name, cur, event.capture, event.passive, event.params)
    } else if (cur !== old) {
      old.fns = cur
      on[name] = old
    }
  }
  for (name in oldOn) {
    if (isUndef(on[name])) {
      event = normalizeEvent(name)
      remove(event.name, oldOn[name], event.capture)
    }
  }
}
```

针对每个事件的处理主要进行了下列操作

1. 根据事件名获取dom事件描述
2. 若事件之前不存在，根据回调函数创建一个调用函数，然后根据dom事件描述与调用函数添加事件监听
3. 若事件之前存在且不相同，复用调用函数并修改调用函数的回调函数指向
4. 将多余的事件移除