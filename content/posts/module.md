---
title: '模块化学习'
date: 2022-10-28T20:39:10+08:00
draft: false
toc: true
images: null
categories:
  - 学习笔记
tags:
  - js
  - 模块化
  - '学习笔记'
slug: ''
---

## 模块化

### 早期

#### 普通函数

```js
function fn1(){
  //...
}
function fn2(){
  //...
}
function fn3() {
  fn1()
  fn2()
}
```

模块成员之间看不出直接关系,后期维护困难，无法解决命名冲突问题

#### 命名空间

```js
var myModule = {
  name: "isboyjc",
  getName: function (){
    console.log(this.name)
  }
}

// 使用
myModule.getName()
```

内部状态可以被外部更改

#### 立即执行函数（IIFE）

```js
// module.js文件
(function(window) {
  let data = 'www.baidu.com'
  //操作数据的函数
  function foo() {
    //用于暴露有函数
    console.log(`foo() ${data}`)
  }
  function bar() {
    //用于暴露有函数
    console.log(`bar() ${data}`)
    otherFun() //内部调用
  }
  function otherFun() {
    //内部私有的函数
    console.log('otherFun()')
  }
  //暴露行为
  window.myModule = { foo, bar } //ES6写法
})(window)
----------------
// otherModule.js模块文件
var otherModule = (function(){
  return {
    a: 1,
    b: 2
  }
})()

// myModule.js模块文件 - 依赖 otherModule 模块
var myModule = (function(other) {
  var name = 'isboyjc'
  
  function getName() {
    console.log(name)
    console.log(other.a, other.b)
  }
  
  return { getName } 
})(otherModule)
```

#### 依赖注入

依赖注册器

```js
let injector = {
  dependencies: {}, // 保存依赖
  register: function(key, value) {
    this.dependencies[key] = value;
  }, // 注册依赖
  // 绑定依赖
  resolve: function(deps, func, scope) {
    var args = [];
    for(var i = 0; i < deps.length, d = deps[i]; i++) {
      if(this.dependencies[d]) {
        // 存在此依赖
        args.push(this.dependencies[d]);
      } else {
        // 不存在
        throw new Error('不存在依赖：' + d);
      }
    }
    return function() {
      func.apply(scope || {}, args.concat(Array.prototype.slice.call(arguments, 0)));
    }   
  }
}
```

使用

```js
// 添加
injector.register('fnA', fnA)
injector.register('fnB', fnB)

// 注入
(injector.resolve(['fnA', 'fnB'], function(fnA, fnB){
  let a = fnA()
  let b = fnB()
  console.log(a, b)
}))()
```

### 模块化规范

#### commonjs

每个文件就是一个模块，有自己的作用域。在一个文件里面定义的变量、函数、类，都是私有的，对其他文件不可见,外部想要调用，必须使用 `module.exports` 主动暴露，而在另一个文件中引用则直接使用 `require(path)` 即可

`CommonJS` 规范适用于服务端(`nodejs`),同步加载

特点：

- 所有代码都运行在模块作用域，不会污染全局作用域。
- 模块可以多次加载，但是只会在第一次加载时运行一次，然后运行结果就被缓存了，以后再加载，就直接读取缓存结果。要想让模块再次运行，必须清除缓存。
- 模块加载的顺序，按照其在代码中出现的顺序。

**加载某个模块，其实是加载该模块的module.exports属性**。

```js
// example.js
var x = 5;
var addX = function (value) {
  return value + x;
};
module.exports.x = x;
module.exports.addX = addX;
---------------------
var example = require('./example.js');//如果参数字符串以“./”开头，则表示加载的是一个位于相对路径
console.log(example.x); // 5
console.log(example.addX(1)); // 6
console.log(example.x); // 5
```

**CommonJS模块的加载机制是，输入的是被输出的值的拷贝**

#### AMD

`require.js`依赖前置

异步加载模块，一般用于浏览器

```js
define(id?: String, dependencies?: String[], factory: Function|Object)
```

- `id` 即模块的名字，字符串，可选
- `dependencies` 指定了所要依赖的模块列表，它是一个数组，也是可选的参数，每个依赖的模块的输出将作为参数一次传入 `factory` 中。如果没有指定 `dependencies`，那么它的默认值是 `["require", "exports", "module"]`
- `factory` 包裹了模块的具体实现，可为函数或对象，如果是函数，返回值就是模块的输出接口或者值

```js
// 定义依赖 myModule，该模块依赖 JQ 模块
define('myModule', ['jquery'], function($) {
  // $ 是 jquery 模块的输出
  $('body').text('isboyjc')
})

// 引入依赖
require(['myModule'], function(myModule) {
  // todo...
})
```



#### CMD

`Sea.js` 依赖就近

CMD规范专门用于浏览器端，模块的加载是异步的，模块使用时才会加载执行。整合了`CommonJS`和`AMD`规范的特点

```js
define(function(require, exports, module) {
  var a = require('./a')
  a.doSomething()
  
  // 依赖就近原则：依赖就近书写，什么时候用到什么时候引入
  var b = require('./b')
  b.doSomething()
})
```

```js
define(function(require, exports, module) {
  // 同步引入
  var a = require('./a')
  
  // 异步引入
  require.async('./b', function (b) {
  })
  
  // 条件引入
  if (status) {
      var c = requie('./c')
  }
  
  // 暴露模块
  exports.aaa = 'hahaha'
})
```

对于依赖的模块，`AMD` 是提前执行，`CMD` 是延迟执行

#### UMD

Universal Module Definition

通用模块定义

```js
((root, factory) => {
  if (typeof define === 'function' && define.amd) {
    // AMD
    define(factory);
  } else if (typeof exports === 'object') {
    // CommonJS
    module.exports = factory();
  } else if (typeof define === 'function' && define.cmd){
		// CMD
    define(function(require, exports, module) {
      module.exports = factory()
    })
  } else {
    // 都不是
    root.umdModule = factory();
  }
})(this, () => {
  console.log('我是UMD')
  // todo...
});
```

#### ES module

```js
// 模块定义 add.js
export function add(a, b) {
  return a + b;
}

// 模块使用 main.js
import { add } from "./add.js";
console.log(add(1, 2)); // 3
```



### 参考文章

[模块化：isboyjc](https://juejin.cn/post/7007946894605287432#heading-5)

[模块化：浪里行舟](