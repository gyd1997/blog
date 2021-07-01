# Webpack5实现ESModule模块化的原理

Webpack 作为前端使用最广泛的打包工具（之一），它支持 `CommonJS`、`AMD` 和 `ES module` 模块化。本文将从源码分析 Webpack5 是如何帮助我们实现代码中支持 `ES Module`，以此来深入学习其中的过程和原理。

## 准备工作

在此，先对源码的准备工作做简要说明：

1.src 目录下新建 js 文件夹，js 文件夹中新建 `math.js` 文件，使用 `ES Module` 模块化导出两个简单的函数，代码如下：

```javascript
// src/js/math.js
export const sum = (num1, num2) => {
  return num1 + num2
}

export const mul = (num1, num2) => {
  return num1 * num2
}
```

2.src 目录下新建 `index.js` 文件，作为 webpack 打包入口。在 `index.js` 中，使用 `ES Module` 方式引入 `math.js` 中的两个函数，代码如下：

```javascript
// src/index.js 入口
import { sum, mul } from './js/math'
console.log(sum(10, 20))
console.log(mul(10, 20))
```

3.在 `webpack.config.js` 中，将 `mode` 设置为 `'development'`，将 `devtool` 设置为 `source-map`，以便于我们打包后输出的源码更容易阅读和分析。`webpack.config.js` 代码如下：

```javascript
const path = require('path')
const { CleanWebpackPlugin } = require('clean-webpack-plugin')
const HtmlWebpackPlugin = require('html-webpack-plugin')

module.exports = {
  mode: 'development',
  entry: './src/index.js',
  devtool: 'source-map',
  output: {
    filename: 'js/bundle.js',
    path: path.resolve(__dirname, './build')
  },
  plugins: [
    new CleanWebpackPlugin(),
    new HtmlWebpackPlugin({
      title: 'webpack'
    })
  ]
}
```

代码目录的其他部分和打包过程不做详细展示，下面我们将开始分析打包后的 `bundle.js` 代码。

## bundle.js 的简要说明

在代码打包成功后，`bundle.js` 会在里面插入生成较多的注释，我将提前删掉这些注释来减少干扰。

我们先来简要的看一下打包出来的代码的大致情况，简单的做一个认识，方便我们下文详细分析。代码如下：

```javascript
// 最外层使用立即执行函数包裹，分析中可以忽略
;(() => {
  // 严格模式
  'use strict'
  // 1.定义了一个 __webpack_modules__ 对象
  var __webpack_modules__ = {
    './src/js/math.js': (
      __unused_webpack_module,
      __webpack_exports__,
      __webpack_require__
    ) => {
      __webpack_require__.r(__webpack_exports__)
      __webpack_require__.d(__webpack_exports__, {
        sum: () => sum,
        mul: () => mul
      })
      const sum = (num1, num2) => {
        return num1 + num2
      }

      const mul = (num1, num2) => {
        return num1 * num2
      }
    }
  }

  // 2.定义了一个缓存对象
  var __webpack_module_cache__ = {}

  // 3.定义了 webpack 自己的 require 函数
  function __webpack_require__(moduleId) {
    var cachedModule = __webpack_module_cache__[moduleId]
    if (cachedModule !== undefined) {
      return cachedModule.exports
    }
    var module = (__webpack_module_cache__[moduleId] = {
      exports: {}
    })
    __webpack_modules__[moduleId](module, module.exports, __webpack_require__)
    return module.exports
  }

  // 4.一个立即执行函数
  ;(() => {
    __webpack_require__.d = (exports, definition) => {
      for (var key in definition) {
        if (
          __webpack_require__.o(definition, key) &&
          !__webpack_require__.o(exports, key)
        ) {
          Object.defineProperty(exports, key, {
            enumerable: true,
            get: definition[key]
          })
        }
      }
    }
  })()

  // 5.一个立即执行函数
  ;(() => {
    __webpack_require__.o = (obj, prop) =>
      Object.prototype.hasOwnProperty.call(obj, prop)
  })()

  // 6.一个立即执行函数
  ;(() => {
    __webpack_require__.r = exports => {
      if (typeof Symbol !== 'undefined' && Symbol.toStringTag) {
        Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' })
      }
      Object.defineProperty(exports, '__esModule', { value: true })
    }
  })()

  // 7.定义__webpack_exports__变量，该变量被初始化一个空对象
  var __webpack_exports__ = {}

  // 8.一个立即执行函数
  ;(() => {
    __webpack_require__.r(__webpack_exports__)
    var _js_math_js__WEBPACK_IMPORTED_MODULE_0__ =
      __webpack_require__('./src/js/math.js')
    console.log((0, _js_math_js__WEBPACK_IMPORTED_MODULE_0__.sum)(10, 20))
    console.log((0, _js_math_js__WEBPACK_IMPORTED_MODULE_0__.mul)(10, 20))
  })()
})()

```

