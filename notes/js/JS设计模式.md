## JS高阶函数
## 1、JS中的单列模式
```js
// 惰性单列（单一原则，只返回单列）
let getSingle = function (fn) {
  let instance = null;
  return () => {
    return instance || (instance = fn.apply(this, arguments))
  }
}
```
## 2、策略模式
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

## 3、代理模式
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
## 4、迭代器模式
比较老的一种设计模式，很多语言已经内置实现。比如JavaScript中的`Array.prototype.forEach`。
## 5、发布—订阅模式
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
## 6、命令模式
命令模式最常见的应用场景是：有时候需要向某些对象发送请求，但是并不知道请的接收者是谁，也不知道被请求的操作是什么。命令模式还支持撤销、排队等操作。

命令模式在 JavaScript 语言中是一种隐形的模式。
## 7、组合模式
组合模式将对象组合成树形结构，以表示“部分-整体”的层次结构。 除了用来表示树形结构之外，组合模式的另一个好处是通过对象的多态性表现，使得用户对单个对象和合对象的使用具有一致性，下面分别说明。
  - 表示树形结构。通过回顾上面的例子，我们很容易找到组合模式的一个优点：提供了一种遍历树形结构的方案，通过调用组合对象的 execute 方法，程序会递归调用组合对象下面的叶对象的 execute 方法，所以我们的万能遥控器只需要一次操作，便能依次完成关门、打开电脑、登录 QQ 这几件事情。组合模式可以非常方便地描述对象部分-整体层次结构。
  - 利用对象多态性统一对待组合对象和单个对象。利用对象的多态性表现，可以使客户端忽略组合对象和单个对象的不同。在组合模式中，客户将统一地使用组合结构中的所有对象，而不需要关心它究竟是组合对象还是单个对象。

**一些值得注意的地方**
- 组合模式不是父子关系
- 对叶对象操作的一致性
  - 组合模式除了要求组合对象和叶对象拥有相同的接口之外，还有一个必要条件，就是对一组叶对象的操作必须具有一致性。
- 双向映射关系
- 用职责链模式提高组合模式性能

### 何时使用组合模式
- 表示对象的部分-整体层次结构
- 客户希望统一对待树中的所有对象。

## 8、模板方法模式
模板方法模式是一种只需使用继承就可以实现的非常简单的模式。

模板方法模式由两部分结构组成，第一部分是抽象父类，第二部分是具体的实现子类。通常在抽象父类中封装了子类的算法框架，包括实现一些公共方法以及封装子类中所有方法的执行顺序。子类通过继承这个抽象类，也继承了整个算法结构，并且可以选择重写父类的方法。

假如我们有一些平行的子类，各个子类之间有一些相同的行为，也有一些不同的行为。如果相同和不同的行为都混合在各个子类的实现中，说明这些相同的行为会在各个子类中重复出现。但实际上，相同的行为可以被搬移到另外一个单一的地方，模板方法模式就是为解决这个问题而生的。在模板方法模式中，子类实现中的相同部分被上移到父类中，而将不同的部分留待子类来实现。这也很好地体现了泛化的思想。

模板方法模式是一种严重依赖抽象类的设计模式。JavaScript 在语言层面
并没有提供对抽象类的支持，所以也很难模拟抽象类的实现。（TypeScript可以很好的实现）

## 9、享元模式
享元（flyweight）模式是一种用于性能优化的模式，“fly”在这里是苍蝇的意思，意为蝇量级。享元模式的核心是运用共享技术来有效支持大量细粒。如果系统中因为创建了大量类似的对象而导致内存占用过高，享元模式就非常有用了。

享元模式的过程是剥离外部状态，并把外部状态保存在其他地方，在合适的时刻
再把外部状态组装进共享对象。

### 内部状态与外部状态：
- 内部状态存储于对象内部。
- 内部状态可以被一些对象共享。
- 内部状态独立于具体的场景，通常不会改变。
- 外部状态取决于具体的场景，并根据场景而变化，外部状态不能被共享。

