# Git 基础

Git 是一个开源的分布式版本控制系统，Git 主要分为三个区域：工作区，暂存区和版本库

![Git 三个分区](https://i.loli.net/2020/05/01/wDuvC83x7r94tX2.png)

上图中左侧为工作区域，右侧为版本库。在版本库中标记为 index 的区域是暂存区，标记 master 的是代表 master 分支锁代表的目录树。


## Git 常用命令

### Git 配置命令

```bash
git config --list 列出所有的配置信息
git config --global user.name "User Name"
git config --global user.email "User Email"
```

### Git 基本命令

```bash
git init # 把当前目录变成 git 管理的版本库
git status # 查看当前 git 版本仓库的状态
git add <file> # 添加文件到暂存区
git commit -m "desc" # 将暂存区的文件添加到版本库
git commit -a -m "desc" # 将修改文件添加到版本库
git commit --amend # 撤销和修改最后一次提交

git diff <file> # 查看工作区和暂存区的区别
git reset --hard e7a8a # 回退到指定的版本
git reset --hard HEAD^ # 回退到上一个版本 HEAD 表示当前版本，HEAD^ 表示上一个版本，HEAD^^ 表示上上一个版本
# reset 三个选项
# --soft 移动 HEAD 指针，不会修改index和working tree，本地文件的内容并没有发生变化，而index中仍然有最近一次提交的修改
# --mixed  与--soft的不同之处在于，--mixed修改了index，使其与第二个版本匹配。index中给定commit之后的修改被unstaged。
# --hard  那么最后一次提交的修改，包括本地文件的修改都会被清除，彻底还原到上一次提交的状态且无法找回。
git revert  # 回退一次提交

git restore <file> # 丢弃工作区中的改动，使用暂存区的覆盖工作区
git restore --staged <file> # 取消暂存

git log # 看到 git 提交的记录
git reflog # 记录每一次的 git 命令

git rm <file> # 从版本库中删除文件
git rm -f <file> # 同时从工作区和暂存区删除指定的文件
git rm --cached <file> # 从暂存区删除指定的文件, 删除已经跟踪的文件

git remote add origin <远程仓库地址> # 添加远程仓库地址，origin 就是远程库
git clone <远程仓库地址>  # 从远程地址 clone 仓库

git fetch # 从远程仓库拉数据
git pull # 从远程仓库拉数据，比 git fetch 多了 merge 操作
git push origin master # 将修改提交到远程仓库
git checkout -b dev origin/dev # 从远程仓库 checkout branch

```

### Git 分支

```bash
git branch # 查看当前分支
git checkout -b dev # 创建分支
git checkout master # 切换分支
git switch -c dev # 创建分支
git switch master # 切换分支

git merge dev # 合并分支
git branch -d dev # 删除分支
git log --graph # 分支合并图

git stash # 暂存工作中的内容
git stash list # 查看所有的 stash
git stash pop # 弹出 stash 中的内容并删除
git stash apply <stash@{0}> # 应用 stash 中内容但是不删除
git stash drop <stash@{0}> # 删除 stash 中内容

git cherry-pick b7235ae # 将某次提交从另外一个 branch 拉倒当前 branch 上

git rebase # 变基
```

### Git 标签

```bash
git tag v1.0 <commit id> # 创建标签
git show v1.0 # 打印 tag 信息
git tag -d v1.0 # 删除一个标签
git push origin master <tagname> # 将一个标签提交到远程仓库
git push origin master --tags # 将所有标签提交到远程仓库
git push origin :refs/tags/v1.0 # 删除远程仓库的 tag
```

*几个缩写*

```bash
# type gaa 查看缩写
gaa # git add --all
gcmsg # git commit -m
gp # git push
```


## Git 其他操作

### Git 忽略文件

可以需要忽略的文件或文件夹配置在 `.gitignore` 中文件中或 `.git/info/exclude`，区别是前者是会提交到远程仓库，多人共享的。后者则是本地的不会提交到版本仓库中。且者两个都只能排除未入库的文件。如果要排除已经入库的文件则可以使用命令  `git update-index --no-assume-unchanged FILENAME` 忽略。

`.gitignore` 文件格式

1. 以斜杠"/"开头表示目录；"/"结束的模式只匹配文件夹以及在该文件夹路径下的内容，但是不匹配该文件；"/"开始的模式匹配项目跟目录；如果一个模式不包含斜杠，则它匹配相对于当前 .gitignore 文件路径的内容，如果该模式不在 .gitignore 文件中，则相对于项目根目录
2. 以星号"*"通配多个字符，即匹配多个任意字符；使用两个星号"**" 表示匹配任意中间目录，比如`a/**/z`可以匹配 a/z, a/b/z 或 a/b/c/z等
3. 以问号"?"通配单个字符，即匹配一个任意字符
4. 以方括号"[]"包含单个字符的匹配列表，即匹配任何一个列在方括号中的字符。比如[abc]表示要么匹配一个a，要么匹配一个b，要么匹配一个c；如果在方括号中使用短划线分隔两个字符，表示所有在这两个字符范围内的都可以匹配。比如[0-9]表示匹配所有0到9的数字，[a-z]表示匹配任意的小写字母）
5. 以叹号"!"表示不忽略(跟踪)匹配到的文件或目录，即要忽略指定模式以外的文件或目录，可以在模式前加上惊叹号（!）取反

```bash
*.idea          # 所有以 '.idea' 为后缀的文件都屏蔽掉
dist.zip        # 仓库中所有名为 dist.zip 的文件都屏蔽
*prd*.*         # 仓库中所有包含 prd 的文件都屏蔽
/TODO           # 表示仅仅忽略项目根目录下的 TODO 文件，不包括 subdir/TODO
build/          # 表示忽略 build/目录下的所有文件，过滤整个build文件夹；
log/*           # 屏蔽目录 log 下的所有文件，但不屏蔽 log 目录本身
/dist.zip       # 只屏蔽根目录下的 dist.zip  文件，其他目录中的不屏蔽
readme.md       # 屏蔽所有名为 readme.md 的文件
!/readme.md     # 在上一条屏蔽规则的条件下，不屏蔽根目录下的 readme.md 文件
**/foo:         # 表示忽略/foo,a/foo,a/b/foo等
a/**/b:         # 表示忽略a/b, a/x/b,a/x/y/b等
```