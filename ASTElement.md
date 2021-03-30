```
src/compiler/parser/index.js
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
src/compiler/parser/index.js
start (tag, attrs, unary, start, end) {
  ...
  
  let element: ASTElement = createASTElement(tag, attrs, currentParent)
  
  ...

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

  ...

  if (inVPre) {
    processRawAttrs(element)
  } else if (!element.processed) {
    // structural directives
    processFor(element)
    processIf(element)
    processOnce(element)
  }
  ...
}
```

