[TOC]

# 添加到工作区的文件怎么撤销（添加到工作区了）

- rm readme.txt ：从工作区删除readme.txt文件
- git checkout -- readme.txt：当你不小心改错了工作区的内容时，想丢弃工作区的修改

# git add . 后如何撤销（添加到暂存区了）

- git add . （空格+ 点） 表示当前目录所有文件，不小心就会提交其他文件
- git add 如果添加了错误的文件的话
- 撤销操作
  - git status 先看一下add 中的文件 
  - git reset HEAD 如果后面什么都不跟的话 就是上一次add 里面的全部撤销了 
  - git reset HEAD XXX/XXX/XXX.java 就是对某个文件进行撤销了

# git commit 后如何撤销（添加到本地版本库了）

- git reset --hard HEAD^ ：将版本回退到上一个版本，仅仅是指针的变化，很快
- git reset --hard <版本号> ：将版本回退到特定的版本号

# git push之后如何撤销

- 首先本地回退到上一个版本
- 再git push





git clone https://github.com/wangkang09/JavaTest.git # 克隆远程仓库到本地
git clone -o configRepo https://github.com/wangkang09/config-repo.git # 克隆并指定主机名为configRepo，默认是origin

# 获取仓库状态
git status 

# 将工作区的文件放入暂存区
git add <file/directory> 

# 将暂存区的文件放入版本库
git commit -m "提交说明" 
# 将所有提交文件，放入版本库
git commit -am "提交说明" 

# 获取历史提交记录
git log 
# 同上
git log --pretty=oneline 

# 将版本回退到上一个版本，仅仅是指针的变化，很快
git reset --hard HEAD^ 
# 将版本回退到特定的版本号
git reset --hard <版本号> 

# 查询每一次命令的版本号，方便回退
git reflog 

# 查看工作区和暂存区的区别
git diff readme.txt 
# 查看工作区和版本库的区别
git diff HEAD readme.txt 

# 当你不小心改错了工作区的内容时，想丢弃工作区的修改。如果没有 -- ，是切换分支的命令
git checkout -- readme.txt 

# 当你不小心改错了工作区的内容，并且添加到了暂存区，想丢弃。先使用这个命令，用版本库的内容覆盖暂存区，在使用上面的命令，用暂存区的覆盖工作区。也可以直接提交后，在回退git reset --hard HEAD^
git reset HEAD readme.txt 

# 从工作区删除readme.txt文件
rm readme.txt 
# 提交到暂存区
git add/rm readme.txt 
# 等于上面两步操作
git rm readme.txt 

# 创建分支，仅仅是改下dev的指针指向，当前分支最新版本
git branch dev 
# 切换到dev分支
git checkout dev 
# 相当于以上两个步骤
git checkout -b dev 

# 查看分支和当前分区，标`*`号的是当前分支
git branch 
# 合并某个分支到当前分支中
git merge <branchName> 
git fetch origin test

# 合并两个远程分支，将dev分支的内容合并到test分支中
git checkout -b dev origin/dev
git checkout test
git merge dev # 将dev合并到test中
git push # 将test分支提交到远程

# 删除某个分支
git branch -d <branchName> 

#查看分支合并图
git log --graph 

# 存储工作现场，经过这个命令后，工作区就是干净的了
# 作用：当临时有个bug要修复，先把工作现场stash一下，在创建bug分支修复，这样就不会提示未提交的信息
git stash
# 查看存储的工作现场
git stash list 
# 恢复工作现场，并将stash删除
git stash pop 
git stash apply + git stash drop


# 在本地创建和远程分支对应的分支
git fetch origin 
git checkout -b branchName origin/branchName

# 将远程的test分支fetch到本地的example分支
git fetch origin test:example

# git push，推送本地分支到远程分支
git push <远程主机名> <本地分支名>:<远程分支名>
git push origin test # 表示将本地分支test推送到远程主机的test分支中，如果后者不存在，则会被新建

git push origin # 将当前分支推送到远程origin的同名（有追踪关系）分支
git push # 如果当前分支只有一个追踪分支，远程主机名可以不写

# 删除远程分支
git push origin :test # 删除远程test分支
git push origin --delete test # 同上


# git pull
git pull <远程主机> <远程分支>:<本地分支>
git pull origin master:test # 将远程master分支，拉取并合并到本地test分支中

git pull origin master # 将远程master分支，拉取并合并到当前分支



# 本地创建仓库，并和远程仓库关联

```
git init
git add README.md
git commit -m "first commit"
git remote add origin https://github.com/wangkang09/sentinel-test.git
git push -u origin master
```