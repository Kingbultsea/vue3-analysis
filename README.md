# Vue3源码流程图化解释

vue3做了最大的变化就是api的细分，适配typescript。給我一种感觉就是，vue3像乐高，一个个拼接起来成模块，
模块之间的互相组合，来构成一个整体，
这样更利于团队开发了，可以根据团队情况来定制合适的开发架构。先简单说一下vue3中最重要的包：Reactivity、runtime和compiler。

## 如何阅读源码
单步调试单元测试。

使用webstrom，Run -> Edit Configurations -> + -> npm
![avatar](https://res.psy-1.com/FoHPNlyOB_b3UEazTaiIxQz-jER4)

[使用vscode](https://blog.csdn.net/weixin_30597269/article/details/99215170)

然后修改jest.config.js文件
```typescript
testMatch: ['<rootDir>/packages/runtime-core/__tests__/**/rendererAttrsFallthrough.spec.[jt]s?(x)']
```
比如这里是启动runtime-core中的rendererAttrsFallthrough.spec.ts测试用例。

##### 如何调试promise(microtask微任务)?
在微任务内部打上断点debugger，单步调试时，直接点击跳转到下一个断点即可。

## Reactivity
Reactivity是响应式数据包，vue3中可以使用ref和relative来进行响应式数据化，这个包涉及三个概念effect、track、trigger，
effect(fn)就好比添加影响，fn执行的过程中，遇到响应式数据取值，则触发响应式数据所获取的字段的track，
添加追踪当前激活的effect，当有事件触发响应式数据对应的字段的修改值的行为时，将会trigger，触发所追踪的effect，再次执行fn的流程。

测试用例(reactivity/__tests__/effect.spec.ts)：
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

测试用例(runtime-core/__test__/rendererElement.spec.ts)：
```typescript
beforeEach(() => {
  root = nodeOps.createElement('div')
})

it('should create an element', () => {
  render(h('div'), root)
  expect(inner(root)).toBe('<div></div>')
})
```
