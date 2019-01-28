---
layout: post
title: "Git常用命令（二）"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - Git
---

#### git reset

* 与 `git add` 相对的，`git reset` 可以将添加到暂存区的文件还原到工作区：

  * `git reset filePath` 还原指定文件

  * `git reset .` 还原所有文件

* `git reset` 还有回退版本的功能：

  * `git reset --hard HEAD^` 将当前版本回退到上一个版本

    `HEAD` 代表当前版本，`HEAD^` 代表上一个版本，托字符 `^` 可以用来表示回溯版本的个数，写几个就回溯几个版本。但如果我们要回溯80个版本，git 也不会让我们就真的傻乎乎的写80个 `^`，这时可以这么写 `HEAD~80`。

  * `git reset --hard commitId` 还可以回退到指定版本：

    其中 commitId 可以不用写全，写前几位就好，git 会帮我们自动去找。但也不能只写两三位，如果匹配到多个 commitId 的话 git 就不知道我们要回退到哪一个了。

#### git diff

  * `git diff` 不带任何参数，会显示本地工作区和暂存区代码之间的差异。

  * `git diff --staged` 会显示暂存区和已经 commit 的代码之间的差异。

  * `git diff commitIdA..commitIdB` 显示 A commit 和 B commit 之间的差异。

  * `git diff master..dev` 显示 master 分支和 dev 分支之间的差异。

#### git rebase

* `git rebase branchName` 将当前分支变基到 branchName 分支

  更多内容可查看 [Rebase 代替合并](https://www.git-tower.com/learn/git/ebook/cn/command-line/advanced-topics/rebase#start)

#### git branch

* `git branch -v` 查看所有本地分支的最后一次提交

* `git branch -vv` 查看所有本地分支的跟踪分支

* `git branch --merged` 查看所有已合并到当前分支的分支

* `git branch --no-merged` 查看所有未合并到当前分支的分支

#### git show

* `git show HEAD` 显示当前版本（也就是最新一次 commit）的详细内容

* `git show HEAD^ / git show HEAD~80` 同样可以使用托字符或数字

* `git show commitId` 显示某个提交的详细内容

* `git show-branch` 显示当前分支历史

* `git show-branch -a` 显示所有分支历史

#### git revert

* `git revert commitId` 撤销某次提交所做的全部修改

#### git push

* `git push remoteName branchName` 将本地仓库代码推到 remoteName 远程仓库中的 branchName 分支

* `git push --set-upstream remoteName branchName` 在 remoteName 远程仓库创建 branchName 分支，并将当前仓库当前分支的代码推到新建的分支中

* `git push remoteName -d branchName` 删除 remoteName 远程仓库中的 branchName 分支

#### git fetch

* `git fetch --prune` 获取所有远程分支（不自动合并），并清除本地已经在远程删除掉的分支（由于存在缓存信息，远程已经删掉的分支仍会存在于本地）

#### git log

* `git log` 显示提交日志

* `git log -p` 显示提交日志以及每次提交所对应的代码变动

* `git log -n` 显示 n 个 commit 的日志

* `git log --pretty=format:'%h %s' --graph` 图示提交日志