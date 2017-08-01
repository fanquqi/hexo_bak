---
title: Git使用总结
date: 2017-06-30 16:28:55
tags: git
categories: git
---
> 很长一段时间使用git都是只有 `add`,`commit`,`pull`,`push`等这几个简单的命令。想想自己也是使用github的人，怎么能只会这些皮毛。

首先我们知道git主要是做版本控制工具，所以一些概念逻辑都是为了更好地实现这个功能。

## 三个目录概念

首先要对概念清楚 
- Working Directory：工作目录，这个可以简单的理解为你在文件系统里真实看到的文件
- Stage（Index）：暂存“目录”，用git add命令添加的文件就到了这里，即将被commit的文件
- Repository：项目“目录”，用git commit提交的文件就到了这里

## commit

介绍几种比较6的commit的操作，记住使用git的一贯原则还是少量改动频繁提交，方便做版本控制。

- 平常我们都是`git add ./`然后`git commit -m "fix"`提交代码
- 这两条可以结合到一起直接`git commit -am "fix"`就做到了 
或者指定文件`git commit test.txt -m "fix2.0"`.

- 修改上一次提交 `git commit --amend -am "fix2.0, 2.5` 这样之前2.0的commit_id直接被覆盖了。

## checkout


- 分支相关操作：git checkout 分支名/commit hash切换到相应的分支或commit，加上-b参数则会创建分支并切换过去
    + git checkout -b branch3 1a222c3  注意这里commit_id为新分支起点

- 恢复文件相关操作：git checkout [分支名/commit hash/HEAD快捷方式] -- 文件名恢复指定分支的最新commit或指定commit或快捷方式指向的commit的文件到工作目录，若省略中间的参数，则

    - 暂存区有内容且暂存区内容与工作目录不同，则恢复暂存区的状态到工作目录(之前`git add`过恢复到当时的状态)

    - 暂存区无内容，则恢复HEAD（最新的commit）的状态到工作目录

## diff 
- 使用方式基本就是 `git diff [source] [target]` 也就是说 `target`相对于`source`有哪些变化 ,这里的target,source 可以是commit_id也可以是两个分支,同时`git diff master branch2`和`git diff HEAD branch2`显示结果是一样的
- 只给一个参数 这个参数默认是 `source` 而`target` 默认是当前分支最新的commit
- 不给参数 `source`为暂存区，`target`为工作目录
- 如果想要使暂存目录作为target的话，需要使用--cached参数

## reset 
- git reset [commit hash/分支名/快捷方式] [文件名]类似“git add的反操作”，直接将所在commit的文件状态恢复到暂存区域。省略commit则默认为HEAD，省略文件名默认为所有文件。只改变暂存目录，不改变工作目录，当前commit不变。
- git reset --soft [commit hash/分支名/快捷方式]软恢复，将恢复前所在commit的文件状态恢复到暂存区，当前最新commit为参数中的commit。只改变暂存目录，不改变工作目录，当前commit改变。
- git reset --hard [commit hash/分支名/快捷方式]硬恢复，强制将整个项目恢复为参数中的commit时的文件状态，清空暂存目录，工作目录clean。暂存目录和工作目录同时被改变，当前commit改变。

关于reset命令的其他补充：当前HEAD已经位于“不是最新”，是不是前面的commit都找不回来了？当然不会，reset过的操作也是可以被reset的。有两种方法：

1. 如果记得“最新”的hash（，则直接git reset --hard 1a222c3，则项目直接强制恢复到“最新”所在的状态。
2. 如果不记得的话，运行git reflog，这个命令会输出一个列表，包含HEAD发生的所有变化。

## cherry-pick
> cherry-pick其实在工作中还挺常用的，就像copy一样，把一个分之上的某个或者某几个commit复制到另一个分之。一种常见的场景就是，比如我在A分支做了几次commit以后，发现其实我并不应该在A分支上工作，应该在B分支上工作，这时就需要将这些commit从A分支复制到B分支去了，这时候就需要cherry-pick命令了

模拟下上面场景 在master分之上提交了两个commit  一个commit1 一个commit2，发现不应该在master上面操作，应该新建分之branch1
```
首先新建分之 从commit0开始，并切到branch1
git checkout -b branch1 commit0
复制master分之 两个commit到当前分之
git cherry-pick commit1 commit2
在master分支上将这两个commit删除。先切回master分支：
git checkout master，运行git reset --hard commit0

```

## merge


## rebase


