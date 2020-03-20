#### 模拟实现call，apply，bind
```js
// call实现
Function.prototype.call =  function (thisArg, args) {
}
// apply实现
Function.prototype.apply =  function (thisArg, argsArray) {
}
// ----bind实现----
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