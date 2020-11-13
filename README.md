# Vue3源码流程图化解释

（不懂的，想来学习的，或者是想交流的欢迎进群讨论。目前gitbook还没有整理，有些知识点可能比较乱，表述得不清楚，可以进群探讨）

vue3做了最大的变化就是api的细分，适配typescript。給我一种感觉就是，vue3像乐高，一个个拼接起来成模块，
模块之间的互相组合，来构成一个整体，
这样更利于团队开发了，可以根据团队情况来定制合适的开发架构。composition-api的出现，如果想真正利用好，弄懂vue3源码是必须的。
vue3中重要的包：Reactivity、runtime-x和 compiler-x。

gitbook链接：https://kingbultsea.github.io/vue3-analysis/book/index.html

## 如何阅读源码
单步调试单元测试。

使用webstrom，Run -> Edit Configurations -> + -> npm
![avatar](https://res.psy-1.com/FoHPNlyOB_b3UEazTaiIxQz-jER4)

[使用vscode](https://blog.csdn.net/weixin_30597269/article/details/99215170)

设置好debugger之后，修改jest.config.js文件指向一个你所需要测试的文件即可。
```typescript
testMatch: ['<rootDir>/packages/runtime-core/__tests__/**/rendererAttrsFallthrough.spec.[jt]s?(x)']
```
比如这里是启动runtime-core中的rendererAttrsFallthrough.spec.ts测试用例。

#### reactivity包的单元测试阅读顺序
ref.spec.ts -> reactive.spec.ts -> readonly.spec.ts -> 
reactiveArray.spec.ts -> shallowReactive.spec.ts ->
effect__.spec.ts -> computed.spec.ts -> collections(Map Set weekMap weekSet的处理)

#### runtime-core包的单元测试阅读顺序
vnode.spec.ts -> h.spec.ts -> vnodeHooks.spec.ts -> scheduler.spec.ts ->
rendererElement.spec.ts -> rendererFragment.spec.ts -> 
rendererComponent.spec.ts -> rendererChildren.spec.ts(了解diff算法) -> 
rendererAttrsFallthrough.spec.ts

#### 如何调试promise(microtask微任务)?
在微任务内部打上断点debugger，单步调试时，直接点击跳转到下一个断点即可。

## Reactivity
Reactivity是响应式数据包，vue3中可以使用ref和relative来进行响应式数据化，这个包涉及三个概念effect、track、trigger，
effect(fn)就好比添加影响，fn执行的过程中，遇到响应式数据取值，则触发响应式数据所获取的字段的track，
添加追踪当前激活的effect，当有事件触发响应式数据对应的字段的修改值的行为时，将会trigger，触发所追踪的effect，再次执行fn的流程。

测试用例(reactivity/\_\_tests\_\_/effect.spec.ts)：
```typescript
it('should observe basic properties', () => {
    let dummy
    const counter = reactive({ num: 0 })
    effect(() => (dummy = counter.num))
    expect(dummy).toBe(0)
    counter.num = 7
    expect(dummy).toBe(7)
})
```
## runtime
涉及到runtime的包有runtime-core、runtime-dom和runtime-test。把vnode渲染成真实的dom，而runtime-core
和runtime-test的区别是，runtime-core是用到生产环境中，runtime-test是用在测试环境中的，两者的代码并没有本质性的区别，
只不过是涉及到真实的dom的时候，runtime-test使用nodeOps来模拟真实dom。

runtime涉及到两个对象，一个是instance，是函数式组件中所用到的（STATEFUL_COMPONENT），用来记录生命周期钩子，
组件之间的上下文交互。
另外一个是vnode，渲染属性的记录。

测试用例(runtime-core/\_\_test\_\_/rendererElement.spec.ts)：
```typescript
beforeEach(() => {
  root = nodeOps.createElement('div')
})

it('should create an element', () => {
  render(h('div'), root)
  expect(inner(root)).toBe('<div></div>')
})
```

## compiler
我们使用webpack的vue-loader插件，总是会帮我们把template标签转换成instance.render函数，如果没有该插件，在手写OPTION API中的template，将会在render渲染过程中的第二个步骤，检测当前使用的版本有没有使用compile，有则调用compile转换。

这里compile所做的还有静态标记，静态dom的缓存，重复dom的内存收集。