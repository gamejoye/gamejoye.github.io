---
title: 《你不知道的Javascript》记录 part-1
date: 2024-10-02 17:11:49
tags: 前端
cover: cover.jpeg
---
# 作用域与闭包
## 作用域是什么
作用域是根据名称查找变量的一套规则
### LHS和RHS
当变量出现在赋值操作的左侧时进行 LHS 查询，出现在右侧时进行 RHS 查询。

RHS 查询与简单地查找某个变量的值别无二致，而 LHS 查询则是试图找到变量的容器本身，从而可以对其赋值。

Q：为什么会有LHS和RHS？

A：LHS和RHS找不到变量的处理是不一样的，LHS找不到会创建一个变量， RHS则会抛出错误（Unknow Variable）。 区分LHS和RHS可以让我们在查找变量方式上有其各自对应的处理方法

## 词法作用域
![词法作用域](lex-scope.png)
1. 包含着整个全局作用域，其中只有一个标识符：foo。
2. 包含着 foo 所创建的作用域，其中有三个标识符：a、bar和 b。
3. 包含着 bar 所创建的作用域，其中只有一个标识符：c。

词法作用域意味着作用域是由书写代码时函数声明的位置来决定的。编译的词法分析阶段基本能够知道全部标识符在哪里以及是如何声明的，从而能够预测在执行过程中如何对它们进行查找。

## 块作用域
```
for (let i = 0; i < 5; i++) {
  console.log(i)
}
```
上面的代码声明了一个块作用域的变量i
变量i会在每次循环运行函数体的时候重新声明然后赋值。相当于：
```
let j;
for (j=0; j<10; j++) {
  let i = j; // 每个迭代重新绑定！
  console.log( i );
}
```
在没有let和const的语法下。 声明块作用域可以通过在try 中抛出错误， 然后catch里面捕获这个错误来实现
```
try {
  throw 2
} catch(a) {
  console.log('a', a)
}
```

## 提升
下面的语句分为两步

1. 先**提升**声明变量a
2. 将2赋值为变量a
```
var a = 2
```
函数优先
```
foo() // 1

var foo = function() {
  console.log(2)
}

function foo() {
  console.log(1)
}

foo() // 2
```

## 闭包
**当函数可以记住并访问所在的词法作用域，即使函数是在当前词法作用域之外执行，这时就产生了闭包。**


# this和对象原型

## 关于this

this 是在运行时进行绑定的，并不是在编写时绑定，它的上下文取决于函数调用时的各种条件。this 的绑定和函数声明的位置没有任何关系，只取决于函数的调用方式。
当一个函数被调用时，会创建一个活动记录（有时候也称为执行上下文）。这个记录会包含函数在哪里被调用（调用栈）、函数的调用方法、传入的参数等信息。this 就是记录的其中一个属性，会在函数执行的过程中用到。
this 既不指向函数自身也不指向函数的词法作用域，this 实际上是在函数被调用时发生的绑定，它指向什么完全取决于函数在哪里被调用


## this全面解析

### 绑定规则
1. 默认绑定
```
function foo() {
  console.log( this.a );
}
var a = 2;
foo(); // 2
```
在代码中，foo() 是直接使用不带任何修饰的函数引用进行调用的，因此只能使用
默认绑定，无法应用其他规则。

2. 隐式绑定
```
function foo() {
  console.log( this.a );
}
var obj = {
  a: 2,
  foo: foo
};
obj.foo(); // 2
```
当函数引用有上下文对象时，隐式绑定规则会把函数调用中的 this 绑定到这个上下文对象。
对象属性引用链中只有最顶层或者说最后一层会影响调用位置

#### 引用丢失
一个最常见的 this 绑定问题就是被隐式绑定的函数会丢失绑定对象，也就是说它会应用默认绑定，从而把 this 绑定到全局对象或者 undefined 上，取决于是否是严格模式
```
function foo() {
  console.log( this.a );
}
var obj = {
  a: 2,
  foo: foo
};
var bar = obj.foo; // 函数别名！
var a = "oops, global"; // a 是全局对象的属性
bar(); // "oops, global"
```
一种更微妙、更常见并且更出乎意料的情况发生在传入回调函数时
```
function foo() {
  console.log( this.a );
}
function doFoo(fn) {
  // fn 其实引用的是 foo
  fn(); // <-- 调用位置！
}
var obj = {
  a: 2,
  foo: foo
};
var a = "oops, global"; // a 是全局对象的属性
doFoo( obj.foo ); // "oops, global"
```

