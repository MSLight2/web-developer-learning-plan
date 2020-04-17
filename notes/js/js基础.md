### 模拟实现call，apply，bind
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

### new实现
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

### 多维数组转换一维数组
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

### 数组、对象去重

### 柯里化函数
- 函数柯里化的基本方法和函数绑定是一样的：使用一个闭包返回一个函数。两者的区别
在于，当函数被调用时，返回的函数还需要设置一些传入的参数（柯里化就是将一个接受多个参数的函数转化为接受单一参数的函数的技术）
```js
// 基本示例
function curry (fn) {
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

### js深拷贝
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

### 实现JSON.parse
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
    var rx_one = /^[\],:{}\s]*$/; // 匹配：] , : { } \s
    // 匹配：\\ \/ \b \f \n \r \t \u(0-9a-fA-F)
    var rx_two = /\\(?:["\\\/bfnrt]|u[0-9a-fA-F]{4})/g;
    // 匹配： "非(双引号 \ \n \r)" true false null 数值（类似：3, -3, 3.22, 3.22e+2, 3.22e5）
    var rx_three = /"[^"\\\n\r]*"|true|false|null|-?\d+(?:\.\d*)?(?:[eE][+\-]?\d+)?/g;
    // 匹配：(: [) (,[)
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
}
```
参考链接：

