```
src/compiler/parser/index.js
export function parse (
  template: string,
  options: CompilerOptions
): ASTElement | void {
  ...
  parseHTML(template, {
    warn,
    expectHTML: options.expectHTML,
    isUnaryTag: options.isUnaryTag,
    canBeLeftOpenTag: options.canBeLeftOpenTag,
    shouldDecodeNewlines: options.shouldDecodeNewlines,
    shouldDecodeNewlinesForHref: options.shouldDecodeNewlinesForHref,
    shouldKeepComment: options.comments,
    outputSourceRange: options.outputSourceRange,
    start (tag, attrs, unary, start, end) {
      ...
    },
    end (tag, start, end) {
      ...
    },
    chars (text: string, start: number, end: number) {
      ...
    },
    comment (text: string, start, end) {
      ...
    }
  })
  return root
}
```

parse为parseHTML提供钩子函数与参数生成ASTElement实例，然后根据维护的堆栈来进行层级处理，最终生成ast

```
export function createASTElement (
  tag: string,
  attrs: Array<ASTAttr>,
  parent: ASTElement | void
): ASTElement {
  return {
    type: 1,
    tag,
    attrsList: attrs,
    attrsMap: makeAttrsMap(attrs),
    rawAttrsMap: {},
    parent,
    children: []
  }
}
```

ast节点的type有下列值：

- 1：ASTElement实例，可以认为是标签节点会有子节点
- 2：含有声明式插值的文本（{{ val }}）
- 3：纯文本

```
start (tag, attrs, unary, start, end) {
  const ns = (currentParent && currentParent.ns) || platformGetTagNamespace(tag)

  // handle IE svg bug
  /* istanbul ignore if */
  if (isIE && ns === 'svg') {
    attrs = guardIESVGBug(attrs)
  }

  let element: ASTElement = createASTElement(tag, attrs, currentParent)
  if (ns) {
    element.ns = ns
  }

  if (isForbiddenTag(element) && !isServerRendering()) {
    element.forbidden = true
  }

  // apply pre-transforms
  for (let i = 0; i < preTransforms.length; i++) {
    element = preTransforms[i](element, options) || element
  }

  if (!inVPre) {
    processPre(element)
    if (element.pre) {
      inVPre = true
    }
  }
  if (platformIsPreTag(element.tag)) {
    inPre = true
  }
  if (inVPre) {
    processRawAttrs(element)
  } else if (!element.processed) {
    // structural directives
    processFor(element)
    processIf(element)
    processOnce(element)
  }

  if (!root) {
    root = element
  }

  if (!unary) {
    currentParent = element
    stack.push(element)
  } else {
    closeElement(element)
  }
}
```

start用于生成标签ASTElement实例然后进行数据解析并维护一个堆栈来管理层级关系

在start中构建ASTElement实例并进行扩展，主要进行了下列操作：

1. 获取当前命名空间（svg，match）
2. 创建实例，若有ns则添加ns属性
3. 判断是否为忽略处理的标签（style、js script），若是则添加forbidden属性
4. 调用模块预处理钩子
5. 判断当前是否处于非编译状态，若不是且存在v-pre特性则进入非编译状态
6. 若是pre标签则进入保留空白符状态
7. 若当前是非编译状态进行特性原始化（有特性时将所有将value通过JSON.stringify处理，无特性且pre不为truly时添加plain属性）
8. 若当前不是非编译状态且ASTElement实例未处理过特性时
   - 处理v-for特性
   - 处理v-if相关特性（v-if、v-else-if、v-else）
   - 处理v-once特性
9. 若无根节点则保存
10. 若不是一元节点则入栈并修改父节点指向，否则关闭标签

```
end (tag, start, end) {
  const element = stack[stack.length - 1]
  // pop stack
  stack.length -= 1
  currentParent = stack[stack.length - 1]
  if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
    element.end = end
  }
  closeElement(element)
}
```
end用于闭合栈顶的ASTElement实例并维护父节点指向

```
function closeElement (element) {
  trimEndingWhitespace(element)
  if (!inVPre && !element.processed) {
    element = processElement(element, options)
  }
  // tree management
  if (!stack.length && element !== root) {
    // allow root elements with v-if, v-else-if and v-else
    if (root.if && (element.elseif || element.else)) {
      if (process.env.NODE_ENV !== 'production') {
        checkRootConstraints(element)
      }
      addIfCondition(root, {
        exp: element.elseif,
        block: element
      })
    }
  }
  if (currentParent && !element.forbidden) {
    if (element.elseif || element.else) {
      processIfConditions(element, currentParent)
    } else {
      if (element.slotScope) {
        // scoped slot
        // keep it in the children list so that v-else(-if) conditions can
        // find it as the prev node.
        const name = element.slotTarget || '"default"'
        ;(currentParent.scopedSlots || (currentParent.scopedSlots = {}))[name] = element
      }
      currentParent.children.push(element)
      element.parent = currentParent
    }
  }

  // final children cleanup
  // filter out scoped slots
  element.children = element.children.filter(c => !(c: any).slotScope)
  // remove trailing whitespace node again
  trimEndingWhitespace(element)

  // check pre state
  if (element.pre) {
    inVPre = false
  }
  if (platformIsPreTag(element.tag)) {
    inPre = false
  }
  // apply post-transforms
  for (let i = 0; i < postTransforms.length; i++) {
    postTransforms[i](element, options)
  }
}

function trimEndingWhitespace (el) {
  // remove trailing whitespace node
  if (!inPre) {
    let lastNode
    while (
      (lastNode = el.children[el.children.length - 1]) &&
      lastNode.type === 3 &&
      lastNode.text === ' '
    ) {
      el.children.pop()
    }
  }
}
```

