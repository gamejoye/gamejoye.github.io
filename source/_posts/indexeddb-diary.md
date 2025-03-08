---
title: indexeddb 日记
date: 2025-03-08 14:15:43
tags:
  - indexeddb
  - 浏览器
  - 本地存储
  - 前端
cover: cover.jpg
---

## 什么是indexeddb
indexeddb 是浏览器提供的一种本地存储机制，可以存储大量的支持[结构化克隆算法](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API/Structured_clone_algorithm)的数据。

## 如何使用indexeddb

### 打开indexeddb
```
const request = indexedDB.open('todo_database');
// with version
const request = indexedDB.open('todo_database', 2);
```

### 创建 / 删除 indexeddb objectStore
```
// onupgradeneeded 事件会在version变化的时候触发 （创建database也可以理解一种version change， 0 -> 大于0）
request.onupgradeneeded = () => {
  const db = request.result;
  // 创建objectStore
  const ob = db.createObjectStore('todo_store');
  // 创建索引
  ob.createIndex('id', 'id', { unique: true });

  objectStore.transaction.oncomplete = () => {
    // 你可以操作 todo_store objectStore 了
    // ...

    // 删除objectStore
    db.deleteObjectStore('todo_store');
  };
};

```
// 只有在onupgradeneeded中才可以 创建 / 删除 objectStore
// 所以，如果你需要创建 / 删除 objectStore，需要监听 onupgradeneeded 事件，同时你要在调用open的时候设置一个version数值比当前的version大。

### 处理indexeddb
```
request.onsuccess = () => {
  const db = request.result;
  // 获取数据库中的数据

  const transaction = db.transaction('todo_store');
  const store = transaction.objectStore('todo_store');
  const request = store.get(1);
  request.onerror = () => {
    console.log('error');
  };
  request.onsuccess = () => {
    const data = request.result;
    console.log(data);
  };
};
```
indexeddb处理请求都是通过事件来处理的，所以需要监听事件（onsuccess, onerror）。
db.transaction() 会返回一个事务对象，事务对象可以处理多个store。同时可以配置事务的mode（readonly, readwrite）。

### 关闭indexeddb
```
// db type is IDBDatabase
db.close();
```

### IDBDatabase 事件
1. close
close 事件会在数据库关闭的时候触发 （注意如果是调用的db.close()， 那么这个事件不会触发）

2. versionchange
versionchange 事件会在数据库的version变化的时候触发

> #### versionchange有什么妙用吗？
>> 背景：对于多个连接到同一个database的实例，如果某个触发onupgradeneeded事件，那么其他实例需要断开连接，否则该onupgradeneeded事件会blocked。
> - 考虑这样的一个场景：你需要更新database的schema，但是呢，你又有很多个关于该database的实例，你需要一个一个的去调用db.close()吗？
> - 答案是：不需要，你可以监听versionchange事件，当versionchange事件触发的时候，你再调用db.close()。
> ```
> db.onversionchange = () => {
>   db.close();
>   // 如果你需要重新连接到database，你可以在未来某个时间节点重新调用open方法进行reconnect
> }
> ```

### IDBCursor
IDBCursor 是 indexeddb 提供的一种游标，可以遍历数据库中的数据。

```
const request = this.db
  .transaction('todo_store', 'readonly')
  .objectStore('todo_store')
  .openCursor();
const keys: string[] = [];
request.onsuccess = e => {
  const cursor = (e.target as any).result as IDBCursorWithValue;
  if (!cursor) {
    // 没有更多数据了
    return;
  }
  const value = cursor.value
  // do something with value
  // ...

  cursor.continue();
};
```

### IDBKeyRange
IDBKeyRange 是 indexeddb 提供的一种范围，可以用于查询数据库中的数据。

```
const lower = 'your lower bound';
const upper = 'your upper bound';
const range = IDBKeyRange.bound(lower, upper, false, true);
range 包含范围key: `[lower, upper)`
```

### IDBTransaction
IDBTransaction 是 indexeddb 提供的一种事务，所有的读取和写入数据均在事务中完成。

```
const transaction = db.transaction('todo_store');
const store = transaction.objectStore('todo_store');
```

提起这个主要是踩了一个坑
最小example：
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>IndexedDB Demo</title>
</head>
<body>

    <button id="loadButton">Load Data</button>
    <div id="output"></div>

    <script>
        // 1. 打开数据库
        const request = indexedDB.open("myDatabase", 1);

        let objectStore;

        // 2. 数据库创建或更新时的回调
        request.onupgradeneeded = function(event) {
            const db = event.target.result;
            if (!db.objectStoreNames.contains("myStore")) {
                const store = db.createObjectStore("myStore", { keyPath: "id" });
                store.transaction.oncomplete = function() {
                    // 初始化数据
                    const transaction = db.transaction("myStore", "readwrite");
                    const objectStore = transaction.objectStore("myStore");
                    objectStore.add({ id: 1, name: "Alice", age: 25 });
                    objectStore.add({ id: 2, name: "Bob", age: 30 });
                };
            }
        };

        // 3. 打开数据库成功时的回调
        request.onsuccess = function(event) {
            const db = event.target.result;

            // 获取 objectStore 并赋值给外部变量
            const transaction = db.transaction("myStore", "readwrite");
            objectStore = transaction.objectStore("myStore");

            console.log("Database opened successfully");

            // 绑定 button 点击事件
            const loadButton = document.getElementById("loadButton");
            loadButton.addEventListener("click", function() {
                // 在按钮点击时使用 objectStore 的 get 方法
                const getRequest = objectStore.get(1); // 假设我们查询 id 为 1 的数据

                getRequest.onsuccess = function(event) {
                    const result = event.target.result;
                    document.getElementById("output").textContent = `Name: ${result.name}, Age: ${result.age}`;
                };

                getRequest.onerror = function(event) {
                    console.error("Error retrieving data: ", event.target.error);
                };
            });
        };

        // 4. 打开数据库失败时的回调
        request.onerror = function(event) {
            console.error("Database open failed: ", event.target.error);
        };
    </script>

</body>
</html>

```
报错：
```
Uncaught TransactionInactiveError: Failed to execute 'get' on 'IDBObjectStore': The transaction has finished.
    at HTMLButtonElement.<anonymous> (indexedDB-transaction-issue.html:48:48)
```

原因：
- 所有请求结束并且控制权返回事件循环之前没有发出其他请求，transaction 会自动提交（finished）。换句话，你可以在上一个请求的成功/失败回调中发出请求，但不能在其他异步中处理该事务。对于finished transaction，任何请求都会报错。
- 在这个demo中，transaction 在onupgradeneeded中创建，button click callback通过闭包获取到了objectStore。但此时我们的transaction已经finished了，所以报错
- 解决方案：在callback重新创建一个transaction
