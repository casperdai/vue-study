ASTElement实例化后会对attrs进行处理

src/compiler/parser/index.js

##### processPre

```
function processPre (el) {
  if (getAndRemoveAttr(el, 'v-pre') != null) {
    el.pre = true
  }
}
```

检测v-pre指令，若使用了添加pre属性（值为true）进行标识表示将进入非编译模式

```
function processRawAttrs (el) {
  const list = el.attrsList
  const len = list.length
  if (len) {
    const attrs: Array<ASTAttr> = el.attrs = new Array(len)
    for (let i = 0; i < len; i++) {
      attrs[i] = {
        name: list[i].name,
        value: JSON.stringify(list[i].value)
      }
      if (list[i].start != null) {
        attrs[i].start = list[i].start
        attrs[i].end = list[i].end
      }
    }
  } else if (!el.pre) {
    // non root node in pre blocks with no attributes
    el.plain = true
  }
}
```

若在v-pre范围内时，所有特性将使用字面量，若无特性则添加plain属性（值为true）进行标识

##### processFor

```
export function processFor (el: ASTElement) {
  let exp
  if ((exp = getAndRemoveAttr(el, 'v-for'))) {
    const res = parseFor(exp)
    if (res) {
      extend(el, res)
    } else if (process.env.NODE_ENV !== 'production') {
      warn(
        `Invalid v-for expression: ${exp}`,
        el.rawAttrsMap['v-for']
      )
    }
  }
}
```

检测v-for指令

```
export const forAliasRE = /([\s\S]*?)\s+(?:in|of)\s+([\s\S]*)/
export const forIteratorRE = /,([^,\}\]]*)(?:,([^,\}\]]*))?$/
const stripParensRE = /^\(|\)$/g

export function parseFor (exp: string): ?ForParseResult {
  const inMatch = exp.match(forAliasRE)
  if (!inMatch) return
  const res = {}
  res.for = inMatch[2].trim()
  const alias = inMatch[1].trim().replace(stripParensRE, '')
  const iteratorMatch = alias.match(forIteratorRE)
  if (iteratorMatch) {
    res.alias = alias.replace(forIteratorRE, '').trim()
    res.iterator1 = iteratorMatch[1].trim()
    if (iteratorMatch[2]) {
      res.iterator2 = iteratorMatch[2].trim()
    }
  } else {
    res.alias = alias
  }
  return res
}
```

若有则解析v-for指令，主要进行了下列操作：

1. 检测表达式是否正确，若不正确直接跳出
2. 添加for属性（进行循环的对象的表达式）
3. 获取变量名
4. 添加alias属性（循环过程中的值对应的变量名）
5. 若有多个变量名则添加iterator1和iterator2属性进行存储

##### processIf

```
function processIf (el) {
  const exp = getAndRemoveAttr(el, 'v-if')
  if (exp) {
    el.if = exp
    addIfCondition(el, {
      exp: exp,
      block: el
    })
  } else {
    if (getAndRemoveAttr(el, 'v-else') != null) {
      el.else = true
    }
    const elseif = getAndRemoveAttr(el, 'v-else-if')
    if (elseif) {
      el.elseif = elseif
    }
  }
}

export function addIfCondition (el: ASTElement, condition: ASTIfCondition) {
  if (!el.ifConditions) {
    el.ifConditions = []
  }
  el.ifConditions.push(condition)
}
```

检测v-if、v-else、v-else-if指令

- 若有v-if特性
  - 添加if属性（值为特性对应的值）
  - 添加一条条件进入ifConditions数组，值格式为{ exp: string, block: ASTElement }，分别记录条件表达式与对应的ASTElement 实例
- 否则
  - 若有v-else，添加else属性（值为true）
  - 若有v-else-if，添加elseif属性（值为特性对应的值）

根据处理可知，v-if与v-else或v-else-if同时存在时，仅v-if会在编译时处理其余将作为自定义指令待运行时解析

##### processIfConditions

