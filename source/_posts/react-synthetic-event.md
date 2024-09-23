---
title: react合成事件
date: 2024-09-22 15:35:16
tags: 
  - react
  - 前端
cover: cover.jpeg
---

## 介绍
**React**基于浏览器的事件机制自身实现了一套事件机制，包括事件注册、事件的合成、事件冒泡、事件派发等。
**React**为了性能会在容器监听所有事件（事件委托机制），当事件发生时会先触发原生事件再触发合成事件最后触发容器的原生事件：

> 原生事件：子元素 DOM 事件监听！
> 原生事件：父元素 DOM 事件监听！
> React 事件：子元素事件监听！
> React 事件：父元素事件监听！
> 原生事件：容器 DOM 事件监听！

## 伪代码
```javascript

let stopPro = false;
// 1. React 初始化时，将事件处理程序附加到根容器上
function attachEventHandlers(rootContainer) {
  const eventTypes = ['click', 'change', 'input', 'keydown', 'keyup'];
  eventTypes.forEach(eventType => {
    rootContainer.addEventListener(eventType, handleEvent, false);
  });
}

// 2. 捕获到事件后处理事件
// 会从发生事件的Element一直冒泡到rootContainer触发handleEvent
// 所以这里就已经是执行完毕所有的原生事件了
function handleEvent(nativeEvent) {
  stopPro = false;
  // 3. 创建合成事件对象
  const syntheticEvent = createSyntheticEvent(nativeEvent);

  // 4. 查找事件目标和冒泡路径
  // 根据发生事件的Element获取真实的Element链（current => rootContainer）
  const eventPath = getEventPath(nativeEvent.target);

  // 5. 依次调用冒泡路径上的事件处理程序
  for (let i = 0; i < eventPath.length; i++) {
    // 对于这里， 其实在react的源码里面， rdom和vdom其实是彼此可以获取到对方的
    const element = eventPath[i];
    const handlers = element.__reactHandlers || {};
    const handler = handlers[nativeEvent.type];
    if (handler) {
      // 如果有合成事件， 就触发它
      handler(syntheticEvent);
    }
    // 合成事件已经阻止了冒泡
    if (stopPro) break;
  }
}

// 3. 创建合成事件对象
// 这里模拟合成事件的创建
function createSyntheticEvent(nativeEvent) {
  const syntheticEvent = {};
  for (let key in nativeEvent) {
    syntheticEvent[key] = nativeEvent[key];
  }
  syntheticEvent.nativeEvent = nativeEvent;
  syntheticEvent.preventDefault = () => nativeEvent.preventDefault();
  syntheticEvent.stopPropagation = () => {
    // 阻止合成事件冒泡
    stopPro = true;
  }
  return syntheticEvent;
}

// 4. 获取事件冒泡路径
function getEventPath(target) {
  const path = [];
  while (target) {
    path.push(target);
    target = target.parentElement;
  }
  return path;
}

// 示例组件中的事件处理程序
function handleClick(event) {
  console.log('合成事件: ', event);
  console.log('事件目标: ', event.target);
}

// 模拟组件挂载和事件处理程序绑定
function mountComponent(rootContainer) {
  const button = document.createElement('button');
  button.textContent = '点击我';
  button.__reactHandlers = { click: handleClick };
  rootContainer.appendChild(button);
}
```

## 合成事件和原生事件的对比
|  特性    |  合成事件 | 原生事件   |
|  ----   |  ----    | ----     |
|  注册方式  | 通过React的属性(比如 onClick)    |  addEventListener, el.onxxx = ()  => {}  |
|  绑定位置  | 绑定在容器上面    |  直接绑定到指定 DOM 元素   |
| 跨浏览器兼容性 | 提供一致的事件处理接口 | 不同浏览器的行为可能不同 |