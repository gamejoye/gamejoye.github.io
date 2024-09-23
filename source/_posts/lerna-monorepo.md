---
title: lerna搭载monorepo实践
date: 2024-09-22 15:34:39
tags: 
  - lerna
  - monorepo
cover: cover.jpeg
---

## 安装
安装并查看lerna版本
```bash
npx lerna -v
```

## 创建项目并初始化lerna项目
```bash
mkdir new-lerna-demo
cd new-lerna-demo
lerna init
```

## 创建a包
```bash
lerna create a
```
一路回车，将packages/a/package.json的**name**字段改成 **@gamejoye/a**，设置 **type**为 **"module"**，修改packages/a/lib/a.js的内容为ESM模块的形式

## 为a包添加依赖
给a包添加依赖，这里以loadsh为例子
```bash
npm i lodash -w @gamejoye/a
```

## 创建b包
```bash
lerna create b
```
一路回车，将packages/b/package.json的**name**字段改成 **@gamejoye/b**，设置 **type**为 **"module"**，修改packages/b/lib/b.js的内容为ESM模块的形式

## 创建cli包
```bash
npx lerna create cli --access public --bin --es-module
```
这里标志为public，并配置bin字段提供一个命令行命令，并配置为ESM模块形式
修改 bin/cli：
```javascript
#!/usr/bin/env node

'use strict';

import cli from '../src/cli.js';

cli().parse(process.argv.slice(2));
```

修改 src/cli
```javascript
import factory from 'yargs/yargs';

export default function cli(cwd) {
  const parser = factory(null, cwd);

  parser.alias('h', 'help');
  parser.alias('v', 'version');

  parser.usage(
    "$0",
    "TODO: description",
    yargs => {
      yargs.options({
        // TODO: options
      });
    },
  );

  return parser;
}


```

## 同步依赖
更新所有 workspaces 中的依赖
```bash
npm i -ws
```

## 配置<root>/package.json
添加
```json
{
  "scripts": {
    "lerna-cli": "cli"
  }
}
```

## 运行命令
这行命令会输出 **10.5.0** 代表npm的版本
```
npm run lerna-cli -v
```

## 打通包之间的关系
要在cli中使用a包和b包
```bash
npm i @gamejoye/a @gamejoye/b -w packages/cli
```

改造 bin/cli
```javascript
#!/usr/bin/env node

'use strict';

import cli from '../src/cli.js';

import a from '@gamejoye/a';
import b from '@gamejoye/b';
console.log(a());
console.log(b());

cli().parse(process.argv.slice(2));

```

运行如下命令会输出
```bash
npm run lerna-cli
```
> Hello from a
> Hello from b

## 发布版本
**lerna version**会让你选择下一个版本号，并自动更新子包下package.json的version字段，同时进行push到远程并打上tag
```
git add .
git commit -m 'first commit'
git remote add origin https://github.com/gamejoye/new-lerna-demo.git
git push -u origin master
lerna version
```

**lerna publish**