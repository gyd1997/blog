# Webpack5解析ESModule和CommonJS模块的互相导入

在前两篇文章中，已经详细分析了 Webpack5 是如何实现 ESModule 和 CommonJS 的模块化，本文将从源码分析 Webpack5 如何实现 ESModule 和 CommonJS 模块的互相导入。由于源码内容紧密的关系到前两篇文章的内容，所以阅读本文前必须先熟悉前两篇文章。本文中出现的一些 webpack 的函数由于前两篇文章已详细分析过，本文不再详细分析其执行过程。

## 准备工作

1. 在 src/js 文件夹下创建 format.js 文件，其中使用 CommonJS 的方式导出一个函数，代码如下：

   ```javascript
   const priceFormat = price => {
     return '￥' + price
   }
   
   module.exports = {
     priceFormat
   }
   ```

2. 在 src/js 文件夹下创建 math.js 文件，其中使用 ESModule 方式导出一个函数，代码如下：

   ```javascript
   export const sum = (num1, num2) => {
     return num1 + num2
   }
   ```

3. 创建 src/index.js 文件，作为主入口。在 index.js 中，使用 ESModule 方式导入 format.js，使用 CommonJS 方式导入 math.js，做简单的执行打印操作。代码如下：

   ```javascript
   const math = require('./js/math.js')
   import format from './js/format'
   console.log(math.sum(10, 20))
   console.log(format.pricaFormat(200))
   ```

## bundle.js 的简要说明

将代码打包后获得 bundle.js 文件，首先对它做一个简单的认识，了解其大致结构，而后再分析其执行过程。

> bundle.js 最外层是一个立即执行函数，此处已删除
>
> 打包默认包含大量注释，此处已删除

代码如下：

```javascript
// 1.包含引入模块的变量，key 为模块路径，value 为模块内容
var __webpack_modules__ = {
  './src/js/format.js': module => {
    const priceFormat = price => {
      return '￥' + price
    }

    module.exports = {
      priceFormat
    }
  },

  './src/js/math.js': (
    __unused_webpack_module,
    __webpack_exports__,
    __webpack_require__
  ) => {
    'use strict'
    __webpack_require__.r(__webpack_exports__)
    __webpack_require__.d(__webpack_exports__, {
      sum: () => sum
    })
    const sum = (num1, num2) => {
      return num1 + num2
    }
  }
}

// 2.用于缓存的对象
var __webpack_module_cache__ = {}

// 3.webpack实现的加载模块内容的函数
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

// 4.__webpack_require__挂载了一个 n 方法
;(() => {
  __webpack_require__.n = module => {
    var getter =
      module && module.__esModule ? () => module['default'] : () => module
    __webpack_require__.d(getter, { a: getter })
    return getter
  }
})()

// 5.__webpack_require__挂载了一个 d 方法，功能是对 exports 做一层代理
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

// 6.__webpack_require__上挂载了一个 o 方法，功能是返回 obj 上是否有没有自己的 prop 属性
;(() => {
  __webpack_require__.o = (obj, prop) =>
    Object.prototype.hasOwnProperty.call(obj, prop)
})()

// 7.__webpack_require__上挂载了一个 r 方法，功能是对 ESModule 的模块做一个标记
;(() => {
  __webpack_require__.r = exports => {
    if (typeof Symbol !== 'undefined' && Symbol.toStringTag) {
      Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' })
    }
    Object.defineProperty(exports, '__esModule', { value: true })
  }
})()

// 8.__webpack_exports__变量初始化为一个空对象
var __webpack_exports__ = {}

// 9.真正开始执行的地方
;(() => {
  'use strict'
  // 9.1
  __webpack_require__.r(__webpack_exports__)
  // 9.2
  var _js_format__WEBPACK_IMPORTED_MODULE_0__ =
    __webpack_require__('./src/js/format.js')
  // 9.3
  var _js_format__WEBPACK_IMPORTED_MODULE_0___default = __webpack_require__.n(
    _js_format__WEBPACK_IMPORTED_MODULE_0__
  )
  // 9.4
  const math = __webpack_require__('./src/js/math.js')
  // 9.5
  console.log(math.sum(10, 20))
  // 9.6
  console.log(
    _js_format__WEBPACK_IMPORTED_MODULE_0___default().priceFormat(200)
  )
})()
```

bundle.js 的内容一共包括 9 个小模块，基本上所有的小模块在之前的两篇文章中都出现过并且一模一样，下面将分析大概的执行流程，对每个函数的功能不做详细讲解。

## 详细分析

```javascript
__webpack_require__.r(__webpack_exports__)
```

第一句将 `__webpack_exports__`标记为 ESModule，但是后面没有用到这个变量，此处忽略这一句。

