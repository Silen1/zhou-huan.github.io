---
layout: post
title: "Git常用命令（一）"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Git
---

#### git clone

* `git clone url` 拷贝一个 git 仓库到本地

#### git checkout

* `git checkout branchName` 切换到 branchName 分支

* `git checkout -b branchName` 从当前分支创建新分支 branchName，并切换到该分支

* `git checkout -b [branch] [remotename]/[branch]` 创建本地分支跟踪远程分支

#### git add

* `git add filePath` 添加指定文件到缓存区

* `git add .` 将所有改动添加到缓存区

#### git commit

* `git commit -m"commit message"` 将暂存区代码添加本地到仓库中

* `git commit -am"commit message"` git add 和 git commit 的合成命令，将工作区**所有文件的更改**添加到缓存区同时提交到仓库

#### git pull

* `git pull` 获取远程分支并合并到当前分支

#### git push

* `git push` 将本地仓库代码推到远程仓库

#### git branch

* `git branch` 显示本地分支

* `git branch -a` 显示所有分支，包括本地分支和远程分支

* `git branch -d branchName` 删除 branchName 分支

* `git branch -D branchName` 强制删除 branchName 分支

#### git status

* `git status` 查看上次提交之后是否有修改

* `git status -s` 查看简短的输出结果

#### git merge

* `git merge branchName` 合并 branchName 分支到当前分支

#### git cherry-pick

* `git cherry-pick commitId` 将某一次提交合并到当前分支

#### git fetch

* `git fetch` 获取所有远程分支（不自动合并）

#### git remote

* `git remote` 查看已经配置的远程仓库服务器

* `git remote -v` 显示需要读写远程仓库使用的 Git 保存的简写与其对应的 URL

* `git remote add <shortname> <url>` 添加一个新的远程 Git 仓库，同时指定一个别名

* `git remote show remoteName` 查看 remoteName 远程仓库的详细信息

#### git tag

* `git tag` 查看所有的标签

* `git tag newName` 创建一个轻量级的标签

* `git tag -a newName -m"annotate"` 创建一个带注释的标签

* `git tag -d tagName` 删除某个标签

* `git checkout tagName` 切换到某个标签

* `git tag -l` 查看所有标签

  也可以按照特定的表达式搜索某些标签：

  ```bash
  $ git tag -l v1.4.2.*
  v1.4.2.1
  v1.4.2.2
  v1.4.2.3
  v1.4.2.4
  ```

* `git show tagName` 查看该标签的信息

  **tag 指向的是一次 commit，show tagName 命令展示出来的 diff 也是该次 commit 的**

* `git push --tags` 将本地的标签推到远程

  默认情况下，`git push`不会将标签上传到远程服务器，为了共享这些标签，须在`git push`后加 `--tags` 选项

#### git stash

* `git stash` 将工作区和暂存区的代码临时储存起来，每 stash 一次就在储存栈的顶部新增一个储存项

* `git stash list` 查看当前储存栈

* `git stash pop` 恢复储存栈顶部的储存项，同时将其从储存栈中移除

* `git stash show / git stash show stashName` 查看顶部或指定储存项的简要 diff 信息，增加参数 `-p` 可以查看详细 diff 信息

* `git stash drop / git stash drop stashName` 删除储存栈顶部或指定的储存项，可以使用 git stash list 查看 stashName

* `git stash clear` 清除储存栈中所有储存项

* `git stash apply` / `git stash apply stashName` 恢复顶部或目标储存项，但不从储存栈中删除它

#### git config

* `git config user.name` 查看本地用户名

* `git config user.email` 查看本地用户邮箱地址

* `git config --global user.name "username"` 修改本地用户名

* `git config --global user.email "email-address"` 修改本地用户邮箱地址

如果发现将某些文件加入 gitignore 的忽略规则之后未生效，原因可能是：.gitignore 只能忽略那些原来没有被 track 的文件，如果某些文件已经被纳入了版本管理中，则修改 .gitignore 文件是无效的，解决方法就是把本地缓存删除（改变成未 track 状态），然后再提交：
```bash
git rm -r --cached .
git add .
git commit -m "update .gitignore"
```