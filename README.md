# Vue3源码流程图化解释

vue3做了最大的变化就是api的细分，适配typescript。給我一种感觉就是，vue3像乐高，一个个拼接起来成模块，
模块之间的互相组合，来构成一个整体，
这样更利于团队开发了，可以根据团队情况来定制合适的开发架构。先简单说一下vue3中最重要的包：Reactivity、runtime和compiler。

#### Reactivity
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
