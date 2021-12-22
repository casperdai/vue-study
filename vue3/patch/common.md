##### Text

```
const processText: ProcessTextOrCommentFn = (n1, n2, container, anchor) => {
  if (n1 == null) {
    hostInsert(
      (n2.el = hostCreateText(n2.children as string)),
      container,
      anchor
    )
  } else {
    const el = (n2.el = n1.el!)
    if (n2.children !== n1.children) {
      hostSetText(el, n2.children as string)
    }
  }
}
```

文本节点通过`hostCreateText`进行节点创建，通过`hostSetText`更新

##### Comment

```
const processCommentNode: ProcessTextOrCommentFn = (
  n1,
  n2,
  container,
  anchor
) => {
  if (n1 == null) {
    hostInsert(
      (n2.el = hostCreateComment((n2.children as string) || '')),
      container,
      anchor
    )
  } else {
    // there's no support for dynamic comments
    n2.el = n1.el
  }
}
```

注释节点只有创建，更新仅传递dom元素