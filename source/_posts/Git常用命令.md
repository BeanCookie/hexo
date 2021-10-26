---
title: Git常用命令
date: 2020-04-15 11:43:59
categories:
- 开发工具
tags:
- Git
---

## Git常用命令

### 远程地址

1. 显示所有远程仓库

   ```bash
   git remote -v
   ```

2. 添加远程地址

   > 通常我们把origin指向私有分支，当协同开发时为了避免误操作我们会新建一个upstrem分支指向协同分支
   
   ```bash
git remote add [-t <branch>] [-m <master>] <name> <url>
   ```
   
3. 删除远程地址

   ```bash
   git remote remove <name>
   ```

   