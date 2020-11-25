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



## 测试用例

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

在创建effect的时候，createReactiveEffect会设置activeEffect = effect，表示当前执行的effect，运行传递进effect的Function：

```ty
// Function:
() => {
  console.log(a.foo)
  console.log(rf.value)
}
```



**a**被设置成Proxy，可以监听赋值和取值等操作的行为，此处取值键为foo，查看内部getter。（需要注意的是**track**方法）

### getter：

```typescript
/*
  @target 目标对象
  @key 对象参数
  @receiver 自身Proxy
*/
function get(target: object, key: string | symbol, receiver: object) {
    // 特定Key相关s's
    if (key === ReactiveFlags.isReactive) {
      return !isReadonly
    } else if (key === ReactiveFlags.isReadonly) {
      return isReadonly
    } else if (
      key === ReactiveFlags.raw &&
      receiver ===
        (isReadonly
          ? (target as any).__v_readonly
          : (target as any).__v_reactive)
    ) {
      return target
    }

    // 数组相关
    const targetIsArray = isArray(target)
    if (targetIsArray && hasOwn(arrayInstrumentations, key)) {
      // 拦截 ['includes', 'indexOf', 'lastIndexOf'] 方法 每一个遍历的 返回一个methods
      return Reflect.get(arrssayInstrumentations, key, receiver)
    } 

    // 这个其实就相当于 new Proxy() 返回target[key]
    const res = Reflect.get(target, key, receiver)

    // 获取原型
    if ((isSymbol(key) && builtInSymbols.has(key)) || key === '__proto__') {
      return res
    }

    // 非readonly 初始创建的时候的值 进行追踪，这里的重点
    if (!isReadonly) {
      track(target, TrackOpTypes.GET, key)
    }

    // shallow 初始创建的时候的值
    if (shallow) {
      return res
    }

    // 转换
    if (isRef(res)) {
      // ref unwrapping, only for Objects, not for Arrays.
      return targetIsArray ? res : res.value
    }

    if (isObject(res)) {
      // Convert returned value into a proxy as well. we do the isObject check
      // here to avoid invalid value warning. Also need to lazy access readonly
      // and reactive here to avoid circular dependency.
      // 如果是对象 那么递归 res
      // 如果是链式调用 检查到__v_reactive 就跳过处理 track 也是会执行的
      // readonly 就 readonly 递归循环
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
```



### track：

```typescript
function track(target: object, type: TrackOpTypes, key: unknown) {
  if (!shouldTrack || activeEffect === undefined) {
    return
  }
  // WeakMap
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key) // value
  if (!dep) {
    depsMap.set(key, (dep = new Set()))
  }
  // cleanup(effect) 每次执行trigger effect的时候 都会删除dep
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
    }
  }
}
```

询问targetMap有没有该target（引用属性），没有则设置target为键，value为空的Map，depsMap。

查看value存在不存在key，不存在则创建一个空的Set，dep。

查看dep存在不存在effect，不存在则存放当前activeEffect，給当前activeEffect.deps.push(dep)。

**一个全局收集器收集target，key，activeEffect，由于追踪响应行为的是key，所以在key的层级存放activeEffect。**

```typescript
// 如果用对象来描述Map 用数组描述Set，那么以下就是当前测试变量a得出的结果
{ target: { key: [ activeEffect ] } }
```



# 触发过程（trigger）

### setter:

```typescript
function createSetter(shallow = false) {
  return function set(
    target: object, // 目标
    key: string | symbol, // Key
    value: unknown, // 设置的值
    receiver: object // receiver就是它的Proxy
  ): boolean {
    const oldValue = (target as any)[key]
    if (!shallow) { // shallow 浅
      value = toRaw(value)
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    // 如果没有hasOwn 则调用的是TriggerOptypes.Add 标记是set还是add
    const hadKey = hasOwn(target, key)

    // target直接赋值 这个操作不会再次触发setter
    const result = Reflect.set(target, key, value, receiver)
    
    // 如果target是原型链中的某个东西，就不要触发
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        // 触发器trigger
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        // 触发器trigger  
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
}
```

是不是看不懂为什么最后要target ===  toRaw(receiver) 才触发？

```typescript
// 来调试一下你就知道啦
const ob1 = reactive({ foo: 2 })
const ob2 = reactive({ foo: 2 })

Object.setPrototypeOf(ob1, ob2)
ob1.a = 99
```

ob1.a的第一次setter，跑到:

```typescript
 const result = Reflect.set(target, key, value, receiver)
```

这引用别人的一个例子：

