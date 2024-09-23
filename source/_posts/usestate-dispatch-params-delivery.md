---
title: useState dispatch 参数传递方式
date: 2024-09-22 15:37:11
tags: 
  - react
  - 前端
cover: cover.jpeg
---

在 **React** 的 **useState** hook中我们可以传递函数或者其state对应的类型。 以下是他们的区别：

## 当传递的参数是state对应的类型的时候
```javascript
const [state, setState] = useState(0);
// state => 0
setState(1);
// state => 0
setState(2)
// state => 0
```
这种方式在React18里面会被批量化处理，实际只会触发一次dispatch，同时在当时的环境中state一直都是0，直到下次commitRoot完成

## 当传递的参数是函数的时候
```javascript
const [state, setState] = useState(0);
setState((state) => {
  // state => 0
  return state + 1;
});
setState((state) => {
  // state => 1
  return state + 2;
});
setState((state) => {
  // state => 3
  return state + 3;
});
// state 0
```
这种方式在React18里面同样也会被批量化处理，实际只会触发一次dispatch。但在对应函数里面我们可以获得state的最新状态

## 疑问
发现了一个有趣的现象，但是我无法解释。看下面这段代码：
```javascript
import { useState } from "react";

function App() {

  const [state1, setState1] = useState(0);

  const handleOnClick1 = () => {
    setState1((newState) => {
      console.log('newState1:', newState);
      return newState + 1;
    })
    setState1((newState) => {
      console.log('newState2:', newState);
      return newState + 2;
    })
    setState1((newState) => {
      console.log('newState3:', newState);
      return newState + 3;
    })
    console.log('state1:', state1);
  }
  return (
    <div>
      <button onClick={handleOnClick1}>点击-1</button>
    </div>
  );
}

export default App;
```

当我们点击按钮的时候， 我们会期望的到这样的一个输出：
> state1 0
> newState1 0
> newState2 1
> newState3 3

但是实际的输出却是：
> newState1 0
> state1 0
> newState2 1
> newState3 3

但是当我们第二次点击的时候会得到：
> state1 6
> newState1 6
> newState2 7
> newState3 9

按照源码来看，在我们函数里面同步执行完毕之后， React Scheduler才会调度更新执行我们dispatch的actions呀。 希望有人帮我解答一下。
> React版本：**18.3.1**