## 关于compile转换成render方法的一些例子

在带有编译版本的vue中，finishComponentSetup会对没有render方法，但是有template的Component，做编译处理。

finishComponentSetup的代码片段:

```typescript
Component.render = compile(Component.template, {
        isCustomElement: instance.appContext.config.isCustomElement || NO
})
```



去查看一下compile的CompilerOptions相关配置。

```typescript
type CompilerOptions = ParserOptions & TransformOptions & CodegenOptions

// ParserOptions:
export interface ParserOptions {
  // 平台上本地元素, 例如<div>是网页上的本地标签
  isNativeTag?: (tag: string) => boolean
  
  // 不需要闭合的原生元素, 例如 <img>, <br>, <hr>
  isVoidTag?: (tag: string) => boolean
 
  // 应该保留内部空白的元素, 例如<pre> 
  // https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/pre
  isPreTag?: (tag: string) => boolean
 
  // 特定于平台的内置组件，例如<Transition>
  isBuiltInComponent?: (tag: string) => symbol | void
  
  // 自定义于平台的本地元素
  isCustomElement?: (tag: string) => boolean
  
  // 获取标签的名称
  getNamespace?: (tag: string, parent: ElementNode | undefined) => Namespace
  
  // 获取此元素的文本分析模式
  getTextMode?: (
    node: ElementNode,
    parent: ElementNode | undefined
  ) => TextModes
    
  // 默认 ['{{', '}}']，可修改为你喜欢的
  delimiters?: [string, string]
  
  // Only needed for DOM compilers，解码实体
  decodeEntities?: (rawText: string, asAttr: boolean) => string
  
  // 错误  
  onError?: (error: CompilerError) => void
}


// TransformOptions：
export interface TransformOptions {
  /**
   * An array of node trasnforms to be applied to every AST node.
   */
  nodeTransforms?: NodeTransform[]
  /**
   * An object of { name: transform } to be applied to every directive attribute
   * node found on element nodes.
   */
  directiveTransforms?: Record<string, DirectiveTransform | undefined>
  /**
   * An optional hook to transform a node being hoisted.
   * used by compiler-dom to turn hoisted nodes into stringified HTML vnodes.
   * @default null
   */
  transformHoist?: HoistTransform | null
 
  // 如果提供了其他的内置元素，可以使用该选项标记为内置的，这样编译器将为这些标签生成组件vnode  
  isBuiltInComponent?: (tag: string) => symbol | void
  /**
   * 把表达式 {{ foo }} 转换成 `_ctx.foo`
   * 如果该选项为false, the generated code will be wrapped in a
   * `with (this) { ... }` block.
   * - This is force-enabled in module mode, since modules are by default strict
   * and cannot use `with`
   * @default mode === 'module'
   */
  prefixIdentifiers?: boolean
    
    
  // 静态提升到 `_hoisted_x` 变量
  // 默认值 false
  hoistStatic?: boolean
    
  /**
   * Cache v-on handlers to avoid creating new inline functions on each render,
   * also avoids the need for dynamically patching the handlers by wrapping it.
   * e.g `@click="foo"` by default is compiled to `{ onClick: foo }`. With this
   * option it's compiled to:
   * ```js
   * { onClick: _cache[0] || (_cache[0] = e => _ctx.foo(e)) }
   * ```
   * - Requires "prefixIdentifiers" to be enabled because it relies on scope
   * analysis to determine if a handler is safe to cache.
   * @default false
   */
  cacheHandlers?: boolean
  /**
   * A list of parser plugins to enable for `@babel/parser`, which is used to
   * parse expressions in bindings and interpolations.
   * https://babeljs.io/docs/en/next/babel-parser#plugins
   */
  expressionPlugins?: ParserPlugin[]
  
    // Single File Component组件中scoped styles ID
  scopeId?: string | null
  /**
   * Generate SSR-optimized render functions instead.
   * The resulting funciton must be attached to the component via the
   * `ssrRender` option instead of `render`.
   */
  ssr?: boolean
  onError?: (error: CompilerError) => void
}
```





```html
<div id="foo" :class="bar.baz">
  {{ world.burn() }}
  <div v-if="ok">yes</div>
  <template v-else>no</template>
  <div v-for="(value, index) in list"><span>{{ value + index }}</span></div>
</div>
```

转换成render:

```typescript
const _Vue = Vue

return function render(_ctx, _cache) {
  with (_ctx) {
    const { toDisplayString: _toDisplayString, createVNode: _createVNode, openBlock: _openBlock, createBlock: _createBlock, createCommentVNode: _createCommentVNode, createTextVNode: _createTextVNode, Fragment: _Fragment, renderList: _renderList } = _Vue

    return (_openBlock(), _createBlock("div", {
      id: "foo",
      class: bar.baz
    }, [
      _createTextVNode(_toDisplayString(world.burn()) + " ", 1 /* TEXT */),
      ok
        ? (_openBlock(), _createBlock("div", { key: 0 }, "yes"))
        : (_openBlock(), _createBlock(_Fragment, { key: 1 }, [
            _createTextVNode("no")
          ], 64 /* STABLE_FRAGMENT */)),
      (_openBlock(true), _createBlock(_Fragment, null, _renderList(list, (value, index) => {
        return (_openBlock(), _createBlock("div", null, [
          _createVNode("span", null, _toDisplayString(value + index), 1 /* TEXT */)
        ]))
      }), 256 /* UNKEYED_FRAGMENT */))
    ], 2 /* CLASS */))
  }
}
```

