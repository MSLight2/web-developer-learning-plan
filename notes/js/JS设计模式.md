## JS中的单列模式
```js
// 惰性单列（单一原则，只返回单列）
let getSingle = function (fn) {
  let instance = null;
  return () => {
    return instance || (instance = fn.apply(this, arguments))
  }
}
```
## 策略模式
策略模式的定义是：定义一系列的算法，把它们一个个封装起来，并且使它们可以相互替换。

一个基于策略模式的程序至少由两部分组成。第一个部分是一组策略类，略类封装了具体的算法，并负责具体的计算过程。第二个部分是环境类 Context，Context 接受客户的请求，随后把请求委托给某一个策略类。

### 缓动算法
算法接收4 个参数，这 4 个参数的含义分别是动画已消耗的时间、小球原始位置、小球目标位置、动画持续的总时间，返回的值则是动画元素应该处在的当前位置。
```js
let tween = {
  linear: function( t, b, c, d ){
    return c*t/d + b;
  },
  easeIn: function( t, b, c, d ){
    return c * ( t /= d ) * t + b;
  },
  strongEaseIn: function(t, b, c, d){
    return c * ( t /= d ) * t * t * t * t + b;
  },
  strongEaseOut: function(t, b, c, d){
    return c * ( ( t = t / d - 1) * t * t * t * t + 1 ) + b;
  },
  sineaseIn: function( t, b, c, d ){
    return c * ( t /= d) * t * t + b;
  },
  sineaseOut: function(t,b,c,d){
    return c * ( ( t = t / d - 1) * t * t + 1 ) + b;
  }
};
```
### 表单校验
```js
// -----------策略对象-----------
var strategies = {
  isNonEmpty: function (value, errorMsg) {
    if (value === '') {
      return errorMsg;
    }
  },
  minLength: function (value, length, errorMsg) {
    if (value.length < length){
      return errorMsg;
    }
  },
  isMobile: function(value, errorMsg) {
    if (!/(^1[3456789][0-9]{9}$)/.test(value)) {
      return errorMsg;
    }
  }
};

// -----------Validator类-----------
var Validator = function(){
  this.cache = [];
};
Validator.prototype.add = function( dom, rules ){
  var self = this;
  for ( var i = 0, rule; rule = rules[ i++ ]; ){
    (function( rule ){
      var strategyAry = rule.strategy.split( ':' ); // 把 strategy 和参数分开
      var errorMsg = rule.errorMsg;
      self.cache.push(function(){ // 把校验的步骤用空函数包装起来，并且放入 cache
        var strategy = strategyAry.shift(); // 用户挑选的 strategy
        strategyAry.unshift( dom.value ); // 把 input 的 value 添加进参数列表
        strategyAry.push( errorMsg ); // 把 errorMsg 添加进参数列表
        return strategies[ strategy ].apply( dom, strategyAry );
      });
    })(rule)
  }
};
Validator.prototype.start = function(){
  for ( var i = 0, validatorFunc; validatorFunc = this.cache[ i++ ]; ){
    var errorMsg = validatorFunc();
    if ( errorMsg ){
      return errorMsg;
    }
  }
};

// -----------客户调用代码-----------
var validataFunc = function(){
  var validator = new Validator(); // 创建一个 validator 对象
  /***************添加一些校验规则****************/
  validator.add( registerForm.userName, 'isNonEmpty', '用户名不能为空' );
  validator.add( registerForm.password, 'minLength:6', '密码长度不能少于 6 位' );
  validator.add( registerForm.phoneNumber, 'isMobile', '手机号码格式不正确' );
  var errorMsg = validator.start(); // 获得校验结果
  return errorMsg; // 返回校验结果
}
var registerForm = document.getElementById( 'registerForm' );
  registerForm.onsubmit = function(){
  var errorMsg = validataFunc(); // 如果 errorMsg 有确切的返回值，说明未通过校验
  if ( errorMsg ){
    alert ( errorMsg );
    return false; // 阻止表单提交
  }
};
```
> 在JS这种以函数为一等对象的语言中，策略模式是隐形的。

## 代理模式
### 保护代理和虚拟代理
保护代理用于控制不同权限的对象对目标对象的访问。过滤一些不符合或不需要的请求。虚拟代理把一些开销很大的对象，延迟到真正需要它的时候才去创建。

代理对象的接口要和本体保持一致。代理对于用户是透明的，用户感知和使用本体一样。
### 虚拟代理（图片懒加载）
```js
var proxyImage = (function(){
  var img = new Image;
  img.onload = function(){
    myImage.setSrc( this.src );
  }
  return {
    setSrc: function( src ){
      myImage.setSrc('loading.gif');
      img.src = src;
    }
  }
})();
proxyImage.setSrc('show.jpg');
```
## 迭代器模式
比较老的一种设计模式，很多语言已经内置实现。比如JavaScript中的`Array.prototype.forEach`。
## 发布—订阅模式
发布—订阅模式又叫观察者模式，它定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都将得到通知。

- 首先要指定好谁充当发布者（比如售楼处）；
- 然后给发布者添加一个缓存列表，用于存放回调函数以便通知订阅者（售楼处的花名册）；
- 最后发布消息的时候，发布者会遍历这个缓存列表，依次触发里面存放的订阅者回调函数（遍历花名册，挨个发短信）。
```js
var Event = (function(){
  var clientList = {},
  listen,
  trigger,
  remove;
  listen = function( key, fn ){
    if ( !clientList[ key ] ){
      clientList[ key ] = [];
    }
    clientList[ key ].push( fn );
  };
  trigger = function(){
    var key = Array.prototype.shift.call( arguments ),
    fns = clientList[ key ];
    if ( !fns || fns.length === 0 ){
      return false;
    }
    for( var i = 0, fn; fn = fns[ i++ ]; ){
      fn.apply( this, arguments );
    }
  };
  remove = function( key, fn ){
    var fns = clientList[ key ];
    if ( !fns ){
      return false;
    } 
    if ( !fn ){
      fns && ( fns.length = 0 );
    } else {
      for ( var l = fns.length - 1; l >=0; l-- ){
        var _fn = fns[ l ];
        if ( _fn === fn ){
          fns.splice( l, 1 );
        }
      }
    }
  };
  return {
    listen: listen,
    trigger: trigger,
    remove: remove
  }
})();
Event.listen( 'squareMeter88', function( price ){ // 小红订阅消息
 console.log( '价格= ' + price ); // 输出：'价格=2000000'
});
Event.trigger( 'squareMeter88', 2000000 ); // 售楼处发布消息
```
## 命令模式