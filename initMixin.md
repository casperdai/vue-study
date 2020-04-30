##### 第一步

解析合并options，将解析后的值挂载至$options

分内部组件和非内部组件，通过_isComponent判断

##### 第二步

```
initLifecycle(vm)
initEvents(vm)
initRender(vm)
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
callHook(vm, 'created')
```

beforeCreate时并未有属性的挂载等操作，初始化了事件可以调用事件相关接口

created时已进行了数据挂载并获取到了inject，已可以进行非dom相关的操作

**重点为initState**

##### 第三部

若有el则进行$mount操作