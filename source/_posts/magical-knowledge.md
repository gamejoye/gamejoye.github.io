---
title: 那些我第一时间没反应过来的知识
date: 2024-10-10 21:56:48
tags: 
  - 前端
  - 杂项
cover: cover.jpeg
---
## apply, bind联合调用对函数参数进行展开
```
function spread(fn) {
  return Function.apply.bind( fn, null );
}
Promise.all(
  foo( 10, 20 )
)
.then(
  spread( function(x,y){
    console.log( x, y ); // 200 599
  })
)
```
主要在`return Function.apply.bind( fn, null );`这段代码上。

`bind`将`fn`绑定到`apply`函数的`this`上面， 第二个`null`参数则是将`this`绑定到`fn`函数上面

思考一下，如果需要调用 函数原型上的`apply`进行显示绑定`this`，我们会这样调用`fn.apply(null)`，这样调用的话，在`apply`函数里面的`this`其实就是`fn`函数。这样也就能够解释上面`spread`函数里面`bind`的第一个参数是`fn`了

至于第二个参数，其实就是相当于`fn.apply(null)`的调用了