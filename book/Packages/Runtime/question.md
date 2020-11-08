# 一些问题
#### 1.当连续触发响应式的trigger会导致组件的effectComponent连续执行更新吗？
并不会。第一次触发trigger的时候会把事件队列状态改成Pending，把flushJobs丢进microtask来执行，第二次触发trigger会检测到当前状态如果是Pending或者是Flushing状态的时候将不进行下去。
可在packages/runtime-core/\_\_tests\_\_/rendererAttrsFallthrough.spec.ts 'should allow attrs to fallthrough'测试用例中调试。

![流程图](https://res.psy-1.com/FuStWNAasmJ0eOOjTm_nC_7ON0uz)

#### 2.patch到底有多少种类型？分别是干什么的？
在patch中查看有processText、processCommentNode、mountStaticNode、patchStaticNode、processFragment、processElement（流程图已有）、processComponent（流程图已有）、Teleport、SUSPENSE。


#### 3.instance.props、instance.attrs和vnode.props分别是什么？关系是什么？
 先说一下设置的阶段，和如何设置的：
 1.instance.props和instance.attrs是在第二步骤中setupComponent中的initProps实现的，是根据vnode.props来设置的，instance和组件类型相关。
 2.vnode.props创建vnode的时候传入的参数，比如```h(Child, { foo: 1, class: 'parent' })```。
 
使用阶段，拿这个做例子:
```typescript
const Hello = {
  setup() {
    const count = ref(0)

    function inc() {
      count.value++
    }

    return () =>
      h(Child, {
        foo: count.value + 1,
        id: 'test',
        class: 'c' + count.value,
        style: { color: count.value ? 'red' : 'green' },
        onClick: inc,
        'data-id': count.value + 1
      })
  }
}

const Child = {
  setup(props: any) {
    return () =>
      h(
        'div',
        {
          class: 'c2',
          style: { fontWeight: 'bold' }
        },
        props.foo
     )
  }
}

const root = document.createElement('div')
document.body.appendChild(root)
render(h(Hello), root)

```

步骤一：第一次渲染，流程图②setupComponent的initProps，使用Hello组件的vnode.props（当前为空，所以输出的attrs都是为空的），标准化生成instance.attrs和instance.props
```typescript
if (!instance.type.props) {
  // functional w/ optional props, props === attrs
  instance.props = attrs
} else {
  // functional w/ declared props
  instance.props = props
}
```

步骤二：②setupConponent中，renderComponentRoot(instance)会把instance.attrs与调用instance.render（Hello.setup返回的方法）所返回的vnode的props进行合并。

步骤三：子组件进行patch，同样经过initProps处理，
在②中调用setup方法，并传入instance.props, instance.setupConetxt，这个时候子组件就会利用到父组件传入的props了。

_**那更新的时候，是怎么样的？**_

触发父组件instance，renderComponentRoot(instance)中调用instance.render（这个过程就是vnode.props的更新了）
再次进行当前track追踪（effect运行的特性，是会删除当前追踪），patch(旧vnode1, 新vnode2)，processComponent updateComponent，
新vnode2引用vnode1.component（instance2，子组件instance2），删除queue任务中的instance2.update，instance2.next = n2，执行instance2.update，
检测到有next字段，执行updateComponentPreRender,这里instance2是为了标识是子instance。

updateComponentPreRender：
```typescript
nextVNode.component = instance2 // 对当前没用，因为已经引用了
const prevProps = instance2.vnode.props
instance2.vnode = nextVNode // 更新vnode
instance2.next = null // 删除next
updateProps(instance2, nextVNode.props, prevProps, optimized)
updateSlots(instance2, nextVNode.children)
```
updateProps会利用setFullProps来更新attrs和props，这里的更新过的props和attrs可以提供給Child内部使用，如当前使用了props.foo。
