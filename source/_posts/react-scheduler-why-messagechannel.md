---
title: React Scheduler 调度为什么使用 MessageChannel
date: 2025-03-14 16:01:52
tags:
  - react
  - 前端

---

## 为什么需要 React Scheduler 调度

如果组件Render或者组件树节点非常多的情况下，一次loop处理所有节点会非常耗时，并且占用主线程导致页面无法交互

React Scheduler可以结合 React Fiber进行任务的调用，使得能够独立执行每个VDOM的更新，并且能够中断和恢复。

## React Scheduler 调度过程
在了解MessageChannel之前，我们先结合源码了解一下React调度过程

在源码中，schedulePerformWorkUntilDeadline就是调度一个messageChannel，在messageChannel的回调中执行performWorkUntilDeadline函数
```js
if (typeof MessageChannel !== 'undefined') {
  // DOM and Worker environments.
  // We prefer MessageChannel because of the 4ms setTimeout clamping.
  const channel = new MessageChannel();
  const port = channel.port2;
  channel.port1.onmessage = performWorkUntilDeadline;
  schedulePerformWorkUntilDeadline = () => {
    port.postMessage(null);
  };
}
```

请注意scheduledHostCallback实际就是flushWork函数
```js
function requestHostCallback(callback) {// 过期任务请求调度
  
  scheduledHostCallback = callback;
  //判断是否有messageChanel在运行
  if (!isMessageLoopRunning) { //初始为false
    isMessageLoopRunning = true;
    schedulePerformWorkUntilDeadline();
  }
}

// ...
requestHostCallback(flushWork);
```

这是调度逻辑
```js
const performWorkUntilDeadline = () => { // 调度时候执行的函数
  if (scheduledHostCallback !== null) { // scheduledHostCallback为flushWork
    const currentTime = getCurrentTime();
    // Keep track of the start time so we can measure how long the main thread
    // has been blocked.
    startTime = currentTime;
    const hasTimeRemaining = true;

    // If a scheduler task throws, exit the current browser task so the
    // error can be observed.
    //
    // Intentionally not using a try-catch, since that makes some debugging
    // techniques harder. Instead, if `scheduledHostCallback` errors, then
    // `hasMoreWork` will remain true, and we'll continue the work loop.
    let hasMoreWork = true;
    try {
      // scheduledHostCallback为我们requestHostCallback传入的函数 flushwork，实则执行 workLoop
      hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
    } finally {
      // 表示是否还有任务需要执行，taskqueue不为空
      if (hasMoreWork) {
        // If there's more work, schedule the next message event at the end
        // of the preceding one.
        // 还有剩余的任务，则重新发起调度
        schedulePerformWorkUntilDeadline();
      } else {
        // hasMoreWork为false表示taskqueue执行完了
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      }
    }
  } else {
    isMessageLoopRunning = false;
  }
  // Yielding to the browser will give it a chance to paint, so we can
  // reset this.
  needsPaint = false;
};
```

其余部分不重要就不附带源码了，稍微总结一下过程：
1. React Scheduler 通过 MessageChannel调度 performWorkUntilDeadline 执行 Task
2. performWorkUntilDeadline 执行 flushWork， flushWork 实际执行 workLoop
3. flushWork返回值为是否还有任务
4. performWorkUntilDeadline 会根据 flushWork 的返回值决定是否继续调度（schedulePerformWorkUntilDeadline）

## Scheduler 与 MessageChannel

为什么Scheduler需要使用MessageChannel？
Scheduler需要满足
1. 能够暂停JS执行，主线程返回给浏览器
2. 能够在未来重新调度该任务

### 宏任务
事件循环loop每次执行一个宏任务。
本轮注册的宏任务会在下轮事件循环中执行，不会阻塞本次更新

### 微任务
每次执行宏任务完都会检查是否还有新的微任务，执行，直到没有新的微任务
本轮注册的微任务会在本轮事件循环中执行，会阻塞本次更新

所以？我们需要：能够注册宏任务的API。

> MessageChannel的目的就是为了产生宏任务

## 为什么不使用 setTimeout(callback, 0) 产生宏任务？

setTimeout(callback, 0) 会产生宏任务，但如果`setTimeout递归层数太深`会有4ms的限制


## 总结
Scheduler需要满足
1. 能够暂停JS执行，主线程返回给浏览器
2. 能够在未来重新调度该任务

微任务为什么不行
1. 微任务会在本轮事件循环全部执行完毕，达不到将主线程还给浏览器

setTimeout为什么不行
1. 递归太深会有4ms限制


