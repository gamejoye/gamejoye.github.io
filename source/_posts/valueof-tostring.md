---
title: valueof-tostring
date: 2024-10-10 21:08:38
tags: 前端
cover: cover.jpeg
---
# 概述
valueOf 和 toString 是对象的两个内置方法，它们用于将对象转换为原始值或字符串。每个对象都从 Object.prototype 继承了这两个方法，并且可以根据需要重写它们。

# valueOf
强制数字类型转换和强制基本类型转换优先会调用valueOf

# toString
强制字符串转换会优先调用 toString

# valueOf 与 toString 的区别

| 特性 | valueOf | toString |
| ---- | ---- | ---- |
| 目的 | 返回对象的原始值 | 返回对象的字符串表示 |
| 默认行为 | 返回对象本身（引用），多数对象不会覆盖 |	返回 "[object Object]"，可被覆盖 |
| 调用场景 | 数学运算、比较操作	| 字符串拼接、显示对象信息 |

# 代码
```
const myObj = {
  toString: function () {
    return "Hello World";
  },
  valueOf() {
    return 1;
  }
};

console.log('jkl: '.concat(myObj)); // jkl: Hello World
console.log(myObj + "!!!");         // 1!!!
console.log(myObj + 1);             // 2
console.log(String(myObj));         //Hello World
console.log(+myObj);                // 1

```

# 总结
1. **valueOf** 用于将对象转换为原始值，常用于数学运算或比较。
2. **toString** 用于将对象转换为字符串，常用于字符串拼接或显示对象信息。
3. JavaScript 会根据上下文自动选择调用 **valueOf** 或 **toString**，也可以通过重写这两个方法来控制对象的行为。