---
title: react useEffect 原理杂谈
date: 2024-09-22 15:36:17
tags: 
  - react
  - 前端
cover: cover.jpeg
---

> 关于react useEffect的一些逻辑梳理

## 初始化阶段
1. 调用mountEffect， 设置fiberFlags
2. mountEffect调用mountWorkInProgrsss把当前effect hook加到currentlyRenderingFiber.memoizedState
3. 此时直到commitRoot阶段effect.destory还是undefinded
4. commitRoot异步调用刷新PassiveEffect， 执行useEffect传递的回调， 同时把回调的返回值设置为effect.destory

## 更新阶段
1. 调用updateEffect
2. updateEffect调用updateWorkInProgress把当前effect hook加到currentlyRenderingFiber.memoizedState
3. 根据prevDeps跟nextDeps， 判断是否需要给当前fiber打上Flag用于commitRoot阶段的处理
4. 此时直到commitRoot阶段effect.destory都还是上一次设置的destory

## 总结
无论是初始化阶段还是更新阶段都需要使用pushEffect设置currentlyRenderingFiber.updateQueue
最终commit阶段会根据hook.tag判断该hook的回调是否需要重新执行