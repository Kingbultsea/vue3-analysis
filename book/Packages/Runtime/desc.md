## vue3渲染流程
### 测试用例
```typescript
it('should allow attrs to fallthrough', async () => {
    debugger
    const click = jest.fn()
    const childUpdated = jest.fn()

    const Hello = {
      setup() { // 如果有render 就优先渲染render了
        const count = ref(0)

        function inc() { // 需要了解onClick事件系统
          count.value++
          click()
        }

        // 这里的任何东西 都不会被本身effect收集 只有return 后的方法 才会

        return () => // 这里还可以当redner来使用 我平时会的只是 return {}  template所需要的事件或属性 所以是检查function 还是 object来判断的吧
          h(Child, { // 那是不是有一个props effect? 标记也要注意下
            foo: count.value + 1,
            id: 'test',
            class: 'c' + count.value,
            style: { color: count.value ? 'red' : 'green' },
            onClick: inc,
            'data-id': count.value + 1
          })
      },
      mounted() {
        console.log('?')
      }
    }

    const Child = { // 原来是这样传参数的 那么就不需要this 什么的了
      setup(props: any) {
        onUpdated(childUpdated)
        return () =>
          h(
            'div',
            {
              class: 'c2',
              style: { fontWeight: 'bold' }
            },
            props.foo // 这个为什么是undefinded呢 因为setFullProps执行的时候，判断到的赋值Props方式为attrs，所以instance.props为空
         )
      }
    }

    const root = document.createElement('div')
    document.body.appendChild(root)
    render(h(Hello), root) // 这个render 是'@vue/runtime-dom' 我们之前用的 是'@vue/runtime-test' 里面的 测试用的... 但是区别不一样的就是 会不会初始化而已

    const node = root.children[0] as HTMLElement

    expect(node.getAttribute('id')).toBe('test')
    expect(node.getAttribute('foo')).toBe('1')
    expect(node.getAttribute('class')).toBe('c2 c0')
    expect(node.style.color).toBe('green')
    expect(node.style.fontWeight).toBe('bold')
    expect(node.dataset.id).toBe('1')

    node.dispatchEvent(new CustomEvent('click')) // 事件触发a
    node.dispatchEvent(new CustomEvent('click')) // 事件触发a
    expect(click).toHaveBeenCalled()

    await nextTick()
    expect(childUpdated).toHaveBeenCalled()
    expect(node.getAttribute('id')).toBe('test')
    expect(node.getAttribute('foo')).toBe('2')
    expect(node.getAttribute('class')).toBe('c2 c1')
    expect(node.style.color).toBe('red')
    expect(node.style.fontWeight).toBe('bold')
    expect(node.dataset.id).toBe('2')
  })
```
请使用该测试用例进行单步调试。[processComponent流程图](https://www.processon.com/view/link/5f85c9321e085307a0892f7e)

**什么是vnode？**网上搜索到的都是说visual dom用来描述真实的DOM标签，作用是可以通过render渲染成DOM，然后挂载。
这些对于入门的人，肯定一脸懵逼，其实vnode就是一个object记录了渲染过程所需要用到的数据而已，对于每一个字段的作用，深入下去，一个一个慢慢认识。
这里附带上[vnode字段图](https://www.processon.com/view/link/5f963f37f346fb06e1ec35b2)

**什么是instance？**
和vnode一样的行为，但是这只应用于Component类型的组件，所以和vnode分开，作为一个vnode的拓展，当然vnode.component字段就包含对应的instance。
[instance](https://www.processon.com/view/link/5f963f5fe401fd06fda22681)

**什么是h？**[渲染函数&JSX](https://cn.vuejs.org/v2/guide/render-function.html)
你为什么要了解这个？比如你的template是
```html
<template>
  <div id="1">123</div>
</template>
```  
会被compiler转换成h('div', { id: 1 }, 123)，这里不去会去说compiler的转换，单纯的讲render流程，
compiler做的东西还有静态标记之类的，这里只是举个简单的例子，将会在compiler章节详细说。

### 渲染过程
![熊猫紧张](https://res.psy-1.com/Fr9pcXuMBigc_ofuRmebvi-XsUx_)前排提示多喝热水，请打开
[渲染流程图](https://www.processon.com/view/link/5f85c9321e085307a0892f7e)、
[instance](https://www.processon.com/view/link/5f963f5fe401fd06fda22681)、
[vnode](https://www.processon.com/view/link/5f963f37f346fb06e1ec35b2)
和单步调试来食用，
如果你没有使用电脑，无法进行单步调试，没关系，看着我的图也不是不行，我会以文字和概念的方式，尽量和你科普明白。<font color=#ff8000>这个颜色是运行的代码</font>。

入口<font color=#ff8000>render(①vnode，②HtmlElement)</font>，①参数是vnode，②参数是你实际浏览器中的DOM，这一整个东西就是把vnode，变成一个真的DOM，然后插入到②参数DOM中。
我们先来了解一下这个①，这个①在当前测试用例中使用<font color=#ff8000>h(Hello)</font>制作出来的，这做了啥？设置<font color=#ff8000>vnode.type = Hello</font>，设置vnode.shapeFlag，
检测到传入的参数Hello是Object类型的，所以<font color=#ff8000>vnode.shapeFlag = ShapeFlags.STATEFUL_COMPONENT</font>，顺便说一下normalizeChildren，如果有chlidren的情况下是用来加密vnode.shapeFlag的
，这里有没有chlidren的传入，可以忽略，啥？啥又是chlidren的传入？
```typescript
const testVnode = h('span', { id: 1 }, 'hello')
render(testVnode, HTMLDivElement) 
// 最终生成<div><span id="1">hello</span></div>
```
这下了解了吧，这个hello就是chldren。

![wocao](https://res.psy-1.com/FimJsXIrCCMb7NhhduJbEBUPD2tF)什么，还不懂？那从头看一遍，懂了的可以继续往下看<font color=#ff8000>render</font>到底做了啥。
