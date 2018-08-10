---
layout: post
title: JavaScript中的call和apply
date: '2018-08-06 07:16'
categories:
  - JavaScript设计模式
tags:
  - call
  - apply
  - bind
---
ECAMScript 3给 Function 的原型定义了两个方法，它们是 `Function.prototype.call` 和 `Function.prototype.apply` 。这俩方法是每个函数都包含的非继承而来的方法,它们的用途都是在特定的作用域中调用函数，实际上等于设置函数体内this对象的值。在实际开发中，特别是在一些函数式风格的代码编写中，`call` 和 `apply` 方法尤为有用。在 JavaScript版本的设计模式中，这两个方法的应用也非常广泛，能熟练运用这两个方法，是我们真正成为一名 JavaScript程序员的重要一步。

> 先奉上整理的脑图：
> ![](https://i.loli.net/2018/08/06/5b6782f1cbc91.png)

## 一、call 和 apply 的区别

它们的作用一模一样，区别仅在于传入参数形式的不同。

`apply` 接受两个参数，第一个参数指定了函数体内 `this` 对象的指向，第二个参数为一个带下标的集合，这个集合可以为数组，也可以为类数组，`apply` 方法把这个集合中的元素作为参数传递给调用的函数：

```js
var func = function(a,b,c){
  alert([a,b,c]) //=> [1,2,3]
}
func.apply(null,[1,2,3])
```

在这段代码中，参数 1、2、3 被放在数组中一起传入 func 函数，它们分别对应 func 参数列表中的 a 、b 、c。

`call` 传入的参数数量不固定，跟 `apply` 相同的是，第一个参数也是代表函数体内的 `this` 指向，从第二个参数开始往后，每个参数被依次传入函数：

```js
var func = function(a,b,c){
  alert([a,b,c]) //=> [1,2,3]
}
func.call(null,1,2,3)
```

当调用一个函数时，JavaScript 的解释器并不会计较形参和实参在数量、类型以及顺序上的区别，JavaScript 的参数在内部就是用一个数组来表示的。从这个意义上说，`apply` 比 `call` 的使用率更高，我们不关心具体多少参数被传入函数，只要用 `apply` 一股脑地推过去就可以了。

`call` 是的包装在 `apply` 上的一颗语法糖，如果我们明确地知道函数接收多少个参数，而且想一目了然地表达形参和实参的对应关系，那么可以用 `call` 来传送参数。

当使用 call 或者 apply 的时候，如果我们传入的第一个参数为 null ，函数体内的 this 会指向默认的宿主对象，在浏览器中则是 window ：

```js
var func = function( a, b, c ){
  alert ( this === window ); //=> true
};
func.apply( null, [ 1, 2, 3 ] );
```

但如果是在严格模式下，函数体内的 this 还是为 null ：

```js
var func = function( a, b, c ){
  "use strict";
  alert ( this === null ); //=> true
}
func.apply( null, [ 1, 2, 3 ] );
```

有时候我们使用 call 或者 apply 的目的不在于指定 this 指向，而是另有用途，比如借用其他对象的方法。那么我们可以传入 null 来代替某个具体的对象：

```js
Math.max.apply( null, [ 1, 2, 5, 3, 4 ] ) //=> 5
```

## 二、call 和 apply 的用途

### 1、改变 this 指向

call 和 apply 最常见的用途是改变函数内部的 this 指向，我们来看个例子：

```js
var obj1 = {
  name: 'yang'
}

var obj2 = {
  name: 'xiao'
}

window.name = 'window'

var getName = function(){
  alert(this.name)
}

getName() //=> window
getName.call( obj1 ) //=> yang
getName.call( obj2 ) //=> xiao
```

当执行 `getName.call( obj1 )` 这句代码时， `getName` 函数体内的 `this` 就指向 `obj1` 对象，所以此处的

```js
var getName = function(){
  alert ( this.name )
}
```

实际上相当于：

```js
var getName = function(){
  alert ( obj1.name ) //=> yang
}
```

### 2、借用其他对象的方法

借用方法的第一种场景是“借用构造函数”，通过这种技术，可以实现一些类似继承的效果：

```js
var A = function(name){
  this.name = name
}

var B = function(){
  A.apply(this, arguments)
}

B.prototype.getName = function(){
  return this.name
}

var b = new B( 'yang' )
console.log(b.getName()) //=> yang
```

借用方法的第二种运用场景跟我们的关系更加密切。

函数的参数列表 arguments 是一个类数组对象，虽然它也有“下标”，但它并非真正的数组，所以也不能像数组一样，进行排序操作或者往集合里添加一个新的元素。这种情况下，我们常常会借用 `Array.prototype` 对象上的方法。比如想往 `arguments` 中添加一个新的元素，通常会借用
`Array.prototype.push` ：

```js
(function(){
Array.prototype.push.call( arguments, 3 );
  console.log ( arguments ) //=> [1,2,3]
})( 1, 2 )
```

在操作 `arguments` 的时候，我们经常非常频繁地找 `Array.prototype` 对象借用方法。

想把 arguments 转成真正的数组的时候，可以借用 `Array.prototype.slice` 方法；想截去 arguments 列表中的头一个元素时，又可以借用 `Array.prototype.shift` 方法。那么这种机制的内部实现原理是什么呢？我们不妨翻开 V8的引擎源码，以 `Array.prototype.push` 为例，看看 V8引
擎中的具体实现：

```js
function ArrayPush() {
  var n = TO_UINT32( this.length ); // 被 push 的对象的 length
  var m = %_ArgumentsLength(); // push 的参数个数
  for (var i = 0; i < m; i++) {
    this[ i + n ] = %_Arguments( i ); // 复制元素 (1)
  }
  this.length = n + m; // 修正 length 属性的值 (2)
  return this.length;
};
```

通过这段代码可以看到， `Array.prototype.push` 实际上是一个属性复制的过程，把参数按照下标依次添加到被 push 的对象上面，顺便修改了这个对象的 length 属性。至于被修改的对象是谁，到底是数组还是类数组对象，这一点并不重要。

由此可以推断，我们可以把“任意”对象传入 `Array.prototype.push`：

```js
var a = {};
Array.prototype.push.call( a, 'first' )
alert ( a.length ) //=> 1
alert ( a[ 0 ] ) //=> first
```