剥离了外部状态的对象成为共享对象，外部状态在必要时被传入共享对象来组装成一个完整的对象。虽然组装外部状态成为一个完整对象的过程需要花费一定的时间，但却可以大大减少系统中的对象数量，相比之下，这点时间或许是微不足道的。因此，享元模式是一种用时间换空间的优化模式。

通过一个对象工厂来解决对象创建问题，只有当某种共享对象被真正需要时，它才从工厂中被创建出来。用一个管理器来记录对象相关的外部状态，使这些外部状态通过某个钩子和共享对象联系起来。

### 使用享元模式时机
- 一个程序中使用了大量的相似对象。
- 由于使用了大量对象，造成很大的内存开销。
- 对象的大多数状态都可以变为外部状态。
- 剥离出对象的外部状态之后，可以用相对较少的共享对象取代大量对象。

> 对象池

## 10、职责链模式
职责链模式的定义是：使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系，将这些对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。
```js
var order500 = function( orderType, pay, stock ){
  if ( orderType === 1 && pay === true ){
    console.log( '500 元定金预购，得到 100 优惠券' );
  }else{
    return 'nextSuccessor'; // 我不知道下一个节点是谁，反正把请求往后面传递
  }
};
var order200 = function( orderType, pay, stock ){
  if ( orderType === 2 && pay === true ){
    console.log( '200 元定金预购，得到 50 优惠券' );
  }else{
    return 'nextSuccessor'; // 我不知道下一个节点是谁，反正把请求往后面传递
  }
};
var orderNormal = function( orderType, pay, stock ){
  if ( stock > 0 ){
    console.log( '普通购买，无优惠券' );
  }else{
    console.log( '手机库存不足' );
  }
};

// this.successor，表示在链中的下一个节点
// Chain.prototype.setNextSuccessor 指定在链中的下一个节点
// Chain.prototype.passRequest 传递请求给某个节点
var Chain = function( fn ){
  this.fn = fn;
  this.successor = null;
};
Chain.prototype.setNextSuccessor = function( successor ){
  return this.successor = successor;
};
Chain.prototype.passRequest = function(){ 
  var ret = this.fn.apply( this, arguments );
  if ( ret === 'nextSuccessor' ){
    return this.successor && this.successor.passRequest.apply( this.successor, arguments );
  }
  return ret;
};
// 现在我们把 3 个订单函数分别包装成职责链的节点：
var chainOrder500 = new Chain( order500 );
var chainOrder200 = new Chain( order200 );
var chainOrderNormal = new Chain( orderNormal );
// 然后指定节点在职责链中的顺序：
chainOrder500.setNextSuccessor( chainOrder200 );
chainOrder200.setNextSuccessor( chainOrderNormal ); 
// 最后把请求传递给第一个节点：
chainOrder500.passRequest( 1, true, 500 ); // 输出：500 元定金预购，得到 100 优惠券
chainOrder500.passRequest( 2, true, 500 ); // 输出：200 元定金预购，得到 50 优惠券
chainOrder500.passRequest( 3, true, 500 ); // 输出：普通购买，无优惠券
chainOrder500.passRequest( 1, false, 0 ); // 输出：手机库存不足
```
异步的职责链需手动调用，定义一个`next`方法即可；
```js
Chain.prototype.next= function(){
  return this.successor && this.successor.passRequest.apply( this.successor, arguments );
};
```
### AOP实现职责链
```js
Function.prototype.after = function( fn ){
  var self = this;
  return function(){
    var ret = self.apply( this, arguments );
    if ( ret === 'nextSuccessor' ){
      return fn.apply( this, arguments );
    }
    return ret;
  }
};
var order = order500yuan.after( order200yuan ).after( orderNormal );
order( 1, true, 500 ); // 输出：500 元定金预购，得到 100 优惠券
order( 2, true, 500 ); // 输出：200 元定金预购，得到 50 优惠券
order( 1, false, 500 ); // 输出：普通购买，无优惠券
```
## 11、中介者模式