# 测试用例
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

## processComponent流程图：
https://www.processon.com/view/link/5f85c9321e085307a0892f7e

![processComponent流程图](https://res.psy-1.com/FkR9Rnsq2SKkx9C274ypMAL1MUss)


## vnode：
https://www.processon.com/view/link/5f963f37f346fb06e1ec35b2
![vnode](https://res.psy-1.com/Fl3_9xTvszqf7gRIPdOWG4eqT4k0)


## instance：
https://www.processon.com/view/link/5f963f5fe401fd06fda22681
![instance](https://res.psy-1.com/FmF17k7lj1iysIogq31ZIkj8HGf5)
