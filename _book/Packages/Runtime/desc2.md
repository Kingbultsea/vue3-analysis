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
componentUpdate，根据instance.isMounted来判断是否已经挂载，如果没有挂载则调用instance.render，
调用instance.render的过程中，如果有响应式数据，将会track到当前instance.update的effect，最终生成subVnode，把subVnode.props与instance.attrs合并。
看渲染流程1中的测试用例：
```typescript
const node = root.children[0] as HTMLElement
expect(node.getAttribute('class')).toBe('c2 c0')
```
c2 和c0合并了，就是基于这一行代码。
```typescript
subVnode = cloneVNode(subVnode, instance.attrs) // subVnode.props与instance.attrs的合并
```

接着
```typescript
invokeArrayFnc(instance.bm) // 触发使用composition API创建的beforeMount事件。

invokeVNodeHook(vnode.props, instance.parent, vnode) // vnode.props中的钩子
```

再次进行patch：
```typescript
patch(null, instance.subTree, container) // container 为 render中的第二个参数②
```

再来一次循环，但是这次需要patch的是Child，经过同样的循环，不同的是Child是拥有props的，所以会设置Child的instance.attrs。
然后继续执行到第三次patch，简称patch3，需要patch的vnode，简称vnode3，vnode3拥有从所有上级所传递的props，因为vnode3.type === 'div'，是操作真实的DOM，
所以shapeFlags被标记成ELEMENT，执行processElement，创建'div'的HTMLELEMENT，把vnode.props植入该HTMLELEMENT，执行vnode.props.onVnodeBeforeMount钩子，
hostInsert(el, container)，container为render中的第二个参数②，把该HTMLELEMENT插入到container中。

整个初始化渲染就完成了
