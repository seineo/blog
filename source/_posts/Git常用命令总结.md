---
title: Git常用命令总结
date: 2024-12-14 10:07:57
tags:
---

## Git提交

### 提交变更

Git仓库的每次提交可以看作是对当前目录所有文件的一次快照，但是并不是盲目地复制保存整个目录，而是会对比当前版本和上一版本，存储差异化的内容作为这一次的提交记录。那为了可以回溯之前的版本，Git的每次提交节点都会有一个父节点。

<img src="https://oss.seineo.cn/images/202412251008297.png" alt="截屏2024-12-16 16.09.40" style="zoom:50%;" />

### 创建分支

git分支可以让我们可以并行地进行代码的开发/修复工作，假设我们现在在`main`分支，那么可以有两种方法创建新分支`bugFix`：

```shell
# 方法一
git branch bugFix
git checkout bugFix

# 方法二
git checkout -b bugFix
```

### 合并分支

在新分支开发新功能或修复bug之后，我们会需要将工作合并到原来的分支上，这时候有两种做法：

-   `git merge`：`git merge`是会创建一个新的提交节点，它包含的两个分支合并后的修改，也就是会将这两个分支的最新节点作为父节点。

    <center><img src="https://oss.seineo.cn/images/202412161730059.png" alt="截屏2024-12-16 16.28.37" style="zoom:50%;" /></center>

-   `git rebase`：`rebase`的字面意思就是将一系列提交记录“变基”，指的是复制一系列提交记录，放到另一个分支上。

    <center><img src="https://oss.seineo.cn/images/202412161636215.png" alt="截屏2024-12-16 16.36.34" style="zoom:50%;" /></center>

    如上图所示，假定我们在bugFix分支上提交了一个hash为`C3`的节点，然后我们在这个分支使用`git rebase main`命令，那么命令会找到bugFix和main分支的共同祖先节点（即`C1`节点），将bugFix分支在这之后的提交记录（`C3`节点）放到main分支上作为新的提交记录，可以看到`rebase`相比于`merge`会让提交记录更简洁与线性。

### 重写提交历史

当着手开发一个新的特性时，我们通常首先会基于主分支创建一个特性分支，例如使用`git checkout -b feat`命令来创建名为`feat`的特性分支。在这个特性分支上，我们不断地进行代码的编写、修改和提交，这个过程中可能会产生一系列较为杂乱的提交记录，这可能是由于频繁地修复小问题、调整代码结构或者阶段性地保存工作进度等原因导致的。

当特性开发完成，准备将其合并回主分支时，为了让代码的提交历史更加清晰、简洁，易于其他团队成员理解和后续的代码维护，我们就可以利用`git rebase -i`命令来重写提交历史。比如，当我们在特性分支上已经进行了多次提交，想要对最近的5次提交记录进行整理时，可以使用`git rebase -i HEAD~5`命令，进入交互式的`rebase`操作界面。在如下的界面，我们可以通过**修改每行开头的单词**来对提交记录进行诸如合并、编辑提交信息等操作，也可以直接**修改行之间的顺序**来对提交重新排序，从而使得提交历史更加符合逻辑和项目的开发流程：

```text
pick c50221f commit B
pick 73deeed commit C
pick d9623b0 commit D
pick e7c7111 commit E
pick 74199ce commit F

# 变基 ef13725..74199ce 到 ef13725（5 个提交）
#
# 命令:
# p, pick <提交> = 使用提交
# r, reword <提交> = 使用提交，但修改提交说明
# e, edit <提交> = 使用提交，进入 shell 以便进行提交修补
# s, squash <提交> = 使用提交，但融合到前一个提交
# f, fixup <提交> = 类似于 "squash"，但丢弃提交说明日志
# x, exec <命令> = 使用 shell 运行命令（此行剩余部分）
# b, break = 在此处停止（使用 'git rebase --continue' 继续变基）
# d, drop <提交> = 删除提交
......
```

在本地完成了对特性分支提交历史的整理后，我们可以将修改后的分支强制推送到远程仓库，使用如`git push -f o/feat`的命令，以覆盖远程也同样杂乱的提交记录。最后，为了确保特性分支与主分支的最新代码保持一致，并能够顺利地合并到主分支，我们还需要执行`git rebase main`命令，将主分支上的最新提交合并到特性分支上，解决可能出现的代码冲突后，特性分支就可以以一个整洁、清晰的提交历史状态合并回主分支，为整个项目的代码库维护和团队协作提供更好的基础。

完整的git命令示例如下：