```
function processIfConditions (el, parent) {
  const prev = findPrevElement(parent.children)
  if (prev && prev.if) {
    addIfCondition(prev, {
      exp: el.elseif,
      block: el
    })
  }
}

function findPrevElement (children: Array<any>): ASTElement | void {
  let i = children.length
  while (i--) {
    if (children[i].type === 1) {
      return children[i]
    } else {
      children.pop()
    }
  }
}
```

当ASTElement实例有else或elseif属性时，为前一个兄弟节点添加一个条件

根据处理可知：

- v-else和v-else-if同时存在时，v-else将无效
- v-else或v-else-if存在时，若前兄弟节点为文本节点时将被移除，若前兄弟节点不存在if属性将自身丢弃

##### processOnce

```
function processOnce (el) {
  const once = getAndRemoveAttr(el, 'v-once')
  if (once != null) {
    el.once = true
  }
}
```

检测v-once指令，若有则添加once属性（值为true）

##### processKey

```
function processKey (el) {
  const exp = getBindingAttr(el, 'key')
  if (exp) {
    el.key = exp
  }
}
```

检测动态key特性，若有则添加key属性（值为特性对应的值）

##### processRef

```
function processRef (el) {
  const ref = getBindingAttr(el, 'ref')
  if (ref) {
    el.ref = ref
    el.refInFor = checkInFor(el)
  }
}
```

检测动态ref特性，若有则添加ref属性（值为特性对应的值）和refInFor属性（值为true或false，标识是否在v-for指令范围内）

##### processSlotContent（2.6 v-slot syntax）

```
const slotRE = /^v-slot(:|$)|^#/

function processSlotContent (el) {
  f (el.tag === 'template') {
    // v-slot on <template>
    const slotBinding = getAndRemoveAttrByRegex(el, slotRE)
    if (slotBinding) {
      const { name, dynamic } = getSlotName(slotBinding)
      el.slotTarget = name
      el.slotTargetDynamic = dynamic
      el.slotScope = slotBinding.value || emptySlotScopeToken // force it into a scoped slot for perf
    }
  } else {
    // v-slot on component, denotes default slot
    const slotBinding = getAndRemoveAttrByRegex(el, slotRE)
    if (slotBinding) {
      // add the component's children to its default slot
      const slots = el.scopedSlots || (el.scopedSlots = {})
      const { name, dynamic } = getSlotName(slotBinding)
      const slotContainer = slots[name] = createASTElement('template', [], el)
      slotContainer.slotTarget = name
      slotContainer.slotTargetDynamic = dynamic
      slotContainer.children = el.children.filter((c: any) => {
        if (!c.slotScope) {
          c.parent = slotContainer
          return true
        }
      })
      slotContainer.slotScope = slotBinding.value || emptySlotScopeToken
      // remove children as they are returned from scopedSlots now
      el.children = []
      // mark el non-plain so data gets generated
      el.plain = false
    }
  }
}
```

检测v-slot指令，若存在主要进行了下列操作：

- 若为template标签时
  - 添加slotTarget属性（插槽名），slotTargetDynamic属性（值为true或false，标识是否为动态插槽名），slotScope属性（插槽作用域表达式）
- 否为组件时
  - 获取插槽描述
  - 新建ASTElement实例A并以插槽名为关键字加入scopedSlots
  - 实例A添加slotTarget属性（插槽名），slotTargetDynamic属性（值为true或false，标识是否为动态插槽名），slotScope属性（插槽作用域表达式）并将当前实例中不是默认具名插槽的子节点移至实例A
  - 清空当前实例的子节点

根据处理可知，组件不能作为组件的具名插槽

##### processSlotOutlet

```
function processSlotOutlet (el) {
  if (el.tag === 'slot') {
    el.slotName = getBindingAttr(el, 'name')
  }
}
```

检测是否为slot标签，若是则添加slotName属性（值为动态name特性的值）

##### processComponent

```
function processComponent (el) {
  let binding
  if ((binding = getBindingAttr(el, 'is'))) {
    el.component = binding
  }
  if (getAndRemoveAttr(el, 'inline-template') != null) {
    el.inlineTemplate = true
  }
}
```

检测动态is特性，若有则添加component属性（值为特性对应的值）标识组件tag的表达式

检测inline-template特性，若有则添加inlineTemplate属性（值为true）

##### processAttrs

