#### 模拟实现call，apply，bind
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
Function.prototype.applyFn =  function (thisArg, argArray) {
  // 规范1
  if (typeof this !== 'function') throw new Error('Type Error: ' + this + 'must be a function.')
  // 规范2
  if (argArray == null) argArray = []
  // 规范3
  if (argArray !== new Object(argArray)) throw new Error('TypeError: CreateListFromArrayLike called on non-object.')
  // 规范 注
  thisArg = thisArg ? new Object(thisArg) : window
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
Function.prototype.call =  function (thisArg, args) {
}

// --------------------------bind实现--------------------------
// 不使用原生call、apply
Function.prototype.bind =  function (thisArg, argsArray) {
}
// 使用原生call、apply
Function.prototype.bind2 =  function (thisArg, argsArray) {
}
```
#### 实现JSON.parse
#### es5和es6对于继承的实现
#### Promise实现
#### async await实现