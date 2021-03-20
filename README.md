### Vue Note

##### 基本

[前置](./前置.md)
[入口](./入口.md)
[初始化](./初始化.md)
[答疑](./答疑.md)

##### 响应式

[依赖收集](./依赖收集.md)
[observe](./observe.md)
[provide/inject](./provide2inject.md)
[props](./props.md)
[计算属性](./computed.md)

##### 虚拟DOM

[vnode](./patch1.md)
[基本流程](./patch2.md)
[diff](./patch3.md)

##### 源码

[属性合并](./mergeOptions.md)
[Vue.extend](./Vue.extend.md)
[slot](./slot.md)
[ref](./ref.md)
[v-model](./v-model.md)
[自定义指令](./directives.md)

##### 流程

new Vue().$mount('#app')

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