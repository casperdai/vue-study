```
src/compiler/codegen/index.js
import baseDirectives from '../directives/index'

export class CodegenState {
  ...

  constructor (options: CompilerOptions) {
    ...
    this.directives = extend(extend({}, baseDirectives), options.directives)
    ...
  }
}

src/compiler/directives/index.js
export default {
  on,
  bind,
  cloak: noop
}

src/platforms/web/compiler/options.js
import directives from './directives/index'

export const baseOptions: CompilerOptions = {
  ...
  directives,
  ...
}

src/platforms/web/compiler/directives/index.js
export default {
  model,
  text,
  html
}
```

编译时主要进行了6中指令的转化

##### v-on

```
src/compiler/directives/on.js
export default function on (el: ASTElement, dir: ASTDirective) {
  if (process.env.NODE_ENV !== 'production' && dir.modifiers) {
    warn(`v-on without argument does not support modifiers.`)
  }
  el.wrapListeners = (code: string) => `_g(${code},${dir.value})`
}
```

针对v-on="listeners"的情况，v-on带事件名的情况在生成ast时已进行了转化

添加wrapListeners用于扩展生成的render函数内容

##### v-bind

```
src/compiler/directives/bind.js
export default function bind (el: ASTElement, dir: ASTDirective) {
  el.wrapData = (code: string) => {
    return `_b(${code},'${el.tag}',${dir.value},${
      dir.modifiers && dir.modifiers.prop ? 'true' : 'false'
    }${
      dir.modifiers && dir.modifiers.sync ? ',true' : ''
    })`
  }
}
```

针对v-bind="attrs"的情况，v-bind带属性名的情况在生成ast时已进行了转化

##### v-cloak

空解析，用于通过dom元素内容来进行编译的情况，原理是编译结束后将不会有v-cloak特性存在于挂载至的dom元素上

##### v-model

```
src/platforms/web/compiler/directives/model.js
export default function model (
  el: ASTElement,
  dir: ASTDirective,
  _warn: Function
): ?boolean {
  warn = _warn
  const value = dir.value
  const modifiers = dir.modifiers
  const tag = el.tag
  const type = el.attrsMap.type

  if (el.component) {
    genComponentModel(el, value, modifiers)
    // component v-model doesn't need extra runtime
    return false
  } else if (tag === 'select') {
    genSelect(el, value, modifiers)
  } else if (tag === 'input' && type === 'checkbox') {
    genCheckboxModel(el, value, modifiers)
  } else if (tag === 'input' && type === 'radio') {
    genRadioModel(el, value, modifiers)
  } else if (tag === 'input' || tag === 'textarea') {
    genDefaultModel(el, value, modifiers)
  } else if (!config.isReservedTag(tag)) {
    genComponentModel(el, value, modifiers)
    // component v-model doesn't need extra runtime
    return false
  }

  // ensure runtime directive metadata
  return true
}
```

主要进行了下列操作：

- 若是动态组件或非保留标签时
  - 添加model属性，{ value, expression, callback }
  - 不需添加进directives
- 若是select标签
  - 添加一个change事件监听
- 若是checkbox类型的input
  - props添加checked属性（若已存在会合并）
  - 添加一个change事件监听
- 若是radio类型的input
  - props添加checked属性（若已存在会合并）
  - 添加一个change事件监听
- 若是input或textarea标签
  - props添加value属性（若已存在会合并）
  - 根据修饰符添加事件监听
    - lazy，change事件
    - trim或number，blur事件

##### v-text

```
src/platforms/web/compiler/directives/text.js
export default function text (el: ASTElement, dir: ASTDirective) {
  if (dir.value) {
    addProp(el, 'textContent', `_s(${dir.value})`, dir)
  }
}
```

props添加textContent属性，会将v-text对应的值字符串化

##### v-html

```
src/platforms/web/compiler/directives/html.js
export default function html (el: ASTElement, dir: ASTDirective) {
  if (dir.value) {
    addProp(el, 'innerHTML', `_s(${dir.value})`, dir)
  }
}
```

为ASTElement实例的props添加innerHTML属性，会将v-html对应的值字符串化