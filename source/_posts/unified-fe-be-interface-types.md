---
title: 前后端接口类型同步最佳实践
date: 2025-01-15 20:13:38
tags:
  - monorepo
  - 软件工程
  - 全栈
cover: cover.jpg
---

> 背景：在使用 TypeScript 做前端（React）与后端（NestJS）开发时，如何让双方的接口类型保持同步，尽量减少手动维护、避免前后端接口不一致问题？


# 为什么需要“前后端接口类型同步”？
在传统的前后端分离开发中，接口文档往往依赖手工书写与维护，容易出现以下问题：
1. 接口字段更新后忘记同步：后端新增或者修改字段，前端未能来得及同步修改，导致调用出错
2. 类型不匹配：前端以为age是number类型，后端期望得到string，则需要再进行二次处理
3. 维护麻烦：接口一多，维护手动文档、手动在前后端各写同样的dto，非常繁琐且易出错

## Monorepo + Shared Types包
### 如何做
1. 将前端项目（比如React）和后端项目（比如NestJs）放在一个`monorepo`里
2. 创建types包，前后端都通过安装`@cotask/types`来获取类型
```
my-monorepo/
├── packages/
│   ├── types/                // 类型包
│   │   ├── src/
│   │   │   └── dto.ts        // 统一类型比如CreateTaskDto等等
│   │   └── package.json
│   ├── cotask-fe/            // 前端
│   └── cotask-be/            // 后端
└── pnpm-workspace.yaml
```
### 优点
1. 开发体验好：改完接口类型，前后端立马同步
2. 维护成本：所有类型都集中在types包，统一发布、管理

### 缺点
1. 如果需要对外暴露接口时，其他语言无法使用，需要其他一些工具做转换

### 适用性
- 前后端在同一个仓库，采用 TS + TS 技术栈
- 想在项目初期就快速获得一致的类型，而不想写注解或额外脚本

## Swagger / OpenAPI 自动生成类型
### 如何做
1. 后端（NestJS）使用 @nestjs/swagger 在 Controller 与 DTO 上编写装饰器，自动生成 OpenAPI (Swagger) JSON
2. 前端使用 openapi-typescript 将 JSON 转成 .d.ts
3. 写一个脚本用于在后端编译出来 JSON，同时前端将 JSON 转成 .d.ts类型声明

### 优点
1. 对外友好：有了Swagger文档，如果第三方需要同步类型，则引入 JSON
2. 自动生成：后端接口修改了 DTO 之后只需要执行脚本即可生成最新的类型声明
3. 多语言支持：OpenAPI 是通用规范，Java、Go、Python 等都可用类似方式生成客户端SDK

### 缺点
1. 需要额外编写 @ApiProperty()
2. 需要脚本/CI 流程 来保持接口文档及时更新、生成

### 适用性
1. 中大型项目，或打算对外提供 API、或多语言客户端
2. 前后端不在同一个仓库，需要更正式的 OpenAPI 文档

## 其他做法？？？

TODO


# 我在[Cotask](https://github.com/gamejoye/cotask)里面的做法和思考
## 做法
我采用了 `monorepo`，使用了以下的方法来做类型统一：
1. Shared Types：在`packages/cotask-fe`和`packages/cotask-be`之外我还创建了`packages/types`包存储类型
2. OpenApi：在`packages/cotask-be`有用于自动生成接口类型的脚本，并将生成的类型保存在`packages/types/generated`下

## 为什么我要这样做？
1. OpenApi 用于更方便的同步类型，同时如果未来有需要向第三方提供文档的需要只需要提供`pacakges/types/generated`就可以了
2. Shared Types 主要用于保存一些其他类型：比如一个待办事项有**优先级**、**频率**这样的常量类型，如果是使用 OpenApi 做类型同步，需要自己从 OpenApi 生成的一层层嵌套中提取这些类型，所以在 Cotask 中我便抽离了一个 Shared Types来保存这些类型，同时保存 OpenApi 自动生成的类型