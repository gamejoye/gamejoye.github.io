---
title: 前端开发中的"pacakge.json"
date: 2024-12-18 22:15:21
tags: 
  - 前端
  - SDK
cover: cover.jpg
---

> 对于前端 package 开发者，或多或少都会涉及到配置pacakge.json的属性

# main module types
1. main字段 ==> commonjs模块的入口文件
2. module字段 ==> esmodule模块的入口文件
3. types字段 ==> ts类型声明

- 如果不配置main字段，则commonjs和esmodule入口为main字段

# files
files字段用于指定哪些文件/目录会被发布到 `npm`
```
{
  "files": [
    "dist",
  ]
}
包含整个dist文件夹
```

某些文件总是被包含
1. pacakge.json
2. README
3. LICENSE

# exports

> 待完结
