---
title: 'vue源码(1)'
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
[toc]

# vue2 初始化

src目录

```
├─compiler                        // 模板编译相关文件，将 template 编译为 render 函数
│  ├─codegen                     // 把AST(抽象语法树)转换为Render函数
│  ├─directives                  // 生成Render函数之前需要处理的东西
│  └─parser                      // 模板编译成AST
├─core       // Vue核心代码，包括了组件、全局API、Vue实例化、响应式原理、vdom(虚拟DOM)、等等。
│  ├─components  
│  ├─global-api    
│  ├─instance        
│  ├─observer                    // 响应式核心
│  ├─util    
│  └─vdom                        // 虚拟DOM相关的代码
├─platforms
├─server                          // 服务端渲染相关内容（ssr）
├─sfc                             // 转换单文件组件（*.vue）
└─shared                          // 全局共享的方法和常量
```

## 构造函数

### 调试查找构造函数

new Vue实例时，先调用Vue的构造函数

Vue构造函数

```js
// src/core/instance/index.js
// 构造函数
function Vue (options) {
  // ...
  this._init(options)
}
initMixin(Vue)  // 初始化混入
stateMixin(Vue)  // state混入
eventsMixin(Vue)  // events混入
lifecycleMixin(Vue) // 生命周期混入
renderMixin(Vue)  //render混入

export default Vue
```

### 入口文件查找构造函数

先前自己查看源码时是通过调试直接找到构造函数再一步一步看的，后面梳理过程中参考一些大佬博客，通过打包入口文件一层一层查看，学到了一些新内容

```js
// package.json
"scripts": {
    "dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev --sourcemap"
    ......
}
```

打包工具：`rollup`, 配置文件：`scripts/config.js`，环境变量中`TARGET`值为：

`scripts/config.js`中`target` 为 `web-full-dev`

```js
// Runtime+compiler development build (Browser)
  'web-full-dev': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.js'),
    format: 'umd',
    env: 'development',
    alias: { he: './entity-decoder' },
    banner
  }

// 最终导出config opts为'web-full-dev'
const config = {
    input: opts.entry,
    external: opts.external,
    plugins: [
      flow(),
      alias(Object.assign({}, aliases, opts.alias))
    ].concat(opts.plugins || []),
    output: {
      file: opts.dest,
      format: opts.format,
      banner: opts.banner,
      name: opts.moduleName || 'Vue'
    },
    onwarn: (msg, warn) => {
      if (!/Circular/.test(msg)) {
        warn(msg)
      }
    }
}
```

`resolve`方法

```js
const aliases = require('./alias')
// resolve方法底层依赖的是path.resolve()方法
const resolve = p => {
  const base = p.split('/')[0]
  // aliases[base] => src/platforms/web
  if (aliases[base]) {
    return path.resolve(aliases[base], p.slice(base.length + 1))
  } else {
    return path.resolve(__dirname, '../', p)
  }
}
```

打包入口文件为`src/platforms/web/entry-runtime-with-compiler.js`

入口文件

```typescript
import config from 'core/config'
import { warn, cached } from 'core/util/index'
import { mark, measure } from 'core/util/perf'

import Vue from './runtime/index'
import { query } from './util/index'
import { compileToFunctions } from './compiler/index'
import { shouldDecodeNewlines, shouldDecodeNewlinesForHref } from './util/compat'


// 获取id对应的元素
const idToTemplate = cached(id => {
  // 获取元素
  const el = query(id)
  return el && el.innerHTML
})

// $mount
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  const options = this.$options
  // resolve template/el and convert to render function 
  // 解析 template/el 并转换为渲染函数
  // 不存在render选项
  if (!options.render) {
    let template = options.template
    // template存在
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') { // 如果是选择器
          template = idToTemplate(template)
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        return this
      }
    } else if (el) {  // el 存在
      template = getOuterHTML(el)
    }
    if (template) {
      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns
    }
  }
  return mount.call(this, el, hydrating)
}
// 同时设置了 el、template、render的时候，优先级的判断为：render > template > el
/**
 * Get outerHTML of elements, taking care
 * of SVG elements in IE as well.
 */
function getOuterHTML (el: Element): string {
  if (el.outerHTML) {
    return el.outerHTML
  } else {
    const container = document.createElement('div')
    container.appendChild(el.cloneNode(true))
    return container.innerHTML
  }
}
Vue.compile = compileToFunctions

export default Vue

```

入口文件中的`Vue`来自`./runtime/index`

