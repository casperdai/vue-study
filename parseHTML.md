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
    end: function end (tag, start, end$1) {
      ...
    },
    chars: function chars (text, start, end) {
      ...
    },
    comment: function comment (text, start, end) {
      ...
    }
  })
  return root
}
```

parse主要同parseHTML进行模板解析

```
src/compiler/parser/html-parser.js
export function parseHTML (html, options) {
  const stack = []
  const expectHTML = options.expectHTML
  const isUnaryTag = options.isUnaryTag || no
  const canBeLeftOpenTag = options.canBeLeftOpenTag || no
  let index = 0
  let last, lastTag
  while (html) {
    while (html) {
    last = html
    // Make sure we're not in a plaintext content element like script/style
    if (!lastTag || !isPlainTextElement(lastTag)) {
      ...
    } else {
      let endTagLength = 0
      const stackedTag = lastTag.toLowerCase()
      const reStackedTag = reCache[stackedTag] || (reCache[stackedTag] = new RegExp('([\\s\\S]*?)(</' + stackedTag + '[^>]*>)', 'i'))
      const rest = html.replace(reStackedTag, function (all, text, endTag) {
        endTagLength = endTag.length
        if (!isPlainTextElement(stackedTag) && stackedTag !== 'noscript') {
          text = text
            .replace(/<!\--([\s\S]*?)-->/g, '$1') // #7298
            .replace(/<!\[CDATA\[([\s\S]*?)]]>/g, '$1')
        }
        if (shouldIgnoreFirstNewline(stackedTag, text)) {
          text = text.slice(1)
        }
        if (options.chars) {
          options.chars(text)
        }
        return ''
      })
      index += html.length - rest.length
      html = rest
      parseEndTag(stackedTag, index - endTagLength, index)
    }

    if (html === last) {
      options.chars && options.chars(html)
      break
    }
  }

  // Clean up any remaining tags
  parseEndTag()

  function advance (n) {
    index += n
    html = html.substring(n)
  }

  function parseStartTag () {
    ...
  }

  function handleStartTag (match) {
    ...
  }

  function parseEndTag (tagName, start, end) {
    ...
  }
}
```

parseHTML主要通过一个while循环来解析模板

while循环主要进行下列操作：

1. 若无未闭合标签或未闭合标签不是特殊标签时查找标签起始符进行标签解析
2. 否则将标签内容进行chars处理
3. 若一轮循环未发生有效解析则剩余未解析内容进行chars处理

```
let textEnd = html.indexOf('<')
if (textEnd === 0) {
  // Comment:
  if (comment.test(html)) {
    const commentEnd = html.indexOf('-->')

    if (commentEnd >= 0) {
      if (options.shouldKeepComment) {
        options.comment(html.substring(4, commentEnd), index, index + commentEnd + 3)
      }
      advance(commentEnd + 3)
      continue
    }
  }

  // http://en.wikipedia.org/wiki/Conditional_comment#Downlevel-revealed_conditional_comment
  if (conditionalComment.test(html)) {
    const conditionalEnd = html.indexOf(']>')

    if (conditionalEnd >= 0) {
      advance(conditionalEnd + 2)
      continue
    }
  }

  // Doctype:
  const doctypeMatch = html.match(doctype)
  if (doctypeMatch) {
    advance(doctypeMatch[0].length)
    continue
  }

  // End tag:
  const endTagMatch = html.match(endTag)
  if (endTagMatch) {
    const curIndex = index
    advance(endTagMatch[0].length)
    parseEndTag(endTagMatch[1], curIndex, index)
    continue
  }

  // Start tag:
  const startTagMatch = parseStartTag()
  if (startTagMatch) {
    handleStartTag(startTagMatch)
    if (shouldIgnoreFirstNewline(startTagMatch.tagName, html)) {
      advance(1)
    }
    continue
  }
}

let text, rest, next
if (textEnd >= 0) {
  rest = html.slice(textEnd)
  while (
    !endTag.test(rest) &&
    !startTagOpen.test(rest) &&
    !comment.test(rest) &&
    !conditionalComment.test(rest)
  ) {
    // < in plain text, be forgiving and treat it as text
    next = rest.indexOf('<', 1)
    if (next < 0) break
    textEnd += next
    rest = html.slice(textEnd)
  }
  text = html.substring(0, textEnd)
}

if (textEnd < 0) {
  text = html
}

if (text) {
  advance(text.length)
}

if (options.chars && text) {
  options.chars(text, index - text.length, index)
}
```

解析标签主要进行下列操作：

1. 查询<符号
2. <符号在第一个字符时
   1. 若为注释且有结束符号，进行comment处理然后跳至结束符号后一位进入下次循环
   2. 若为条件注释且有结束符号，跳至结束符号后一位进入下次循环
   3. 若为doctype，跳至doctype后一位进入下次循环
   4. 若parseStartTag能检测到有效标签，进行标签解析handleStartTag后进入下次循环
3. 存在<符号时，将第一个有效<符号之前的内容作为文本内容
4. 不存在<符号时，剩余模板均作为文本内容
5. 若存在文本内容则进行chars处理

```
const attribute = /^\s*([^\s"'<>\/=]+)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const dynamicArgAttribute = /^\s*((?:v-[\w-]+:|@|:|#)\[[^=]+\][^\s"'<>\/=]*)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const ncname = `[a-zA-Z_][\\-\\.0-9_a-zA-Z${unicodeRegExp.source}]*`
const qnameCapture = `((?:${ncname}\\:)?${ncname})`
const startTagOpen = new RegExp(`^<${qnameCapture}`)
const startTagClose = /^\s*(\/?)>/

function parseStartTag () {
  const start = html.match(startTagOpen)
  if (start) {
    const match = {
      tagName: start[1],
      attrs: [],
      start: index
    }
    advance(start[0].length)
    let end, attr
    while (!(end = html.match(startTagClose)) && (attr = html.match(dynamicArgAttribute) || html.match(attribute))) {
      attr.start = index
      advance(attr[0].length)
      attr.end = index
      match.attrs.push(attr)
    }
    if (end) {
      match.unarySlash = end[1]
      advance(end[0].length)
      match.end = index
      return match
    }
  }
}
```

parseStartTag用于获取标签描述，主要进行了下列操作：

1. 检测是否为合法的起始符，若是则获取标签名否则直接跳出
2. 获取动态特性与静态特性
3. 若存在结束符则是有效标签进行返回，否则直接跳出

```
function handleStartTag (match) {
  const tagName = match.tagName
  const unarySlash = match.unarySlash

  if (expectHTML) {
    if (lastTag === 'p' && isNonPhrasingTag(tagName)) {
      parseEndTag(lastTag)
    }
    if (canBeLeftOpenTag(tagName) && lastTag === tagName) {
      parseEndTag(tagName)
    }
  }

  const unary = isUnaryTag(tagName) || !!unarySlash

  const l = match.attrs.length
  const attrs = new Array(l)
  for (let i = 0; i < l; i++) {
    const args = match.attrs[i]
    const value = args[3] || args[4] || args[5] || ''
    const shouldDecodeNewlines = tagName === 'a' && args[1] === 'href'
      ? options.shouldDecodeNewlinesForHref
      : options.shouldDecodeNewlines
    attrs[i] = {
      name: args[1],
      value: decodeAttr(value, shouldDecodeNewlines)
    }
    if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
      attrs[i].start = args.start + args[0].match(/^\s*/).length
      attrs[i].end = args.end
    }
  }

  if (!unary) {
    stack.push({ tag: tagName, lowerCasedTag: tagName.toLowerCase(), attrs: attrs, start: match.start, end: match.end })
    lastTag = tagName
  }

  if (options.start) {
    options.start(tagName, attrs, unary, match.start, match.end)
  }
}
```

handleStartTag用于根据标签描述来预处理数据，主要进行了下列操作：

1. 若有未闭合标签，根据一定规则（是否能嵌套等）进行闭合
2. 预处理attrs
3. 若不是一元标签（不能有子节点）时，入栈作为当前正在处理的未闭合标签
4. 进行start处理

```
function parseEndTag (tagName, start, end) {
  let pos, lowerCasedTagName
  if (start == null) start = index
  if (end == null) end = index

  // Find the closest opened tag of the same type
  if (tagName) {
    lowerCasedTagName = tagName.toLowerCase()
    for (pos = stack.length - 1; pos >= 0; pos--) {
      if (stack[pos].lowerCasedTag === lowerCasedTagName) {
        break
      }
    }
  } else {
    // If no tag name is provided, clean shop
    pos = 0
  }

  if (pos >= 0) {
    // Close all the open elements, up the stack
    for (let i = stack.length - 1; i >= pos; i--) {
      if (process.env.NODE_ENV !== 'production' &&
        (i > pos || !tagName) &&
        options.warn
      ) {
        options.warn(
          `tag <${stack[i].tag}> has no matching end tag.`,
          { start: stack[i].start, end: stack[i].end }
        )
      }
      if (options.end) {
        options.end(stack[i].tag, start, end)
      }
    }

    // Remove the open elements from the stack
    stack.length = pos
    lastTag = pos && stack[pos - 1].tag
  } else if (lowerCasedTagName === 'br') {
    if (options.start) {
      options.start(tagName, [], true, start, end)
    }
  } else if (lowerCasedTagName === 'p') {
    if (options.start) {
      options.start(tagName, [], false, start, end)
    }
    if (options.end) {
      options.end(tagName, start, end)
    }
  }
}
```

parseEndTag用于闭合标签，主要进行了下列操作：

1. 根据tagName在堆栈中查询对应未闭合标签的位置
2. 若有对应未闭合标签时，从栈顶至自身依次进行end处理闭合标签
3. 若无对应未闭合标签时，处理一些特殊tagName

