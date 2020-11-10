# vue3渲染流程2（setupRenderEffect）
![托脸](https://res.psy-1.com/FkYnyYqXEj0EDfF5IlRr5L2dz5zR)setupRenderEffect，
涉及到响应式数据的东西的，建议先去看去看Reactivity的响应式数据包再来。



setupRenderEffect中实际用到的是一个叫componentUpdate的方法，传递給effect，创建响应，赋值給instance.update。
```typescript
const prodEffectOptions = {
  scheduler: queueJob
}
instance.update = effect(componentEffect, prodEffectOptions)
```



## componentEffect

当前测试用例instance.mounted === undefinded，利用<font color=#ff8000>renderComponentRoot</font>，生成subTree，subTree是根据instance.type来生成的。



## renderComponentRoot
如果组件是ShapeFlags.STATEFUL_COMPONENT类型，调用<font color=#ff8000>instance.render(instance.withProxy || instance.proxy, instance.renderCache)</font>生成subTree，检测<font color=#ff8000>instance.type.inheritAttrs !== false</font>，inheriAttrs的初始默认值为undefided，再次检测<font color=#ff8000>subTree.shapeFlag是否属于ShapeFlags.ELEMENT类型
或者ShapeFlags.COMPONENT类型</font>，如果是则通过<font color=#ff8000>cloneVNode内部调用mergeProps把subTree.props与subTree.attrs合并</font>，是不是忘记了attrs是什么东西？在initProps中，根据当前组件vnode.props所遍历得到的，本质上是vnode.props。



如果组件是ShapeFlage.FUNCTIONAL_COMPONENT，调用instance.vnode.type生成，和ShapeFlags.STATEFUL_COMPONENT类型中的合并subTree.props与instance.attrs行为一致，
但是<font color=#ff8000>attrs = instance.type.props ? instance.attrs : getFallthroughAttrs(instance.attrs)</font>。getFallthroughAttrs在API DOC中有描述，这里主要是筛选出key为class、style和on开头的key。



```typescript
export function renderComponentRoot(
  instance: ComponentInternalInstance
): VNode {
  const {
    type: Component, // 具名匹配 传递給Component
    parent,
    vnode, // instance 也传递了vnode 可以说是一个component的核心了吧
    proxy,
    withProxy,
    props,
    slots,
    attrs,
    emit,
    renderCache
  } = instance

  let result
  currentRenderingInstance = instance // 当前正在render的组件
  if (__DEV__) {
    accessedAttrs = false
  }
  try {
    let fallthroughAttrs
    if (vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT) { // STATEFUL_COMPONENT渲染subtree
      // withProxy is a proxy with a different `has` trap only for
      // runtime-compiled render functions using `with` block.
      const proxyToUse = withProxy || proxy
      result = normalizeVNode(
        instance.render!.call(proxyToUse, proxyToUse!, renderCache)
      )
      fallthroughAttrs = attrs
    } else {
      // FUNCTINAL
      const render = Component as FunctionalComponent
      // in dev, mark attrs accessed if optional props (attrs === props)
      if (__DEV__ && attrs === props) {
        markAttrsAccessed()
      }
      result = normalizeVNode(
        render.length > 1
          ? render(
              props,
              __DEV__
                ? {
                    get attrs() {
                      markAttrsAccessed()
                      return attrs
                    },
                    slots,
                    emit
                  }
                : { attrs, slots, emit }
            )
          : render(props, null as any /* we know it doesn't need it */)
      )
      fallthroughAttrs = Component.props ? attrs : getFallthroughAttrs(attrs)
    }

    // attr merging
    // in dev mode, comments are preserved, and it's possible for a template
    // to have comments along side the root element which makes it a fragment
    let root = result // 先默认自身return的vnode为root组件
    let setRoot: ((root: VNode) => void) | undefined = undefined
    if (__DEV__) {
      ;[root, setRoot] = getChildRoot(result) // chilid1 undefinded
    }

    if ( // undefined !== false
      Component.inheritAttrs !== false && // 默认为undefinded
      fallthroughAttrs &&
      Object.keys(fallthroughAttrs).length
    ) {
      if (
        root.shapeFlag & ShapeFlags.ELEMENT ||
        root.shapeFlag & ShapeFlags.COMPONENT
      ) {
        root = cloneVNode(root, fallthroughAttrs)
      } else if (__DEV__ && !accessedAttrs && root.type !== Comment) {
        const allAttrs = Object.keys(attrs)
        const eventAttrs: string[] = []
        const extraAttrs: string[] = []
        for (let i = 0, l = allAttrs.length; i < l; i++) {
          const key = allAttrs[i]
          if (isOn(key)) {
            // remove `on`, lowercase first letter to reflect event casing accurately
            eventAttrs.push(key[2].toLowerCase() + key.slice(3))
          } else {
            extraAttrs.push(key)
          }
        }
        if (extraAttrs.length) {
          warn(
            `Extraneous non-props attributes (` +
              `${extraAttrs.join(', ')}) ` +
              `were passed to component but could not be automatically inherited ` +
              `because component renders fragment or text root nodes.`
          )
        }
        if (eventAttrs.length) {
          warn(
            `Extraneous non-emits event listeners (` +
              `${eventAttrs.join(', ')}) ` +
              `were passed to component but could not be automatically inherited ` +
              `because component renders fragment or text root nodes. ` +
              `If the listener is intended to be a component custom event listener only, ` +
              `declare it using the "emits" option.`
          )
        }
      }
    }

    const parentScopeId = parent && parent.type.__scopeId
    if (parentScopeId) {
      root = cloneVNode(root, { [parentScopeId]: '' })
    }

    if (vnode.dirs) {
      if (__DEV__ && !isElementRoot(root)) {
        warn(
          `Runtime directive used on component with non-element root node. ` +
            `The directives will not function as intended.`
        )
      }
      root.dirs = vnode.dirs
    }

    if (vnode.transition) {
      if (__DEV__ && !isElementRoot(root)) {
        warn(
          `Component inside <Transition> renders non-element root node ` +
            `that cannot be animated.`
        )
      }
      root.transition = vnode.transition
    }
    // inherit ref
    if (Component.inheritRef && vnode.ref != null) {
      root.ref = vnode.ref
    }

    if (__DEV__ && setRoot) {
      setRoot(root)
    } else {
      result = root
    }
  } catch (err) {
    handleError(err, instance, ErrorCodes.RENDER_FUNCTION)
    result = createVNode(Comment)
  }
  currentRenderingInstance = null // 完成整个render

  return result
}
```

componentUpdate，根据instance.isMounted来判断是否已经挂载，如果没有挂载则调用instance.render，
调用instance.render的过程中，如果有响应式数据，将会track到当前instance.update（effect），最终生成subTree，把subTrree.props与instance.attrs合并。

看渲染流程1中的测试用例：

```typescript
const node = root.children[0] as HTMLElement
expect(node.getAttribute('class')).toBe('c2 c0')
```
c2 和c0合并了，就是基于这一行代码：
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


接着触发使用API所创建的beforeMounted，与vnode.props.onVnodeBeforeMount（创建vnode的时候传入的）：
```typescript
invokeArrayFnc(instance.bm) // 触发使用composition API创建的beforeMount事件。

// onVnodeBeforeMount
if ((vnodeHook = props && props.onVnodeBeforeMount)) { // vnode.props.onVnodeBeforeMount
  invokeVNodeHook(vnodeHook, parent, initialVNode)
}
```



再次进行patch：

```typescript
patch(null, instance.subTree, container) // container 为 render中的第二个参数②，没有改变。
```

再来一次同样类型的patch循环，<font color=#ff8000>Child</font>，经过同样的processComponent，不同的是Child的vnode是拥有props的。

> 注意区分vnode.type.props，这里没有vnode.type.props，所以全部vnode.props都杯合并到，由renderComponentRoot创建的subTree.props



接上，继续执行到第三次patch，简称patch3，所需要patch的vnode（上面的subTree），简称vnode3，vnode3拥有从所有上级所传递的props，因为vnode3.type  ===  'div'，是真实的DOM标签，所以shapeFlags被标记成ELEMENT，执行<font color=#ff8000>processElement</font>，创建'div'的HTMLDivElement，把vnode.props赋值給当前HTMLDivElement。

触发vnode.props.onVnodeBeforeMount钩子。

<font color=#ff8000>hostInsert(el, container)</font>，container为render中的第二个参数②，把该HTMLELEMENT插入到container中。

patch完成后，类似回溯的写法，从内往外依次执行，设置vnode.el = instance.subTree（ShapeFlags.COMPONENT都会赋值），触发instance.m的钩子（在组件setup中使用composition API Mounted方法传递进的事件）



当前测试用例整个初始化渲染就完成了。

