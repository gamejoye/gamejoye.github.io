---
title: 全自动化前端监控SDK
date: 2024-12-11 14:29:14
tags: 
  - 前端
cover: cover.jpg
---

## 异常类型
### JS代码执行异常
举个例子
```
var a = {};
a.b.c = 123;
```
针对JS代码执行异常我们可以`try...catch`包裹代码块，或者采用`window.addEventListener('error', listener)`来进行捕获

### Promise reject异常
这类异常通常出现promise出现reject但是未被catch的情况
针对这类异常通常采用`window.addEventListener('unhandledrejection', listener)`来进行捕获

### 资源加载异常
这类异常出现在我们请求的js css img等链接资源失效的时候，这类异常并**不会冒泡**，所以需要在***捕获阶段***进行处理，`window.addEventListener('error', listener, true)` 第三个参数表示在**捕获阶段**进行处理

### 接口请求异常
1. fetch请求
```
fetchReplacer(originalFetch: typeof fetch) {
  return function (...args: Parameters<typeof fetch>) {
    return originalFetch(...args)
      .then(res => {
        if (!res.ok) {
          // TODO 错误处理
        }
        return res;
      })
      .catch(error => {
        // TODO 错误处理
        throw error;
      });
  };
}

const originalFetch = window.fetch;
if (!originalFetch) {
  console.warn('fetch is not supported');
  return;
}
window.fetch = fetchReplacer(originalFetch);
```

2. xhr请求
```
replacer(originalXHR: typeof XMLHttpRequest) {
  const originalSend = originalXHR.prototype.send;
  const originalOpen = originalXHR.prototype.open;
  const shouldSkipHandler = (e: ProgressEvent<XMLHttpRequestEventTarget>) => {
    const status = (e.target as XMLHttpRequest & ExtraXMLHttpRequest).status;
    const type = e.type;
    // 正常情况 + 超时情况/错误情况 触发的loadend
    return (type === 'loadend' && status === 0) || (status + '').startsWith('2');
  };
  // 重写open方法 方便错误处理阶段获取接口请求信息
  originalXHR.prototype.open = function open(
    this: XMLHttpRequest & ExtraXMLHttpRequest,
    method: string,
    url: string | URL,
    async: boolean = true,
    username?: string | null,
    password?: string | null
  ) {
    this.method = method;
    this.requestURL = typeof url === 'string' ? url : url.toString();
    return originalOpen.apply(this, [method, url, async, username, password]);
  };
  originalXHR.prototype.send = function send(
    this: XMLHttpRequest & ExtraXMLHttpRequest,
    ...args: Parameters<typeof originalSend>
  ) {
    const handler = function (e: ProgressEvent<XMLHttpRequestEventTarget>) {
      // 获取请求URL 请求状态 请求方法 
      const { responseURL, requestURL, status, statusText, method, responseText } =
        e.target as XMLHttpRequest & ExtraXMLHttpRequest;
      if (shouldSkipHandler(e)) return;
      // TODO 错误处理
    };
    // 监听错误 （网络中断 / 跨域等）
    this.addEventListener('error', handler);
    // 监听请求错误 （请求体的内容非2xx）
    this.addEventListener('loadend', handler);
    // 监听请求超时
    this.addEventListener('timeout', handler);
    return originalSend.apply(this, args);
  };
}

const originalXHR = window.XMLHttpRequest;
if (!originalXHR) {
  console.warn('XMLHttpRequest is not supported');
  return;
}
replacer(originalXHR);
```

### 跨域脚本执行异常


## 前端如何做全自动化的异常监控呢
