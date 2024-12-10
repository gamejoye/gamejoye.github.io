---
title: git-specification
date: 2024-12-10 20:37:57
tags:
  - git
  - 软件工程
cover: cover.jpg
---
# git commit 规范

## commit格式: \<type\>(\<scope\>): \<subject\>

type（必须）：
- feat: 新功能、新特性
- fix: 修改 bug
- perf: 更改代码，性能优化
- refactor: 代码重构（重构，在不影响代码内部行为、功能下的代码修改）
- docs: 文档修改
- style: 代码格式修改, 注意不是 css 修改（例如分号修改）
- test: 测试用例新增、修改
- build: 影响项目构建或依赖项修改
- revert: 恢复上一次提交
- ci: 持续集成相关文件修改
- chore: 构建过程或辅助工具的变动
- release: 发布新版本
- workflow: 工作流相关文件修改
scope（可选）： commit影响的范围
subject（必须）： subject是commit目的的简短描述，不超过50个字符

# 通过git pre-commit/commit-msg 钩子函数进行自动化提交验证

> pre-commit可以用于在commit之前检查代码格式
> commit-msg可以用于检查commit message的格式

pre-commit可以配置eslint使用

这里主要介绍commit-msg

## commitlint.config.js
```
// @ts-check

/** @type {import('@commitlint/types').UserConfig} */
const config = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',
        'fix',
        'docs',
        'style',
        'refactor',
        'perf',
        'test',
        'chore',
        'revert',
        'build',
        'ci',
        'workflow',
      ],
    ],
    'type-case': [2, 'always', 'lower-case'],
    'type-empty': [2, 'never'],
    'scope-empty': [0],
    'scope-case': [2, 'always', 'lower-case'],
    'subject-empty': [2, 'never'],
    'subject-full-stop': [2, 'never', '.'],
    'subject-case': [2, 'never', ['sentence-case', 'start-case', 'pascal-case', 'upper-case']],
    'header-max-length': [2, 'always', 72],
  },
};

module.exports = config;

```

## .husky/commit-msg
```
npx --no -- commitlint --edit ${1} 
```

## 使配置生效
```
npx husky
```

