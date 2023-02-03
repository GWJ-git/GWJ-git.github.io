---
title: 'vue源码阅读'
date: 2022-12-14T20:39:10+08:00
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
[todo] 看完后需要再梳理一遍思路
# vue2

new Vue实例时，先调用Vue的构造函数

Vue构造函数

```js
// src/core/instance/index.js
// 构造函数
function Vue (options) {
  // ...
  this._init(options)
}
initMixin(Vue)
// stateMixin(Vue)
// eventsMixin(Vue)
// lifecycleMixin(Vue)
// renderMixin(Vue)

export default Vue
```

improt Vue时，会调用`initMixin(Vue),stateMixin(Vue),eventsMixin(Vue),lifecycleMixin(Vue),renderMixin(Vue)`这些方法，给Vue的原型添加属性及方法

`initMixin`为`Vue`的原型添加`_init`方法

`stateMixin` 通过`Object.defineProperty()`劫持vue原型的`$data`和`$props`,为Vue的原型添加`$set` ,`$delete`和`$watch` 

`eventsMixin`为Vue的原型添加`$on` ,`$once`,`$off` ,`$emit`

`lifecycleMixin`为Vue的原型添加`_update`,`$forceUpdate`,`$destroy`

`renderMixin`为Vue的原型添加render相关方法及`$nextTick`,`_render`,



`initMixin`方法

```typescript
let uid = 0

export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    // 实例对象
    const vm: Component = this
    // uid
    vm._uid = uid++

    // a flag to avoid this being observed
    vm._isVue = true
    
    // 合并options  初始化vm.$options
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
   
    // vm._renderProxy
    
    // expose real self
    vm._self = vm
    // 初始化render，生命周期状态等及调用生命周期
    // 初始化生命周期状态 vm._isMounted vm._isDestroyed 等  
    initLifecycle(vm)
    // 初始化 vm._events，vm._hasHookEvent
    initEvents(vm)
    // 初始化 vm._vnode,vm._staticTrees ,vm.$slots,vm.$scopedSlots,vm.$createElement等
    initRender(vm)
    // 调用beforeCreate生命周期
    callHook(vm, 'beforeCreate')
    // 初始化Injection
    initInjections(vm) // resolve injections before data/props
      
    // 初始化 props=>methods=>data=>computed=>watch
    initState(vm)
    
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')


    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```



合并options方法处 `mergeOptions`

```js
if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
}
```



`initInternalComponent(vm, options)`

对于组件（`options._isComponent`为`true`）调用

`initInternalComponent(vm, options)`传入实例和构造函数的`options`

```typescript
export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  // 父传向子的props  propsData
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```

以`vm.constructor.options`为原型，在vm上挂载`$options`,并添加`父级实例(parent)、Vnode(_parentVnode)、Listeners(_parentListeners)，props(propsData)，_renderChildren，_componentTag`,option.render存在，添加`render`，`staticRenderFns`



`function mergeOptions (parent: Object,child: Object,vm?: Component): Object`

非组件情况下`mergeOptions` 传入父级`options`,子级，`options`及`vm`实例                                                                                              `resolveConstructorOptions`  递归返回 `vm.constructor.options`

`vm.constructor.super`存在，调用 `resolveConstructorOptions(vm.constructor.super)`，进行一些判断赋值

```typescript
export function resolveConstructorOptions (Ctor: Class<Component>) {
  let options = Ctor.options
  if (Ctor.super) {
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
    if (superOptions !== cachedSuperOptions) {
      // super option changed,
      // need to resolve new options.
      Ctor.superOptions = superOptions
      // check if there are any late-modified/attached options (#4976)
      const modifiedOptions = resolveModifiedOptions(Ctor)
      // update base extend options
      if (modifiedOptions) {
        extend(Ctor.extendOptions, modifiedOptions)
      }
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
  return options
}
```

` mergeOptions (parent: Object,child: Object,vm?: Component)`

parent=> vm.constructor.options  child=>options  

规范props,inject,Directive,合并extends，mixins的options

`option`上部分属性(components,directives,filters,_base),以`parent--(vm.constructor.options)`为原型

`option`上部分属性（el,props,template）,值为`child[key]--(options)`

```js
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }
  // 规范数组、对象形式的prop 均转换成对象形式(name:{type: null})
  normalizeProps(child, vm)
  // 规范inject 
  normalizeInject(child, vm)
  // 规范自定义指令
  normalizeDirectives(child)

  // Apply extends and mixins on the child options,
  // but only if it is a raw options object that isn't
  // the result of another mergeOptions call.
  // Only merged options has the _base property.
  if (!child._base) {
    if (child.extends) {
      parent = mergeOptions(parent, child.extends, vm)
    }
    if (child.mixins) {
      for (let i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm)
      }
    }
  }

  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```





`initState`方法 props=>methods=>data=>computed=>watch

```typescript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // 初始化Props
  if (opts.props) initProps(vm, opts.props)
  // 初始化methods 判断方法名是否已定义或方法名对应的是否为函数  vm.method
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
    //vm._props.key
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

`observe (value: any, asRootData: ?boolean)`方法

```typescript
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
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
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```



`Observer`类

```typescript
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
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
      // value为对象
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