![blog](https://res.psy-1.com/Fupesc8bpq0Zf7qW4kXX1YHe0M7y)



运行在浏览器上：

![blog-change](https://res.psy-1.com/FvN1wTA7YmQoBw9F4HqjELL-6wbu)



再来看看Reflect.set的文档：

![img](https://res.psy-1.com/FoZDmo3zpR9ik3KM_yRmo7EZ0-Jt)

![熊猫紧张](https://res.psy-1.com/Fr9pcXuMBigc_ofuRmebvi-XsUx_)啊这... 虽然我不懂，但是这不影响我去分析它：

第一次触发Reflect.set的过程会调用到原型中的set，等于触发parentProxy的

set(原本的target, Reflect.set传入的key, Reflect.set传入的value，Reflect.set传入的receiver)

第二次的Reflect.set，如果我把这行代码去掉，那么childProxy将不会被设置成功，只有这行代码才是真正设置childProxy的，但是对parentProxy没有任何影响，继续分析。

代码在set中，只返回true，或者在第二次Reflect中直接return true，也是不会有任何设置的成功，所以和返回的值无关。

第一次和第二的Reflect只有target是不一样的。



![wocao](https://res.psy-1.com/FimJsXIrCCMb7NhhduJbEBUPD2tF)那我第一次的Reflect.set随便丢一个target是不是会设置成功？？

![img](https://res.psy-1.com/Fj8d73bIc5gOieJIArlVG6AJUjgz)

果真成功了....



### trigger

```typescript
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  const depsMap = targetMap.get(target)
  if (!depsMap) {
    return
  }

  const effects = new Set<ReactiveEffect>() // 收集当前target要触发的effect
  const computedRunners = new Set<ReactiveEffect>()

  // 添加要触发的effect
  const add = (effectsToAdd: Set<ReactiveEffect> | undefined) => {
    if (effectsToAdd) {
      effectsToAdd.forEach(effect => {
        // activeEffect 和 shouldTrack 都是effect.ts 的全局值
        if (effect !== activeEffect || !shouldTrack) {
          if (effect.options.computed) {
            computedRunners.add(effect) // computed相关
          } else {
            effects.add(effect)
          }
        } else {
          // 不能是运行中的effect 或者 当前effect不能追踪
          // 因为 effect(() => { reactiveData.foo++ })
          // 首先track，赋值操作trigger，再触发track，死循环  
        }
      })
    }
  }

  if (type === TriggerOpTypes.CLEAR) {
    // 使用Map Set WeakMap WeakSet类型的响应式数据使用clear方法，才会触发
    // 测试用例在reactivity/__test__/collections/Map.sepc.ts 
    // 全局搜索 'should observe size mutations'
    depsMap.forEach(add)
  } else if (key === 'length' && isArray(target)) {
    // taregt是数组的情况下 我们修改其length长度，是一种修改操作
    depsMap.forEach((dep, key) => {
      // 缩短长度的操作等于删除元素 触发effect
      // 追踪过length的effect 都需要触发
      if (key === 'length' || key >= (newValue as number)) {
        add(dep)
      }
    })
  } else {
    // 使用void 0 代替 undefinded 因为undefinded又可能被重写  
    if (key !== void 0) {
      add(depsMap.get(key))
    }
      
    // iteration key on ADD | DELETE | Map.SET
    const isAddOrDelete =
      type === TriggerOpTypes.ADD ||
      (type === TriggerOpTypes.DELETE && !isArray(target))
    if (
      isAddOrDelete ||
      (type === TriggerOpTypes.SET && target instanceof Map)
    ) {
      add(depsMap.get(isArray(target) ? 'length' : ITERATE_KEY))
    }
    if (isAddOrDelete && target instanceof Map) {
      add(depsMap.get(MAP_KEY_ITERATE_KEY))
    }
  }

  const run = (effect: ReactiveEffect) => {
    if (__DEV__ && effect.options.onTrigger) {
      effect.options.onTrigger({
        effect,
        target,
        key,
        type,
        newValue,
        oldValue,
        oldTarget
      })
    }
    if (effect.options.scheduler) { // 渲染processComponent的时候在setupRenderEffect方法中，instance.update = effect(()=>{}, prodEffectOptions)
      effect.options.scheduler(effect)
    } else {
      effect()
    }
  }

  // effect.options.computed 类型的优先级比较高
  computedRunners.forEach(run)
  effects.forEach(run)
}
```



# 总结

![img](file:///C:\Users\MyPC\AppData\Roaming\Tencent\QQ\Temp\ZVZEBH@X$1Z]CIYS}8DSU[X.gif)不总结了，就这么简单。