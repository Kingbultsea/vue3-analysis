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

#### render
入口<font color=#ff8000>render(①vnode，②HtmlElement)</font>，①参数是vnode，②参数是你实际浏览器中的DOM，这一整个东西就是把vnode，变成一个真的DOM，然后插入到②参数DOM中。
我们先来了解一下这个①，这个①在当前测试用例中使用<font color=#ff8000>h(Hello)</font>制作出来的，这做了啥？设置<font color=#ff8000>vnode.type = Hello</font>，设置vnode.shapeFlag，
检测到传入的参数Hello是Object类型的，所以<font color=#ff8000>vnode.shapeFlag = ShapeFlags.STATEFUL_COMPONENT</font>，顺便说一下normalizeChildren，如果有chlidren的情况下是用来加密vnode.shapeFlag的
，这里有没有chlidren的传入，可以忽略，啥？啥又是chlidren的传入？
```typescript
const testVnode = h('span', { id: 1 }, 'nihao')
render(testVnode, HTMLDivElement)
// 最终生成<div><span id="1">nihao</span></div>
```
这下了解了吧，这个nihao就是chldren。

![wocao](https://res.psy-1.com/FimJsXIrCCMb7NhhduJbEBUPD2tF)什么，还不懂？那从头看一遍，懂了的可以继续往下看<font color=#ff8000>render</font>到底做了啥。

#### shapeFlage 
把②参数命名为container，传入render的①vnode称为n2，container.vnode称为n1。

判断到n2不为空，进行<font color=#ff8000>patch</font>，判断到n1 == null，且n2.shapeFlag进行decode，为ShapeFlags.COMPONENT类型，执行processComponent。
这里的decode和encode是什么来的？
我们在生成vnode的时候，vnode.type数据被<font color=#ff8000>normalizeChildren</font>加密过，因为当前children为null,所以type为0，加密方式为<font color=#ff8000>vnode.shapeFlag |= type</font>
解密方式为<font color=#ff8000>const a = vnode.shapeFlag & ShapeFlags.COMPONENT</font>，只要a > 0就为true，ShapeFlags.COMPONENT为一个常数。
```typescript
export declare const enum ShapeFlags {
    ELEMENT = 1, // 1
    FUNCTIONAL_COMPONENT = 2, // 10
    STATEFUL_COMPONENT = 4, // 100
    TEXT_CHILDREN = 8, // 1000
    ARRAY_CHILDREN = 16, // 10000
    SLOTS_CHILDREN = 32, // 100000
    TELEPORT = 64, // 1000000
    SUSPENSE = 128, // 10000000
    COMPONENT_SHOULD_KEEP_ALIVE = 256, // 100000000
    COMPONENT_KEPT_ALIVE = 512, // 1000000000
    COMPONENT = 6 // 110
}
```
![托脸](https://res.psy-1.com/FkYnyYqXEj0EDfF5IlRr5L2dz5zR)想了解这套运行机制麽？上面被normalizeChildren过的，
是按位与的意思，比如二进制中
```typescript
01 | 01 === 01
01 | 10 === 11
10 | 10 === 10 
```
对应位置0 | 1 === 1, 0 | 0 === 0，就像||的逻辑，只要满足其中一个为真值。

判断类型的按位且
```typescript
11 & 11 === 11
01 & 10 === 00
10 & 10 === 10
```
对应位置1 & 1 === 1, 1 | 0 === 0，就像&&的逻辑，要满足所有为真值。

有没有注意到上面typescript的ShapeFlags的枚举,后面有二进制注释,有没有注意到判断类型渲染，都是使用if来判断的，就是说只要大于0就行。
有没有发现了什么？再提醒一点，二进制标记法？没错<font color=#ff8000>normalizeChildren</font>中，
只不过是两个类型利用二进制标记的合并，比如100代表STATEFUL_COMPONENT类型，10000代表ARRAY_CHILDREN类型，两个使用100 | 10000是不是等于10100，而在
<font color=#ff8000>patch</font>中<font color=#ff8000>if(vnode.shapeFlag & ShapeFlags.COMPONENT)</font>，
<font color=#ff8000>ShapeFlags.COMPONENT</font>为110，再看看那个表，是不是FUNCTIONAL_COMPONENT | STATEFUL_COMPONENT可以得到110？
所以10100 & 110为true，以上方法就是通过|来进行两种类型二进制占位符来合并，通过&判断该位置的值是否存在。

![bailefolun](https://res.psy-1.com/FmdgGYpuhYvAxJDIYXyxZEaHTkdW)是不是顿时大悟，感觉自己的代码质量大升，下次写代码也可以通过二进制标记来进行类型判断了！学废了没？
那继续下面的东西。

#### processComponent 
<font color=#ff8000>processComponent</font>，判断到<font color=#ff8000>n1 == null && n2.shapeFlag & ShapeFlags.COMPONENT_KEPT_ALIVE</font>
则执行<font color=#ff8000>mountComponent</font>，这个东西作用是挂载组件，这里挂载组件分三步走：

* **createComponentInstance** 创建instance，一个组件相关的Object。
* **setupComponent**，对attrs的一些处理 还有setup的处理 instance.type中的所有关键词字段的处理。
* **setupRenderEffect**，处理instance.render。

#### createComponentInstance
没有做什么，就是像vnode一样，生成了一个对象<font color=#ff8000>instance</font>，该对象记录了当前组件的信息，比如parent字段为父组件的instance，
root根组件的instance，render根据STATEFUL_COMPONENT、FUNCTIONAL_COMPONENT所生成的一个返回vnode的函数，也会记录一些生命周期的钩子相关字段。

#### setupComponent
<font color=#ff8000>initProps</font>，标准化生成vnode.attrs和vnode.props。
instance.props = shallowReactive(props) 浅响应式化，这里的浅响应式化我是认为没有任何作用的，可以去看一些问题章节中的3,props的更新，是根本用不着的。
<font color=#ff8000>initSlots</font>
