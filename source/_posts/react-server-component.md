---
title: react-server-component
date: 2024-09-23 01:07:30
tags:
  - react
  - 前端
cover: cover.jpeg
---

## 概述
React Server Components 允许服务器和客户端（浏览器）协作渲染您的 React 应用程序。考虑为您的页面渲染的典型 React 元素树，它通常由不同的 React 组件组成，这些组件渲染更多的 React 组件。RSC 使得此树中的某些组件可以由服务器渲染，而某些组件可以由浏览器渲染

![rsc树图](react-server-components.png)

RSC跟SSR是两个不同的概念，使用RSC并不需要SSR

## 好处
1. 更接近数据源
2. 某些情况下不需要下载某些npm包

## RSC做了什么
1. 服务器将根组件序列化为JSON（RSC payload）
  - HTML标记：原本就是可以序列化的，直接序列化
  - server component：调用服务端组件函数，序列化返回的结果
  - client component：序列化为模块引用对象
  ```
  {
    $$typeof: Symbol(react.element),
    type: {
      $$typeof: Symbol(react.module.reference),
      name: "default",
      filename: "./src/ClientComponent.client.js"
    },
    props: {
      children: {
        $$typeof: Symbol(react.element),
        type: "span",
        props: {
          children: "Hello from server land"
        }
      }
    }
  }
  ```
2. 将RSC payload发送到客户端进行水合（hydration）

## 最终的vdom树
![最终的vdom树](react-server-components-client.png)


## 参考资料
[How React server components work: an in-depth guide](https://www.plasmic.app/blog/how-react-server-components-work)