3. 显式绑定

所有“函数”都可以通过函数原型上的call、apply、bind进行显式的this绑定


4. new绑定

使用 new 来调用函数，或者说发生构造函数调用时，会自动执行下面的操作：
1. 创建（或者说构造）一个全新的对象。
2. 这个新对象会被执行 [[ 原型 ]] 连接。
3. 这个新对象会绑定到函数调用的 this。
4. 如果函数没有返回其他对象，那么 new 表达式中的函数调用会自动返回这个新对象。
```
function foo(a) {
  this.a = a;
}
var bar = new foo(2);
console.log( bar.a ); // 2
```
使用 new 来调用 foo(..) 时，我们会构造一个新对象并把它绑定到 foo(..) 调用中的 this上。new 是最后一种可以影响函数调用时 this 绑定行为的方法，我们称之为 new 绑定。

### 绑定的优先级
按照下面的规则：
1. 函数是否在 new 中调用（new 绑定）？如果是的话 this 绑定的是新创建的对象。var bar = new foo()
2. 函数是否通过 call、apply（显式绑定）或者硬绑定调用？如果是的话，this 绑定的是指定的对象。var bar = foo.call(obj2)
3. 函数是否在某个上下文对象中调用（隐式绑定）？如果是的话，this 绑定的是那个上下文对象。var bar = obj1.foo()
4. 如果都不是的话，使用默认绑定。如果在严格模式下，就绑定到 undefined，否则绑定到全局对象。var bar = foo()

### 绑定例外
1. 把 null 或者 undefined 作为 this 的绑定对象传入 call、apply 或者 bind，这些值在调用时会被忽略，实际应用的是默认绑定规则

2. 赋值表达式 p.foo = o.foo 的返回值是目标函数的引用，因此调用位置是 foo() 而不是
p.foo() 或者 o.foo()。这里会应用默认绑定
```
function foo() {
  console.log( this.a );
}
var a = 2;
var o = { a: 3, foo: foo };
var p = { a: 4 };
o.foo(); // 3
(p.foo = o.foo)(); // 2
```

### this词法
箭头函数并不是使用 function 关键字定义的，而是使用被称为“胖箭头”的操作符 => 定义的。箭头函数不使用 this 的四种标准规则，而是根据外层（函数或者全局）作用域来决定 this
```
function foo() {
  var self = this; // lexical capture of this
  setTimeout( function(){
    console.log( self.a );
  }, 100 );
}
var obj = {
  a: 2
};
foo.call( obj ); // 2
```


### this全面解析 *小结*
如果要判断一个运行中函数的 this 绑定，就需要找到这个函数的直接调用位置。找到之后就可以顺序应用下面这四条规则来判断 this 的绑定对象。
1. 由 new 调用？绑定到新创建的对象。
2. 由 call 或者 apply（或者 bind）调用？绑定到指定的对象。
3. 由上下文对象调用？绑定到那个上下文对象。
4. 默认：在严格模式下绑定到 undefined，否则绑定到全局对象。

一定要注意，有些调用可能在无意中使用默认绑定规则。如果想“更安全”地忽略 this 绑定，你可以使用一个 DMZ 对象，比如 ø = Object.create(null)，以保护全局对象。

ES6 中的箭头函数并不会使用四条标准的绑定规则，而是根据当前的词法作用域来决定this，具体来说，箭头函数会继承外层函数调用的 this 绑定（无论 this 绑定到什么）。这其实和 ES6 之前代码中的self = this 机制一样。


## 对象

### 内容

#### 属性描述符

