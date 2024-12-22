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
3. ~~startTransitio的回调中不会将await以后的dispatch标记为Transition优先级，需要手动再调用startTransition~~
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

最重要的还是startTransition的实现 (react v18.x.x)
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
>~~注意，上面的 <span style="color: red;">startTransitio的回调中不会将await以后的dispatch标记为Transition优先级，需要手动再调用startTransition</span> 其实我们也可以通过源码很明显的知道为什么。~~
```
try {
  setPending(false);
  callback();
} finally {
  setCurrentUpdatePriority(previousPriority);
  ReactCurrentBatchConfig.transition = prevTransition;
}
```
> ~~在执行完callback之后，会同步的恢复之前的优先级，而await之后的执行是异步执行的。所以，如果我们需要将await之后的更新标记为Transition我们需要再次调用startTransition~~

> 在react19中，已经添加异步函数的支持，以自动处理待定状态、错误、表单和乐观更新。

以下是react19的实现，主要设计3个主要函数 `startTransition` `dispatchOptimisticSetState` `updateReducerImpl`

1. startTransition
```
function startTransition<S>(
  fiber: Fiber,
  queue: UpdateQueue<S | Thenable<S>, BasicStateAction<S | Thenable<S>>>,
  pendingState: S, // 这里是true 由hook传进来
  finishedState: S, // 这里是false 由hook传进来
  callback: () => mixed,
  options?: StartTransitionOptions,
): void {
  const previousPriority = getCurrentUpdatePriority();
  setCurrentUpdatePriority(
    higherEventPriority(previousPriority, ContinuousEventPriority),
  );

  const prevTransition = ReactSharedInternals.T;
  const currentTransition: BatchConfigTransition = {};

  if (enableAsyncActions) {
    // 设置Transition环境 调度乐观更新
    ReactSharedInternals.T = currentTransition;
    dispatchOptimisticSetState(fiber, false, queue, pendingState);
  } else {
    // 这里就是react18会走的逻辑
  }

  try {
    if (enableAsyncActions) {
      const returnValue = callback();

      if (
        returnValue !== null &&
        typeof returnValue === 'object' &&
        typeof returnValue.then === 'function'
      ) {
        const thenable = ((returnValue: any): Thenable<mixed>);
        // 这里会为promise添加监听 注册Promise.resolve(thenable).then(() => result)
        // 为后面调用.then的那些回调传入finishedState
        const thenableForFinishedState = chainThenableValue(
          thenable,
          finishedState,
        );
        // 将更新添加到updateQueue等待调度
        dispatchSetStateInternal(
          fiber,
          queue,
          (thenableForFinishedState: any),
          requestUpdateLane(fiber),
        );
      } else {
        dispatchSetStateInternal(
          fiber,
          queue,
          finishedState,
          requestUpdateLane(fiber),
        );
      }
    } else {
      // Async actions are not enabled.
      dispatchSetStateInternal(
        fiber,
        queue,
        finishedState,
        requestUpdateLane(fiber),
      );
      callback();
    }
  } catch (error) {
    // 错误的处理就暂且不看
  } finally {
    // 这里就没有 setPending(false); 了
    setCurrentUpdatePriority(previousPriority);

    ReactSharedInternals.T = prevTransition;
  }
}
```

2. dispatchOptimisticSetState
```
function dispatchOptimisticSetState<S, A>(
  fiber: Fiber,
  throwIfDuringRender: boolean,
  queue: UpdateQueue<S, A>,
  action: A,
): void {
  const transition = requestCurrentTransition();

  // 调度一个同步优先级的update
  // revertLane主要就是用于updateReducerImpl识别是否是一个乐观更新
  const update: Update<S, A> = {
    // An optimistic update commits synchronously.
    lane: SyncLane,
    // After committing, the optimistic update is "reverted" using the same
    // lane as the transition it's associated with.
    revertLane: requestTransitionLane(transition),
    action,
    hasEagerState: false,
    eagerState: null,
    next: (null: any),
  };

  if (isRenderPhaseUpdate(fiber)) {
    // ...
  } else {
    const root = enqueueConcurrentHookUpdate(fiber, queue, update, SyncLane);
    if (root !== null) {
      // 这里直接调度同步更新，保证Sync更新是一定处于Transition更新之前
      startUpdateTimerByLane(SyncLane);
      scheduleUpdateOnFiber(root, fiber, SyncLane);
      // Optimistic updates are always synchronous, so we don't need to call
      // entangleTransitionUpdate here.
    }
  }
}
```

