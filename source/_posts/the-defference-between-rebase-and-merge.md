---
title: git merge 与 git rebase
date: 2017-03-12 
tags:
  - Android
---

我们在使用 git 的过程中，经常会遇到将两个分支进行合并的情况，这时我们可以有两种选择，`git merge`和`git rebase`。本文分别介绍这两个命令的使用方式以及他们的异同。

## git merge

`git merge`涉及到两种合并的方式，其中一种简单的是`Fast forward`，如果一个分支走下去可以到达另一个分支的话，那么 Git 在合并两者时，只会简单地把指针向前移动。现在我们讨论一种可以称为`三方合并`的方式。

如下图，我们从`C2`新建了名为`develop`的分支，且`master`指针在`C3`上。现在，我们需要合并`master`与`develop`分支，那么我们输入的命令：

```
git checkout master
git merge develop
// 命令行一般会有如下输出
Auto-merging README
Merge made by the 'recursive' strategy.
 README | 1 +
 1 file changed, 1 insertion(+)
```
<!--more-->
![Alt text](/images/git_merge.png)

注意，这次合并的方式和`Fast forward`的底层实现并不相同，由于当前`master`分支所指向的提交对象（C3）并不是`develop`分支的直接祖先，Git 不得不进行一些额外处理。在这里，Git 会用两个分支的末端（C3 和 C5）以及它们的共同祖先（C2）进行一次简单的三方合并计算，如下图所示：

![Alt text](/images/git_merge_1.png)

这次，Git 没有简单的把分支指针右移，而是对三方合并后的结果重新做一个新的快照，并自动创建一个指向它的提交对象（C6），如下图所示：

![Alt text](/images/git_merge_2.png)

当然，合并的过程并非每次都这么顺风顺水，也会碰到冲突的时候，那就解决冲突喽。

## git rebase

在这里我们还是以第一张图片的情形作为例子，我们这里不用`git merge`，使用`git rebase`，我们输入命令：

```
git rebase master develop
```

这条命令的的原理是：回到两个分支最近的共同祖先，然后根据`develop`分支的后续提交对象（C4和C5），生成一系列文件补丁，然后以基底分支`master`最后一个提交对象（C3）为新的出发点，逐个应用之前准备好的补丁文件，最后会生成若干个合并提交对象（C4'和C5’），从而改写`develop`的提交历史，使它成为`master`分支的直接下游，如下图所示：

![Alt text](/images/git_merge_3.png)

现在我们就可以快进主干分支`master`了：
```
git checkout master
git merge develop
```
最终我们的提交历史会变成下图的样子：

![Alt text](/images/git_merge_4.png)

`git rebase`和`git merge`最后整合得到的结果没有任何区别，但是`git rebase`能产生一个更为整洁的提交历史，仿佛所有修改都是在一根线上先后进行的，尽管实际上它们原本是同时并行发生的。So，我的做法是每次`push`之前先把本地修改与服务器修改`rebase`一次，这样就能得到一个较为简洁的提交历史。

## 区别

`git rebase`是按照每次的修改次序重演一遍修改，而`git merge`是把最终结果合并在一起。

## 参考资料
1. https://git-scm.com/book/zh/v1/Git-%E5%88%86%E6%94%AF-%E5%88%86%E6%94%AF%E7%9A%84%E8%A1%8D%E5%90%88