```
root = vue instance
vnode = render()  createElement
root._vnode ?
	yes: oldVnode = root._vnode
	no: oldVnode = $el
root._vode = vnode
patch(oldVnode, vnode)
sameVnode(oldVnode, vnode) ?
	yes: patchVnode  cbs.update update hook prepatch hook
		children ?
			yes: sameVnode(oldChildVnode, childVnode)
			no: update dom
		postpatch hook
	no: createElm(vnode) removeVnodes([oldVnode])
root.$el = vnode.elm
```

vdom的根vnode通过`render`函数而来，vnode实例通过`createElement`生成

patch过程是生成DOM的过程，通过`createElm`为每个vnode生成对应的elm

```
vnode = createElement(context, tag, data, children, normalizationType, alwaysNormalize)  $createElement _c
tag ?
	null: a comment vnode
	html element: a html element vnode
	component: tag = Ctor, a component vnode
	function: async component vnode ?
	other: a comment vnode
```

`createElement`构造vnode，可简单分为组件vnode和元素vnode

元素vnode将会被直接生成DOM元素，组件vnode会以自身再次构造vdom

```
createElm(vnode)
create component instance ?  init hook
	yes: create component instance(vnode.elm = vm.$el)
	no: create html element(vnode.elm = document.create)
    	children ?
    		yes: createElm(child)
insert into parent (insert hook)
```

`createElm`是vdom到DOM的过程

对于组件vnode是通过实例化获取DOM元素

```
vm -> render -> vm vnode -> patch -> child vnode -> createElm -> childVm -> render -> childVm vnode -> patch -> ...
```

从根实例初始化后开始就是个递归的问题

一个Vue实例通过`render`生成自身的`根vnode`，然后通过patch过程中调用`createElm`来将vnode映射成DOM元素，若vnode为组件vnode时则会重启一次流程