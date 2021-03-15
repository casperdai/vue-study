### Vue Note

源码阅读

[前置](./前置.md)
[命名转换](./命名转换.md)
[入口](./入口.md)
[属性合并](./mergeOptions.md)
[初始化](./初始化.md)
[Vue.extend](./Vue.extend.md)

响应式

[数据动态响应](./数据动态响应.md)

虚拟DOM

[生成vnode](./patch1.md)
[patch](./patch2.md)
[生成dom](./vdom2dom.md)

##### 流程

new Vue({}).$mount('#app')

```
src/core/instance/lifecycle.js
src/core/instance/render.js

src/core/vdom/vnode.js
src/core/vdom/patch.js

updateComponent = () => {
  vm._update(vm._render(), hydrating)
}

// we set this to vm._watcher inside the watcher's constructor
// since the watcher's initial patch may call $forceUpdate (e.g. inside child
// component's mounted hook), which relies on vm._watcher being already defined
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

lifecycle（mountComponent） => render（\_render） =>  lifecycle（\_update） => patch

$mount只是创建了一个Watcher