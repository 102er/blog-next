---
title: git版本回退
date: 2021-10-10 22:05:14
tags: git
---

### 代码回滚

git在代码回滚提供了多种方式，比如：git reset...，git revert...，git rebase...等，使用哪种方式回退，取决于实际回退场景。

<!-- more -->

#### git reset

回退版本，直接把代码回退到某个提交，以实际例子，演示下需要如何回退：

![avatar](https://102er.github.io/images/git-reset-log.png)

看提交记录，我们把dev分支不小心合并到了当前特性分支，需要剪掉dev的分支，保证功能分支的特性唯一。从分支线图看出，我们需要回退到 commit id = a94308f 这个版本，问题点是，我们在dev合并之后，又commit了几个记录，我们的目的是剪掉dev合并这次记录，如果是普通的commit 那么我们可以用git revert 撤销这个提交。但是这个是merge操作，没办法。所以，只能回退到a94308f这个版本，然后把dev后的提交pick进来。具体命令如下：

1. git reset --hard  a94308f  #回退到这个版本，并清空其他提交内容
2. git cherry-pick fb9e36f    #找回消失的提交记录
3. git cherry-pick 6d60f55   #找回消失的提交记录
4. 如果有冲突，需要解决冲突，然后执行：git cherry-pick --continue
5. 最后再提交到远程 需要强制覆盖远程：git push -f

这样就实现了版本的回退。有时候，我们发现要回退版本的时候，已经有很多人提交了。这时候 如果在使用git reset+cherry-pick方式，那不得疯了。基于cherry-pick之上，我们可以使用**git rebase -i** 互动模式，帮助我们解决多量commit的场景。

常用命令：

- 使用 `git reset HEAD^` 回到上一個 patch（檔案內容不清空）
- 使用 `git reset --hard HEAD^` 回到上一個 patch，並且強制清除修改的內容 
- 使用 `git reset --hard <commit id>` 直接 reset 成指定的 patch
- 使用 git reset --soft HEAD^  回到前一個 patch，但保持檔案狀態為 ***Changes to be committed\***
- **-- hard  要慎用此参数，它会删除回退点之前的所有修改内容。**如果代码未提交，那就很难补救了，如果代码已经提交，那可以通过找到commit id 找回代码

#### git revert

revert一般在剔除部分提交。比如，我不需要commit=057c752,就可以执行：`git revert -n 057c752`  它可以帮忙我们移除一个commit；它仅仅只是踢掉这个commit，其他的commit还是在的。常用命令：

- 使用 `git revert <commit id>` 還原指定的 patch
- 使用 `git revert --continue` 告知 git 已經解完衝突
- 使用 `git revert --abort` 來要放棄這次 revert
- 不适用于合并操作的commit

### 总结

reset 适用于回退版本到当前版本没有新的commit，而revert是剪掉某个commit记录。
