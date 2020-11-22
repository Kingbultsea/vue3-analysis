# 为什么使用slot，事件没有更新？

代码：

```html
<!-- parent -->
<template>
    <button @click="changeText">change test</button>
    <A :name="test" ref="A">
        <template v-slot="{ record }">
            {{ record }}
            <B ref="B">
                <template v-slot:over>
                    <button @click="handleClick(record)">click me 啊2
                    </button>
                </template>
            </B>
        </template>
    </A>
</template>

<!-- A -->
<template>
    <div>
        <slot :record="name" />
    </div>
</template>
```



使用compile转换后的代码：

```typescript
import { createVNode as _createVNode, toDisplayString as _toDisplayString, resolveComponent as _resolveComponent, withCtx as _withCtx, createTextVNode as _createTextVNode, Fragment as _Fragment, openBlock as _openBlock, createBlock as _createBlock } from "vue"

export function render(_ctx, _cache, $props, $setup, $data, $options) {
  const _component_B = _resolveComponent("B")
  const _component_A = _resolveComponent("A")

  return (_openBlock(), _createBlock(_Fragment, null, [
    _createVNode("button", {
      onClick: _cache[1] || (_cache[1] = (...args) => (_ctx.changeText(...args)))
    }, "change test"),
    _createVNode(_component_A, {
      name: _ctx.test,
      ref: "A"
    }, {
      default: _withCtx(({ record }) => [
        _createTextVNode(_toDisplayString(record) + " ", 1 /* TEXT */),
        _createVNode(_component_B, { ref: "B" }, {
          over: _withCtx(() => [
            _createVNode("button", {
              onClick: $event => (_ctx.handleClick(record))
            }, "click me 啊2 ", 8 /* PROPS */, ["onClick"])
          ]),
          _: 1
        }, 512 /* NEED_PATCH */)
      ]),
      _: 1
    }, 8 /* PROPS */, ["name"])
  ], 64 /* STABLE_FRAGMENT */))
}
```

A:

```typescript
const { renderSlot: _renderSlot, createVNode: _createVNode, openBlock: _openBlock, createBlock: _createBlock } = Vue

function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", null, [
    _renderSlot(_ctx.$slots, "default", { record: _ctx.name }) // { record: _ctx.name } 传入 default 方法中
    // createBlock(Fragment, { key: props.key /* 这里参数没有key */ }, default(props), PatchFlags.STABLE_FRAGMENT)
  ]))
}
```



点击handle为什么传入的东西还是旧的值？为什么不会更新？



我现在是假设你已经熟悉首次渲染。

我们在更新的时候，在组件A的update中，生成了vnode，该vnode，在patch第一个元素的时候发现record那段textVnode有变化，故更新该元素。

第二个元素是组件B，更新组件B前会使用shouldUpdateComponent方法：

```typescript
function shouldUpdateComponent(prevVNode, nextVNode, optimized) {
    const { props: prevProps, children: prevChildren } = prevVNode;
    const { props: nextProps, children: nextChildren, patchFlag } = nextVNode;
    // Parent component's render function was hot-updated. Since this may have
    // caused the child component's slots content to have changed, we need to
    // force the child to update as well.
    if ((process.env.NODE_ENV !== 'production') && (prevChildren || nextChildren) && isHmrUpdating) {
        return true;
    }
    // force child update for runtime directive or transition on component vnode.
    if (nextVNode.dirs || nextVNode.transition) {
        return true;
    }
    if (patchFlag > 0) {
        if (patchFlag & 1024 /* DYNAMIC_SLOTS */) {
            // slot content that references values that might have changed,
            // e.g. in a v-for
            return true;
        }
        if (patchFlag & 16 /* FULL_PROPS */) {
            if (!prevProps) {
                return !!nextProps;
            }
            // presence of this flag indicates props are always non-null
            return hasPropsChanged(prevProps, nextProps);
        }
        else if (patchFlag & 8 /* PROPS */) {
            const dynamicProps = nextVNode.dynamicProps;
            for (let i = 0; i < dynamicProps.length; i++) {
                const key = dynamicProps[i];
                if (nextProps[key] !== prevProps[key]) {
                    return true;
                }
            }
        }
    }
    else if (!optimized) {
        // this path is only taken by manually written render functions
        // so presence of any children leads to a forced update
        if (prevChildren || nextChildren) {
            if (!nextChildren || !nextChildren.$stable) {
                return true;
            }
        }
        if (prevProps === nextProps) {
            return false;
        }
        if (!prevProps) {
            return !!nextProps;
        }
        if (!nextProps) {
            return true;
        }
        return hasPropsChanged(prevProps, nextProps);
    }
    return false;
}
```

这个东西是判断你的props和children有没有改变，没有则返回false，所以B组件不会更新。



那么，你是不是想问，record是指引属性，为什么还是旧的？你可以去认真看看initSlot和renderSlot，是调用为Object的子组件来实行的，旧的record就是在首次渲染的时候，调用defalut中的初始值，应用的地方也就是首次的环境。

而新的record根本就没有被更新到组件B上。



**那为什么我直接调用组件B的update方法，还是不更新呢？**

这边建议你再去熟悉一下渲染流程，英文组件B的vnode依旧没有发生变化，需要的是组件A创建新的组件B的vnode才能达到你的效果。



### 解决方法：

为了使组件B更新，可以随便绑定一个参数給B，比如

```html
<B :asd="record"></B>
```

或者

```html
<B>
  <span style="opacity: 0">{{ record }}</span>
</B>
```



### 关于修复

##### 原始问题：https://github.com/vuejs/vue-next/issues/2618

**修复方法**：https://github.com/vuejs/vue-next/pull/2568/files