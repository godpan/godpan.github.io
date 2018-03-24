---
title: Git删除本地分支（批量删除）
layout: post
guid: urn:uuid:85cc4b63-e9f0-4454-8ca7-fbdsdc2a2fad
tags:
  - git
---

分享一个小技巧，我们在很多时候需要删除一些本地无用分支，假如我们想要删除具体分支，我们可以这么做：

```shell
git branch -D branchName
```

但是有些时候我们要删除很多分支，比如除了master外的所有分支，那么我们可以这么做：

```shell
git checkout master
git branch | grep -v 'master' | xargs git branch -D
```

具体执行步骤是：

1. 切换到master分支
2. 将git branch的结果进行筛选，除去master
3. 将处理后的结果作为git branch -D的参数来进行删除分支