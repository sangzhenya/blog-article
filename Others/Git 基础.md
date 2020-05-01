# Git 基础

Git 是一个开源的分布式版本控制系统，Git 主要分为三个区域：工作区，暂存区和版本库

![Git 三个分区](https://i.loli.net/2020/05/01/wDuvC83x7r94tX2.png)

上图中左侧为工作区域，右侧为版本库。在版本库中标记为 index 的区域是暂存区，标记 master 的是代表 master 分支锁代表的目录树。


## Git 常用命令

```bash
git init # 把当前目录变成 git 管理的版本库
git status # 查看当前 git 版本仓库的状态
git add <file> # 添加文件到暂存区
git commit -m "desc" # 将暂存区的文件添加到版本库
git commit -a -m "desc" # 将修改文件添加到版本库
git diff <file> # 查看工作区修改的区别
git log # 看到 git 提交的记录
git reset --hard e7a8a # 回退到指定的版本
git reset --hard HEAD^ # 回退到上一个版本 HEAD 表示当前版本，HEAD^ 表示上一个版本，HEAD^^ 表示上上一个版本
git reflog # 记录每一次的 git 命令
```