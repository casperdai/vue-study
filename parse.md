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

parse根据parseHTML提供的标签描述生成ASTElement，然后根据维护的堆栈来进行层级处理，最终生成ast

```
start (tag, attrs, unary, start, end) {
  ...

  let element: ASTElement = createASTElement(tag, attrs, currentParent)
  
  ...

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

start用于生成标签ASTElement并维护一个堆栈来管理层级关系

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

end用于闭合栈顶的ASTElement实例

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

- 非编译模式下或内容不为空时
  - 当前父节点为script与style时直接保存
  - 其他情况均需经过decodeHTMLCached处理（`he.decode`，转义并去除HTML标签）
- trim后为空字符且栈顶ASTElement实例无子节点则直接转换位空字符
- 其他情况若preserveWhitespace为truly则解析成一个空格否则为空字符

节点添加有以下规则：

- 不能为空字符
- 非编译模式才能添加有声明式插值的文本节点
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

