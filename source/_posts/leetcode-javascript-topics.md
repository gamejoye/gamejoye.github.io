---
title: 力扣javascript专题
date: 2024-10-04 01:32:06
tags: 
  - 前端
  - 算法
---

## 概述
收集一些常见的前端手撕题

## [转换回调函数为Promise函数](https://leetcode.cn/problems/convert-callback-based-function-to-promise-based-function/description/)

### 描述
> 编写一个函数，接受另一个函数 fn ，并将基于回调函数的函数转换为基于 Promise 的函数

> 一个可以传递给 promisify 的函数示例
```
function sum(callback, a, b) {
  if (a < 0 || b < 0) {
    const err = Error('a and b must be positive');
    callback(undefined, err);
  } else {
    callback(a + b);
  }
}
```
### 代码骨架
```
/**
 * @param {Function} fn
 * @return {Function<Promise<number>>}
 */
var promisify = function(fn) {
  return async function(...args) {
  }
};

/**
 * const asyncFunc = promisify(callback => callback(42));
 * asyncFunc().then(console.log); // 42
 */
```

### 思路
1. fn函数的第一个参数是callback回调，callback接受 data和error
2. 在callback里面根据data和error调用resolve或者reject

### 解决方案
```
var promisify = function (fn) {
  return async function (...args) {
    return new Promise((resolve, reject) => {
      fn(
        (data, error) => {
          if (error) reject(error)
          else resolve(data)
        },
        ...args,
      )
    })
  }
};
```