```
const onRE = /^@|^v-on:/
const dirRE = /^v-|^@|^:|^#/
const argRE = /:(.*)$/
const bindRE = /^:|^\.|^v-bind:/
const propBindRE = /^\./
const modifierRE = /\.[^.\]]+(?=[^\]]*$)/g

function processAttrs (el) {
  const list = el.attrsList
  let i, l, name, rawName, value, modifiers, syncGen, isDynamic
  for (i = 0, l = list.length; i < l; i++) {
    name = rawName = list[i].name
    value = list[i].value
    if (dirRE.test(name)) {
      // mark element as dynamic
      el.hasBindings = true
      // modifiers
      modifiers = parseModifiers(name.replace(dirRE, ''))
      // support .foo shorthand syntax for the .prop modifier
      if (process.env.VBIND_PROP_SHORTHAND && propBindRE.test(name)) {
        (modifiers || (modifiers = {})).prop = true
        name = `.` + name.slice(1).replace(modifierRE, '')
      } else if (modifiers) {
        name = name.replace(modifierRE, '')
      }
      if (bindRE.test(name)) { // v-bind
        name = name.replace(bindRE, '')
        value = parseFilters(value)
        isDynamic = dynamicArgRE.test(name)
        if (isDynamic) {
          name = name.slice(1, -1)
        }
        if (modifiers) {
          if (modifiers.prop && !isDynamic) {
            name = camelize(name)
            if (name === 'innerHtml') name = 'innerHTML'
          }
          if (modifiers.camel && !isDynamic) {
            name = camelize(name)
          }
          if (modifiers.sync) {
            syncGen = genAssignmentCode(value, `$event`)
            if (!isDynamic) {
              addHandler(
                el,
                `update:${camelize(name)}`,
                syncGen,
                null,
                false,
                warn,
                list[i]
              )
              if (hyphenate(name) !== camelize(name)) {
                addHandler(
                  el,
                  `update:${hyphenate(name)}`,
                  syncGen,
                  null,
                  false,
                  warn,
                  list[i]
                )
              }
            } else {
              // handler w/ dynamic event name
              addHandler(
                el,
                `"update:"+(${name})`,
                syncGen,
                null,
                false,
                warn,
                list[i],
                true // dynamic
              )
            }
          }
        }
        if ((modifiers && modifiers.prop) || (
          !el.component && platformMustUseProp(el.tag, el.attrsMap.type, name)
        )) {
          addProp(el, name, value, list[i], isDynamic)
        } else {
          addAttr(el, name, value, list[i], isDynamic)
        }
      } else if (onRE.test(name)) { // v-on
        name = name.replace(onRE, '')
        isDynamic = dynamicArgRE.test(name)
        if (isDynamic) {
          name = name.slice(1, -1)
        }
        addHandler(el, name, value, modifiers, false, warn, list[i], isDynamic)
      } else { // normal directives
        name = name.replace(dirRE, '')
        // parse arg
        const argMatch = name.match(argRE)
        let arg = argMatch && argMatch[1]
        isDynamic = false
        if (arg) {
          name = name.slice(0, -(arg.length + 1))
          if (dynamicArgRE.test(arg)) {
            arg = arg.slice(1, -1)
            isDynamic = true
          }
        }
        addDirective(el, name, rawName, value, arg, isDynamic, modifiers, list[i])
      }
    } else {
      addAttr(el, name, JSON.stringify(value), list[i])
      // #6887 firefox doesn't update muted state if set via attribute
      // even immediately after element creation
      if (!el.component &&
          name === 'muted' &&
          platformMustUseProp(el.tag, el.attrsMap.type, name)) {
        addProp(el, name, 'true', list[i])
      }
    }
  }
}
```

处理特性

- 若是指令或指令缩写
  - 添加hasBindings属性（值为true，标识有绑定值）
  - 获取modifiers
  - 获取去除修饰符后的特性名
  - 解析特性名
    - 若为属性绑定，根据修饰符处理后通过addProp或addAttr进行添加
    - 若为事件，通过addHandler进行添加
    - 其他，通过addDirective进行添加
- 否则通过addAttr添加一个值为字面量值的属性