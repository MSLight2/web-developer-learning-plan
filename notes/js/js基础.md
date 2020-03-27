#### 模拟实现call，apply，bind
[ECMAScript5.1中文版](http://yanhaijing.com/es5/#324)
```js
// apply，call共同注意点：
// 在非严格模式下指定this值为null或undefined时，会自动替换为全局对象，原始值会被包装
// 返回值：使用调用者提供的 this 值和参数调用该函数的返回值。若该方法没有返回值，则返回 undefined。

// ------------------------apply实现---------------------------
/**apply规范
  1、如果 IsCallable(func) 是 false, 则抛出一个 TypeError 异常 .
  2、如果 argArray 是 null 或 undefined, 则
    返回提供 thisArg 作为 this 值并以空参数列表调用 func 的 [[Call]] 内部方法的结果。
  3、如果 Type(argArray) 不是 Object, 则抛出一个 TypeError 异常 .
  4、令 len 为以 "length" 作为参数调用 argArray 的 [[Get]] 内部方法的结果。
  5、令 n 为 ToUint32(len).
  6、令 argList 为一个空列表 .
  7、令 index 为 0.
  8、只要 index < n 就重复
    8.1、令 indexName 为 ToString(index).
    8.2、令 nextArg 为以 indexName 作为参数调用 argArray 的 [[Get]] 内部方法的结果。
    8.3、将 nextArg 作为最后一个元素插入到 argList 里。
    8.4、设定 index 为 index + 1.
  9、提供 thisArg 作为 this 值并以 argList 作为参数列表，调用 func 的 [[Call]] 内部方法，返回结果。
    apply 方法的 length 属性是 2。

  注：在外面传入的 thisArg 值会修改并成为 this 值。thisArg 是 undefined 或 null 时它会被替换成全局对象，所有其他值会被应用 ToObject 并将结果作为 this 值，这是第三版引入的更改。
*/
Function.prototype.applyFun = function (thisArg, argArray) {
  // 规范1
  if (typeof this !== 'function') throw new Error('TypeError: ' + this + 'must be a function.')
  // 规范2
  if (argArray == null) argArray = []
  // 规范3
  if (argArray !== new Object(argArray)) throw new Error('TypeError: CreateListFromArrayLike called on non-object.')
  // 规范 注
  thisArg = thisArg ? Object(thisArg) : window

  // 唯一key，以防在属性重名
  var fnKey = '_' + Date.now()
  thisArg[fnKey] = this
  // 执行函数有三个方案
  // 方案一: new Function形式执行函数
  // 方案二: es6：实现方式 -> var result = thisArg[fnKey](...argArray)
  // 方案三: eval执行函数
  var strCode = generateString(argArray.length)
  var result = (new Function(strCode))(thisArg, fnKey, argArray)
  delete thisArg[fnKey]

  return result
}

// 参考自：https://juejin.im/post/5bf6c79bf265da6142738b29
function generateString(argArraylenth) {
  var code = 'return arguments[0][arguments[1]]('
    for(var i = 0; i < argArraylenth; i++){
      if(i > 0) code += ','
      code += 'arguments[2][' + i + ']'
    }
    code += ')'
    return code
}
// -------------------------call实现---------------------------
/**call规范
  1、如果 IsCallable(func) 是 false, 则抛出一个 TypeError 异常。
  2、令 argList 为一个空列表。
  3、如果调用这个方法的参数多余一个，则从 arg1 开始以从左到右的顺序将每个参数插入为 argList 的最后一个元素。
  4、提供 thisArg 作为 this 值并以 argList 作为参数列表，调用 func 的 [[Call]] 内部方法，返回结果。
    call 方法的 length 属性是 1
*/
Function.prototype.callFun = function (thisArg) {
  if (typeof this !== 'function') throw new Error('TypeError: ' + this + 'must be a function.')
  var argList = []
  thisArg = thisArg ? Object(thisArg) : window

  // 理论上不使用push更好，因为push内部还会有层循环
  // 可以这样写：argList[i] = arguments[i + 1]
  var len = arguments.length
  for (var i = 0; i < len; i++) {
    argList.push(arguments[i + 1])
  }

  var fnKey = '_' + Date.now()
  thisArg[fnKey] = this
  var strCode = generateString(argList.length)
  var result = (new Function(strCode))(thisArg, fnKey, argList)
  delete thisArg[fnKey]

  return result
}
// 通过apply实现call
Function.prototype.callFun2 = function (thisArg) {
  var argList = []
  var len = arguments.length
  for (var i = 0; i < len; i++) {
    argList[i] = arguments[i + 1]
  }
  var result = this.apply(thisArg, argList)
  return result
}


// --------------------------bind实现--------------------------
// bind简单实现
Function.prototype.bindFun = function (thisArg) {
  var target = this
  if (typeof this !== 'function') throw new Error('TypeError: ' + this + 'must be a function.')
  var slice = Array.prototype.slice
  var args = slice.call(arguments, 1)

  var bound = function () {
    // 简单实现
    var funArgs = args.concat(slice.apply(arguments))
    return target.apply(thisArg, funArgs)
  }

  return bound
}
// 更完整的bind
// bind函数有个特点：
// 绑定函数自动适应于使用 new 操作符去构造一个由目标函数创建的新实例。
// 当一个绑定函数是用来构建一个值的，原来提供的 this 就会被忽略。
// 不过提供的参数列表仍然会插入到构造函数调用时的参数列表之前。
Function.prototype.bindFun2 = function (thisArg) {
  var target = this
  if (typeof this !== 'function') {
    throw new Error('Function.prototype.bind - what is trying to be bound is not callable')
  }
  var slice = Array.prototype.slice
  var args = slice.call(arguments, 1)

  var bound = function () {
    var funArgs = args.concat(slice.apply(arguments))
    if (this instanceof bound) {
      // 使用new时会进入这个判断，这忽略传入的thisArg
      var result = target.apply(this, funArgs)
      if (Object(result) === result) {
        return result
      }
      return this
    } else {
      return target.apply(thisArg, funArgs)
    }
  }

  // 因为使用new 操作符之后，this会绑定到新创建的对象，所以要更正this指向。
  var Empty = function Empty() {}
  if (target.prototype) {
    Empty.prototype = target.prototype
    bound.prototype = new Empty()
    Empty.prototype = null
  }
  return bound
}

// Polyfill
Function.prototype.bind = Function.prototype.bind || function bindPolyfill () { ... }
```
> PS: 以上`bind`版本的实现还没实现`bound`的`name`和`length`属性。
> 可以使用`Object.defineProperties`实现或参照`es5-shim`[源码](https://github.com/es-shims/es5-shim/blob/master/es5-shim.js#L201-L335)

#### new实现
```js
/**
  1.令 ref 为解释执行 NewExpression 的结果 .
  2.令 constructor 为 GetValue(ref).
  3.如果 Type(constructor) is not Object ，抛出一个 TypeError 异常 .
  4.如果 constructor 没有实现 [[Construct]] 内置方法 ，抛出一个 TypeError 异常 .
  5.返回调用 constructor 的 [[Construct]] 内置方法的结果 , 传入按无参数传入参数列表 ( 就是一个空的参数列表 ).
    产生式 MemberExpression : new MemberExpression Arguments 按照下面的过程执行 :
    5.1、令 ref 为解释执行 MemberExpression 的结果 .
    5.2、令 constructor 为 GetValue(ref).
    5.3、令 argList 为解释执行 Arguments 的结果 , 产生一个由参数值构成的内部列表类型 (11.2.4).
    5.4、如果 Type(constructor) is not Object ，抛出一个 TypeError 异常 .
    5.5、如果 constructor 没有实现 [[Construct]] 内置方法，抛出一个 TypeError 异常 .
    5.6、返回以 argList 为参数调用 constructor 的 [[Construct]] 内置方法的结果。

  MDN: new 关键字会进行如下操作
  1、创建一个空的简单JavaScript对象（即{}）；
  2、链接该对象（即设置该对象的构造函数）到另一个对象 ；
  3、将步骤1新创建的对象作为this的上下文 ；
  4、如果该函数没有返回对象，则返回this。
 */
 function newOperator () {
   var ref = {}
   var ctor = arguments[0]
   if (typeof ctor !== 'function') {
     throw new Error ('TypeError: Type(constructor) is not Object')
   }
   var argList = Array.prototype.slice.call(arguments, 1)
   ref.__proto__ = ctor.prototype
   var result = ctor.apply(ref, argList)
   if (
     result &&
     result !== null &&
     (typeof result === 'function' || typeof result === 'object')
   ) {
     return result
   }
   return ref
 }
```
#### 多维数组转换一维数组
- 使用`concat`和`apply`结合
```js
var arr = [1,[2,3],4,[5,6]]
console.log([].concat.apply([], arr)) // [1, 2, 3, 4, 5, 6]
```
> 缺点：超过二维数组不行

- 使用`concat`和`toString`结合
```js
console.log([].concat(arr.toString().split(','))) // ["1", "2", "3", "4", "5", "6"]
```
> 缺点：每一项值都是字符串，之后还需处理。

- 使用es6`flat`
```js
var arr = [1,[2,3],4,[5,6]]
console.log(arr.flat()) // [1,2,3,4,5,6]
```
> 缺点：兼容性（现在基本不存在了[手动狗头]）

- 模拟实现flat
```js
/**
 * @param arr 展开的数组
 * @param depth 展开的深度
 */
function _flatten (arr, depth) {
  var target = new Array()
  _FlattenIntoArray(target, arr, depth, 0)
  return target
}
function _FlattenIntoArray (target, arrArg, depth, start) {
  var targetIndex = start || 0
  var sourceIndex = 0
  var sourceLen = arrArg.length
  var IsArray = Array.isArray || function isArray (val) {
    return Object.prototype.toString.call(val) === '[object Array]'
  }
  while (sourceIndex < sourceLen) {
    var key = String(sourceIndex)
    if (key in arrArg) {
      var val = arrArg[key]
      var shouldFlatten = false
      // 传递大于0的值，这说明需要深度展开
      if (depth > 0) {
        shouldFlatten = IsArray(val)
      }
      if (shouldFlatten) {
        targetIndex = _FlattenIntoArray(target, val, depth - 1, targetIndex)
      } else {
        target[targetIndex] = val
        targetIndex += 1
      }
    }
    sourceIndex += 1
  }
  return targetIndex
}

// 使用
var arr = [1,[2,3],4,[5,6]]
console.log(_flatten(arr, 1)) // [1,2,3,4,5,6]
arr = [1,[2,3],4,[5,6, [7,8,[9,10]]]]
console.log(_flatten(arr, 3)) // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

#### 柯里化函数
- 函数柯里化的基本方法和函数绑定是一样的：使用一个闭包返回一个函数。两者的区别
在于，当函数被调用时，返回的函数还需要设置一些传入的参数（柯里化就是将一个接受多个参数的函数转化为接受单一参数的函数的技术）
```js
// 基本示例
function curry(fn){
  var args = Array.prototype.slice.call(arguments, 1)
  return function(){
    var innerArgs = Array.prototype.slice.call(arguments)
    var finalArgs = args.concat(innerArgs)
    return fn.apply(null, finalArgs)
  }
}
// 用法
function curry (fn) {
  // fn.length 返回 fn 函数定义中形式参数的数量
  var fnArg = fn.length
  return function () {
    var innerArgs = Array.prototype.slice.call(arguments)
    if (innerArgs.length >= fnArg) {
      // 实参数量满足形参的情况
      fn.call(null, innerArgs)
    } else {
      // 实参数量少于形参的情况
      return function () {
        var finalArgs = Array.prototype.slice.call(arguments)
        fn.call(null, innerArgs.concat(finalArgs))
      }
    }
  }
}
const check = curry(function (what, x) {
  return x.match(what);
})
const checkPhone = check(phoneRegExp) // check('/^1[0-9]{10}/g')
const checkEmail = check(emailRegExp) // check('/^(https|http):\/\/)+.+/g')
checkPhone('13511111111')
checkEmail('www123@qq.com')

// 一道经典面试题: add(1)(2)(3)
function add() {
  var firstArgs = Array.prototype.slice.call(arguments)
  var sum = function() {
    firstArgs.concat(Array.prototype.slice.call(arguments))
    // 再次返回函数，用于持续调用
    return sum
  }
  // 利用toString隐式转换的特性，当最后执行时隐式转换，并计算最终的值返回
  sum.toString = function () {
    return firstArgs.reduce(function (a, b) {
      return a + b
    })
  }
  return sum
}
```
> PS: 函数柯里化可以使函数成为一种预加载函数

#### js深拷贝
- 使用`JSON.parse`和`JSON.stringify`
```js
var obj = {
  a: 2,
  b: 2,
  c: { d: 4 }
}
var objCopy = JSON.parse(JSON.stringify(obj))
objCopy.a = 6
console.log(objCopy.a) // 6
console.log(obj.a)     // 2
```
> 使用`JSON.parse`和`JSON.stringify`存在局限性；就是你要转换的对象必须是合法`json`对象，不能拷贝`undefined、function`等

- 完整深拷贝
```js
// 递归拷贝
function deepCopy (origin) {
  if (
    Object.prototype.toString.call(origin) === '[object Array]' ||
    Object.prototype.toString.call(origin) === '[object Object]'
  ) {
    let result = Array.isArray(origin) ? [] : {}
    for (key in origin) {
      if (Object.prototype.hasOwnProperty.call(origin, key)) {
        let val = origin[key]
        if (val && typeof val === 'object') {
          result[key] = deepCopy(val)
        } else {
          result[key] = val
        }
      }
    }
    return result
  }
  return origin
}
```
- 其它
```js
// 数组常用拷贝
let arr = [1,[2,3],4,[5,6, [7,8,[9,10]]]]
let copy = arr.slice()  // slice
copy = [].concat(arr)   // concat
[...copy] = arr         // ... es6扩展运算符
copy = Array.from(arr)  // es6的Array.from
```

#### 实现JSON.parse
```js
if (typeof JSON.parse !== 'function') {
  JSON.parse = function (strVal, reviver) {
    let result
    function walk (holder, key) {
      var k
      var v
      var value = holder[key]
      for (k in value) {
        if (Object.prototype.hasOwnProperty.call(value, k)) {
          v = walk(value, k)
          if (val != undefined) {
            value[k] = v
          } else {
            delete value[k]
          }
        }
      }
      return reviver.call(holder, key, value)
    }

    // TODO: 字符串解析，移除不合法字符
    var rx_one = /^[\],:{}\s]*$/;
    var rx_two = /\\(?:["\\\/bfnrt]|u[0-9a-fA-F]{4})/g;
    var rx_three = /"[^"\\\n\r]*"|true|false|null|-?\d+(?:\.\d*)?(?:[eE][+\-]?\d+)?/g;
    var rx_four = /(?:^|:|,)(?:\s*\[)+/g;

    if (
      rx_one.test(
        strVal
          .replace(rx_two, "@")
          .replace(rx_three, "]")
          .replace(rx_four, "")
      )
    ) {
    // 用于eval编译的字符串必须是安全的字符串。
    result = eval("(" + strVal + ")")
    return (typeof reviver === "function")
      ? walk({"": result}, "")
      : result
  }
}
```
参考链接：

[正则可视化工具](https://regexper.com/)

[统一码所有区段](https://www.fuhaoku.net/blocks)

[JSON.parse 三种实现方式](https://github.com/youngwind/blog/issues/115)

[JSON之父Douglas Crockford写的Ployfill](https://github.com/douglascrockford/JSON-js/blob/master/json2.js)

[json3](https://github.com/bestiejs/json3/blob/master/lib/json3.js)

附上`JSON.parse`源码(C语言): [JSON.parse](https://github.com/v8/v8/blob/master/src/json/json-parser.h)

#### es5和es6对于继承的实现
- es5的继承
- es6的继承
#### Promise实现
#### async await实现
#### 执行上下文和作用域
#### cookie、sessionStorage和localStorage
- cookie
- sessionStorage
- localStorage
#### MVVM
什么是`MVVM`：即`Model-View-ViewModel`简称。

参考：

[廖雪峰 MVVM](https://www.liaoxuefeng.com/wiki/1022910821149312/1108898947791072)

[基于Vue实现简易的MVVM](https://juejin.im/post/5cd8a7c1f265da037a3d0992#heading-0)

#### 前端工程化
- 为什么要前段工程化：
  - 随着前端项目的日益复杂，前端已经不是以前的简简单单的页面，写个html、css、引入几个js的事情了，由webpage模式转化成webApp模式为主了。更为复杂和多样化
  - 随着项目工程的复杂和多样化就会产生许多的问题：如`项目的维护，项目的协作、项目的风险把控、项目的开发质量、开发效率甚至于开发成本`等等问题
  - 让前端的开发流程、技术、工具、经验等规范化、标准化。让前端开发能够"自成体系"，可以最大程度地提高前端工程师的开发效率，降低技术选型、前后端联调等带来的协调沟通成本。
- 如何前端工程化：
  
  **前端工程化是一个很开发的话题，不同人有不同的理解。但最终目的不都是为了提高开发效率，保证代码质量，降低成本、提高风险可控性。个人认为可从`可控性`和`稳定性`两大方面理解**
  - 可控性
    对项目进行`组件化`、`模块化`、`规范化`、`自动化`使项目的可维护性和可用性高
  - 稳定性
    对项目进行测试（自动化测试：ui测试。逻辑测试，端到端测试、性能测试），自动化部署，持续集成；减少人为操作重复性的工作，提高项目的稳定性。
#### js浮点精度问题
- js浮点数遵循IEEE二进制浮点数算术标准（IEEE 754），所以在浮点计算时会产生误差；至于为什么会产生误差，可以了解下`IEEE 754`标准
- 推荐使用js库：
  [mathjs、](https://mathjs.org/docs/getting_started.html)
  [decimal.js、](http://mikemcl.github.io/decimal.js/#)
  [big.js](http://mikemcl.github.io/big.js/)
- 一个GitHub上的项目: [number-precision](https://github.com/nefe/number-precision)
- 知道小数点位数的前提下，手写个简单的转换方法
  ```js
  function calcFloat (num, digit) {
    let times = Math.pow(10, digit)
    return Math.round(num * times) / times
  }
  ```
> 计算思路：先放大数字，使之能够精确表示，计算之后再缩小数字，得到实际值