---
title: cookie
date: 2025-03-13 21:16:31
tags:
  - 计算机网络
  - HTTP
---

> cookie特性记录

## Domain
用于配置cookie作用的域（无视端口）

Domain属性规则：
1. 只能将值设置为当前域名或者更高级别的域名
比如当前请求 *https://auth.xxx.com* 则 response 的set-cookie只能配置 auth.xxx.com 或者 xxx.com 的 Domain
2. 不提供则默认为当前域 比如当前请求 *https://auth.xxx.com* 则 默认 cookie为 auth.xxx.com

## HttpOnly
禁止JavaScript通过Document.cookie进行访问cookie。如果通过JavaScript发送的请求还是会随着请求发送的

## Path
只有满足当前Path要求才会一起发送
比如当前Path属性为/abc，则只有请求 *https://auth.xxx.com/abc* 会伴随发送Path为/abc的cookie（同理只要其他cookie符合也会一并发送）。但是请求 *https://auth.xxx.com/get-cookie* 则不行

## SameSite
- Strict
只有请求设置该cookie的站点时才会发送。如果请求来自不同的域名或协议（即使是相同域名）则不会发送
> 会受到Domain的影响

如果一个Domain为 xxx.com，如果a.xxx.com访问b.xxx.com同样会附带 Domain为 xxx.com 的cookie

- Lax
这意味着 cookie 不会在跨站请求中被发送
举例前提：在 `https://example.com` 设置了cookie：`Set-Cookie: session_id=abc123; SameSite=Lax; Secure; Path=/;`
  - 跨站请求时，Cookie 会发送的情况
    浏览器会发送 Cookie（例如：用户从 https://other-site.com 点击链接跳转到 https://example.com，此时会附带 Cookie）
  - 跨站请求时，Cookie 不会发送的情况：
    通过嵌入资源时（如图像、iframe 或 AJAX 请求），浏览器不会发送 Cookie。例如，如果在 https://example.com 页面中嵌入了一个来自 https://other-site.com 的图像或 iframe，请求 https://other-site.com 的 Cookie 不会被发送。

- None
在所有跨站请求中都发送，但cookie必须要标记为Secure

## Secure
仅当请求通过 https: 协议发送时才会将该 cookie 发送到服务器


## 应用
1. 实现SSO单点登录
auth.xxx.com设置 Domain为xxx.com的cookie。使得请求a.xxx.com和b.xxx.com都能发送cookie