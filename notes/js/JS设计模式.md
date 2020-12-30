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
中介者模式的作用就是解除对象与对象之间的紧耦合关系。增加一个中介者对象后，所有的相关对象都通过中介者对象来通信，而不是互相引用，所以当一个对象发生改变时，只需要通知中介者对象即可。中介者使各对象之间耦合松散，而且可以独立地改变它们之间的交互。

使用时机：一般来说，如果对象之间的复杂耦合确实导致调用和维护出现了困难，而且这些耦合度随项目的变化呈指数增长曲线，那我们就可以考虑用中介者模式来重构代码。

## 12、装饰者模式
在程序开发中，许多时候都并不希望某个类天生就非常庞大，一次性包含许多职责。那么我们就可以使用装饰者模式。装饰者模式可以动态地给某个对象添加一些额外的职责，而不会影响从这个类中派生的其他对象。**给对象动态地增加职责的方式称为装饰者（decorator）模式。**
```js
var plane = {
  fire: function(){
    console.log( '发射普通子弹' );
  }
}
var missileDecorator = function(){
  console.log( '发射导弹' );
}
var atomDecorator = function(){
  console.log( '发射原子弹' );
}
var fire1 = plane.fire;
plane.fire = function(){
  fire1();
  missileDecorator();
}
var fire2 = plane.fire;
plane.fire = function(){
  fire2();
  atomDecorator();
}
plane.fire();
// 分别输出： 发射普通子弹、发射导弹、发射原子弹
```
###  AOP 装饰函数
```JS
var before = function( fn, beforefn ){
  return function(){
    beforefn.apply( this, arguments );
    return fn.apply( this, arguments );
  }
}
var a = before(
  function(){alert (3)},
  function(){alert (2)}
);
a = before( a, function(){alert (1);} );
a(); 
```
### 装饰者模式和代理模式
代理模式和装饰者模式最重要的区别在于它们的意图和设计目的。代理模式的目的是，当直接访问本体不方便或者不符合需要时，为这个本体提供一个替代者。本体定义了关键功能，而代理提供或拒绝对它的访问，或者在访问本体之前做一些额外的事情。装饰者模式的作用就是为对象动态加入行为。换句话说，代理模式强调一种关系（Proxy 与它的实体之间的关系），这种关系可以静态的表达，也就是说，这种关系在一开始就可以被确定。而装饰者模式用于一开始不能确定对象的全部功能时。代理模式通常只有一层代理本体的引用，而装饰者模式经常会形成一条长长的装饰链。

## 13、状态模式(一种要好好掌握的设计模式)
状态模式的关键是区分事物内部的状态，事物内部状态的改变往往会带来事物的行为改变。

### 状态模式和策略模式的关系
策略模式和状态模式的相同点是，它们都有一个上下文、一些策略或者状态类，上下文把请求委托给这些类来执行。
```js
```
它们之间的区别是策略模式中的各个策略类之间是平等又平行的，它们之间没有任何联系，所以客户必须熟知这些策略类的作用，以便客户可以随时主动切换算法；而在状态模式中，状态和状态对应的行为是早已被封装好的，状态之间的切换也早被规定完成，“改变行为”这件事情发生在状态模式内部。对客户来说，并不需要了解这些细节。这正是状态模式的作用所在。

## 14、适配器模式
适配器模式的作用是解决两个软件实体间的接口不兼容的问题。使用适配器模式之后，原本由于接口不兼容而不能工作的两个软件实体可以一起工作。

适配器模式主要用来解决两个已有接口之间不匹配的问题，它不考虑这些接口是怎样实
现的，也不考虑它们将来可能会如何演化。适配器模式不需要改变已有的接口，就能够
使它们协同作用。

装饰者模式和代理模式也不会改变原有对象的接口，但装饰者模式的作用是为了给对象
增加功能。装饰者模式常常形成一条长的装饰链，而适配器模式通常只包装一次。代理
模式是为了控制对对象的访问，通常也只包装一次。

外观模式的作用倒是和适配器比较相似，有人把外观模式看成一组对象的适配器，但外
观模式最显著的特点是定义了一个新的接口。

## 设计原则和编程技巧
### 单一职责原则
单一职责原则更多地是被运用在对象或者方法级别上。

单一职责原则（SRP）的职责被定义为“引起变化的原因”。如果我们有两个动机去改写一
个方法，那么这个方法就具有两个职责。每个职责都是变化的一个轴线，如果一个方法承担了过多的职责，那么在需求的变迁过程中，需要改写这个方法的可能性就越大。

**SRP 原则体现为：一个对象（方法）只做一件事情。**

#### 何时分离原则
SRP 原则是所有原则中最简单也是最难正确运用的原则之一。

要明确的是，并不是所有的职责都应该一一分离。

一方面，如果随着需求的变化，有两个职责总是同时变化，那就不必分离他们。比如在 ajax请求的时候，创建 xhr 对象和发送 xhr 请求几乎总是在一起的，那么创建 xhr 对象的职责和发送xhr 请求的职责就没有必要分开。

另一方面，职责的变化轴线仅当它们确定会发生变化时才具有意义，即使两个职责已经被耦合在一起，但它们还没有发生改变的征兆，那么也许没有必要主动分离它们，在代码需要重构的时候再进行分离也不迟。

### 最少知识原则