从 ES5 开始，所有的属性都具备了属性描述符。
```
var myObject = {
  a:2
};
Object.getOwnPropertyDescriptor( myObject, "a" );
// {
//  value: 2,
//  writable: true,
//  enumerable: true,
//  configurable: true
// }
```
使用 Object.defineProperty(..)来添加一个新属性或者修改一个已有属性（如果它是 configurable）并对特性进行设置
```
var myObject = {};
Object.defineProperty( myObject, "a", {
  value: 2,
  writable: true,
  configurable: true,
  enumerable: true
});
myObject.a; // 2
```
1. writable 决定是否可以修改属性的值。
2. configurable 只要属性是可配置的，就可以使用 defineProperty(..) 方法来修改属性描述符。把 configurable 修改成false 是单向操作，无法撤销！除了无法修改，configurable:false 还会禁止删除这个属性
    - tips：即便属性是 configurable:false，我们还是可以把 writable 的状态由 true 改为 false，但是无法由 false 改为 true。
3. 这个描述符控制的是属性是否会出现在对象的属性枚举中（比如 for...in 循环）

#### 不变性
以下的方法创建的都是浅不变形，它们只会影响目标对象和它的直接属性。如果目标对象引用了其他对象（数组、对象、函数，等），其他对象的内容不受影响，仍然是可变的
```
myImmutableObject.foo; // [1,2,3]
myImmutableObject.foo.push( 4 );
myImmutableObject.foo; // [1,2,3,4]
```

##### 对象常量
结合 writable:false 和 configurable:false 就可以创建一个真正的**常量属性**（不可修改、重定义或者删除）
```
var myObject = {};
Object.defineProperty( myObject, "FAVORITE_NUMBER", {
  value: 42,
  writable: false,
  configurable: false
} );
```
##### 禁止扩展
Object.preventExtensions() 静态方法可以防止新属性被添加到对象中（即防止该对象被扩展）。它还可以防止对象的原型被重新指定。
```
var myObject = {
  a:2
};
Object.preventExtensions( myObject );
myObject.b = 3;
myObject.b; // undefined
```

##### 密封
Object.seal(..) 会创建一个“密封”的对象，这个方法实际上会在一个现有对象上调用Object.preventExtensions(..) 并把所有现有属性标记为 configurable:false。所以，密封之后不仅不能添加新属性，也不能重新配置或者删除任何现有属性（虽然可以修改属性的值）。

##### 冻结
Object.freeze(..) 会创建一个冻结对象，这个方法实际上会在一个现有对象上调用Object.seal(..) 并把所有“数据访问”属性标记为 writable:false，这样就无法修改它们的值。

#### Getter Setter
> 如果配置的描述符里面有getter则会忽略value， 如果有setter则会忽略writeable

在 ES5 中可以使用 getter 和 setter 部分改写默认操作
```
var myObject = {
  // 给 a 定义一个 getter
  get a() {
    return 2;
  }
};

Object.defineProperty(
  myObject, // 目标对象
  "b", // 属性名
  { // 描述符
    // 给 b 设置一个 getter
    get: function(){ return this.a * 2 },
    // 确保 b 会出现在对象的属性列表中
    enumerable: true
  }
);

myObject.a; // 2
myObject.b; // 4
```
如果没有定义setter的话，会忽略赋值操作（不会报错）

#### 存在性
```
var myObject = {
  a:2
};
("a" in myObject); // true
("b" in myObject); // false
myObject.hasOwnProperty( "a" ); // true
myObject.hasOwnProperty( "b" ); // false
```
in 还会检查原型链是否有属性， hasOwnProperty只会检查当前对象是否有属性

1. 可枚举性
“可枚举”就相当于“可以出现在对象属性的遍历中”
Object.keys(..) 会返回一个数组，包含所有可枚举属性，Object.getOwnPropertyNames(..)会返回一个数组，包含所有属性，无论它们是否可枚举。
in 和 hasOwnProperty(..) 的区别在于是否查找原型链，然而，Object.keys(..)和 Object.getOwnPropertyNames(..) 都只会查找对象直接包含的属性。

### 遍历
for..in 会遍历所有可枚举的属性（包括原型链上的）

for..of 循环首先会向被访问对象请求一个迭代器对象，然后通过调用迭代器对象的
next() 方法来遍历所有返回值。
数组有内置的 @@iterator，因此 for..of 可以直接应用在数组上。我们使用内置的 @@
iterator 来手动遍历数组，看看它是怎么工作的：
```
var myArray = [ 1, 2, 3 ];
var it = myArray[Symbol.iterator]();
it.next(); // { value:1, done:false }
it.next(); // { value:2, done:false }
it.next(); // { value:3, done:false }
it.next(); // { done:true }
```

