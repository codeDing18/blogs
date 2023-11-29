# 常用命令

---

## 切换分支

checkout最常用的用法莫过于对于工作分支的切换了：

`git checkout branchName`

该命令会将当前工作分支切换到branchName。另外，可以通过下面的命令在新分支创建的同时切换分支：

`git checkout -b newBranch`



## 修改分支名

**1. 本地分支重命名(还没有推送到远程)**

`git branch -m oldName newName `

**2. 远程分支重命名 (已经推送远程-假设本地分支和远程对应分支名称相同)**

a. 重命名远程分支对应的本地分支

`git branch -m oldName newName`

b. 删除远程分支

`git push --delete origin oldName`

c. 上传新命名的本地分支

`git push origin newName`

d.把修改后的本地分支与远程分支关联

`git branch --set-upstream-to origin/newName`

> 注意：如果本地分支已经关联了远程分支，需要先解除原先的关联关系：

`git branch --unset-upstream `





## 保存当前修改

```bash
git stash save "修改的信息"
#列出保存的所有版本，出现如下形式
git stash list 
stash@{0}: On order-master-bugfix: 22222
stash@{1}: On order-master-bugfix: 22222
#选择指定栈中的一个版本
git stash apply stash@{0}
#将stash栈中最后一个版本取出来
git stash pop
# 将指定index的储藏从储藏记录列表中删除
git stash drop stash@{index}
```



## git tag

[参考链接](https://blog.csdn.net/qq_39505245/article/details/124705850)

```bash
# 检出标签的理解 ： 我想在这个标签的基础上进行其他的开发或操作。
#检出标签的操作实质 ： 就是以标签指定的版本为基础版本，新建一个分支，继续其他的操作。
#因此 ，就是 新建分支的操作了。
git checkout -b 分支名称 标签名称


```



## git cherry-pick 

`git cherry-pick`命令的作用，就是将指定的提交（commit）**（git log查看commit id）**应用于其他分支。

```bash
git checkout 分支
git cherry-pick commit id
git cherry-pick <HashA> <HashB>  #将 A 和 B 两个提交应用到当前分支
```



