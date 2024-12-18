---
title: 《你不知道的Javascript》记录 part-2
date: 2024-10-03 20:58:36
tags: 前端
cover: cover.jpeg
---

# 类型与语法
## 值
### 数组
#### 稀疏数组
```
var a = [ ];
a[0] = 1;
// 此处没有设置a[1]单元
a[2] = [ 3 ];
a[1]; // undefined
a.length; // 3
```
**a[1]实际上跟undefined还是有区别的**

#### 类数组
1. 具有索引属性：类数组对象的元素以数字索引的方式存储，比如 obj[0]、obj[1] 等。
2. 具有 length 属性：类数组对象通常拥有 length 属性，表示它包含的元素个数。
3. 不是数组实例：类数组对象并不是 Array 的实例，所以它没有数组的一些内置方法，如 push()、pop()、map() 等。


### 值和引用
1. 简单值（即标量基本类型值，scalar primitive）总是通过值复制的方式来赋值 / 传递，包括null、undefined、字符串、数字、布尔、symbol、 bigint。
2. 复合值（compound value）——对象和函数，则总是通过引用复制的方式来赋值 / 传递。

## 原生函数
- String()
- Number()
- Boolean()
- Array()
- Object()
- Function()
- RegExp()
- Date()
- Error()
- Symbol()

通过构造函数（如 new String("abc")）创建出来的是封装了基本类型值（如 "abc"）的封装对象

```
var a = new String( "abc" );
typeof a; // 是"object"，不是"String"
a instanceof String; // true
Object.prototype.toString.call( a ); // "[object String]"
```

### 封装对象的包装和拆封
1. JavaScript 会自动为基本类型值包装一个封装对象
2. 拆封封装对象调用valueOf函数或者强制转换

```
// 自动包装
const s = '123'
s.length // 3

// 拆封
const os = new String('abc')
os.valueOf()    // abc
'' + os         // abc
```

# 异步与性能
## Promise
### Promise信任问题

#### Promise调用过早
什么意思？通俗的解释就是一个任务有时同步完成有时异步完成

立即完成的 Promise（类似于 `new Promise(function(resolve){ resolve(42); })`）也无法被同步观察到，也就是 then之后的任务一定是会以异步的方式被调用

#### Promise调用过晚
Promise 创建对象调用 resolve(..) 或 reject(..) 时，这个 Promise 的then(..) 注册的观察回调就会被自动调度。可以确信，这些被调度的回调在下一个异步事件点上一定会被触发

考虑下面这种情况
```
p.then(function () {
  p.then(function () {
    console.log("C");
  });
  console.log("A");
});
p.then(function () {
  console.log("B");
});
// A B C 
```
"C" 无法打断或抢占 "B"，这是因为 Promise 的运作方式。因为 "C" 的任务被注册到下一轮去了

tips：**永远都不应该依赖于不同 Promise 间回调的顺序和调度** 也就是对于一轮里面的Promise任务，不要假定这一轮任务的执行顺序


#### Promise未调用问题
这里优雅的解决办法可以通过`Promise.race`设置超时来实现

#### Promise建立信任
```
var p = {
  then: function (cb, errcb) {
    cb(42);
    errcb("evil laugh");
  }
};
p
  .then(
    function fulfilled(val) {
      console.log(val); // 42
    },
    function rejected(err) {
      // 不应该运行！
      console.log(err);
    }
  );
```
上面的代码不符合预期且是同步执行的， 需要进行**Promise的规范化**

改造后的
```
Promise.resolve(p)
  .then(
    function fulfilled(val) {
      console.log(val); // 42
    },
    function rejected(err) {
      // 永远不会到达这里
    }
  );
```
`Promise.resolve(p)` 如果p是一个Promise的话，直接就返回该Promise，否则会对p进行规范化，保证符合Promise的规范

## 生成器
### 生成器产生值
#### 可迭代协议
可迭代协议允许JavaScript对象定义或定制它们的迭代行为，例如，在一个`for..of`结构中，哪些值可以被遍历到。一些内置类型同时是内置的可迭代对象，并且有默认的迭代行为，比如 Array 或者 Map，而其他内置类型则不是（比如 Object）

要成为可迭代对象，该对象必须实现`[Symbol.iterator]()`方法

##### [Symbol.iterator]
> 一个无参数的函数，其返回值为一个符合**迭代器协议**的对象

#### 迭代器协议
##### next(value?)
返回一个 `IteratorResult` 接口类型的对象

```
// IteratorResult 接口类型
{
  done: boolean,
  value: any
}
```
`next`函数传递的value参数是可选的，传递给生成器 `next` 方法的值将成为相应 `yield` 表达式的值

##### return(value?)
无参数或者接受一个参数的函数，并返回符合 `IteratorResult` 接口的对象，其 value 通常等价于传递的 value，并且 done 等于 true。调用这个方法表明迭代器的调用者不打算调用更多的 next()，并且可以进行清理工作。

##### throw(exception?)
无参数或者接受一个参数的函数，并返回符合 `IteratorResult` 接口的对象，通常 done 等于 true。调用这个方法表明迭代器的调用者监测到错误的状况


#### 迭代器实现
```
var something = (function () {
  var nextVal;
  return {
    // for..of循环需要
    [Symbol.iterator]: function () { return this; },
    // 标准迭代器接口方法
    next: function () {
      if (nextVal === undefined) {
        nextVal = 1;
      }
      else {
        nextVal = (3 * nextVal) + 6;
      }
      return { done: false, value: nextVal };
    }
  };
})();
```
调用
```
for (var v of something) {
  console.log(v);
  if (v > 500) {
    break;
  }
}
```
`for..of` 循环在每次迭代中自动调用 next()，它不会向 next() 传入任何值，并且会在接收
到 done:true 之后自动停止

#### 生成器迭代器
##### 示例
```
function* myGenerator() {
  try {
    yield 1;
    yield 2;
  } catch (e) {
    console.log('Error caught:', e);
  }
  yield 3;
}

const gen = myGenerator();

console.log(gen.next());    // { value: 1, done: false }
console.log(gen.throw('Something went wrong')); // Error caught: Something went wrong
// { value: 3, done: false }
console.log(gen.next());    // { value: undefined, done: true }
```
`something`是**生成器** 并不是一个**可迭代对象**
`something()`会产生一个**可迭代对象**
```
function *something() {
  var nextVal;
  while (true) {
    if (nextVal === undefined) {
      nextVal = 1;
    }
    else {
      nextVal = (3 * nextVal) + 6;
    }
    yield nextVal;
  }
}
for (var v of something()) {
  console.log( v );
  if (v > 500) {
    break;
  }
}
```

### 异步迭代生成器


# 其他一些要点
1. `Promise.resolve`用于规划化Promise，如果传递给resolve的是一个Promise或者thenable，这会对其进行展开
2. `Promise.reject`跟`Promise.resolve`不同，reject如果接受Promise或者thenable并不会对其进行展开，而是直接把这个内容当成reject的理由
3. try-catch无法跨异步捕获错误
```
function foo() {
  setTimeout(function () {
    baz.bar();
  }, 100);
}
try {
  foo();
  // 后面从 `baz.bar()` 抛出全局错误
}
catch (err) {
  // 永远不会到达这里
}
```
4. 生成器函数第一次传递的value是无效的
