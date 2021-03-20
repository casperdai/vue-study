```
<Component v-model="someVal"/>

_c('component',{model:{value:(someVal),callback:function ($$v) {someVal=$$v},expression:"someVal"}})],1)}}

src/core/vdom/create-component.js
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  ...
  // transform component v-model data into props & events
  if (isDef(data.model)) {
    transformModel(Ctor.options, data)
  }
  ...
}

// transform component v-model info (value and callback) into
// prop and event handler respectively.
function transformModel (options, data: any) {
  const prop = (options.model && options.model.prop) || 'value'
  const event = (options.model && options.model.event) || 'input'
  ;(data.attrs || (data.attrs = {}))[prop] = data.model.value
  const on = data.on || (data.on = {})
  const existing = on[event]
  const callback = data.model.callback
  if (isDef(existing)) {
    if (
      Array.isArray(existing)
        ? existing.indexOf(callback) === -1
        : existing !== callback
    ) {
      on[event] = [callback].concat(existing)
    }
  } else {
    on[event] = callback
  }
}
```

模板解析后会有一个model属性，记录了当前值与设置方法

当生成vnode前会去解析model属性

根据组件的自定义model映射或默认映射在data.attrs中注入属性并在data.on中添加事件

默认映射为({ prop: 'value', event: 'input' })，这也是平常使用v-model后在组件内定义的接收prop的key为‘value’，变更需触发input事件的原因