如果你看过我的第一篇 CommonJS 分析，你会发现 ES Module 打包出的内容要稍多一些，不过不要紧，我们下面一起慢慢分析。

## 详细分析

简单的浏览过 `bundle.js` 代码之后，我们对它的主要内容和结构已经有了一定的认识，下面我们来详细分析它执行的过程。

先来简单梳理一下，抛开外层立即执行函数不看，里面主要有 8 个部分，其中包括：

1. 定义了三个变量（`__webpack_modules__`，`__webpack_module_cache__`，`__webpack_exports__`）
2. 定义了一个函数（`__webpack_require__`）
3. 四个立即执行函数

首先我们来看定义的三个变量：

```javascript
// 1.定义了一个 __webpack_modules__ 对象（内容先不用详细看，后续用到时有解释）
var __webpack_modules__ = {
  './src/js/math.js': (
    __unused_webpack_module,
    __webpack_exports__,
    __webpack_require__
  ) => {
    __webpack_require__.r(__webpack_exports__)
    __webpack_require__.d(__webpack_exports__, {
      sum: () => sum,
      mul: () => mul
    })
    const sum = (num1, num2) => {
      return num1 + num2
    }
    const mul = (num1, num2) => {
      return num1 * num2
    }
  }
}
```

第一个变量：`__webpack_modules__`，它的 key 是 `math.js` 的路径，value 是一个函数。函数中先执行了两个方法，然后下面是原来 `math.js` 中导出的两个函数。

```javascript
// 2.定义了一个缓存对象
var __webpack_module_cache__ = {}
```

第二个变量：`__webpack_module_cache__`，是一个缓存对象，初始化为空对象。

```javascript
// 7.定义__webpack_exports__变量，该变量被初始化一个空对象
var __webpack_exports__ = {}
```

第三个变量：`__webpack_exports__`，初始化为空对象，目前作用不明。

接下来我们来看标号第4/5/6这三个立即执行函数：

```javascript
// 4.一个立即执行函数
;(() => {
  __webpack_require__.d = (exports, definition) => {
    for (var key in definition) {
      if (
        __webpack_require__.o(definition, key) &&
        !__webpack_require__.o(exports, key)
      ) {
        Object.defineProperty(exports, key, {
          enumerable: true,
          get: definition[key]
        })
      }
    }
  }
})()
```

该部分的作用是给 `__webpack_require__` 函数的 `d` 属性赋值为一个函数，函数内容我们暂且不看。

> 在 JavaScript 中，函数是一种特殊的对象，所以也可以在它上面再添加属性和属性对应的值。

```javascript
// 5.一个立即执行函数
;(() => {
  __webpack_require__.o = (obj, prop) =>
    Object.prototype.hasOwnProperty.call(obj, prop)
})()
```

该部分的作用是给 `__webpack_require__` 函数的 `o` 属性赋值为一个函数，函数接收 obj 和 prop 两个参数，函数的作用是判断 prop 是否是 object 自身的属性。

```javascript
// 6.一个立即执行函数
;(() => {
  __webpack_require__.r = exports => {
    if (typeof Symbol !== 'undefined' && Symbol.toStringTag) {
      Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' })
    }
    Object.defineProperty(exports, '__esModule', { value: true })
  }
})()
```

该部分的作用是给 `__webpack_require__` 函数的 `r` 属性赋值为一个函数，函数有一个 export 参数，该函数的作用是给 exports 对象加上一个标记，标记 exports 对象是 `ES Module` 模块，方便以后识别。

简单的从语法上分析一下：