工具：[正则可视化工具](https://regexper.com/)

[统一码所有区段](https://www.fuhaoku.net/blocks)

[JSON.parse 三种实现方式](https://github.com/youngwind/blog/issues/115)

[JSON之父Douglas Crockford写的Ployfill](https://github.com/douglascrockford/JSON-js/blob/master/json2.js)

[json3](https://github.com/bestiejs/json3/blob/master/lib/json3.js)

附上`JSON.parse`源码(C语言): [JSON.parse](https://github.com/v8/v8/blob/master/src/json/json-parser.h)

### es5和es6对于继承的实现
- es5的继承
  |继承|特性|实例|优点|缺点|
  |----|----|----|----|----|
  |借用构造函数  | 在子类型构造函数的内部调用超类型构造函数 | function SubType(){ SuperType.call(this); }  | 可以在子类型构造函数中向超类型构造函数传递参数 | 方法都在构造函数中定义，因此函数复用就无从谈起。在超类型的原型中定义的方法，对子类型而言也是不可见|
  |组合继承      | 是将原型链和借用构造函数的技术组合到一块，从而发挥二者之长的一种继承模式。其背后的思路是使用原型链实现对原型属性和方法的继承，而通过借用构造函数来实现对实例属性的继承。 |SubType.prototype = new SuperType();| 避免了原型链的和借用构造函数的缺陷 | 无论什么情况下，都会调用两次超类型构造函数：一次是在创建子类型原型的时候，另一次是在子类型构造函数内部。 |
  |原型式继承    |借助原型可以基于已有的对象创建新对象，同时还不必因此创建自定义类型。要求你必须有一个对象可以作为另一个对象的基础。ECMAScript 5 通过新增 Object.create()方法规范化了原型式继承|function object(o){<br> function F(){};<br> F.prototype = o;<br> return new F();<br> }| 所有对象实例共享它所包含的属性和方法 | 原型属性会被所有实例共享；在创建子类型的实例时，不能向超类型的构造函数中传递参数。 |
  |寄生式继承    |是与原型式继承紧密相关的一种思路；寄生式继承的思路与寄生构造函数和工厂模式类似，即创建一个仅用于封装继承过程的函数，该函数在内部以某种方式来增强对象，最后再像真地是它做了所有工作一样返回对象| function createAnother(original){<br>var clone = object(original);<br>clone.sayHi = function(){ alert("hi")};<br>return clone; //返回这个对象<br>} | 解决组合继承模式由于多次调用超类型构造函数而导致的低效率问题 | 使用寄生式继承来为对象添加函数，会因为做不到函数复用而降低效率，这个与构造函数模式类似 |
  |寄生组合式继承|即通过借用构造函数来继承属性，通过原型链的混成形式来继承方法。其背后的基本思路是：不必为了指定子类型的原型而调用超类型的构造函数，我们所需要的无非就是超类型原型的一个副本而已。|function inheritPrototype(subType, superType){<br>var prototype = object(superType.prototype);<br> prototype.constructor = subType;<br> subType.prototype = prototype; <br>} |只调用了一次 SuperType 构造函数，并且因此避免了在 SubType.prototype 上面创建不必要的、多余的属性。与此同时，因此，原型链还能保持不变；寄生组合式继承被认为是引用类型最理想的继承范式| / |
- es6的继承
  
  es6的继承只是es5的语法糖，**本质上也是原型链的继承。**
  |继承|特性|
  |----|----|
  |es6继承|1、关键字`extends`。<br/>2、子类必须在`constructor`方法中调用`super`方法，否则新建实例时会报错。<br>3、ES5 的继承，实质是先创造子类的实例对象this，然后再将父类的方法添加到this上面（`Parent.apply(this)`）。ES6 的继承机制完全不同，实质是先将父类实例对象的属性和方法，加到this上面（所以必须先调用super方法），然后再用子类的构造函数修改this。<br>4、子类的构造函数中，只有调用`super`之后，才可以使用`this`关键字，否则会报错。<br>5、父类的静态方法，也会被子类继承。|
  | | 6、`Object.getPrototypeOf`方法可以用来从子类上获取父类。可以使用这个方法判断，一个类是否继承了另一个类<br>7、`extends`关键字不仅可以用来继承类，还可以用来继承原生的构造函数，而ES5不可以继承原生的构造函数<br>8、注意，继承`Object`的子类，有一个行为差异：无法通过`super`方法向父类`Object`传参。这是因为 ES6 改变了`Object`构造函数的行为，一旦发现`Object`方法不是通过`new Object()`这种形式调用，ES6 规定Object构造函数会忽略参数。|
  |`super`关键字|**super这个关键字，既可以当作函数使用，也可以当作对象使用。**<br>一、作为函数调用<br>`this`指向的不是父类，而是子类，且`super()`只能用在子类的构造函数之中<br>二、`super`作为对象时<br>`super`指向父类的原型对象，所以定义在父类实例上的方法或属性，是无法通过`super`调用的。<br><br>ES6 规定，在子类普通方法中通过`super`调用父类的方法时，方法内部的`this`指向当前的子类实例。<br>在子类的静态方法中通过super调用父类的方法时，方法内部的this指向当前的子类，而不是子类的实例。|
  |类的 `prototype` 属性和`__proto__`属性|1、子类的`__proto__`属性，表示构造函数的继承，总是指向父类。<br>2、子类`prototype`属性的`__proto__`属性，表示方法的继承，总是指向父类的`prototype`属性|
  |实例的 `__proto__` 属性|子类实例的`__proto__`属性的`__proto__`属性，指向父类实例的`__proto__`属性。|
- es5中构造函数和es6类的对应关系
  |es5构造函数|es6类
  |----|----|
  |ES5 的构造函数| ES6 的类的构造方法(`constructor`)|
  |`prototype`对象的`constructor`属性，直接指向"类"的本身，这与 ES5 的行为是一致的|
  |与 ES5 一样，实例的属性除非显式定义在其本身（即定义在`this`对象上），否则都是定义在原型上（即定义在`class`上）|
  |与 ES5 一样，类的所有实例共享一个原型对象|
  |es5可以枚举|类的内部所有定义的方法，都是不可枚举的|
  |es5构造函数可以作为函数调用|类必须使用`new`调用|
  |默认不是严格模式|类和模块的内部，默认就是严格模式|
  |构造函数存在变量提升|类不存在变量提升|
  |构造函数的`this`默认是`undefined`，因为`this`是运行是确认的。而es6则是必须`new`调用|类的方法内部如果含有`this`，它默认指向类的实例。<br><br>注意：单独使用方法会改变this指向。<br>例如：`let { fn } = A`（假如类A里面有个方法叫`fn`，`fn`里面有使用`this`）<br>此时this指向会改变。可以`在构造方法中绑定this，即fn = fn.bind(this)`或者使用`箭头函数（fn = () => {}）`|
  |es5使用`instanceof`判断是否是构造函数的实例。例如： `this instanceof A`|es6有`new.target`实例化(`new`)时,`new.target`的值就是当前的类（例如：`A --> new A; new.target === A // true`）<br>注意：子类继承父类时，`new.target`会返回子类|
### Promise实现
### async await实现
### 执行上下文和作用域链
  - 执行上下文（执行环境）：
    - 全局执行环境是最外围的一个执行环境
    - 每个函数都有自己的执行环境
    - （每一个环境都有一个变量对象）当代码在一个环境中执行时，会创建变量对象的一个作用域链。作用域链的用途，是保证对执行环境有权访问的所有变量和函数的有序访问。作用域链的前端，始终都是当前执行的代码所在环境的变量对象。如果这个环境是函数，则将其活动对象（activation object）作为变量对象。活动对象在最开始时只包含一个变量，即 arguments 对象（这个对象在全局环境中是不存在的）。作用域链中的下一个变量对象来自包含（外部）环境，而再下一个变量对象则来自下一个包含环境。这样，一直延续到全局执行环境；全局执行环境的变量对象始终都是作用域链中的最后一个对象
  - 作用域
    - 全局作用域：`window`、`global`、其它
    - 函数作用域
    - 快作用域：`for`、`with`、`try/catch`(catch内部)`、let、``const`
### cookie、sessionStorage和localStorage
- cookie
  - 客户端用于存储会话信息的。该标准要求服务器对任意 HTTP 请求发送 Set-Cookie HTTP 头作为响应的一部分，其中包含会话信息
  - 限制
    - IE7 和之后版本每个域名最多 50 个
    - Firefox 限制每个域最多 50 个 cookie
    - Safari 和 Chrome 对于每个域的 cookie 数量限制没有硬性规定
    - cookie 的尺寸一般在`4KB`左右
    - cookie由：名称(name)、值(value)、域(domain)、路径(path)、失效时间(expires)、安全标志(secure...)构成
- sessionStorage
  - sessionStorage 对象存储特定于某个会话的数据，也就是该数据只保持到浏览器关闭。会在浏览器关闭后消失。
  - sessionStorage 中的数据可以跨越页面刷新而存在
- localStorage
  - 要访问同一个 localStorage 对象，页面必须来自同一个域名（子域名无效），使用同一种协议，在同一个端口上（同源限制）
  - 数据保留到通过 JavaScript 删除或者是用户清除浏览器缓存（用户不删除可以长期储存在本地）
  
> `sessionStorage和localStorage`限制：大多数桌面浏览器会设置每个来源 5MB 的限制，Chrome 和 Safari 对每个来源的限制是 2.5MB.而 iOS 版 Safari 和 Android 版 WebKit 的限制也是 2.5MB。IE8+和 Opera 对sessionStorage 的限制是 5MB
### MVVM
什么是`MVVM`：即`Model-View-ViewModel`简称。

参考：

[廖雪峰 MVVM](https://www.liaoxuefeng.com/wiki/1022910821149312/1108898947791072)

[基于Vue实现简易的MVVM](https://juejin.im/post/5cd8a7c1f265da037a3d0992#heading-0)

### 前端工程化
- 为什么要前段工程化：
  - 随着前端项目的日益复杂，前端已经不是以前的简简单单的页面，写个html、css、引入几个js的事情了，由webpage模式转化成webApp模式为主了。更为复杂和多样化
  - 随着项目工程的复杂和多样化就会产生许多的问题：如`项目的维护，项目的协作、项目的风险把控、项目的开发质量、开发效率甚至于开发成本`等等问题
  - 让前端的开发流程、技术、工具、经验等规范化、标准化。让前端开发能够"自成体系"，可以最大程度地提高前端工程师的开发效率，降低技术选型、前后端联调等带来的协调沟通成本。
- 如何前端工程化：
  
  **前端工程化是一个很开发的话题，不同人有不同的理解。但最终目的不都是为了提高开发效率，保证代码质量，降低成本、提高风险可控性。开发-->上线-->维护**
  - 可控性
    对项目进行`组件化`、`模块化`、`规范化`、`自动化`使项目的可维护性和可用性高
  - 稳定性
    对项目进行测试（自动化测试：ui测试。逻辑测试，端到端测试、性能测试），自动化部署，持续集成；减少人为操作重复性的工作，提高项目的稳定性。
### js浮点精度问题
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