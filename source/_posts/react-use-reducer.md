---
title: react18 updateReducer内部原理
date: 2024-09-22 15:36:57
tags: 
  - react
  - 前端
---

> 本文介绍react18 updateReducer内部实现的细节
> 注意！！！： 本章只讨论state类型的hooks

## 前置知识
在介绍实现细节前， 先弄清楚几个概念

1. Hook
```typescript
export type Hook = {|
  memoizedState: any, // 存当前hook的状态 useState返回的数组的第一个参数
  baseState: any, // 在优先级管理会用到 下面会介绍
  baseQueue: Update<any, any> | null, // 同样的 在优先级管理会用到 下面会介绍
  queue: any,  // 同样下面会介绍
  next: Hook | null, // 指向当前fiber的hooks链表的下一个hook

|};
// 每个函数类型的fiber都有各自的hooks链表 存在fiber.memoizedState里面
```

2. UpdateQueue（Hook.queue字段）
```typescript
export type UpdateQueue<S, A> = {
  pending: Update<S, A> | null, // 待处理的update 每次dispatch时添加的update就是加到这里
  dispatch: Dispatch<A> | null, // dispatch函数 也就是useState返回的数组的第二个参数
  /* 忽略其他属性 */
}
```

3. Update（这是每次dispatch(useState返回的数组的第二个参数)调用的时候会添加的更新）
```typescript
type Update<S, A> = {|
  lane: Lane, // 每个update的优先级
  action: A,
  hasEagerState: boolean,
  eagerState: S | null,
  next: Update<S, A>, // 指向当前hook的下一个update
|};
```

## 源码部分
```typescript
function updateReducer(
  reducer,
  initialArg,
): [S, Dispatch<A>] {
  // 这一部分是hook的clone
  // 把当前渲染在屏幕上的fiber的跟当前hook节点对应的内容拷贝下来
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  // currentHook是当前已经渲染在页面上的fiber节点的hooks链表的某个hook state节点
  const current: Hook = (currentHook: any);
  let baseQueue = current.baseQueue;

  // 获取当前屏幕用户触发的更新
  // 合并baseQueue和pendingQueue
  // 需要注意的是pendingQueue和baseQueue都是指向最后一个update
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    if (baseQueue !== null) {
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      // 将baseQueue的最后一个节点指向pendingQueue的第一个节点
      baseQueue.next = pendingFirst;
      // 将pendingQueue的最后一个节点指向baseQueue的最后一个节点
      pendingQueue.next = baseFirst;
      // 最终 整个update链表将形成一个环形链表
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }

  // 以上是合并baseQueue和pendingQueue
  // 下面是处理baseQueue的逻辑

  if (baseQueue !== null) {
    // 我们有update链表需要处理
    const first = baseQueue.next;
    let newState = current.baseState;

    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;
    do {
      const updateLane = update.lane;
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 当前这个update的优先级不够
        const clone: Update<S, A> = {
          lane: updateLane,
          action: update.action,
          hasEagerState: update.hasEagerState,
          eagerState: update.eagerState,
          next: (null: any),
        };
        if (newBaseQueueLast === null) {
          // 之前没有遇到过优先级不够的update
          // 设置newBaseQueueLast为当前优先级不够的update
          newBaseQueueFirst = newBaseQueueLast = clone;
          // 同时 设置newBaseState为当前的newState 因为从现在开始的每个update都要加入newBaseQueue
          newBaseState = newState;
        } else {
          // 将update加入newBaseQueue
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        /* 其他一些不是本文章的重点 */
      } else {
        // 判断之前是否遇到过优先级不够的update
        if (newBaseQueueLast !== null) {
          // 前面说过了：从第一个优先级不够的update开始 到最后一个update都要加入newBaseQueue
          const clone: Update<S, A> = {
            lane: NoLane, // 注意！！ 这里设置为NoLane很重要， 这可以保证下一次的baseQueue中， 该update一定能够被判断为有足够的优先级
            action: update.action,
            hasEagerState: update.hasEagerState,
            eagerState: update.eagerState,
            next: (null: any),
          };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        
        // 执行当前的update
        const action = update.action;
        newState = reducer(newState, action);
      }
      update = update.next;
    } while (update !== null && update !== first);

    if (newBaseQueueLast === null) {
      // 从来没有遇到过优先级不够的update 那么newBaseState就是最新的state
      newBaseState = newState;
    } else {
      // 把newBaseQueue首尾连接起来 形成环形链表
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }

    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;
  }
  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```
至此就是updateReducer源码的解析
如果你还不理解的话， 这里我将提供一个例子来说（假设我们已经处理完了合并baseQueue和pendingQueue了）
合并完的baseQueue: 1 -> 2 -> <span style="color: red;">3</span> -> 4 -> <span style="color: red;">5</span> -> 6
这里红色的表示低优先级的update 黑色表示高优先级的update
我们在处理baseQueue的过程:

1. 遇到update(1)，高优先级update
  - newBaseQueue = null
  - newBaseState = null
  - newState = 1

2. 遇到update(2)，高优先级update
  - newBaseQueue = null
  - newBaseState = null
  - newState = 2

3. 遇到update(3)，低优先级 update
  - newBaseQueue = <span style="color: red;">3</span>
  - newBaseState = newState(上一次的newState) = 2
  - newState = 2

4. 遇到update(4)，高优先级 update
  - newBaseQueue = <span style="color: red;">3</span> -> 4
  - newBaseState = 2
  - newState = 4

5. 遇到update(5)，低优先级 update
  - newBaseQueue = <span style="color: red;">3</span> -> 4 -> <span style="color: red;">5</span>
  - newBaseState = 2
  - newState = 4

6. 遇到update(6)， 高优先级update
  - newBaseQueue = <span style="color: red;">3</span> -> 4 -> <span style="color: red;">5</span> -> 6
  - newBaseState = 2
  - newState = 6

7. 结束将newBaseQueue首尾相连( newBaseQueueLast.next = newBaseQueueFirst )
  - newBaseQueue = <span style="color: red;">3</span> -> 4 -> <span style="color: red;">5</span> -> 6 -> <span style="color: red;">3</span>
  - newBaseState = 2
  - newState = 6

## 参考资料
- [react源码](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js)