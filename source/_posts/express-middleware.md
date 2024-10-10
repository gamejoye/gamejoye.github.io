---
title: express中间件
date: 2024-10-10 21:11:35
tags: 
  - 全栈
  - nodejs
cover: cover.jpeg
---
## 流程
![express流程](output.png)

## 格式
```
function(req,res,next){
  next();
}
```
中间件函数的形参列表中，必须包含`next`参数，而路由处理函数中只包含`res`和`req`

## 使用方式
### 全局注册
```
const express = require('express')
const app = express()

const mw = function (req, res, next) {
  console.log('mw mw mw')
  next()
}

// 将 mw 注册为全局生效的中间件
app.use(mw)

app.listen(80, ()=> {
  console.log('run on 80')
})
```

多个全局中间件
```
const express = require('express')
const app = express()

app.use(function (req, res, next) {
  console.log('调用了第1个全局中间件')
  next()
})

app.use(function (req, res, next) {
  console.log('调用了第2个全局中间件')
  next()
})

app.get(`/`, (req, res) => {
  res.send('home page ')
})

app.listen(80, ()=> {
  console.log('run on 80')
})
```

### 局部注册
```
const express = require('express')
const app = express()

// 定义一个中间件函数
const mw1 = function(req, res, next) {
  console.log('这是中间件函数')
  next()
}

// mw1 中间件只会在当前路由中生效
app.get('/', mw1, function(req, res){
  console.log('经过中间件')
  console.log('home pag')
  res.send('home page')
})

// mw1 中间件不会影响这个路由
app.get('/user', function(req, res){
  console.log('不经过中间件')
  console.log('user page')
  res.send('user page')
})

app.listen(80, ()=> {
  console.log('run on 80')
})
```

多个局部中间价
```
// 等价
app.get('/', mw1,mw2, (req, res) => {})
app.get('/', [mw1,mw2], (req, res) => {})
```

## 注意
1. 在路由前注册中间价
2. 中间价执行完毕后记得执行`next()`
3. 连续调用多个中间价，多个中间件之间，共享`req`和`res`对象