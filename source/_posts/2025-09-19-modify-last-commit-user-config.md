---
layout: next
title: 修改Git最新commit的用户和邮箱信息
date: 2025-09-19 10:00:19
categories: Git
tags: Git
---

`git commit --amend` 命令一键修改

```
git commit --amend --author="新用户名 <新邮箱@example.com>" --no-edit
git push origin your_branch [--force]
```