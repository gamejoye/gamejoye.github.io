---
title: rebase-vs-merge
date: 2024-09-22 15:27:38
tags: git
---

## 如何使用
**git rebase** 与 **git merge**都有相同的作用，都是将一个分支的提交合并到另一分支上，但是在原理上却不相同

1. git merge
将feature分支合并到当前分支
```bash
git merge feature
```
G节点是master和feature所在的节点
![merge流程图](merge.png)

2. git rebase
将master分支变基到当前分支
```bash
git rebase master
```
C节点是master所在的节点
E'节点是feature所在的节点
![rebase流程图](rebase.png)

## 总结
1. merge通常用在master分支合并feature分支
2. rebase用于创建干净的线性commit链 通常用于feature分支更新master分支的内容