# Webpack5实现CommonJS模块化的原理

Webpack 作为前端使用最广泛的打包工具（之一），它支持 `CommonJS`、`AMD` 和 `ES module` 模块化。本文将从源码分析 Webpack5 是如何帮助我们实现代码中支持 `CommonJS`，以此来深入学习其中的过程和原理。

## 准备工作

在此，先对源码的准备工作做简要说明：

1.src 目录下新建 js 文件夹，js 文件夹中新建 `math.js` 文件，使用 `CommonJS` 模块化导出两个简单的函数，代码如下：

```javascript
// src/js/math.js
const sum = (num1, num2) => {
  return num1 + num2
}

const mul = (num1, num2) => {
  return num1 * num2
}

module.exports = {
  sum,
  mul
}
```

2.src 目录下新建 `index.js` 文件，作为 webpack 打包入口。在 `index.js` 中，使用 `CommonJS` 方式引入 `math.js` 中的两个函数，代码如下：

```javascript
const { sum, mul } = require('./js/math.js')
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
  // 1.定义了一个 __webpack_modules__ 对象
  var __webpack_modules__ = {
    './src/js/math.js': module => {
      const sum = (num1, num2) => {
        return num1 + num2
      }
      const mul = (num1, num2) => {
        return num1 * num2
      }
      module.exports = {
        sum,
        mul
      }
    }
  }

  // 2.定义了一个 __webpack_module_cache__ 空对象
  var __webpack_module_cache__ = {}

  // 3.定义 __webpack_require__ 函数
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

  // 定义 __webpack_exports__ 空对象，但并未使用
  var __webpack_exports__ = {}

  // 4.一个立即执行函数，去除外层立即执行函数后真正开始执行的地方
  ;(() => {
    const { sum, mul } = __webpack_require__('./src/js/math.js')
    console.log(sum(10, 20))
    console.log(mul(10, 20))
  })()
})()
```

在打包出的代码中，它使用了一个立即执行函数作为包裹。其中主要的代码分为四个部分，上面已经在注释里简单的标明了。

## 详细分析

简单的浏览过 `bundle.js` 代码之后，我们对它的主要内容和结构已经有了一定的认识，下面我们来详细分析它执行的过程。

抛开最外层的立即执行函数不看，我们可以发现代码主要分为以下四个部分：

1. 定义了一个 `__webpack_modules__` 对象
2. 定义了一个 `__webpack_module_cache__`  空对象
3. 定义 `__webpack_require__ ` 函数
4. 一个立即执行函数，这里是开始执行的地方

我们首先来看一下第一和第二部分的代码：

```javascript
// 1.定义了一个 __webpack_modules__ 对象
  var __webpack_modules__ = {
    './src/js/math.js': module => {
      const sum = (num1, num2) => {
        return num1 + num2
      }
      const mul = (num1, num2) => {
        return num1 * num2
      }
      module.exports = {
        sum,
        mul
      }
    }
  }
```

第一部分代码，定义了一个 `__webpack_modules__` 对象，见名知意，它是 webpack 存放模块内容的一个对象。该对象的 key 为 `'./src/js/math.js'`，它是我们在 index.js 中引入模块的路径。该对象的 value 是一个函数，函数接收一个 `module` 参数，函数体内容和我们自己写的 `math.js` 的内容一样，定义了 sum 和 mul 两个函数，并且将它们放入一个对象中，赋值给参数 `module` 对象的 `export `属性上。

```javascript
var __webpack_module_cache__ = {}
```

第二部分代码，只定义了一个空对象，从名字我们可以猜测它是用来缓存的一个对象。

接下来我们看一下这一个立即执行函数部分的代码。

```javascript
// 4.一个立即执行函数，去除外层立即执行函数后真正开始执行的地方
  ;(() => {
    const { sum, mul } = __webpack_require__('./src/js/math.js')
    console.log(sum(10, 20))
    console.log(mul(10, 20))
  })()
```

```javascript
// 自己编写的 index.js 文件
const { sum, mul } = require('./js/math.js')
console.log(sum(10, 20))
console.log(mul(10, 20))
```

可以发现，它和我们自己编写的 `index.js` 中的代码基本一样，主要是将 `CommonJS` 语法中的 `require` 方法转换成了 webpack 自己实现的 `__webpack_require__` 方法。

这里使用了 `__webpack_require__` 方法，将我们导入函数的 math.js 模块的路径 `'./src/js/math.js'` 作为参数执行。下面我们来看一下第三部分的代码 `__webpack_require__` 中做了什么。

