```
src/compiler/codegen/index.js
export function generate (
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  const state = new CodegenState(options)
  const code = ast ? genElement(ast, state) : '_c("div")'
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns
  }
}
```

generate用来将ast解析成对应的code string，最终返回render函数内容与静态渲染函数内容集合

##### genElement

```
export function genElement (el: ASTElement, state: CodegenState): string {
  if (el.parent) {
    el.pre = el.pre || el.parent.pre
  }

  if (el.staticRoot && !el.staticProcessed) {
    return genStatic(el, state)
  } else if (el.once && !el.onceProcessed) {
    return genOnce(el, state)
  } else if (el.for && !el.forProcessed) {
    return genFor(el, state)
  } else if (el.if && !el.ifProcessed) {
    return genIf(el, state)
  } else if (el.tag === 'template' && !el.slotTarget && !state.pre) {
    return genChildren(el, state) || 'void 0'
  } else if (el.tag === 'slot') {
    return genSlot(el, state)
  } else {
    // component or element
    let code
    if (el.component) {
      code = genComponent(el.component, el, state)
    } else {
      let data
      if (!el.plain || (el.pre && state.maybeComponent(el))) {
        data = genData(el, state)
      }

      const children = el.inlineTemplate ? null : genChildren(el, state, true)
      code = `_c('${el.tag}'${
        data ? `,${data}` : '' // data
      }${
        children ? `,${children}` : '' // children
      })`
    }
    // module transforms
    for (let i = 0; i < state.transforms.length; i++) {
      code = state.transforms[i](el, code)
    }
    return code
  }
}
```

根据节点的各种标识进行函数选择，将节点相关数据转化成字符串进行拼接

##### genStatic

```
function genStatic (el: ASTElement, state: CodegenState): string {
  el.staticProcessed = true
  // Some elements (templates) need to behave differently inside of a v-pre
  // node.  All pre nodes are static roots, so we can use this as a location to
  // wrap a state change and reset it upon exiting the pre node.
  const originalPreState = state.pre
  if (el.pre) {
    state.pre = el.pre
  }
  state.staticRenderFns.push(`with(this){return ${genElement(el, state)}}`)
  state.pre = originalPreState
  return `_m(${
    state.staticRenderFns.length - 1
  }${
    el.staticInFor ? ',true' : ''
  })`
}
```

静态渲染，将**genElement**返回的内容存放至**staticRenderFns**，用**"_m"**包裹下标然后返回

```
<div><span></span></div>

{
  type: 1,
  tag: 'div',
  staticRoot: true,
  children: [
    {
  	  type: 1,
  	  tag: 'span',
  	  ...
	}
  ]
  ...
}

static.staticRenderFns
[ "with(this){return _c('div',[_c('span')])}" ]

_m(0)
```

##### genOnce

```
function genOnce (el: ASTElement, state: CodegenState): string {
  el.onceProcessed = true
  if (el.if && !el.ifProcessed) {
    return genIf(el, state)
  } else if (el.staticInFor) {
    let key = ''
    let parent = el.parent
    while (parent) {
      if (parent.for) {
        key = parent.key
        break
      }
      parent = parent.parent
    }
    if (!key) {
      process.env.NODE_ENV !== 'production' && state.warn(
        `v-once can only be used inside v-for that is keyed. `,
        el.rawAttrsMap['v-once']
      )
      return genElement(el, state)
    }
    return `_o(${genElement(el, state)},${state.onceId++},${key})`
  } else {
    return genStatic(el, state)
  }
}
```

处理有v-once指令的节点

- 若带有**v-if**且未处理返回**genIf**的处理结果
- 若处于**v-for**范围内时
  - 若未定义key返回**genElement**的处理结果
  - 否则通过**genElement**进行处理后用**"_o"**包裹进行返回
- 否则返回**genStatic**的处理结果