```javascript
var _js_format__WEBPACK_IMPORTED_MODULE_0__ =  __webpack_require__('./src/js/format.js')
```

第二句以 format.js 路径作为参数去调用 `__webpack_require__` 方法，返回值赋给 `_js_format__WEBPACK_IMPORTED_MODULE_0__` 变量。

```javascript
var _js_format__WEBPACK_IMPORTED_MODULE_0___default = __webpack_require__.n(
    _js_format__WEBPACK_IMPORTED_MODULE_0__
  )
```

第三句以 `_js_format__WEBPACK_IMPORTED_MODULE_0__` 作为参数，调用 `__webpack_require__.n`方法，将返回值赋值给 `_js_format__WEBPACK_IMPORTED_MODULE_0___default`，来看 `n` 方法的内容：

```javascript
;(() => {
  __webpack_require__.n = module => {
    var getter = module && module.__esModule ? () => module['default'] : () => module
    __webpack_require__.d(getter, { a: getter })
    return getter
  }
})()
```

n 方法通过传入的 module 判断是否有 ESModule 的标记，如果有则 `getter` 值为  **一个返回值为 module['default']**  的箭头函数，否则 `getter` 值为 **一个返回值为 module ** 的箭头函数。此处由于 `format.js` 是一个 CommonJS 模块，所以 `getter` 为  **一个返回值为 module ** 的箭头函数。

下一句又将 `getter` 与 `{a: getter}` 传入 d 方法进行调用：

```javascript
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

d 方法是我们的老朋友了，它给 `getter` 加上了一个 a 属性并做了代理，如果访问 `getter.a` 将代理到 `{a: getter}`，很遗憾后面的代码中并没有用到 `getter.a`，这条线的分析到此为止。

在本实例中相当于做了一些没有用的操作，实际上 `_js_format__WEBPACK_IMPORTED_MODULE_0___default` 是 返回值为 `getter(就是module)` 的函数。

```javascript
console.log(
    // 执行了这个函数，得到了 format 模块的内容，从里面拿出 priceFormat 调用
    _js_format__WEBPACK_IMPORTED_MODULE_0___default().priceFormat(200)
  )
```

到此 使用 ESModule 加载 CommonJS 模块 的内容就结束了，下面来看 CommonJS 加载 ESModule。

```javascript
const math = __webpack_require__('./src/js/math.js')
```

将 math.js 路径作为参数调用 `__webpack_require__`，`__webpack_require__`中又去执行 `__webpack_modules__['./src/js/math.js']`方法。

```javascript
'./src/js/math.js': (
    __unused_webpack_module,
    __webpack_exports__,
    __webpack_require__
  ) => {
    'use strict'
    // r 方法，标记 ESModule
    __webpack_require__.r(__webpack_exports__)
    // d 方法做了一层代理
    __webpack_require__.d(__webpack_exports__, {
      sum: () => sum
    })
    const sum = (num1, num2) => {
      return num1 + num2
    }
  }
```

执行过后 math 对象的 sum 属性的值为 上面的 sum 函数，接着就可以继续执行了：

```javascript
console.log(math.sum(10, 20))
```

## 总结

使用 ESModule 方式导入 CommonJS 模块：

1. 在 `__webpack_modules__`对象中，math.js 还是 以webpack解析 CommonJS 模块的形式存在。
2. 在其加载过程中通过 `__webpack_require__.n` 方法返回了一个箭头函数，使用时候需要多一次调用才能拿到真正的内容。
3. 对 CommonJS 模块的内容也做了一次代理，但是本示例中没有用到通过代理访问。
4. 整个过程类似于 webpack 解析 CommonJS 的过程

使用 CommonJS 方式导入 ESModule 模块：

1. 在 `__webpack_modules__`对象中，format.js 还是 以webpack解析 ESModule 模块的形式存在。
2. 执行过程中，使用 r 方法标记了模块是 ESModule，使用 d 方法做了一次代理。
3. 加载过程也类似于 Webpack 解析ESModule 的过程

纵观全部内容，我们可以发现一些异同点：

- 无论是ESModule还是CommonJS，他们的模块内容都存放在`__webpack_modules__`对象中。一个模块对应一个键值对，key 为 模块路径， value 为模块内容的函数（ESModule在其中又做了处理）。
- 模块都在 `__webpack_module_cache__` 中做了缓存
- 两种模块化使用的是同一个 `__webpack_require__`函数，都用 `__webpack_require__` 函数对象上挂载的方法做了一些不同的处理。

本文到这里就结束了，相信在阅读了这三篇文章之后你一定对 webpack5 解析 ESModule / CommonJS 模块的过程有了深入的理解和认识。