## 原型
### [[Prototype]]
解析整个过程
```
myObject.foo = "bar";
```
1. 如果 myObject 对象中包含名为 foo 的普通数据访问属性，这条赋值语句只会修改已有的属性值。
2. 若果 myObject 对象没有 foo 而是存在myObject的 [[Prototype]]上的话，会有如下几种情况
    1. 如果在 [[Prototype]] 链上层存在名为 foo 的普通数据访问属性（参见第 3 章）并且没有被标记为只读（writable:false），那就会直接在 myObject 中添加一个名为 foo 的新属性，它是屏蔽属性。
    2. 如果在 [[Prototype]] 链上层存在 foo，但是它被标记为只读（writable:false），那么无法修改已有属性或者在 myObject 上创建屏蔽属性。如果运行在严格模式下，代码会抛出一个错误。否则，这条赋值语句会被忽略。总之，不会发生屏蔽。
    3. 如果在 [[Prototype]] 链上层存在 foo 并且它是一个 setter，那就一定会调用这个 setter。foo 不会被添加到（或者说屏蔽于）myObject，也不会重新定义 foo 这个 setter。
  
对于 二、三情况下，如果需要在 myOjbect 上设置 foo 属性可以使用Object.defineProperty(...)

### 总结
1. JS里面的继承不像其他一些OO语言的继承，JS的继承更像是 **委托**
2. 如果要访问对象中并不存在的一个属性，[[Get]] 操作就会查找对象内部[[Prototype]] 关联的对象
3. 关联两个对象最常用的方法是使用 new 关键词进行函数调用，在调用中会创建一个关联其他对象的新对象。

## 行为委托

### 思维模型比较
1. 面向对象风格
```
function Foo(who) {
  this.me = who;
}
Foo.prototype.identify = function() {
  return "I am " + this.me;
};
function Bar(who) {
  Foo.call( this, who );
}
Bar.prototype = Object.create( Foo.prototype );
Bar.prototype.speak = function() {
  alert( "Hello, " + this.identify() + "." );
};
var b1 = new Bar( "b1" );
var b2 = new Bar( "b2" );
b1.speak();
b2.speak();
```
2. 对象委托风格
```
Foo = {
  init: function(who) {
    this.me = who;
  },
  identify: function() {
    return "I am " + this.me;
  }
};
Bar = Object.create( Foo );
Bar.speak = function() {
  alert( "Hello, " + this.identify() + "." );
};
var b1 = Object.create( Bar );
b1.init( "b1" );
var b2 = Object.create( Bar );
b2.init( "b2" );
b1.speak();
b2.speak();
```
- 面向对象风格的原型链
![面向对象风格的原型链](oo-proto-chain.png)

- 面向对象委托风格的原型链
![对象委托风格的原型链](od-proto-chain.png)


# 其他一些内容
1. instanceOf 检查左侧的对象的原型链是否跟右侧的函数的prototype是否引用一致
2. 鸭子类型（典型的就是Promise ）： 如果看起来像鸭子，叫起来像鸭子，那它就是鸭子。 验证是否是Promise的方式就是看该对象是否有then函数， 如果有就当成Promise
3. 行为委托对比类继承是一种更少见但是更强大的设计模式，倡导直接创建对象和关联对象，而不是抽象成为类。行为委托认为对象之间是兄弟关系，互相委托，而不是父类和子类的关系。使用对象关联时，所有的对象都是通过 [[Prototype]] 委托互相关联，下面是内省的方法
    - 自省:
    ```
    var Foo = { /* .. */ };
    var Bar = Object.create( Foo );
    var b1 = Object.create( Bar );
    //-----------------------------
    Foo.isPrototypeOf( Bar ); // true
    Object.getPrototypeOf( Bar ) === Foo; // true
    //-----------------------------
    // 让 b1 关联到 Foo 和 Bar
    Foo.isPrototypeOf( b1 ); // true
    Bar.isPrototypeOf( b1 ); // true
    Object.getPrototypeOf( b1 ) === Bar; // true
    ```
    - 优点：语法更简洁，代码结构更加清晰