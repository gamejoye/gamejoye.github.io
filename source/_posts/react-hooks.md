---
title: react常用hook原理
date: 2024-12-21 21:47:46
tags:
  - 前端
  - react
cover: cover.jpg
---

# useTransition
> Transition是 **react18** 引入的用于区分紧急更新和非紧急更新的hook

## 官方demo
1. `useTransition`返回两个参数 isPending 用于标识是否处于等待，下文会结合源码解释
2. startTransition接收一个回调，回调里面的更新会被标记为Transition
3. startTransitio的回调中不会将await以后的dispatch标记为Transition优先级，需要手动再调用startTransition
```
import {useState, useTransition} from 'react';
import {updateQuantity} from './api';

function CheckoutForm() {
  const [isPending, startTransition] = useTransition();
  const [quantity, setQuantity] = useState(1);

  function onSubmit(newQuantity) {
    startTransition(async function () {
      const savedQuantity = await updateQuantity(newQuantity);
      // 这里后面的更新不会被标记为Transiton
      startTransition(() => {
        // 再调用一次startTransition将更新标记为Transition
        setQuantity(savedQuantity);
      });
    });
  }
  // ……
}
```

## 源码实现
mount和update的实现都很简单，就是调用useState hook来设置isPending。
1. mount阶段
```
function mountTransition(): [
  boolean,
  (callback: () => void, options?: StartTransitionOptions) => void,
] {
  const [isPending, setPending] = mountState(false);
  // The `start` method never changes.
  const start = startTransition.bind(null, setPending);
  const hook = mountWorkInProgressHook();
  hook.memoizedState = start;
  return [isPending, start];
}
```
2. update阶段
```
function updateTransition(): [
  boolean,
  (callback: () => void, options?: StartTransitionOptions) => void,
] {
  const [isPending] = updateState(false);
  const hook = updateWorkInProgressHook();
  const start = hook.memoizedState;
  return [isPending, start];
}
```

最重要的还是startTransition的实现
```
function startTransition(setPending, callback, options) {
  const previousPriority = getCurrentUpdatePriority();
  setCurrentUpdatePriority(
    higherEventPriority(previousPriority, ContinuousEventPriority),
  );

  // 首先`setPending(true);`标记等待状态，此次更新并不是Transition优先级
  setPending(true);

  // 这里很重要，这里会替换ReactCurrentBatchConfig，也就是标记后面的更新是Transition
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = {};
  const currentTransition = ReactCurrentBatchConfig.transition;

  try {
    // 上面标记完了，这里的更新是Transition优先级
    setPending(false);
    // callback就是我们在使用hook时传递给startTransition的回调
    callback();
  } finally {
    // 恢复之前的状态
    setCurrentUpdatePriority(previousPriority);
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}
```
> 注意，上面的 <span style="color: red;">startTransitio的回调中不会将await以后的dispatch标记为Transition优先级，需要手动再调用startTransition</span> 其实我们也可以通过源码很明显的知道为什么。
```
try {
  setPending(false);
  callback();
} finally {
  setCurrentUpdatePriority(previousPriority);
  ReactCurrentBatchConfig.transition = prevTransition;
}
```
> 在执行完callback之后，会同步的恢复之前的优先级，而await之后的执行是异步执行的。所以，如果我们需要将await之后的更新标记为Transition我们需要再次调用startTransition

## useTransition VS debounce
debounce实现：
```
var debounce = function(fn, t) {
  let timer = null;
  return function(...args) {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {
      fn(...args);
    }, t);
  }
};
```
- 执行次数
1. `useTransition`返回的startTransiton，如果我们多次调用，回调里面的更新最终都是会被执行的。
2. `debounce`如果在一定时间内被打断了，之前的函数并不会被执行

- 执行时机
1. Transition优先级的更新是可以被打断的，startTransiton里面的更新会等待没有高优先级的更新之后执行
2. `debounce`则是在函数没有在t时间内被再次执行之后执行