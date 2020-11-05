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

![bailefolun](https://res.psy-1.com/FmdgGYpuhYvAxJDIYXyxZEaHTkdW)是不是顿时大悟，感觉自己的代码质量大升，下次写代码也可以通过二进制标记来进行类型判断了！学废了没？那继续下面的东西。

#### processComponent 
<font color=#ff8000>processComponent</font>，判断到<font color=#ff8000>n1 == null && n2.shapeFlag & ShapeFlags.COMPONENT_KEPT_ALIVE</font>
则执行<font color=#ff8000>mountComponent</font>，这个东西作用是挂载组件，这里挂载组件分三步走：

* createComponentInstance 创建instance，一个组件相关的Object。
* setupComponent，对attrs的一些处理 还有setup的处理 instance.type中的所有关键词字段的处理。
* setupRenderEffect，处理instance.render。

#### createComponentInstance
没有做什么，就是像vnode一样，生成了一个对象<font color=#ff8000>instance</font>，该对象记录了当前组件的信息，比如parent字段为父组件的instance，
root根组件的instance，render根据STATEFUL_COMPONENT、FUNCTIONAL_COMPONENT所生成的一个返回vnode的函数，也会记录一些生命周期的钩子相关字段。

#### setupComponent
<font color=#ff8000>initProps</font>:

```typescript
// 创建了一个props和attrs的数组:
const props = []
const attrs = []
```
<font color=#ff8000>setFullProps(instance, instance.vnode.props, props, attrs)</font>，就是处理props啦，什么，你问这个东西做了什么？

![熊猫紧张](https://res.psy-1.com/FsnFrGmMvgD_GI9YnZeJuc8-xTbk)

**1**.normalizePropsOptions，我们组件中，如果存在props字段。
```typescript
const a = {
  props: ['a-b', 'c-d']
}
const b = {
  props: { 
    foo: Boolean,
    test: {
        type: Number,
        default: 0
    }
  }
}

const normalized = {}
const needCastKeys = []
```

如果拿a做举例，检测到props是一个数组，看代码你就明白了，不用文字解释了![托脸](https://res.psy-1.com/FkYnyYqXEj0EDfF5IlRr5L2dz5zR)
```typescript
const EMPTY_OBJ: { readonly [key: string]: any } = __DEV__
  ? Object.freeze({})
  : {}

normalized = {
    'aB': EMPTY_OBJ,
    'cD': EMPTY_OBJ
}
needCastKeys = undefined
```

b做举例，检测到是对象，遍历props，把key转换为驼峰式写法，
检测到key为foo的type为Boolean类型，<font color=#ff8000>needCastKeys.push(key)</font>，检测到test拥有default字段，
<font color=#ff8000>needCastKeys.push(key)</font>。下面这个代码可以不看。
```typescript
for(const key in instance.props) {
  const opt = raw[key]
  const prop: NormalizedProp = (normalized[normalizedKey] =
          isArray(opt) || isFunction(opt) ? { type: opt } : opt)
  if (prop) {
    const booleanIndex = getTypeIndex(Boolean, prop.type) // 检测boolean类型，如果是数组返回第一个index否则返回0
    const stringIndex = getTypeIndex(String, prop.type) // 检测string类型
    prop[BooleanFlags.shouldCast] = booleanIndex > -1
    prop[BooleanFlags.shouldCastTrue] =
      stringIndex < 0 || booleanIndex < stringIndex
    // if the prop needs boolean casting or default value
    if (booleanIndex > -1 || hasOwn(prop, 'default')) {
      needCastKeys.push(normalizedKey)
    }
  }
}
```

最终输出：
```typescript
normalized = {
  foo: {
    type: Boolean
  },
  test: {
    type: Number,
    default: 0
  }
}
normalized.foo[BooleanFlags.shouldCast] = true
normalized.foo[BooleanFlags.shouldCastTrue] = true

normalized.test[BooleanFlags.shouldCast] = true
normalized.test[BooleanFlags.shouldCastTrue] = true
needCastKeys = ['foo', 'test']
```

这里有一个是instance.type.mixins、instance.type.mixins和全局minxins的循环调用normalizePropsOptions，最终是会合并成一整个:
```typescript
const normalizedEntry: NormalizedPropsOptions = [normalized, needCastKeys]
return normalizedEntry
```

**2**.经过1得到\[<font color=#ff8000>normalized</font>, <font color=#ff8000>needCastKeys</font>\]

![setFullProps](https://res.psy-1.com/Fg7cLitiSUiz-V6wR9ogC5-N4F-B)

最后根据\[<font color=#ff8000>normalized</font>, <font color=#ff8000>needCastKeys</font>\]
遍历vnode.props来赋值給props或是attrs。
还记得一开始我们创建的props,attrs麽？以上都是为了设置这两个的。

**3**.如果有<font color=#ff8000>needCastKeys</font>，这里<font color=#ff8000>normalized</font>改一个名称为<font color=#ff8000>options</font>
```typescript
if (needCastKeys) {
    const rawCurrentProps = toRaw(props) // 避免有引用属性的响应式的干扰
    for (let i = 0; i < needCastKeys.length; i++) {
      const key = needCastKeys[i]
      props[key] = resolvePropValue(
        options!,
        rawCurrentProps,
        key,
        rawCurrentProps[key]
      )
    }
  }
```
我们现在还是用上面的1中的b做例子，这里先说Boolean的。这里的例子因为props没有test这个字段，所以设置默认值为false。
```typescript
// 参数名称：
resolvePropValue(
  options: NormalizedPropsOptions[0],
  props: Data,
  key: string,
  value: unknown
)
 

const opt = options[key] as any
const hasDefault = hasOwn(opt, 'default')

// boolean casting
    if (opt[BooleanFlags.shouldCast]) {
      if (!hasOwn(props, key) && !hasDefault) {
        value = false
      } else if (
        opt[BooleanFlags.shouldCastTrue] &&
        (value === '' || value === hyphenate(key))
      ) {
        value = true
      }
    }
return value // 执行到这里
```

再说一下default的，没有value且有默认这个字段，检测到default不是function而是一个值，所以直接防护default的值0.
```typescript
// 参数名称：
resolvePropValue(
  options: NormalizedPropsOptions[0],
  props: Data,
  key: string,
  value: unknown
)

const opt = options[key] as any
const hasDefault = hasOwn(opt, 'default')

// default values
    if (hasDefault && value === undefined) {
      const defaultValue = opt.default
      value =
        opt.type !== Function && isFunction(defaultValue)
          ? defaultValue()
          : defaultValue
    }
return value
```

看不懂有什么用是吧？**就这么说吧，现在有两个数组一个为props一个为attrs，然后经过setFullProps，这个东西把我们组件中的props的键值給设置为驼峰式，然后遍历vnode.props，
但是跳过键为'key'和'ref'，如果需要驼峰形式的则赋值給props，如果不存在instance.type.emits或者没有被子组件触发emits则赋值給attrs。遍历needCastKeys
，props经过resolvePropValue的处理，给上默认值，就是給默认值而已，只对props的处理。**

![熊猫紧张](https://res.psy-1.com/Fr9pcXuMBigc_ofuRmebvi-XsUx_)什么？你又想让我給你科普一下emit？下次一定下次一定！行了，点开emit章节，你就能了解欸。

最后设置instance的引用。
```typescript
if (isStateful) {
  // stateful
  instance.props = isSSR ? props : shallowReactive(props) // 所以props一层响应
} else {
  if (!instance.type.props) {
    // functional w/ optional props, props === attrs
    instance.props = attrs
  } else {
    // functional w/ declared props
    instance.props = props
  }
}
instance.attrs = attrs
```
![熊猫紧张](https://res.psy-1.com/Fr9pcXuMBigc_ofuRmebvi-XsUx_)流程图有... 写得很明白，希望你能去看几眼。

instance.attrs 将会被subTree.props合并，subTree就是instance.render()返回的vnode。
instance.props 状态类型组件（STATEFUL_COMPONENT）在运行instance.setup中传入的是instance.proxy，只会在instance.render(instance.props)应用到。
函数类型的组件，调用的时候也会传入instance.props。


<font color=#ff8000>initSlots</font>：

这里的测试用例没有slots，相关移步去slots章节。

<font color=#ff8000>setupStatefulComponent</font>：
ShapeFlags.STATEFUL_COMPONENT才会执行，当前组件为ShapeFlags.STATEFUL_COMPONENT类型。设置accessCache和proxy，proxy在处理options，或者说options中使用的this，将指向proxy。
就是methods: { foo(){ console.log(this) } }，会输出instance.proxy。
```
instance.accessCache = {}
instance.proxy = new Proxy(instance.ctx, PublicInstanceProxyHandlers)
```

组件中setup的运行：
```typescript
currentInstance = instance // 使用钩子api的时候会使用到。
pauseTracking() // 暂停track，reactive相关，这里停止track是避免执行setup方法的时候，有响应式数据追踪到其他地方的effect，比如具有父组件进行componentEffect的过程中会patch子组件，子组件更新，父组件也需要更新，这一步仅仅在componentEffect中处理。
setupResult = callWithErrorHandling // 调用组件的setup方法，传入(instance.props, instance.setupConetxt)。
resetTracking() // 恢复上一个track的状态，reactive相关
currentInstance = null // 恢复
```

callWithErrorHandling包裹setup方法，执行<font color=#ff8000>setup(instance.props, instance.setupConetxt)</font>。

setup内部有钩子，当前测试用例调用了onUpdated(CLICKEVENT)钩子，设置instance.u = []，instance.u.push(CLICKEVENT)，可以去看instance.u字段。
```typescript
createHook(LifecycleHooks.UPDATED)

// createHook内部：
injectHook(LifecycleHooks.UPDATED, CLICKEVENT, currentInstance)

// injectHook内部：
const hooks = target[type] || (target[type] = [])
const wrappedHook =
hook.__weh || hook.__weh = (...args: unknown[]) => {
      // 暂不讨论...
}
hooks.push(wrappedHook)
```

最后返回setup的结果給setupResult。执行<font color=#ff8000>handleSetupResult</font>，setupResult是一个Function，
所以:
```typescript
instance.render = setupResult
```
如果setupResult是一个Object类型：
```typescript
instance.setupState = reactive(setupResult) // 进行响应式
```

setupResult为Object类型的时候，进行响应式化有什么好处？（其实我是感觉防止新手不知道怎么处理吧）我们平时会直接返回一个对象，对象里面包裹响应式数据对象，如果再套一层响应式化，可以让我们直接设置字段，
输入数据，不需要再进行一次relative或者ref的调用，这和处理instance.type.data同一个原理，**也就是说你可以直接在setup中的return直接当data(){}那样返回**。

这里再提醒一下，你需要打开[渲染流程图](https://www.processon.com/view/link/5f85c9321e085307a0892f7e)，才能知道流程走向到底去哪了。

执行<font color=#ff8000>finishComponentSetup</font>，这里就涉及到options的设置了。具体去测试用例调试吧，如果需要特别说一下可以在Github提一下issus，这里说几个重点。

**callSyncHook('beforeCreate', instance.type.options)，调用全局mixin、extends、本身mixin，最后才是调用自身的。
这里会有两个钩子被执行'beforeCreate'和'created'，这两个钩子是使用composition API是没有的，这里的钩子比setup中使用api的钩子要提早执行。
如果内部方法使用this，那么都会指向instance.proxy，详情可以查看PublicInstanceProxyHandlers。**

#### setupRenderEffect
![sucide](https://res.psy-1.com/Fn1AHqFf-NFtXHr6tO7AS30GVs-F)终于要看到重点了，这个就分开说吧，因为涉及到的东西很多了。