3. updateReducerImpl 的内容很长我只截取`useTransition`会涉及到的
```
function updateReducerImpl<S, A>(
  hook: Hook,
  current: Hook,
  reducer: (S, A) => S,
): [S, Dispatch<A>] {
  const queue = hook.queue;

  queue.lastRenderedReducer = reducer;
  let baseQueue = hook.baseQueue;

  const baseState = hook.baseState;
  if (baseQueue === null) {
    // ...
  } else {
    // We have a queue to process.
    const first = baseQueue.next;
    let newState = baseState;

    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast: Update<S, A> | null = null;
    let update = first;
    let didReadFromEntangledAsyncAction = false;
    do {
      const updateLane = removeLanes(update.lane, OffscreenLane);
      const isHiddenUpdate = updateLane !== update.lane;

      const shouldSkipUpdate = isHiddenUpdate
        ? !isSubsetOfLanes(getWorkInProgressRootRenderLanes(), updateLane)
        : !isSubsetOfLanes(renderLanes, updateLane);

      if (shouldSkipUpdate) {
        // Priority is insufficient. Skip this update. If this is the first
        // skipped update, the previous update/state is the new base
        // update/state.
        const clone: Update<S, A> = {
          lane: updateLane,
          revertLane: update.revertLane,
          action: update.action,
          hasEagerState: update.hasEagerState,
          eagerState: update.eagerState,
          next: (null: any),
        };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }

        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane,
        );
        markSkippedUpdateLanes(updateLane);
      } else {
        // This update does have sufficient priority.

        // 这里查看是否是一个乐观更新，通过reverLane属性来检查
        const revertLane = update.revertLane;
        if (!enableAsyncActions || revertLane === NoLane) {
          // 不是乐观更新
          if (newBaseQueueLast !== null) {
            const clone: Update<S, A> = {
              // This update is going to be committed so we never want uncommit
              // it. Using NoLane works because 0 is a subset of all bitmasks, so
              // this will never be skipped by the check above.
              lane: NoLane,
              revertLane: NoLane,
              action: update.action,
              hasEagerState: update.hasEagerState,
              eagerState: update.eagerState,
              next: (null: any),
            };
            newBaseQueueLast = newBaseQueueLast.next = clone;
          }

          if (updateLane === peekEntangledActionLane()) {
            didReadFromEntangledAsyncAction = true;
          }
        } else {
          // This is an optimistic update. If the "revert" priority is
          // sufficient, don't apply the update. Otherwise, apply the update,
          // but leave it in the queue so it can be either reverted or
          // rebased in a subsequent render.
          // 这是乐观更新
          if (isSubsetOfLanes(renderLanes, revertLane)) {
            // The transition that this optimistic update is associated with
            // has finished. Pretend the update doesn't exist by skipping
            // over it.
            update = update.next;

            // Check if this update is part of a pending async action. If so,
            // we'll need to suspend until the action has finished, so that it's
            // batched together with future updates in the same action.
            if (revertLane === peekEntangledActionLane()) {
              didReadFromEntangledAsyncAction = true;
            }
            continue;
          } else {
            const clone: Update<S, A> = {
              // Once we commit an optimistic update, we shouldn't uncommit it
              // until the transition it is associated with has finished
              // (represented by revertLane). Using NoLane here works because 0
              // is a subset of all bitmasks, so this will never be skipped by
              // the check above.
              lane: NoLane,
              // Reuse the same revertLane so we know when the transition
              // has finished.
              revertLane: update.revertLane,
              action: update.action,
              hasEagerState: update.hasEagerState,
              eagerState: update.eagerState,
              next: (null: any),
            };
            if (newBaseQueueLast === null) {
              newBaseQueueFirst = newBaseQueueLast = clone;
              newBaseState = newState;
            } else {
              newBaseQueueLast = newBaseQueueLast.next = clone;
            }
            // Update the remaining priority in the queue.
            // TODO: Don't need to accumulate this. Instead, we can remove
            // renderLanes from the original lanes.
            currentlyRenderingFiber.lanes = mergeLanes(
              currentlyRenderingFiber.lanes,
              revertLane,
            );
            markSkippedUpdateLanes(revertLane);
          }
        }

        // Process this update.
        const action = update.action;
        if (shouldDoubleInvokeUserFnsInHooksDEV) {
          reducer(newState, action);
        }
        if (update.hasEagerState) {
          // If this update is a state update (not a reducer) and was processed eagerly,
          // we can use the eagerly computed state
          newState = ((update.eagerState: any): S);
        } else {
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }

    // Mark that the fiber performed work, but only if the new state is
    // different from the current state.
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();

      // 这里就是最重要的 如果状态变了并且依赖于正在进行的async action，直接抛出promise等待suspense处理
      if (didReadFromEntangledAsyncAction) {
        const entangledActionThenable = peekEntangledActionThenable();
        if (entangledActionThenable !== null) {
          // TODO: Instead of the throwing the thenable directly, throw a
          // special object like `use` does so we can detect if it's captured
          // by userspace.
          throw entangledActionThenable;
        }
      }
    }

    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;

    queue.lastRenderedState = newState;
  }

  if (baseQueue === null) {
    // `queue.lanes` is used for entangling transitions. We can set it back to
    // zero once the queue is empty.
    queue.lanes = NoLanes;
  }

  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```

## useTransition VS debounce VS throttle
### useTransition表现出debounce效果
高优先级更新会中断低优先级更新，优先处理。在react19中，react在`dispatchOptimisticSetState`中，会先注册`Sync`优先级的`setPending(true)`的更新，然后还会注册一个`Transition`优先级的`setPending(false)`更新，以及我们在`callback`里面调度的`Transition`优先级更新

### useTransition表现出throttle效果
react为每个不同优先级的更新都设置了过期时间，优先级越低过期时间越长，如果某个任务在超过了过期时间还没被执行的话，优先级会被设置为ImmediatePriority以达到优先处理
```
export const userBlockingPriorityTimeout = 250;
export const normalPriorityTimeout = 5000;
export const lowPriorityTimeout = 10000;
```
### debounce和throttle的实现
> 题目描述：函数防抖 方法是一个函数，它的执行被延迟了 t 毫秒，如果在这个时间窗口内再次调用它，它的执行将被取消。

- debounce
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
> 题目描述：节流 函数首先立即被调用，然后在 t 毫秒的时间间隔内不能再次执行，但应该存储最新的函数参数，以便在延迟结束后使用这些参数调用 fn 。
- throttle
```
var throttle = function(fn, t) {
  let lastArgs = null;
  let timer = null;
  return function throttled(...args) {
    lastArgs = args;
    if (!timer) {
      fn(...lastArgs);
      lastArgs = null;

      timer = setTimeout(() => {
        timer = null;
        if (lastArgs) {
          throttled(...lastArgs);
        }
      }, t);
    }
  }
};
```