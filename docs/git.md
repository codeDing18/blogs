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
git stash "修改的信息"
//列出保存的所有版本，出现如下形式
git stash list 
stash@{0}: On order-master-bugfix: 22222
stash@{1}: On order-master-bugfix: 22222
//选择指定栈中的一个版本
git stash apply stash@{0}
//将stash栈中最后一个版本取出来
git stash pop
```