> Symbol.toStringTag
>
> **`Symbol.toStringTag`** 是一个内置 symbol，它通常作为对象的属性键使用，对应的属性值应该为字符串类型，这个字符串用来表示该对象的自定义类型标签，通常只有内置的 [`Object.prototype.toString()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/toString) 方法会去读取这个标签并把它包含在自己的返回值里。——MDN

- 首先判断支不支持 ES6 的 Symbol，如果支持则通过 `Symbol.toStringTag` 的方式标记 `ES Module`
- 如果不支持 Symbol，则给 exports 对象增加一个键值对，key 是 '__esModule'，值是一个对象

这里使用了两种方式，效果上都是标明 exports 是一个 `ES Module` 模块。



以上是一些比较易读的定义性内容，下面来看真正意义上开始执行的地方：

```javascript
// 8.一个立即执行函数
;(() => {
  // 给 __webpack_exports__ 对象打上标记，标记它为 ES Module
  // 注意在本案例中，只定义 __webpack_exports__ 变量并打上标记，其他地方并没有用到
  __webpack_require__.r(__webpack_exports__)
    
  // 以下将我们自己写的index.js的内容做了一些转换，然后执行
  // 这个变量名太长了，无所谓，它的意思是 ES Module模块化的 math.js 的对象
  // 将 math.js 的路径作为参数，执行 __webpack_require__ 函数，接收返回值给这个超长名字的变量
  var _js_math_js__WEBPACK_IMPORTED_MODULE_0__ =
    __webpack_require__('./src/js/math.js')
  
  console.log((0, _js_math_js__WEBPACK_IMPORTED_MODULE_0__.sum)(10, 20))
  console.log((0, _js_math_js__WEBPACK_IMPORTED_MODULE_0__.mul)(10, 20))
})()
```

> `(0, _js_math_js__WEBPACK_IMPORTED_MODULE_0__.sum)(10, 20)`这是一个 `逗号表达式` 的语法，其实它的作用就相当于：`_js_math_js__WEBPACK_IMPORTED_MODULE_0__.sum(10, 20)`，很简单，有兴趣可以自己查阅以下 逗号表达式 相关的内容。

接下来们来看 `var _js_math_js__WEBPACK_IMPORTED_MODULE_0__ =
    __webpack_require__('./src/js/math.js')` 这一句的执行过程（`__webpack_require__`内容其实和 CommonJS 那一篇里面一模一样，下面大概的分析一下，如果觉得不够详细可以看我的CommonJS 那一篇文章。）

```javascript
function __webpack_require__(moduleId) {
    // moduleId 的值是'./src/js/math.js'
    
    // 这一行代码在 __webpack_module_cache__ 缓存对象中，以 moduleId 为 key 去获取其对应的值，赋值给 cachedModele
    var cachedModule = __webpack_module_cache__[moduleId]
    
    // 如果cachedModule 有值，将 cachedModule 的 export 属性的值直接返回
    if (cachedModule !== undefined) {
      return cachedModule.exports
    }
    
    // 这里是一个连续赋值，将 {exports:{}} 对象既赋值给 "缓存对象 key为 moduleId（'./src/js/math.js'）的值"，也赋值给 module 变量。
    // 因为{exports:{}}是一个对象，也就是说 module 和 __webpack_module_cache__[moduleId] 有了相同的引用，后面如果改了就一起变化
    // 此时，__webpack_module_cache__['./src/js/math.js'] 与 module 的值都是{exports: {}}
    var module = (__webpack_module_cache__[moduleId] = {
      exports: {}
    })
    
    // 这里使用了第一部分的__webpack_modules__对象，对象的 key 为 moduleId，也就是'./src/js/math.js'
    // 也就是说拿到了 __webpack_modules__['./src/js/math.js'] 对应的那个函数，并将后面三个参数传入并执行。这里进入了上面第一部分中的 __webpack_modules__[moduleId] 函数
    __webpack_modules__[moduleId](module, module.exports, __webpack_require__)
    
    // 执行完后返回
    return module.exports
}
```

接下来看 `__webpack_modules__[moduleId]`函数的执行：

```javascript
(__unused_webpack_module, __webpack_exports__, __webpack_require__) => {
// 函数接收了三个参数（CommonJS中只用到了第一个参数，ESModule里都用到了）
// 第一个参数是 module，第二个参数是 module.exports，第三个参数是 __webpack_require__ 函数
    
  // 执行__webpack_require__的r方法，标记为 ES Module
  // 注意：之前在 8号立即执行函数 里，是给 __webpack_exports__ 标记，这次是给 module.exports 标记，这俩是两个变量
  __webpack_require__.r(__webpack_exports__)
    
  // 执行__webpack_require__的d方法，传入两个参数
  // 第一个参数的值是 module.exports
  // 第二个参数是一个对象，key 是导出的函数名，value是一个箭头函数，返回值是 key 对应的函数
  __webpack_require__.d(__webpack_exports__, {
    sum: () => sum,
    mul: () => mul
  })
    
  // 这是我们在 math.js 中导出的两个函数
  const sum = (num1, num2) => {
    return num1 + num2
  }

  const mul = (num1, num2) => {
    return num1 * num2
  }
}
```

接下来再看 d 函数的执行过程：

```javascript
// __webpack_require__ 的 d 函数
// 参数 exports 实际上是 module.exports
// 参数 definition 实际上是 { sum: () => sum, mul: () => mul}
(exports, definition) => {
  // 遍历 definition 对象
  for (var key in definition) {
    // __webpack_require__.o 的作用是判断 prop 是不是 obj 自身的属性
    // 这里判断 key 在 definition 上有，而在 exports 上没有的时候做一个代理操作
    // 在访问 exports[key] 的时候，实际上是去拿 definition[key]
    if (
      __webpack_require__.o(definition, key) &&
      !__webpack_require__.o(exports, key)
    ) {
      Object.defineProperty(exports, key, {
        enumerable: true,
        get: definition[key]
      })
    }
  }
}
```

这部分对 module.exports 做了代理，代理之后访问 module.export[sum] 时候将拿的是 definition 中真正返回的 sum 函数。

至此，`__webpack_require__` 函数执行完毕，它返回的 module.exports 被 `_js_math_js__WEBPACK_IMPORTED_MODULE_0__` 变量接收，该变量上变有了导出的 sum 与 mul 函数，就可以执行后续的操作了。

## 总结

和 Webpack5 处理 CommonJS 模块化的方式相比，处理 ES Module 的过程略显复杂，不过整体上比较相似，尤其是 `__webpack_require__`函数、缓存对象是一模一样的。

### 再来捋一遍大致的过程

1.  定义了三个变量（`__webpack_modules__`，`__webpack_module_cache__`，`__webpack_exports__`）
2. 定义了一个函数（`__webpack_require__`）,并通过三个立即执行函数给该函数对象添加了 d / o / r  方法。
3.   将 模块路径`'./src/js/math.js'` 作为参数，执行 `__webpack_require__` 方法
4. `__webpack_require__`函数中将 `{exports:{}}`连续赋值给 `__webpack_module_cache__` 缓存对象以及 `module` 变量，`module`和 `__webpack_module_cache__`引用相同，一变都变
5. 将 `module` 、`module.exports`、`__webpack_require__`作为参数来执行 `__webpack_modules__`对象中对应的该模块对应的函数，函数执行完之后 `module.exports` 和 `__webpack_module_cache__` 缓存对象中同时都有了原来 ES Module 模块（`'math.js'`）中导出的内容
6. 至此 `__webpack_require__` 执行完毕，内层立即执行函数中已经拿到了模块中的内容进行操作
7. 如果其他地方也引用到了 `math.js` 模块，将直接从缓存对象中去拿

### 和CommonJS处理方式的对比

相同点：

- 大体上的思路和方式都是一样的（感觉说了个废话，大家可以自己品一品）

不同的：

- 对 ES Module 专门增加了标记
- 对模块导出的内容做了一层代理

### 一些可能的难点

- `Object.defineProperty()`
- 函数是特殊的对象，可以像普通对象一样增加属性
- 逗号表达式
- Symbol.toStringTag

对以上内容如果不太清楚的同学，可以自行查阅相关的资料学习。

---

本文到这里就结束了，希望对你有所帮助，有疑问或指正可以在评论区回复。

## 交个朋友 || 加个群

wx号：`gydeee`