```shell
git checkout -b feat
git commit
git push
#...
git rebase -i HEAD~5
git push -f o/feat 
git rebase main
```

## 提交树修改

### 移动HEAD

HEAD默认指向当前分支的最新提交记录，如HEAD->main, main->C1，我们可以通过`git checkout <hash>`来修改HEAD的指向，此处的`hash`并不需要是提交节点的完整哈希值，只需要提供能够唯一标识提交记录的前几个字符即可，一般使用4个字符，如`git checkout fed1`。也可以使用相对引用：

-   `^` 表示向上移动一个提交记录，如`main^`表示将HEAD指向切换到`main`分支最新节点的父节点。

    <center><img src="https://oss.seineo.cn/images/202412161717552.png" alt="截屏2024-12-16 17.17.05" style="zoom:50%;" /></center>

-   `~<num>`表示向上移动多个提交记录，如`HEAD~3`表示将HEAD指向切换到往前数第三个提交节点。

    <center><img src="https://oss.seineo.cn/images/202412161717844.png" alt="截屏2024-12-16 17.17.29" style="zoom: 50%;" /></center>	   

### 撤销变更

撤销变更有两种方式：`git reset`和`git revert`。

-   `git reset`：适合本地提交，回退到指定版本，后面的提交被撤销，如`git reset HEAD~1`，则是撤销本地最新提交。

-   `git revert`：适合有远程分支的提交。会创建一个新提交，代表撤销制定变更后的提交。如下图`git revert HEAD`后，会创建一个`C2'`的提交代表撤销了`C2`变更的提交，revert后即可推送到远程来共享撤销结果。

    <img src="https://oss.seineo.cn/images/202412251008387.png" alt="截屏2024-12-20 10.53.24" style="zoom:50%;" />

### 复制提交

有时候我们会想要“把这个提交放到这里，那个提交放到那里”这样的移动效果，这时候可以使用`git cherry-pick <hash>`，这个命令会将指定的提交复制到当前位置（`HEAD`）下面。比如我们在`C5`处使用`git cherry-pick C2 C4`，则会得到下图的效果：

<img src="https://oss.seineo.cn/images/202412251008559.png" alt="截屏2024-12-20 11.01.00" style="zoom:50%;" />

## 远程仓库

基础知识：在`git clone`之后，本地仓库会出现远程分支如`origin/main`，**远程分支反映了远程仓库在最后一次你与它通信时的状态**。

### 获取数据

如下图所示，`git fetch`会完成两步：

1.   从远程仓库下载本地仓库缺失的提交记录（此处的`C2`、`C3`）
2.   更新本地仓库远程分支的指针（此处的`o/main`）

**注意**：`git fetch`并不会改变本地仓库本地分支（此处的`main*`）的内容，所以其实此时本地仓库并没有与远程仓库同步。

<img src="https://oss.seineo.cn/images/202412251008695.png" alt="截屏2024-12-20 11.18.27" style="zoom:50%;" />

### 推送数据

在使用`git push`推送到远程时，可能会出现冲突的情况。如下图所示，你在本地基于`C1`开发了`C3`，但是你的同事可能已经将远程仓库更新为了`C2`，此时你推送就会失败。

<img src="https://oss.seineo.cn/images/202412251008837.png" alt="截屏2024-12-20 12.53.26" style="zoom:50%;" />

为了解决这一问题，我们需要在本地解决完冲突后再推送。如下图所示，我们可以先拉取远程仓库的数据，可以看到本地就有`main`和`o/main`两个分支，那么就可以按之前“合并分支”小节提到的，使用rebase或者merge来合并分支。

<img src="https://oss.seineo.cn/images/202412251008848.png" alt="截屏2024-12-24 19.37.31" style="zoom:50%;" />

rebase方式：

<img src="https://oss.seineo.cn/images/202412251008999.png" alt="截屏2024-12-24 19.35.36" style="zoom:50%;" />

merge方式：

<img src="https://oss.seineo.cn/images/202412251008012.png" alt="截屏2024-12-24 19.39.16" style="zoom:50%;" />

代码冲突时常发生，如果每次都需要`git fetch` + `git merge/rebase`就有些繁琐了，因此git提供以下两个命令来代替：

```shell
git pull # git fetch + git merge
git pull --rebase # git fetch + git rebase 
```

## 参考

-   [Learn Git Branching](https://learngitbranching.js.org/?locale=zh_CN)
-   [git rebase 用法详解与工作原理](https://waynerv.com/posts/git-rebase-intro/)