```javascript
function __webpack_require__(moduleId) {
    // 从立即执行函数的调用得出，moduleId 的值是'./src/js/math.js'，也就是我们引入模块的路径
    
    // 这一行代码在 __webpack_module_cache__ 缓存对象中，以 moduleId 模块路径为 key 去获取其对应的值，赋值给 cachedModele
    var cachedModule = __webpack_module_cache__[moduleId]
    
    // 如果cachedModule 有值，将 cachedModule 的 export 属性的值直接返回
    if (cachedModule !== undefined) {
      return cachedModule.exports
    }
    
    // 这里是一个连续赋值，将 {exports:{}} 对象既赋值给 缓存对象key为 moduleId（'./src/js/math.js'）的值，也赋值给 module 变量。
    // 因为{exports:{}}是一个对象，也就是说 module 和 __webpack_module_cache__[moduleId] 有了相同的引用，后面如果改了就一起变化
    var module = (__webpack_module_cache__[moduleId] = {
      exports: {}
    })
    
    // 这里使用了第一部分的__webpack_modules__对象，对象key为moduleId（'./src/js/math.js'）
    // 也就是说拿到了了 __webpack_modules__['./src/js/math.js'] 对应的那个函数并将后面三个参数传入并执行。这里进入了上面第一部分中的 __webpack_modules__[moduleId] 函数
    __webpack_modules__[moduleId](module, module.exports, __webpack_require__)
    
    // 执行完后返回
    return module.exports
  }
```

看接下来我们又回看 `__webpack_modules__[moduleId]`函数的执行：

```javascript
// 上面调用的时候传入了三个参数，而该函数执行时只写了 module 参数，因为本文案例比较简单，另外两个参数没有用到
module => {
    // 这里将我们在 math.js 中定义的两个函数，用对象包裹后赋值给参数传入的 module 变量的 export 属性
    const sum = (num1, num2) => {
        return num1 + num2
    }
    const mul = (num1, num2) => {
        return num1 * num2
    }
    module.exports = {
        sum,
        mul
    }
}
```

在这个函数执行完之后，我们再来回看 `__webpack_require__` 函数：

```javascript
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
```

此时已经豁然开朗，`module` 变量的 `export` 属性的值是 `{sum, mul}` 对象，sum 与 mul 是我们定义的函数。而上面的连续赋值又使得 `__webpack_module_cache__[moduleId]` 缓存对象中也有同样的内容，如果其他的地方再次使用将直接从缓存对象中获取。最后将 `module.export` 返回。

再回到一开始的立即执行函数中：

```javascript
;(() => {
    // 至此 __webpack_require__('./src/js/math.js') 执行完毕，返回的是非常普通的一个对象，对象中包含了 sum 与 mul 函数，将它们从返回值解构出来我们就可以愉快的使用了
    const { sum, mul } = __webpack_require__('./src/js/math.js')
    console.log(sum(10, 20))
    console.log(mul(10, 20))
  })()
```

## 总结

以上我们已经非常详细的分析了打包出的 `bundle.js`，做一个简单的总结：

webpack 将我们自己写的 `src/index.js` 以及 `src/js/math.js` 打包到了 `bundle.js` 一个文件中。

bundle.js 使用一个立即执行函数包裹所有的内容，其中主要内容分为四个部分：

- `__webpack_modules__`对象，它的 key 是模块路径，value 是一个函数，函数体是 key 对应的模块的中的内容

- `__webpack_module_cache__` 对象是用来缓存的，它将处理后的内容缓存起来，方便其他地方也引入模块来使用时直接 return 出去

- `__webpack_require__`函数是处理 我们自己写的 ConmonJS 的 `require` 的函数

- 立即执行函数，bundle.js真正开始执行的地方，里面的内容是 index.js 中的内容

  ---

  打包后的 `bundle.js` 执行过程：

  1. 从外层立即执行函数开始执行，真正执行内部代码的地方是第四部分的立即执行函数
  2. 将 模块路径`'./src/js/math.js'` 作为参数，执行 `__webpack_require__` 方法
  3. `__webpack_require__`函数中将 `{exports:{}}`连续赋值给 `__webpack_module_cache__` 缓存对象以及 `module` 变量，`module`和 `__webpack_module_cache__`引用相同，一变都变
  4. 将 `module` 变量作为参数（以及本案例中未用到的 `module.exports, __webpack_require__`）来执行 `__webpack_modules__`对象中对应的该模块对应的函数，函数执行完之后 `module.exports` 和 `__webpack_module_cache__` 缓存对象中同时都有了原来 CommonJS模块（`'math.js'`）中导出的内容
  5. 至此 `__webpack_require__` 执行完毕，内层立即执行函数中已经拿到了模块中的内容进行操作
  6. 如果其他地方也引用到了 `math.js` 模块，将直接从缓存对象中去拿

文章到这里就结束了，希望对你有所帮助。
