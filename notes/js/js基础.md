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

- 使用es6`flat`
```js
var arr = [1,[2,3],4,[5,6]]
console.log(arr.flat()) // [1,2,3,4,5,6]
```

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
#### js深拷贝
#### 实现JSON.parse
#### es5和es6对于继承的实现
#### Promise实现
#### async await实现
#### 执行上下文和作用域
#### cookie、sessionStorage和localStorage