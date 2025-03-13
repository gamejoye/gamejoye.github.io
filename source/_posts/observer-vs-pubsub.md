---
title: 观察者模式 vs 发布-订阅模式的区别
date: 2025-03-13 20:42:12
tags:
  - 设计模式
---

## 观察者模式实现
```js
class Subject {
  constructor() {
    this.observers = [];
  }

  // 添加观察者
  addObserver(observer) {
    this.observers.push(observer);
  }

  // 通知所有观察者
  notifyObservers(message) {
    this.observers.forEach(observer => observer.update(message));
  }
}

class Observer {
  update(message) {
    console.log('Received message:', message);
  }
}

// 创建被观察者和观察者
const subject = new Subject();
const observer1 = new Observer();
const observer2 = new Observer();

// 注册观察者
subject.addObserver(observer1);
subject.addObserver(observer2);

// 通知观察者
subject.notifyObservers('Hello Observers!');

```

## 发布-订阅模式
```js
class EventBus {
  constructor() {
    this.subscribers = {};
  }

  // 订阅消息
  subscribe(event, callback) {
    if (!this.subscribers[event]) {
      this.subscribers[event] = [];
    }
    this.subscribers[event].push(callback);
  }

  // 发布消息
  publish(event, data) {
    if (this.subscribers[event]) {
      this.subscribers[event].forEach(callback => callback(data));
    }
  }
}

// 创建发布-订阅系统
const bus = new EventBus();

// 订阅者1
bus.subscribe('message', (data) => {
  console.log('Subscriber 1 received:', data);
});

// 订阅者2
bus.subscribe('message', (data) => {
  console.log('Subscriber 2 received:', data);
});

// 发布者发布消息
bus.publish('message', 'Hello Subscribers!');

```

## 区别
| 特性 | 观察者模式 | 发布订阅模式 |
| ---- | ---- | ---- |
| 耦合度 | 观察者和被观察者之间存储之间依赖关系（被观察者通常会存下观察者的引用） | 发布者和订阅者之间没有联系，通过EventBus通信 |
| 通信方式 | 被观察者直接通知观察者 | 发布者通过EventBus通知订阅者 |
| 结构 | 观察者手动注册到被观察者中 | 发布者和订阅者无需知道对方的存在 |
| 订阅内容 | 观察者通常订阅的是被观察者的所有状态变化 | 订阅者可以选择只关注某些特定的事件 |