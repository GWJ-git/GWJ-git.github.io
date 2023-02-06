---
title: 'vue源码(2)'
date: 2022-12-23T20:39:10+08:00
draft: false
toc: true
images: null
categories:
  - 学习笔记
tags:
  - vue
  - '学习笔记'
slug: ''
---

#响应式
## initState方法与  响应式

`initState`方法 props=>methods=>data=>computed=>watch

```typescript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // 初始化Props 并添加响应式，代理到vm实例上，this.key获取props[key]
  if (opts.props) initProps(vm, opts.props)
  // 初始化methods 判断方法名是否已定义或方法名对应的是否为函数  vm.key=>methods[key]
  if (opts.methods) initMethods(vm, opts.methods)
  // 初始化data
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  // computed
  if (opts.computed) initComputed(vm, opts.computed)
  // watch
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

`initProps`方法，传入`vm`, `vm.$options.props`

```typescript
function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  // 缓存prop键，以便将来的prop更新可以使用Array而不是动态对象键枚举进行迭代。
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false) // 改变shouldObserve的值
  }
  for (const key in propsOptions) {
    keys.push(key)
    // 检查值的类型，先获取传入值，判断是否是boolean类型(没有传入值和默认值赋予false), 传入值为undefined 赋予默认值，验证类型[,不一致就报错]
    const value = validateProp(key, propsOptions, propsData, vm)
    
    // Object.defineProperty(props=> vm._props,key=>props键值,{...}) 
    //vm._props.key 给props中的属性添加响应式
    defineReactive(props, key, value)

    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.在 Vue.tended ()期间，静态props 已经在组件的原型上进行了代理。我们只需要在这里的实例化中定义代理props 。
                       
    if (!(key in vm)) {
      // Object.defineProperty(vm, key, {...})
      // vm.key => vm._props.key 即this.key
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

`initData (vm: Component)`方法

```typescript
function initData (vm: Component) {
  let data = vm.$options.data
  // 函数形式返回mergedInstanceDataFn.call(vm, vm)
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance 在实例上代理data
  const keys = Object.keys(data)
  let i = keys.length
  while (i--) {
    const key = keys[i]
      // Object.defineProperty(vm, key, {...})
      // vm.key => vm._props.key 即this.key
      proxy(vm, `_data`, key)
  }
  // observe data
  
  observe(data, true /* asRootData */)
}
```

`proxy(target: Object, sourceKey: string, key: string)`方法，在target对象上以key为键值，获取target.sourceKey.key的值

```typescript
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

### observe 获取观察者（创建或返回原有）

`observe (value: any, asRootData: ?boolean)`方法

```typescript
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  // 观察者
  let ob: Observer | void
  
  ob = new Observer(value)
    
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```


```typescript
/**
 * Attempt to create an observer instance for a value,
 * returns the new observer if successfully observed,
 * or the existing observer if the value already has one.
 * 尝试为一个值创建一个观察者实例，如果成功观察到，则返回新的观察者，如果该值已经有观察者，则返回现有的观察者。
 */
export function observe (value: any, asRootData: ?boolean): Observer | void {
  // 检查类型
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  // 做过响应式处理 直接返回现有观察者 ob=>value.__ob__
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 创建观察者
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

### Observer 观察者 

`Observer`类

```typescript
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    
    // 实例化dep
    // 添加响应式时会遍历属性，每个属性创建一个dep 
    this.dep = new Dep()
    this.vmCount = 0
    // 给value对象添加__ob__属性，值为this  Object.defineProperty
    def(value, '__ob__', this) 
    if (Array.isArray(value)) {
      // value 为数组
      if (hasProto) { // 判断__proto__ 能否使用
        // value.__proto__=Object.create(Array.prototype) 重写数组方法
        protoAugment(value, arrayMethods) 
      } else {
        // Object.defineProperty 重写数组方法
        copyAugment(value, arrayMethods, arrayKeys)
      }
      // observe数组的每一项
      this.observeArray(value)
    } else {
      // value为对象 对象响应式处理
      this.walk(value)
    }
  }

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   * 遍历所有属性并将它们转换为getter/setter。仅当值类型为Object时，才应调用此方法。
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

arrayMethods

```js
/*
 * not type checking this file because flow doesn't play well with
 * dynamically accessing methods on Array prototype
 */

import { def } from '../util/index'

// arrayMethods的原型是数组的原型，避免重写原型方法影响原有数组方法
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)

// 这7个方法会改变原数组
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    // 执行原方法
    const result = original.apply(this, args)
    // 获取观察者
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    // 通知更新
    ob.dep.notify()
    return result
  })
})
```



`defineReactive(obj: Object,key: string,val: any,customSetter?: ?Function,shallow?: boolean)` 定义对象上的响应式属性

```typescript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 每个响应式属性存在一个dep，类似消息中介，会存储相关watcher并通知更新
  const dep = new Dep()
  
  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  
  let childOb = !shallow && observe(val)
  
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}
```



`Dep`类

```typescript
let uid = 0

/**
 * A dep is an observable that can have multiple
 * directives subscribing to it.
 * dep是可观察的，可以有多个指令订阅它。
 */
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first 首先维护订阅者列表
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

// The current target watcher being evaluated. 当前目标观察者正在被计算
// This is globally unique because only one watcher
// can be evaluated at a time.  这是全局唯一的，因为一次只能计算一个观察者
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```



计算属性初始化

`initComputed(vm: Component, computed: Object)`

```typescript
function initComputed (vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = vm._computedWatchers = Object.create(null)

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here. 组件定义的计算属性已经在组件原型上定义。我们只需要在这里定义实例化时定义的计算属性。
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}
```











[ndz-vue源码](https://juejin.cn/post/7017758108827664391)


