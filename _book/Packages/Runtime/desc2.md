## vue3渲染流程2（setupRenderEffect）
![托脸](https://res.psy-1.com/FkYnyYqXEj0EDfF5IlRr5L2dz5zR)setupRenderEffect，
涉及到响应式数据的东西的，建议先去看去看Reactivity的响应式数据包再来看。

setupRenderEffect中实际用到的是一个叫componentUpdate的方法，并使用effect添加响应式，最后赋值給instance.update。
```typescript
const prodEffectOptions = {
  scheduler: queueJob
}
instance.update = effect(componentEffect, prodEffectOptions)
```

### renderComponentRoot
如果组件是ShapeFlags.STATEFUL_COMPONENT，调用instance.render(instance.withProxy || instance.proxy, instance.renderCache)生成subTree，检测instance.type.inheritAttrs是否不为false，inheriAttrs的初始默认值为undefided，再次检测subTree.shapeFlag是否属于ShapeFlags.ELEMENT类型
或者ShapeFlags.COMPONENT类型，如果是则通过cloneVNode内部调用mergeProps把subTree.props与subTree.attrs合并，是不是忘记了attrs是什么东西？在initProps中，根据当前组件vnode.props所遍历得到的，本质上是vnode.props。

如果组件是ShapeFlage.FUNCTIONAL_COMPONENT，调用instance.vnode.type生成，和ShapeFlags.STATEFUL_COMPONENT的合并subTree.props行为一致，
但是attrs = instance.type.props ? instance.attrs : getFallthroughAttrs(instance.attrs)。getFallthroughAttrs在API DOC中有描述，这里主要是筛选出key为class、style和on开头的key。attrs是用来与subTree.props合并的。

componentUpdate，根据instance.isMounted来判断是否已经挂载，如果没有挂载则调用instance.render，
调用instance.render的过程中，如果有响应式数据，将会track到当前instance.update的effect，最终生成subTree，把subTrree.props与instance.attrs合并。
看渲染流程1中的测试用例：
```typescript
const node = root.children[0] as HTMLElement
expect(node.getAttribute('class')).toBe('c2 c0')
```
c2 和c0合并了，就是基于这一行代码。
```typescript
subTree = cloneVNode(subTree, instance.attrs) // subTree.props与instance.attrs的合并
```

如果有父parent.type.parentScopeId，则拓展到subTree.props
```typescript
const parentScopeId = parent && parent.type.__scopeId
    if (parentScopeId) {
      root = cloneVNode(root, { [parentScopeId]: '' })
    }
```


接着触发API中的beforeMounted，与vnode.
```typescript
invokeArrayFnc(instance.bm) // 触发使用composition API创建的beforeMount事件。

// onVnodeBeforeMount
if ((vnodeHook = props && props.onVnodeBeforeMount)) { // vnode.props.onVnodeBeforeMount
  invokeVNodeHook(vnodeHook, parent, initialVNode)
}
```

再次进行patch：
```typescript
patch(null, instance.subTree, container) // container 为 render中的第二个参数②
```

再来一次patch循环，但是这次需要patch的是Child，经过同样的循环，不同的是Child的vnode是拥有props的（注意区分vnode.type.props，这里没有vnode.type.props，所以全部vnode.props都会设置到
subTree.props）。


然后继续执行到第三次patch，简称patch3，需要patch的vnode，简称vnode3，vnode3拥有从所有上级所传递的props，因为vnode3.type === 'div'，是操作真实的DOM，
所以shapeFlags被标记成ELEMENT，执行processElement，创建'div'的HTMLELEMENT，把vnode.props植入该HTMLELEMENT，执行vnode.props.onVnodeBeforeMount钩子。

hostInsert(el, container)，container为render中的第二个参数②，把该HTMLELEMENT插入到container中。

执行所有的patch，就是类似回溯的写法，从内往外，设置vnode.el = instance.subTree（ShapeFlags.COMPONENT都会赋值），触发instance.m的钩子，就是使用composition API Mounted方法传递进的事件。

整个初始化渲染就完成了。

现在需要说的是更新的过程。