closeElement会对ASTElement进行扩展，主要进行了下列操作：

1. 非pre标签下，去除尾空格
2. 编译模式下且未处理过特性时通过processElement进行节点处理
3. 不是根节点节点且无父节点时，若有条件特性为根节点添加一条条件
4. 存在父节点且自身未被忽略时
   - 若有else或elseif属性时调用processIfConditions
   - 否则处理scope并与父节点相互关联
5. 过滤掉有scope的子节点（均已添加至scopedSlots）
6. 非pre标签下，去除尾空格（例如空格后接slot）
7. 若有pre属性，退出非编译状态
8. 若是pre标签，退出保留空白符状态（bug：若pre嵌套pre可能会出现空白符异常）
9. 调用模块处理结束钩子

```
export function processElement (
  element: ASTElement,
  options: CompilerOptions
) {
  processKey(element)

  // determine whether this is a plain element after
  // removing structural attributes
  element.plain = (
    !element.key &&
    !element.scopedSlots &&
    !element.attrsList.length
  )

  processRef(element)
  processSlotContent(element)
  processSlotOutlet(element)
  processComponent(element)
  for (let i = 0; i < transforms.length; i++) {
    element = transforms[i](element, options) || element
  }
  processAttrs(element)
  return element
}
```

处理节点主要进行了下列操作：

1. 处理key特性
2. 标记是否为简单节点
3. 处理ref特性，若有则移除ref特性并添加ref属性与refInFor属性
4. 处理slot相关特性
5. 处理slot标签
6. 处理组件相关特性（is，inline-templat）
7. 调用模块节点处理钩子
8. 处理剩余特性（动态绑定等处理）

```
chars (text: string, start: number, end: number) {
  if (!currentParent) {
    return
  }
  // IE textarea placeholder bug
  /* istanbul ignore if */
  if (isIE &&
    currentParent.tag === 'textarea' &&
    currentParent.attrsMap.placeholder === text
  ) {
    return
  }
  const children = currentParent.children
  if (inPre || text.trim()) {
    text = isTextTag(currentParent) ? text : decodeHTMLCached(text)
  } else if (!children.length) {
    // remove the whitespace-only node right after an opening tag
    text = ''
  } else if (whitespaceOption) {
    if (whitespaceOption === 'condense') {
      // in condense mode, remove the whitespace node if it contains
      // line break, otherwise condense to a single space
      text = lineBreakRE.test(text) ? '' : ' '
    } else {
      text = ' '
    }
  } else {
    text = preserveWhitespace ? ' ' : ''
  }
  if (text) {
    if (!inPre && whitespaceOption === 'condense') {
      // condense consecutive whitespaces into single space
      text = text.replace(whitespaceRE, ' ')
    }
    let res
    let child: ?ASTNode
    if (!inVPre && text !== ' ' && (res = parseText(text, delimiters))) {
      child = {
        type: 2,
        expression: res.expression,
        tokens: res.tokens,
        text
      }
    } else if (text !== ' ' || !children.length || children[children.length - 1].text !== ' ') {
      child = {
        type: 3,
        text
      }
    }
    if (child) {
      if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
        child.start = start
        child.end = end
      }
      children.push(child)
    }
  }
}
```

chars用于给栈顶ASTElement实例添加一个文本节点

文本内容处理有以下规则：

- pre标签下或内容不为空时
  - 当前父节点为script与style时直接保存
  - 其他情况均需经过decodeHTMLCached处理（`he.decode`，转义并去除HTML标签）
- trim后为空字符且栈顶ASTElement实例无子节点则直接转换位空字符
- 若whitespaceOption有配置，则根据配置进行转换
- 若preserveWhitespace为truly则解析成一个空格否则为空字符

节点添加有以下规则：

- 不能为空字符
- 编译模式才能添加有声明式插值的文本节点
- 不能有两个连续的空格

```
comment (text: string, start, end) {
  // adding anyting as a sibling to the root node is forbidden
  // comments should still be allowed, but ignored
  if (currentParent) {
    const child: ASTText = {
      type: 3,
      text,
      isComment: true
    }
    if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
      child.start = start
      child.end = end
    }
    currentParent.children.push(child)
  }
```

commet用于给栈顶的ASTElement实例添加一个注释子节点

