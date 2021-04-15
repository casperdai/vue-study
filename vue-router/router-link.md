```
src/components/link.js
export default {
  name: 'RouterLink',
  props: {
    to: {
      type: toTypes,
      required: true
    },
    tag: {
      type: String,
      default: 'a'
    },
    custom: Boolean,
    exact: Boolean,
    exactPath: Boolean,
    append: Boolean,
    replace: Boolean,
    activeClass: String,
    exactActiveClass: String,
    ariaCurrentValue: {
      type: String,
      default: 'page'
    },
    event: {
      type: eventTypes,
      default: 'click'
    }
  },
  render (h: Function) {
    const router = this.$router
    const current = this.$route
    const { location, route, href } = router.resolve(
      this.to,
      current,
      this.append
    )

    const classes = {}
    const globalActiveClass = router.options.linkActiveClass
    const globalExactActiveClass = router.options.linkExactActiveClass
    // Support global empty active class
    const activeClassFallback =
      globalActiveClass == null ? 'router-link-active' : globalActiveClass
    const exactActiveClassFallback =
      globalExactActiveClass == null
        ? 'router-link-exact-active'
        : globalExactActiveClass
    const activeClass =
      this.activeClass == null ? activeClassFallback : this.activeClass
    const exactActiveClass =
      this.exactActiveClass == null
        ? exactActiveClassFallback
        : this.exactActiveClass

    const compareTarget = route.redirectedFrom
      ? createRoute(null, normalizeLocation(route.redirectedFrom), null, router)
      : route

    classes[exactActiveClass] = isSameRoute(current, compareTarget, this.exactPath)
    classes[activeClass] = this.exact || this.exactPath
      ? classes[exactActiveClass]
      : isIncludedRoute(current, compareTarget)

    const ariaCurrentValue = classes[exactActiveClass] ? this.ariaCurrentValue : null

    const handler = e => {
      if (guardEvent(e)) {
        if (this.replace) {
          router.replace(location, noop)
        } else {
          router.push(location, noop)
        }
      }
    }

    const on = { click: guardEvent }
    if (Array.isArray(this.event)) {
      this.event.forEach(e => {
        on[e] = handler
      })
    } else {
      on[this.event] = handler
    }

    const data: any = { class: classes }

    const scopedSlot =
      !this.$scopedSlots.$hasNormal &&
      this.$scopedSlots.default &&
      this.$scopedSlots.default({
        href,
        route,
        navigate: handler,
        isActive: classes[activeClass],
        isExactActive: classes[exactActiveClass]
      })

    if (scopedSlot) {
      if (scopedSlot.length === 1) {
        return scopedSlot[0]
      } else if (scopedSlot.length > 1 || !scopedSlot.length) {
        return scopedSlot.length === 0 ? h() : h('span', {}, scopedSlot)
      }
    }

    if (this.tag === 'a') {
      data.on = on
      data.attrs = { href, 'aria-current': ariaCurrentValue }
    } else {
      // find the first <a> child and apply listener and href
      const a = findAnchor(this.$slots.default)
      if (a) {
        // in case the <a> is a static node
        a.isStatic = false
        const aData = (a.data = extend({}, a.data))
        aData.on = aData.on || {}
        // transform existing events in both objects into arrays so we can push later
        for (const event in aData.on) {
          const handler = aData.on[event]
          if (event in on) {
            aData.on[event] = Array.isArray(handler) ? handler : [handler]
          }
        }
        // append new listeners for router-link
        for (const event in on) {
          if (event in aData.on) {
            // on[event] is always a function
            aData.on[event].push(on[event])
          } else {
            aData.on[event] = handler
          }
        }

        const aAttrs = (a.data.attrs = extend({}, a.data.attrs))
        aAttrs.href = href
        aAttrs['aria-current'] = ariaCurrentValue
      } else {
        // doesn't have <a> child, apply listener to self
        data.on = on
      }
    }

    return h(this.tag, data, this.$slots.default)
  }
}
```

`render`主要进行了下列操作：

1. 根据`to`和当前route获取目标`location`、 `route`和 `href`
2. 生成`classes`
3. 根据`event`生成`on`，`event`不包含 `click`事件时将屏蔽左键点击跳转
4. 有默认具名插槽时，生成子节点数据，若有子节点数据`tag`将失效
   - 子节点数量为`1`时，直接返回唯一的节点
   - 子节点数量为`0`时，返回空节点
   - 子节点数量`>1`时，用`span`标签包裹后返回其他数据均丢失
5. `tag === 'a'`时，扩展`data`的事件处理以及`attrs`
6. `tag !== 'a'`时
   - 若存在标签为`a`的子节点，则扩展该节点的事件处理以及`attrs`
   - 否则将`on`添加至`data`
7. 返回vnode

`exactActiveClass`必须`to`与当前route一致才有效

`activeClass`在`exact`情况下与`exactActiveClass`保持一致，否则根据`isIncludedRoute`（包含关系）来判断有效性

