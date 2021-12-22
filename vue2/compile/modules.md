```
src/compiler/parser/index.js
export function parse (
  template: string,
  options: CompilerOptions
): ASTElement | void {
  ...
  transforms = pluckModuleFunction(options.modules, 'transformNode')
  preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')
  postTransforms = pluckModuleFunction(options.modules, 'postTransformNode')
  ...
}

src/compiler/codegen/index.js
export class CodegenState {
  ...

  constructor (options: CompilerOptions) {
    ...
    this.transforms = pluckModuleFunction(options.modules, 'transformCode')
    this.dataGenFns = pluckModuleFunction(options.modules, 'genData')
    ...
  }
}
```

编译时使用的模块可以提供5种方法

生成ast时使用节点处理钩子，预处理钩子和处理结束钩子

生成render函数内容时使用代码转换函数和属性生成器

```
src/platforms/web/compiler/options.js
import modules from './modules/index'

export const baseOptions: CompilerOptions = {
  ...
  modules,
  ...
}

src/platforms/web/compiler/modules/index.js
export default [
  klass,
  style,
  model
]
```

模块主要包括class、style和model

##### class

```
src/platforms/web/compiler/modules/class.js
export default {
  staticKeys: ['staticClass'],
  transformNode,
  genData
}
```

class模块提供了模块的静态属性关键字、节点处理钩子和属性数据生成器

```
function genData (el: ASTElement): string {
  let data = ''
  if (el.staticClass) {
    data += `staticClass:${el.staticClass},`
  }
  if (el.classBinding) {
    data += `class:${el.classBinding},`
  }
  return data
}
```

属性生成器用于在生成render函数时进行class相关数据生成

```
function transformNode (el: ASTElement, options: CompilerOptions) {
  const warn = options.warn || baseWarn
  const staticClass = getAndRemoveAttr(el, 'class')
  if (process.env.NODE_ENV !== 'production' && staticClass) {
    const res = parseText(staticClass, options.delimiters)
    if (res) {
      warn(
        `class="${staticClass}": ` +
        'Interpolation inside attributes has been removed. ' +
        'Use v-bind or the colon shorthand instead. For example, ' +
        'instead of <div class="{{ val }}">, use <div :class="val">.',
        el.rawAttrsMap['class']
      )
    }
  }
  if (staticClass) {
    el.staticClass = JSON.stringify(staticClass)
  }
  const classBinding = getBindingAttr(el, 'class', false /* getStatic */)
  if (classBinding) {
    el.classBinding = classBinding
  }
}
```

节点处理钩子主要进行了下列操作：

1. 获取class特性，若存在则为实例添加staticClass属性，值为class特性的值的序列化
2. 获取动态class特性，若存在则为实例添加classBinding属性，值为动态class特性的值

##### style

```
src/platforms/web/compiler/modules/style.js
export default {
  staticKeys: ['staticStyle'],
  transformNode,
  genData
}
```

style模块提供了模块的静态属性关键字、节点处理钩子和属性数据生成器

```
function genData (el: ASTElement): string {
  let data = ''
  if (el.staticStyle) {
    data += `staticStyle:${el.staticStyle},`
  }
  if (el.styleBinding) {
    data += `style:(${el.styleBinding}),`
  }
  return data
}
```

属性生成器用于在生成render函数时进行style相关数据生成

```
function transformNode (el: ASTElement, options: CompilerOptions) {
  const warn = options.warn || baseWarn
  const staticStyle = getAndRemoveAttr(el, 'style')
  if (staticStyle) {
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production') {
      const res = parseText(staticStyle, options.delimiters)
      if (res) {
        warn(
          `style="${staticStyle}": ` +
          'Interpolation inside attributes has been removed. ' +
          'Use v-bind or the colon shorthand instead. For example, ' +
          'instead of <div style="{{ val }}">, use <div :style="val">.',
          el.rawAttrsMap['style']
        )
      }
    }
    el.staticStyle = JSON.stringify(parseStyleText(staticStyle))
  }

  const styleBinding = getBindingAttr(el, 'style', false /* getStatic */)
  if (styleBinding) {
    el.styleBinding = styleBinding
  }
}
```

节点处理钩子主要进行了下列操作：

1. 获取style特性，若存在则为实例添加staticStyle属性，值为style特性的值转化成JSON格式后再序列化
2. 获取动态style特性，若存在则为实例添加styleBinding属性，值为动态style特性的值

##### model

```
src/platforms/web/compiler/modules/model.js
export default {
  preTransformNode
}
```

model模块提供了预处理钩子

```
function preTransformNode (el: ASTElement, options: CompilerOptions) {
  if (el.tag === 'input') {
    const map = el.attrsMap
    if (!map['v-model']) {
      return
    }

    let typeBinding
    if (map[':type'] || map['v-bind:type']) {
      typeBinding = getBindingAttr(el, 'type')
    }
    if (!map.type && !typeBinding && map['v-bind']) {
      typeBinding = `(${map['v-bind']}).type`
    }

    if (typeBinding) {
      const ifCondition = getAndRemoveAttr(el, 'v-if', true)
      const ifConditionExtra = ifCondition ? `&&(${ifCondition})` : ``
      const hasElse = getAndRemoveAttr(el, 'v-else', true) != null
      const elseIfCondition = getAndRemoveAttr(el, 'v-else-if', true)
      // 1. checkbox
      const branch0 = cloneASTElement(el)
      // process for on the main node
      processFor(branch0)
      addRawAttr(branch0, 'type', 'checkbox')
      processElement(branch0, options)
      branch0.processed = true // prevent it from double-processed
      branch0.if = `(${typeBinding})==='checkbox'` + ifConditionExtra
      addIfCondition(branch0, {
        exp: branch0.if,
        block: branch0
      })
      // 2. add radio else-if condition
      const branch1 = cloneASTElement(el)
      getAndRemoveAttr(branch1, 'v-for', true)
      addRawAttr(branch1, 'type', 'radio')
      processElement(branch1, options)
      addIfCondition(branch0, {
        exp: `(${typeBinding})==='radio'` + ifConditionExtra,
        block: branch1
      })
      // 3. other
      const branch2 = cloneASTElement(el)
      getAndRemoveAttr(branch2, 'v-for', true)
      addRawAttr(branch2, ':type', typeBinding)
      processElement(branch2, options)
      addIfCondition(branch0, {
        exp: ifCondition,
        block: branch2
      })

      if (hasElse) {
        branch0.else = true
      } else if (elseIfCondition) {
        branch0.elseif = elseIfCondition
      }

      return branch0
    }
  }
}

function cloneASTElement (el) {
  return createASTElement(el.tag, el.attrsList.slice(), el.parent)
}
```

预处理钩子仅对input标签进行处理，当有v-model特性且可能有动态type特性时，主要进行了下列操作：

1. 若不存在v-model特性，直接跳出
2. 若无动态type特性，直接跳出
3. 获取v-if、v-else、v-else-if特性
4. 克隆出实例A
   - 处理v-for特性
   - 新增特性type，值为‘checkbox’
   - 处理节点并将已处理标识processed设为true
   - 添加if条件
5. 克隆出实例B
   - 处理v-for特性
   - 新增特性type，值为‘radio’
   - 处理节点
   - 添加if条件
6. 克隆出实例C
   - 处理v-for特性
   - 新增特性:type，值为获取的动态type特性
   - 处理节点
   - 添加if条件
7. 若有v-else活v-else-if特性时，A添加对应属性
8. 返回实例A