```typescript
// src/platforms/web/runtime/index.js
/* @flow */

import Vue from 'core/index'
import config from 'core/config'
import { extend, noop } from 'shared/util'
import { mountComponent } from 'core/instance/lifecycle'
import { devtools, inBrowser } from 'core/util/index'

import {
  query,
  mustUseProp,
  isReservedTag,
  isReservedAttr,
  getTagNamespace,
  isUnknownElement
} from 'web/util/index'

import { patch } from './patch'
import platformDirectives from './directives/index'
import platformComponents from './components/index'

// install platform specific utils 安装平台特有的utils
Vue.config.mustUseProp = mustUseProp
Vue.config.isReservedTag = isReservedTag
Vue.config.isReservedAttr = isReservedAttr
Vue.config.getTagNamespace = getTagNamespace
Vue.config.isUnknownElement = isUnknownElement

// install platform runtime directives & components  安装平台运行时指令和组件
extend(Vue.options.directives, platformDirectives)
extend(Vue.options.components, platformComponents)

// install platform patch function  补丁 
// 主要作用：虚拟dom 转化为真实的dom（vdom => dom)
Vue.prototype.__patch__ = inBrowser ? patch : noop

// public mount method
// 实现了 $mount 方法：调用mountComponent()方法
// $mount最终目的：把虚拟dom 转化为真实的dom，并且追加到宿主元素中去（vdom => dom => append）
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}

export default Vue
```

`Vue`来自`core/index`

```typescript
// src/core/index.js
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
import { FunctionalRenderContext } from 'core/vdom/create-functional-component'
// 初始化全局api，set/delete/nextTick/observable等
initGlobalAPI(Vue)

Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})

// expose FunctionalRenderContext for ssr runtime helper installation
Object.defineProperty(Vue, 'FunctionalRenderContext', {
  value: FunctionalRenderContext
})

Vue.version = '__VERSION__'

export default Vue
```

主要是`initGlobalAPI`,初始化全局api

`initGlobalAPI`方法

```typescript
import config from '../config'
import { initUse } from './use'
import { initMixin } from './mixin'
import { initExtend } from './extend'
import { initAssetRegisters } from './assets'
import { set, del } from '../observer/index'
import { ASSET_TYPES } from 'shared/constants'
import builtInComponents from '../components/index'
import { observe } from 'core/observer/index'

import {
  warn,
  extend,
  nextTick,
  mergeOptions,
  defineReactive
} from '../util/index'

export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }
  //vue的set/delete/nextTick方法
  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  // 2.6 explicit observable API
  Vue.observable = <T>(obj: T): T => {
    observe(obj)
    return obj
  }

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  initUse(Vue)  // vue.use
  initMixin(Vue)  // Vue.mixin
  initExtend(Vue)  // Vue.extend
  initAssetRegisters(Vue)
}
```

`core/index`的`Vue`来自`./instance/index`，即构造函数所在



improt Vue时，会调用`initMixin(Vue),stateMixin(Vue),eventsMixin(Vue),lifecycleMixin(Vue),renderMixin(Vue)`这些方法，给Vue的原型添加属性及方法

`initMixin`为`Vue`的原型添加`_init`方法

`stateMixin` 通过`Object.defineProperty()`劫持vue原型的`$data`和`$props`,为Vue的原型添加`$set` ,`$delete`和`$watch` 

`eventsMixin`为Vue的原型添加`$on` ,`$once`,`$off` ,`$emit`

`lifecycleMixin`为Vue的原型添加`_update`,`$forceUpdate`,`$destroy`

`renderMixin`为Vue的原型添加render相关方法及`$nextTick`,`_render`,

## initMixin

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
    
    // 合并options(传入options和构造器本身options)  初始化vm.$options
    if (options && options._isComponent) {
      // 子组件
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      //  优化内部组件实例化，因为动态选项合并非常缓慢，并且没有一个内部组件选项需要特殊处理。
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
    // 组件关系属性的初始化 定义：$root、$parent、$children、$refs
    initLifecycle(vm)
    
    // 初始化 vm._events，vm._hasHookEvent
    // 存在父监听，添加到该实例上
    initEvents(vm)
    
    // 初始化 vm._vnode,vm._staticTrees ,vm.$slots,vm.$scopedSlots,vm.$createElement等
    // 初始化render渲染所需的slots、渲染函数等。其实就两件事1、插槽的处理、2、$createElement 也就是 render 函数中的 h 的声明
    initRender(vm)
    
    // 调用beforeCreate生命周期
    callHook(vm, 'beforeCreate')
    // 初始化Injection 先注入上级提供的Provide数据
    initInjections(vm) // resolve injections before data/props
      
    // 初始化 props=>methods=>data=>computed=>watch，响应式处理
    initState(vm)
    
    // 初始化Provide 
    initProvide(vm) // resolve provide after data/props
    
    //created初始化完成   
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

