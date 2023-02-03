---
title: '[译]执行上下文'
date: 2022-09-30T20:39:10+08:00
draft: false
toc: true
images: null
categories:
  - 学习笔记
tags:
  - js
  - 执行上下文
  - '学习笔记'
slug: ''
---

[原文](https://blog.bitsrc.io/understanding-execution-context-and-execution-stack-in-javascript-1c9ea8642dd0)

#执行上下文

简单地说，执行上下文是可以计算和执行 Javascript 代码的一个环境的抽象概念。任何代码在 JavaScript 中运行时，它都是在执行上下文中运行的。

##执行上下文类型

三种

- **Global Execution Context（全局执行上下文）**

  这是默认的或基本的执行上下文。不在任何函数中的代码位于全局执行上下文中。它执行两个操作: 创建一个全局对象，它是一个`window`对象(在浏览器中) ，并将`this`值设置为等于全局对象。一个程序中只能有一个全局执行上下文

- **Functional Execution Context(函数执行上下文)** 

  每次调用一个函数时，都会为该函数创建一个全新的执行上下文。每个函数都有自己的执行上下文，但它是在执行或调用（when the function is invoked or called）该函数时创建的。可以有任意数量的函数执行上下文。无论何时创建一个新的执行上下文，它都会按照已定义的顺序经历一系列步骤，我将在本文后面讨论这些步骤。

- **Eval Function Execution Context（Eval函数执行上下文）**

  在 eval 函数中执行的代码也有自己的执行上下文，但是 eval 通常不被 JavaScript 开发人员使用

##执行栈（Execution Stack）

执行堆栈，在其他编程语言中也称为“调用堆栈”，是一个具有 LIFO (Last in, First out后进先出)结构的堆栈，用于存储在代码执行期间创建的所有执行上下文。

当 JavaScript 引擎第一次遇到脚本时，它会创建一个全局执行上下文并将其推送到当前执行堆栈。每当引擎发现一个函数调用时，它就为该函数创建一个新的执行上下文，并将其推送到栈的顶部。

引擎执行其执行上下文位于堆栈顶部的函数。当这个函数完成时，它的执行栈从栈中弹出，控件到达当前堆栈中它下面的上下文。

```js
let a = 'Hello World!';
function first() {
  console.log('Inside first function');
  second();
  console.log('Again inside first function');
}
function second() {
  console.log('Inside second function');
}
first();
console.log('Inside Global Execution Context');
```

![1_ACtBy8CIepVTOSYcVwZ34Q](https://gitee.com/gong-weijie/pic/raw/master/pic/1_ACtBy8CIepVTOSYcVwZ34Q.webp)

global =>first(执行中调用second)=>second(执行完弹出)=>first(执行完弹出)=> global（执行console）



## 创建执行上下文

执行上下文创建分为两个阶段:

 创建阶段（Creation Phase）和执行阶段（Execution Phase）。

### 创建阶段（Creation Phase）

执行上下文是在创建阶段创建的。在创建阶段发生的事情如下

1. **LexicalEnvironment（词法环境）** 组件被创建.
2. **VariableEnvironment（变量环境） ** 组件被创建.

执行上下文可以在概念上表示如下

```js
ExecutionContext = {
  LexicalEnvironment = <ref. to LexicalEnvironment in memory>,
  VariableEnvironment = <ref. to VariableEnvironment in  memory>,
}
```

#### 词法环境Lexical Environment

ES6官方文档将 Lexical Environment 定义为

> *A* *Lexical Environment* *is a specification type used to define the association of* *Identifiers* *to specific variables and functions based upon the lexical nesting structure of ECMAScript code. A Lexical Environment consists of an Environment Record and a possibly null reference to an* *outer* *Lexical Environment.*
>
> 词法环境(**Lexical Environment**)是一种规范类型，用于根据 ECMAScript 代码的词法嵌套结构定义标识符与特定变量和函数的关联。词法环境由一个环境记录和一个可能为空的外部词法环境引用组成。

简而言之，词法环境是一种保存标识符-变量映射的结构。(这里的标识符是指变量/函数的名称，而变量是对实际对象(包括函数对象和数组对象)或原始值的引用)。

考虑以下代码

```js
var a = 20;
var b = 40;
function foo() {
  console.log('bar');
}
```

```js
lexicalEnvironment = {
  a: 20,
  b: 40,
  foo: <ref. to foo function>
}
```

每个词法环境有三个组成部分:

1. 环境记录（**Environment Record**）
2. 外部环境的引用（**Reference to the outer environment**）
3. `this`绑定（**This binding**）

##### 环境记录（Environment Record）

环境记录是变量和函数声明存储在词法环境中的位置。

环境记录也有两种类型:

- 声明环境记录(**Declarative environment record**)

  函数代码的词法环境包含一个声明型环境记录。存储变量和函数声明

- 对象环境记录（**Object environment record**）

  全局代码的词法环境包含一个对象型环境记录。除了变量和函数声明之外，对象环境记录还存储一个全局绑定对象(浏览器中的`window`对象)。因此，对于每个绑定对象的属性(在浏览器中，它包含由浏览器提供给窗口对象的属性和方法) ，在记录中创建一个新条目。

注意: 对于函数代码，环境记录还包含一个参数(**arguments**)对象，该对象包含传递给函数的索引和参数之间的映射，以及传递给函数的参数的长度(数目)。例如，下面函数的参数对象如下所示:

```js
function foo(a, b) {
  var c = a + b;
}
foo(2, 3);
// argument object
Arguments: {0: 2, 1: 3, length: 2},
```

##### 外部环境的引用(Reference to the Outer Environment)

对外部环境的引用意味着它可以访问外部词法环境。这意味着如果在当前的词法环境中找不到变量，JavaScript 引擎可以在外部环境中查找变量。

##### `this`绑定（This Binding）

在此组件中，确定或设置 `this` 的值。

在全局执行上下文中，`this` 的值指向全局对象。(在浏览器中，`this`指的是 Window 对象)。

在函数执行上下文中，`this` 的值取决于如何调用函数。如果由对象引用调用，则 `this` 的值设置为该对象，否则，`this` 的值设置为全局对象或未定义(在严格模式下)。例如:

```js
const person = {
  name: 'peter',
  birthYear: 1994,
  calcAge: function() {
    console.log(2018 - this.birthYear);
  }
}
person.calcAge(); 
// 'this' refers to 'person', because 'calcAge' was called with //'person' object reference
const calculateAge = person.calcAge;
calculateAge();
// 'this' refers to the global window object, because no object reference was given
```

抽象地说，伪代码中的词法环境类似于这样

```js
GlobalExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier    go here
    }
    outer: <null>,
    this: <global object>
  }
}
FunctionExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
    }
    outer: <Global or outer function environment reference>,
    this: <depends on how function is called>
  }
}
```

#### 变量环境（Variable Environment）

它也是一个词法环境，它的环境记录（**EnvironmentRecord** ）保存了在执行上下文中由变量声明（**VariableStatements**）创建的绑定

变量环境也是一个词法环境，因此它具有上面定义的词法环境的所有属性和组件。

在 ES6中，词法环境（**LexicalEnvironment** ）组件和 变量环境（**VariableEnvironment** ）组件的一个不同之处在于，前者用于存储函数声明和变量(`let` 和 `const`)绑定，而后者仅用于存储变量(`var`)绑定。

### 执行阶段

在这个阶段，所有这些变量的分配都完成了，代码也最终执行了。

### 例子

```js
let a = 20;
const b = 30;
var c;
function multiply(e, f) {
 var g = 20;
 return e * f * g;
}
c = multiply(20, 30);
```

 执行上述代码时，JavaScript 引擎创建一个全局执行上下文来执行全局代码。所以全局执行上下文在创建阶段是这样的:              

```js
GlobalExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",  // 对象环境记录
      // Identifier bindings go here
      a: < uninitialized >,
      b: < uninitialized >,
      multiply: < func >
    }
    outer: <null>, //外部环境引用
    ThisBinding: <Global Object>  // this绑定
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      c: undefined,
    }
    outer: <null>,
    ThisBinding: <Global Object>
  }
}
```

在执行阶段，变量分配完成。因此，在执行阶段，全局执行上下文将类似于下面的内容。

```js
GlobalExectionContext = {
LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      a: 20,
      b: 30,
      multiply: < func >
    }
    outer: <null>,
    ThisBinding: <Global Object>
  },
VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      c: undefined,
    }
    outer: <null>,
    ThisBinding: <Global Object>
  }
}
```

当遇见函数调用`multiply(20, 30)`，创建一个新的函数执行上下文来执行函数代码。

函数执行上下文创建阶段：

```js
FunctionExectionContext = {
LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative", // 声明环境记录
      // Identifier bindings go here
      Arguments: {0: 20, 1: 30, length: 2},
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>,
  },
VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
      g: undefined
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>
  }
}
```

在此之后，执行上下文将经历执行阶段，这意味着完成对函数内部变量的赋值。

函数执行上下文执行阶段：

```js
FunctionExectionContext = {
LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
      Arguments: {0: 20, 1: 30, length: 2},
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>,
  },
VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Declarative",
      // Identifier bindings go here
      g: 20
    },
    outer: <GlobalLexicalEnvironment>,
    ThisBinding: <Global Object or undefined>
  }
}
```

函数完成后，返回值存储在`c`中。因此全局执行上下文更新。之后，全局代码完成程序结束。

注意: 您可能已经注意到，`let` 和 `const` 定义的变量在创建阶段没有任何与它们相关联的值，但是 var 定义的变量被设置为未定义的。

这是因为在创建阶段，代码被扫描以查找变量和函数声明，函数声明被完整的存储在环境中，变量最初被设置为`undefined`(`var`)或保持未初始化（`let`和`const`）

这就是为什么可以在声明`var`定义的变量之前访问它们（`undefined`），但是在声明`let`和`const`变量之前访问它们会出现引用错误的原因。

这就是我们说的提升（`hoisting`）

注意：在执行阶段，如果 JavaScript 引擎无法在源代码中声明的实际位置找到 let 变量的值，那么它将为其赋予未定义的值。

原文结束


执行 Javascript 代码的一个环境的抽象概念，

js引擎第一次遇见脚本时会创建一个全局执行上下文并压入执行栈。执行中遇见函数会创建函数执行上下文并压入栈中，执行完会弹出。

执行上下文的创建分为创建和执行两个阶段。

创建阶段会创建词法环境和变量环境两个组件，两个组件都由环境记录，外部环境引用和this绑定三部分组成，其中词法环境的环境记录存储函数声明和变量`let，const`,而变量环境仅存储变量`var`。

函数执行上下文词法环境的环境记录是声明型，存储变量和函数声明，和一个arguments对象。全局执行上下文的词法环境包含一个对象型的环境记录，除了存储变量和函数声明外，还有一个全局绑定对象（window）。

外部环境引用使执行上下文可以访问外部词法环境。this绑定，全局执行上下文的`this`指向全局对象window对象，而函数执行上下文的`this`取决于如何调用函数。

然后是执行阶段，分配变量，执行代码。