```
<div v-for="val in 10" :key="val"><span v-once></span></div>

{
  type: 1,
  tag: 'dov',
  for: '10',
  key: 'val'
  children: [
    {
  	  type: 1,
  	  tag: 'span',
  	  once: true,
  	  ...
	}
  ]
  ...
}

_o(_c('span'),0,val)
```

##### genFor

```
export function genFor (
  el: any,
  state: CodegenState,
  altGen?: Function,
  altHelper?: string
): string {
  const exp = el.for
  const alias = el.alias
  const iterator1 = el.iterator1 ? `,${el.iterator1}` : ''
  const iterator2 = el.iterator2 ? `,${el.iterator2}` : ''

  el.forProcessed = true // avoid recursion
  return `${altHelper || '_l'}((${exp}),` +
    `function(${alias}${iterator1}${iterator2}){` +
      `return ${(altGen || genElement)(el, state)}` +
    '})'
}
```

处理有v-for指令的节点，用"_l"包裹genStatic的处理结果然后返回

```
<span v-for="(item,index) in arr"></span>

{
  type: 1,
  tag: 'span',
  for: 'arr',
  alias: 'item',
  iterator1: 'index'
  ...
}

_l((arr),function(item,index){return _c('span')})
```

##### genIf

```
export function genIf (
  el: any,
  state: CodegenState,
  altGen?: Function,
  altEmpty?: string
): string {
  el.ifProcessed = true // avoid recursion
  return genIfConditions(el.ifConditions.slice(), state, altGen, altEmpty)
}

function genIfConditions (
  conditions: ASTIfConditions,
  state: CodegenState,
  altGen?: Function,
  altEmpty?: string
): string {
  if (!conditions.length) {
    return altEmpty || '_e()'
  }

  const condition = conditions.shift()
  if (condition.exp) {
    return `(${condition.exp})?${
      genTernaryExp(condition.block)
    }:${
      genIfConditions(conditions, state, altGen, altEmpty)
    }`
  } else {
    return `${genTernaryExp(condition.block)}`
  }

  // v-if with v-once should generate code like (a)?_m(0):_m(1)
  function genTernaryExp (el) {
    return altGen
      ? altGen(el, state)
      : el.once
        ? genOnce(el, state)
        : genElement(el, state)
  }
}
```

处理v-if相关指令，主要进行了下列操作：

- 顺序执行ifConditions中每个条件
- 通过三元表达式连接每个条件的处理
- 单个条件如果条件对应的节点带有**v-once**则通过**genOnce**处理否则通过genElement处理

```
<span v-if="type === 1"></span><span v-else-if="type === 2"></span><span v-else></span>

{
  type: 1,
  tag: 'span',
  if: 'type === 1',
  ifConditions: [
    { exp: 'type === 1', block: span1 },
    { exp: 'type === 2', block: span2 },
    { exp: undefined, block: span3 }
  ]
  ...
}

(type === 1)?genTernaryExp(span1):(type === 2)?genTernaryExp(span2):genTernaryExp(span3)
```

##### genChildren

```
export function genChildren (
  el: ASTElement,
  state: CodegenState,
  checkSkip?: boolean,
  altGenElement?: Function,
  altGenNode?: Function
): string | void {
  const children = el.children
  if (children.length) {
    const el: any = children[0]
    // optimize single v-for
    if (children.length === 1 &&
      el.for &&
      el.tag !== 'template' &&
      el.tag !== 'slot'
    ) {
      const normalizationType = checkSkip
        ? state.maybeComponent(el) ? `,1` : `,0`
        : ``
      return `${(altGenElement || genElement)(el, state)}${normalizationType}`
    }
    const normalizationType = checkSkip
      ? getNormalizationType(children, state.maybeComponent)
      : 0
    const gen = altGenNode || genNode
    return `[${children.map(c => gen(c, state)).join(',')}]${
      normalizationType ? `,${normalizationType}` : ''
    }`
  }
}

function genNode (node: ASTNode, state: CodegenState): string {
  if (node.type === 1) {
    return genElement(node, state)
  } else if (node.type === 3 && node.isComment) {
    return genComment(node)
  } else {
    return genText(node)
  }
}
```

处理子节点，主要进行了下列操作：

- 获取子节点处理方式
  - 默认类别为**0**，代表不处理
  - 子节点存在组件节点时，方式类别为**1**，代表生成vnode实例的子节点数组需进行1级展开操作
  - 子节点存在**v-for**指令或标签为**template**或**slot**时，方式类别为**2**，代表需进行子节点数组可能存在嵌套需多级展开操作
- 根据子节点type进行对应操作

##### genComment

```
export function genComment (comment: ASTText): string {
  return `_e(${JSON.stringify(comment.text)})`
}
```

注释节点，用**“_e”**包裹序列化（加引号）后的内容

##### genText

```
export function genText (text: ASTText | ASTExpression): string {
  return `_v(${text.type === 2
    ? text.expression // no need for () because already wrapped in _s()
    : transformSpecialNewlines(JSON.stringify(text.text))
  })`
}
```

文本节点，用**“_v”**包裹内容，若不是表达式需进行序列化

##### genSlot

```
function genSlot (el: ASTElement, state: CodegenState): string {
  const slotName = el.slotName || '"default"'
  const children = genChildren(el, state)
  let res = `_t(${slotName}${children ? `,${children}` : ''}`
  const attrs = el.attrs || el.dynamicAttrs
    ? genProps((el.attrs || []).concat(el.dynamicAttrs || []).map(attr => ({
        // slot props are camelized
        name: camelize(attr.name),
        value: attr.value,
        dynamic: attr.dynamic
      })))
    : null
  const bind = el.attrsMap['v-bind']
  if ((attrs || bind) && !children) {
    res += `,null`
  }
  if (attrs) {
    res += `,${attrs}`
  }
  if (bind) {
    res += `${attrs ? '' : ',null'},${bind}`
  }
  return res + ')'
}
```

处理插槽，主要进行了下列操作：

1. 拼接通过**genChildren**获取的子节点内容
2. 拼接**attrs**和**dynamicAttrs**
3. 拼接**v-bind内容**
4. 用**“_t”**包裹

```
<slot :prop="val" v-bind="mprops"/>

{
  type: 1,
  tag: 'slot',
  slotName: undefined,
  attrs: [{ name: 'prop', value: 'val' }],
  attrsMap: {
    'v-bind': 'mprops'
    ...
  },
  children: [],
  ...
}

_t("default",null,{"propKey":val},mprops)
```

##### genComponent

```
function genComponent (
  componentName: string,
  el: ASTElement,
  state: CodegenState
): string {
  const children = el.inlineTemplate ? null : genChildren(el, state, true)
  return `_c(${componentName},${genData(el, state)}${
    children ? `,${children}` : ''
  })`
}
```

处理is，和一般的标签节点类似仅tag位置使用ASTElement实例的component属性

##### genData

```
export function genData (el: ASTElement, state: CodegenState): string {
  let data = '{'

  // directives first.
  // directives may mutate the el's other properties before they are generated.
  const dirs = genDirectives(el, state)
  if (dirs) data += dirs + ','

  // key
  if (el.key) {
    data += `key:${el.key},`
  }
  // ref
  if (el.ref) {
    data += `ref:${el.ref},`
  }
  if (el.refInFor) {
    data += `refInFor:true,`
  }
  // pre
  if (el.pre) {
    data += `pre:true,`
  }
  // record original tag name for components using "is" attribute
  if (el.component) {
    data += `tag:"${el.tag}",`
  }
  // module data generation functions
  for (let i = 0; i < state.dataGenFns.length; i++) {
    data += state.dataGenFns[i](el)
  }
  // attributes
  if (el.attrs) {
    data += `attrs:${genProps(el.attrs)},`
  }
  // DOM props
  if (el.props) {
    data += `domProps:${genProps(el.props)},`
  }
  // event handlers
  if (el.events) {
    data += `${genHandlers(el.events, false)},`
  }
  if (el.nativeEvents) {
    data += `${genHandlers(el.nativeEvents, true)},`
  }
  // slot target
  // only for non-scoped slots
  if (el.slotTarget && !el.slotScope) {
    data += `slot:${el.slotTarget},`
  }
  // scoped slots
  if (el.scopedSlots) {
    data += `${genScopedSlots(el, el.scopedSlots, state)},`
  }
  // component v-model
  if (el.model) {
    data += `model:{value:${
      el.model.value
    },callback:${
      el.model.callback
    },expression:${
      el.model.expression
    }},`
  }
  // inline-template
  if (el.inlineTemplate) {
    const inlineTemplate = genInlineTemplate(el, state)
    if (inlineTemplate) {
      data += `${inlineTemplate},`
    }
  }
  data = data.replace(/,$/, '') + '}'
  // v-bind dynamic argument wrap
  // v-bind with dynamic arguments must be applied using the same v-bind object
  // merge helper so that class/style/mustUseProp attrs are handled correctly.
  if (el.dynamicAttrs) {
    data = `_b(${data},"${el.tag}",${genProps(el.dynamicAttrs)})`
  }
  // v-bind data wrap
  if (el.wrapData) {
    data = el.wrapData(data)
  }
  // v-on data wrap
  if (el.wrapListeners) {
    data = el.wrapListeners(data)
  }
  return data
}
```

处理节点数据，最终数据是一个对象的序列化字符串，主要进行了下列操作：

1. 拼接通过**genDirectives**处理而得的数据
2. 若有key，添加**key**属性值为**key对应的值**
3. 若有ref，添加**ref**属性值为**ref对应的值**
4. 若有refInFor，添加**refInFor**属性并设置为**true**
5. 若有pre，添加**pre**属性并设置为**true**
6. 若有component，添加**tag**属性值为**源tag**
7. 拼接通过模块属性生成器而得的数据
8. 若有attrs，添加**attrs**属性值为通过**genProps**处理而得的数据
9. 若有props，添加**domProps**属性值为通过**genProps**处理而得的数据
10. 若有evetns，拼接通过**genHandlers**处理而得的数据
11. 若有nativeEvents，拼接通过**genHandlers**处理而得的数据
12. 若有slotTarget且无自定义作用域slotScope，添加**slot**属性值为**slotTarget**（旧API）
13. 若有scopedSlots，拼接通过**genScopedSlots**处理而得的数据
14. 若有model，添加**model**属性值为**model序列化字符串**
15. 若有inlineTemplate，拼接通过**genInlineTemplate**处理而得的数据
16. 完成data初始拼接
17. 若有**dynamicAttrs**，用**“_b”**包裹通过**genProps**处理而得的数据
18. 若有wrapData，通过**wrapData**进行扩展
19. 若有wrapListeners，通过**wrapListeners**进行扩展

##### genDirectives

```
function genDirectives (el: ASTElement, state: CodegenState): string | void {
  const dirs = el.directives
  if (!dirs) return
  let res = 'directives:['
  let hasRuntime = false
  let i, l, dir, needRuntime
  for (i = 0, l = dirs.length; i < l; i++) {
    dir = dirs[i]
    needRuntime = true
    const gen: DirectiveFunction = state.directives[dir.name]
    if (gen) {
      // compile-time directive that manipulates AST.
      // returns true if it also needs a runtime counterpart.
      needRuntime = !!gen(el, dir, state.warn)
    }
    if (needRuntime) {
      hasRuntime = true
      res += `{name:"${dir.name}",rawName:"${dir.rawName}"${
        dir.value ? `,value:(${dir.value}),expression:${JSON.stringify(dir.value)}` : ''
      }${
        dir.arg ? `,arg:${dir.isDynamicArg ? dir.arg : `"${dir.arg}"`}` : ''
      }${
        dir.modifiers ? `,modifiers:${JSON.stringify(dir.modifiers)}` : ''
      }},`
    }
  }
  if (hasRuntime) {
    return res.slice(0, -1) + ']'
  }
}
```

处理指令

```
v-directive:arg.modifier="val"

`directives:[{
  name:"directive",
  rawName:"v-directive:arg.modifier",
  value:val,
  expression:"val",
  arg:"arg",
  modifiers:{"modifier":true}
}]`
```

理论上指令经过解析后将转化为上述格式，但是必须存在标识了needRuntime的指令，needRuntime标识指令是否需要拼接即需要运行时解析

满足下列情况的指令才会需要进行拼接：

- **state.directives**中不存在对应的处理函数即自定义指令
- 内置指令处理后明确返回需要拼接，即处理函数返回true

##### **genProps**

```
function genProps (props: Array<ASTAttr>): string {
  let staticProps = ``
  let dynamicProps = ``
  for (let i = 0; i < props.length; i++) {
    const prop = props[i]
    const value = __WEEX__
      ? generateValue(prop.value)
      : transformSpecialNewlines(prop.value)
    if (prop.dynamic) {
      dynamicProps += `${prop.name},${value},`
    } else {
      staticProps += `"${prop.name}":${value},`
    }
  }
  staticProps = `{${staticProps.slice(0, -1)}}`
  if (dynamicProps) {
    return `_d(${staticProps},[${dynamicProps.slice(0, -1)}])`
  } else {
    return staticProps
  }
}
```

根据**dynamic**判断是否为动态属性名拼接至**dynamicProps**或**staticProps**

staticProps是一个对象的序列化字符串

dynamicProps是一个数组的序列化字符串

存在dynamicProps时，用**"_d"**包裹进行返回

##### genHandlers

```
src/compiler/codegen/events.js
export function genHandlers (
  events: ASTElementHandlers,
  isNative: boolean
): string {
  const prefix = isNative ? 'nativeOn:' : 'on:'
  let staticHandlers = ``
  let dynamicHandlers = ``
  for (const name in events) {
    const handlerCode = genHandler(events[name])
    if (events[name] && events[name].dynamic) {
      dynamicHandlers += `${name},${handlerCode},`
    } else {
      staticHandlers += `"${name}":${handlerCode},`
    }
  }
  staticHandlers = `{${staticHandlers.slice(0, -1)}}`
  if (dynamicHandlers) {
    return prefix + `_d(${staticHandlers},[${dynamicHandlers.slice(0, -1)}])`
  } else {
    return prefix + staticHandlers
  }
}
```

根据**dynamic**判断是否为动态事件名拼接至**dynamicHandlers**或**staticHandlers**

staticHandlers是一个对象的序列化字符串

dynamicHandlers是一个数组的序列化字符串

存在dynamicHandlers时，用**"_d"**包裹进行返回，最好根据**isNative**决定是**on**还是**nativeOn**进行前缀拼接

##### genScopedSlots

```
function genScopedSlots (
  el: ASTElement,
  slots: { [key: string]: ASTElement },
  state: CodegenState
): string {
  // by default scoped slots are considered "stable", this allows child
  // components with only scoped slots to skip forced updates from parent.
  // but in some cases we have to bail-out of this optimization
  // for example if the slot contains dynamic names, has v-if or v-for on them...
  let needsForceUpdate = el.for || Object.keys(slots).some(key => {
    const slot = slots[key]
    return (
      slot.slotTargetDynamic ||
      slot.if ||
      slot.for ||
      containsSlotChild(slot) // is passing down slot from parent which may be dynamic
    )
  })

  // #9534: if a component with scoped slots is inside a conditional branch,
  // it's possible for the same component to be reused but with different
  // compiled slot content. To avoid that, we generate a unique key based on
  // the generated code of all the slot contents.
  let needsKey = !!el.if

  // OR when it is inside another scoped slot or v-for (the reactivity may be
  // disconnected due to the intermediate scope variable)
  // #9438, #9506
  // TODO: this can be further optimized by properly analyzing in-scope bindings
  // and skip force updating ones that do not actually use scope variables.
  if (!needsForceUpdate) {
    let parent = el.parent
    while (parent) {
      if (
        (parent.slotScope && parent.slotScope !== emptySlotScopeToken) ||
        parent.for
      ) {
        needsForceUpdate = true
        break
      }
      if (parent.if) {
        needsKey = true
      }
      parent = parent.parent
    }
  }

  const generatedSlots = Object.keys(slots)
    .map(key => genScopedSlot(slots[key], state))
    .join(',')

  return `scopedSlots:_u([${generatedSlots}]${
    needsForceUpdate ? `,null,true` : ``
  }${
    !needsForceUpdate && needsKey ? `,null,false,${hash(generatedSlots)}` : ``
  })`
}
```

遍历**scopedSlots**，对每个单独的slot用**genScopedSlot**进行处理，然后用**"_u"**包裹

##### genScopedSlot

```
function genScopedSlot (
  el: ASTElement,
  state: CodegenState
): string {
  const isLegacySyntax = el.attrsMap['slot-scope']
  if (el.if && !el.ifProcessed && !isLegacySyntax) {
    return genIf(el, state, genScopedSlot, `null`)
  }
  if (el.for && !el.forProcessed) {
    return genFor(el, state, genScopedSlot)
  }
  const slotScope = el.slotScope === emptySlotScopeToken
    ? ``
    : String(el.slotScope)
  const fn = `function(${slotScope}){` +
    `return ${el.tag === 'template'
      ? el.if && isLegacySyntax
        ? `(${el.if})?${genChildren(el, state) || 'undefined'}:undefined`
        : genChildren(el, state) || 'undefined'
      : genElement(el, state)
    }}`
  // reverse proxy v-slot without scope on this.$slots
  const reverseProxy = slotScope ? `` : `,proxy:true`
  return `{key:${el.slotTarget || `"default"`},fn:${fn}${reverseProxy}}`
}
```

处理后的数据为一个对象的序列化字符串，格式为{ key: el.slotTarget, fn: fn, proxy?: true }，**proxy**标识是否有自定义作用域**slotScope**

若是**template**返回通过**genChildren**处理的数据，否则返回通过**genElement**处理的数据

```
<component><template #mslot="val"></template></component>

{
  type: 1,
  tag'component',
  scopedSlots: {
    "myslot": {
      type: 1,
      tag: 'template',
      slotScope: 'val'.
      slotTarget: '"mslot"'.
      slotTargetDynamic: false
      ...
    }
  }
  ...
}

_c('component',{scopedSlots:_u([{key:"mslot",fn:function(val){return undefined}}])})
```

##### genInlineTemplate

```
function genInlineTemplate (el: ASTElement, state: CodegenState): ?string {
  const ast = el.children[0]
  if (process.env.NODE_ENV !== 'production' && (
    el.children.length !== 1 || ast.type !== 1
  )) {
    state.warn(
      'Inline-template components must have exactly one child element.',
      { start: el.start }
    )
  }
  if (ast && ast.type === 1) {
    const inlineRenderFns = generate(ast, state.options)
    return `inlineTemplate:{render:function(){${
      inlineRenderFns.render
    }},staticRenderFns:[${
      inlineRenderFns.staticRenderFns.map(code => `function(){${code}}`).join(',')
    }]}`
  }
}
```

当有子节点且第一个子节点为标签节点，通过**generate**解析后添加**inlineTemplate**属性