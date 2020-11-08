# 响应式

先看以下与响应式相关常用API有哪些。

```typescript
// effect
export declare function effect<T = any>(fn: () => T, options?: ReactiveEffectOptions): ReactiveEffect<T>;

// 是不是响应式数据
export declare function isReactive(value: unknown): boolean;

// 是不是只读属性
export declare function isReadonly(value: unknown): boolean;

// 是不是Ref
export declare function isRef<T>(r: Ref<T> | unknown): r is Ref<T>;

// 标记，表示不进行响应式数据化
export declare function markRaw<T extends object>(value: T): T;

// 对象类型的响应式数据
export declare function reactive<T extends object>(target: T): UnwrapNestedRefs<T>;

// 基本数据类型的响应式数据
export declare function ref<T>(value: T): Ref<UnwrapRef<T>>;
```



## 创建过程

```typescript
const a = reactive({ foo: 1 })
const rf = ref('hello')
const ef = effect(() => {
  console.log(a.foo) // 因为a是Proxy类型，可以追踪getter的动作。
  console.log(rf.value)  // 同上
})
```



### reactive:

传入的是对象，但你传基本数据会直接不对target进行处理，直接返回。

```typescript
export function reactive(target: object) { 
  // { msg: 1, __v_isReadonly: true } 这样就会直接返回target什么都不做,ReactiveFlags.isReadonly标记
  if (target && (target as Target)[ReactiveFlags.isReadonly]) {
    return target
  }
    
  // setup执行
  return createReactiveObject(
    target,
    false, // 设置isReadonly为false
    mutableHandlers, // 设置proxy的hander
    mutableCollectionHandlers // 设置Map Set WeakMap WeackSet类型的hander
  )
}
```



### createReactiveObject:

```typescript
function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>
) {
  // 检查是不是Object 如果不是obejct 报错 且在dev环境直接不做任何处理 返回target
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }
  // target is already a Proxy, return it.
  // exception: calling readonly() on a reactive object
  // 检测到target已经是响应式数据，或者是readlyOnly数据
  // 都会直接返回target    
  if (
    target[ReactiveFlags.raw] &&
    !(isReadonly && target[ReactiveFlags.isReactive])
  ) {
    return target
  }
      
  // target already has corresponding Proxy
  if (
    hasOwn(target, isReadonly ? ReactiveFlags.readonly : ReactiveFlags.reactive)
  ) {
    return isReadonly
      ? target[ReactiveFlags.readonly]
      : target[ReactiveFlags.reactive]
  }

  // 拥有__v_skip 属性的 或者被Object.frozen处理过的都不做任何处理
  if (!canObserve(target)) {
    return target
  }

  const observed = new Proxy(
    target,
    // object不属于Set Map WeakSetWeakMap，选用baseHandlers。
    // 如果属于则选用  collectionHandlers 
    collectionTypes.has(target.constructor) ? collectionHandlers : baseHandlers
  )
  
  // target['__v_reactive'] = observed 互相指引
  def(
    // { msg, root } 然后做一个整体观察者proxy
    target,
    // "__v_readonly" /* readonly */ : "__v_reactive" /* reactive */
    isReadonly ? ReactiveFlags.readonly : ReactiveFlags.reactive,
    // proxy
    observed
  )

  return observed
}
```



## createRef，ref等于createRef(target, false)

```typescript
function createRef(rawValue: unknown, shallow = false) {
  // __v_isRef === true，则直接返回  
  if (isRef(rawValue)) {
    return rawValue
  }
  
  // 如果传入的是对象，则直接进行reactive(target),否则直接返回rawValue。
  // shallow 一层响应式 shallowRef的时候可以用到，只监听value属性
  let value = shallow ? rawValue : convert(rawValue)
  
  // 对get和set行为，进行监听
  const r = {
    __v_isRef: true,
    get value() {
      // 追踪
      track(r, TrackOpTypes.GET, 'value')
      return value
    },
    set value(newVal) {
      // toRow 为了避免传入的还是响应式数据结构
      if (hasChanged(toRaw(newVal), rawValue)) {
        // 保存  
        rawValue = newVal
        
        // 再次判是否需要转换  
        value = shallow ? newVal : convert(newVal)

        // 触发所追踪的effect  
        trigger(
          r,
          TriggerOpTypes.SET,
          'value',
          __DEV__ ? { newValue: newVal } : void 0
        )
      }
    }
  }
  return r
}
```



## effect

```typescript
export function effect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions = EMPTY_OBJ // 默认为空 更新组件的effect会用到
): ReactiveEffect<T> {
  // computed -> false
  if (isEffect(fn)) {
    fn = fn.raw
  }
  const effect = createReactiveEffect(fn, options)
  
  // 立即执行
  if (!options.lazy) {
    effect()
  }

  return effect
}
```



## createReactiveEffect

```typescript
function createReactiveEffect<T = any>(
  fn: (...args: any[]) => T,
  options: ReactiveEffectOptions
): ReactiveEffect<T> {
  const effect = function reactiveEffect(...args: unknown[]): unknown {

    if (!effect.active) {
      return options.scheduler ? undefined : fn(...args)
    }

    // effectStack为effect文件的全局属性，如果该effect已经在栈中等待执行，那么跳过 
    if (!effectStack.includes(effect)) {
      cleanup(effect) // 每次运行effect 都会清空对应的所有追踪 为了避免重复追踪
      try {
        // 允许追踪当前effect  
        enableTracking()
        // 入栈  
        effectStack.push(effect)
          
        // activeEffect为effect文件中的全局属性，表示当前正常执行的effect
        // 能够使响应式数据进行track追踪
        activeEffect = effect

        return fn(...args) // 返回执行的函数的结果
      } finally {
        // 出栈  
        effectStack.pop()

        // 恢复上一次effect信息，即本次允许追踪信息出栈 
        resetTracking()
        activeEffect = effectStack[effectStack.length - 1]
      }
    }
  } as ReactiveEffect

  effect.id = uid++ // 组件更新的时候 会根据id从小到大排序
  effect._isEffect = true
  effect.active = true
  effect.raw = fn
  effect.deps = []
  effect.options = options
  return effect
}
```



# 追踪过程（track）



# 触发过程（trigger）