# 1. 基础概念

**仓库**

每个项目整个文件夹称为一个仓库repository，仓库中包含零到多个文件，仓库的改动由链状的提交commit所表示，这些提交又分布在不同的分支branch上。每个仓库都有一个起始默认的仓库master/main，然后新建和删除不同的分支，分支之间可以相互合并，以达到多人共同开发同一个项目。

HEAD表示当前指向的提交。

**远程仓库**

origin表示远程仓库，每个仓库包含本地仓库和对应的远程仓库，例如对应为dev和origin/dev，可以从远程仓库分支拉取更新到本地，以及将本地提交的修改推送到远程分支。

## 1.1 文件状态

对于Git项目，仓库文件夹中的文件有如下几种状态：

**untracked/未追踪**

未追踪的文件就是临时存在当前仓库文件夹中，但是并未包含在仓库中的文件，一般是新增的准备加入仓库中的文件，或是产生的日志文件等临时文件。

未追踪的文件如果不想加到仓库中，而且也不想通过 git status 看到，可以将文件规则加到.gitignore文件中。

如果是需要加到仓库中的文件，通过 git add 可以将文件变为暂存状态，然后就可以提交到仓库中。

**unmodified/未修改**

存在仓库中的所有文件，如果本地文件未被修改，则是未修改的状态。

通过 git status 无法看到未修改的文件，但是可以看到哪些文件是属于已修改和暂存状态的。

**modified/已修改**

存在仓库中的文件，修改后，就会变成已修改状态，已修改的文件所在称为工作区。

通过 git add 可以将已修改的文件转成暂存状态，而 git checkout -- file 则将文件修改撤销，变为未修改状态。

**staged/暂存**

仓库文件转位暂存状态后，就是可以准备提交的状态了，暂存的文件所处暂存区。

通过 git commit 可以将所有暂存的文件提交，文件再次变为未修改状态，而 git reset HEAD file 则会将暂存的文件退回到已修改状态。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/version_control/git_file_stages.png)

从上图中可以看到仓库文件的状态流转。不属于仓库的文件是 untracked 状态，属于仓库的文件是 unmodified 状态，作出更改后变为modified状态，将这些文件添加到暂存区就变成了 staged 状态，最后通过提交更新，将暂存区的文件提交到仓库，又变成了 unmodified 状态。

文件修改在不同状态之间流转、以及对应的命令如下图所示：

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/version_control/git_status_cmd.png)

# 2. 配置

## 2.1 GitHub配置密钥

首先在本机创建SSH key，生成的key将会存在于~/.ssh/id_rsa.pub：

```bash
$ ssh-keygen -t rsa -C "email@example.com"
```

在GitHub打开setting，选择添加SSH Key，然后粘贴id_rsa.pub的内容。

## 2.2 配置文件

### .gitconfig

~/.gitconfig文件存在用户根目录，是git的配置文件。

通过 git config 命令修改该配置文件，也可以直接打开文件作修改。

```bash
$ git config --list # 查看所以配置
$ git config <key> # 查看某个配置
$ git config --global key value # 设置配置
```

以下是一部分常用的配置：

```bash
$ git config --global user.name "name" # 设置用户名
$ git config --global user.email "email@example.com" # 设置邮箱
$ git config --global alias.st status # 设置命令别名
```

### .gitignore

该文件放在每个仓库的根目录，用来列出忽略的文件的模式，对于忽略的文件模式，新增的文件和产生的修改将不会被显示。

当然也可以存在之前已提交的文件，现在被.gitignore的规则所忽略。

### .git-credentials

这个文件会记录用户名和密码，这样操作git的更新和提交的时候，就不需要每次输入用户名和密码了。

首先在~/.git-credentials输入内容：

```
https://<username>:<password>@github.com
```

接着通过命令让密码保存生效：

```bash
$ git config --global credential.helper store
```



# 3. 命令

Git的命令有很多，熟练使用常用的即可，并且通过桌面软件如GitHub Desktop，也可以方便地完成很多操作。

查看参考文档：

```bash
$ git help # 查看常用命令
$ git <cmd> -h # 查看某个命令的用法
```



## 3.1 仓库

### init

在当前文件夹创建仓库，并生成.git/管理版本库。

```bash
$ git init
```

### clone

从远端克隆仓库到本地。

```bash
$ git clone <url> # 克隆仓库
$ git clone -b <branch> <url> # 克隆某个分支
```

### remote

```bash
$ git remote # 远程库名称，一般为origin
$ git remote -v # 查看远程库地址
$ git remote add <name> <url> # 添加新的远程仓库

# 将本地分支映射到远程仓库的分支
git remote add origin https://github.com/a/b.git
```

### submodule

将其他仓库作为本仓库的子模块，并保持父项目和子项目的相互独立。

查看子模块：

```bash
git submodule
git submodule status
```

添加子模块：

```bash
# 将对应仓库作为本仓库的子模块，clone到当前路径
git submodule add <url>

# 将对应仓库作为本仓库的子模块，clone到指定路径
git submodule add <url> <path>

git submodule add https:////github.com/a/b.git component/b
```

当前仓库添加子仓库后，会在仓库根目录添加文件 .gitmodules，记录子模块的信息，内容格式如下：

```
[submodule "themes/ananke"]
	path = themes/ananke
	url = https://github.com/theNewDynamic/gohugo-theme-ananke.git
```

更新子模块：

```bash
git submodule update
```

clone仓库时，对于有子模块的仓库，需要添加 --recurse-submodules，否则会丢失子模块文件。

```bash
git clone --recurse-submodules <url>
```

如果 clone 的时候忘记加这个参数，后续可以再去拉取更新子模块：

```bash
git submodule init
git submodule update

# 初始化拉取更新子模块，先 init 再 update
git submodule update --init --recursive
```

删除子模块：

```bash
git submodule deinit <path>
git rm <path>
rm -rf .git/modules/<path>
# 打开 .gitmodules 删除对应模块
```

让子模块指向不同的 commit 版本：

```bash
cd <submodule_path>
git checkout <commit>
git add <submodule_path> # 将子模块对应的版本改动提交本仓库
```

## 3.2 拉取上传

### pull

拉取远程仓库代码更新，合并到本地。

```bash
$ git pull
$ git pull --rebase # 变基后再合并
```

### fetch

拉取远程仓库代码更新到对应的远程分支，但不合并到本地分支。

```bash
$ git fetch
$ git fetch <remote> # 从远程库抓取
```

### push

将本地提交推送至远程库。

```bash
$ git push # 本地提交推送至当前分支对应的远程库
$ git push <remote> <branch> # 将指定分支推送更新指定远程库
$ git push <remote> HEAD # 将当前分支推送更新指定远程库

# 从本地分支推送至master分支
# -u: 相当于先执行 git branch --set-upstream-to=origin/master master，然后再执行 git push origin master
git push -u origin master
```



## 3.3 分支

### branch

该命令用于显示分支、新建或删除分支。

```bash
$ git branch # 显示本地分支
$ git branch -a # 显示本地和远程分支
$ git branch --show-current # 显示当前分支名称
$ git branch <branch> # 从当前分支创建分支
$ git branch <branch> <commit> # 从某个提交创建分支
$ git branch -d <branch> # 删除分支
$ git branch -D <branch> # 强制删除分支，即使有未合并的提交
$ git branch -u <remote>/<branch> # 设置当前分支对应的远程分支
```

### merge

将指定分支合并到本分支。

```bash
$ git merge <branch> 
```

### merge-base

查询两个分支共同祖先的 commit id。

```
git merge-base <branch1> <branch2>
```

### rebase

将当前分支对于基于的分支变基。

假设当前分支 dev 基于 release 分支开发，并且又提交了两次 commit C1 和 C2，而 release 分支在我们新建 dev 分支后又有了新的提交。

![git_before_rebase](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/version_control/git_before_rebase.png)

变基就是从基于的 release 分支最新的一次提交，参照 commit C1 和 C2，生成新的 commit C1' 和 C2'，然后将 dev 分支的指针指向 C2‘。进行变基操作需要当前的工作区没有未提交的修改，如果有可以先提交或者暂时 stash。当然变基操作的前提是这些分支和基于的分支的改动没有代码冲突，否则建议先执行从 release 分支合并到 dev 分支的操作，解决冲突问题，然后再进行变基。

![git_after_rebase](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/version_control/git_after_rebase.png)

如果不变基而是通过 merge 合并分支，会生成一个两条分支的合并。

![git_merge](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/version_control/git_merge.png)

而通过 rebase 将分支变基再合并则往往会变成一条直线的提交，提交记录更加优雅整洁。当然如果变基过程有文件冲突的话，需要解决冲突，变基还是会产生两条分支合并的提交记录。

```bash
$ git rebase <branch1> # 当前分支基于分支1变基
$ git rebase <branch1> <branch2> # 分支2基于分支1变基
$ git rebase --onto <branch1> <branch2> <branch3> # 当分支3基于分支2而分支2基于分支1，使分支3基于分支1变基
```

有时候当前分支有多个本地提交，准备推送至远端，但是想要将他们合并成一个提交，再推送至远端。先通过 rebase：

```bash
$ git rebase -i HEAD~<n> # 编辑最新的n个提交
$ git rebase -i <commit> # 从指定提交到HEAD的所有提交
```

然后会打开一个临时文本进行编辑，将第二个到最后的 pick 改为 squash，然后保存退出。就可以将多个本地提交合并了。

### cherry-pick

把某个指定提交合并到当前分支。

```bash
$ git cherry-pick <commit> # 把提交合并到当前分支
$ git cherry-pick <commit1>..<commit2> # 把从提交1下一个提交到提交2合并到当前分支
$ git cherry-pick <commit1>^..<commit2> # 把从提交1到提交2合并到当前分支

# 选项
# -n 不自动提交，选择多个提交时会将修改合并
```

### cherry

显示本地尚未推送的提交。

```bash
$ git cherry # 显示本地尚未推送的提交
$ git cherry <commit1> <commit2> # 从提交1到提交2增加的提交

# 选项
# -v 显示注释
```



## 3.4 状态

### status

显示仓库状态：所处分支、和远程库的分支差距、有哪些已修改和暂存的文件。

```bash
$ git status # 显示仓库状态
$ git status -s # 简洁状态
```

### log

查看提交记录。

```bash
$ git log # 查看提交记录
$ git log <file> # 和指定文件或目录相关的提交记录
$ git log <branch1> ^<branch2> # 显示包含分支1不包含分支2提交
$ git log --oneline --decorate --graph --all # 查看所有分叉历史
$ git log --oneline --decorate --graph -<n> # 查看最新分叉历史

# 选项
# -<n> 显示最新n个提交
# --oneline 单行显示
# --no-abbrev 显示完整commit id
# --stat 显示差异统计
# -p 显示差异
```

### reflog

查看版本记录。

```bash
$ git reflog # 查看版本记录
$ git reflog <branch> # 查看分支的版本记录

# 选项
# --no-abbrev 显示完整commit id
```

### diff

查看已修改文件的修改。

```bash
$ git diff # 查看所有文件的修改
$ git diff <file> # 查看文件的修改
$ git diff --staged <file> # 查看暂存文件的修改
$ git diff <commit1> <commit2> # 比较两个提交

# 选项
# --word-diff 旧的和新的比较显示在一行
```

### difftool

使用外部工具查看修改。

```bash
$ git difftool # 查看所有文件的修改
$ git difftool <file> # 查看文件的修改
```



## 3.5 编辑与提交

### add

将已修改文件移动到暂存区。

```bash
$ git add . # 添加所有文件到暂存区
$ git add <file> # 添加文件到暂存区
```

### rm

删除文件。

```bash
$ git rm <file> # 删除文件和在仓库中的索引，放到暂存区等待提交

# 选项
# --cached 只删除仓库中的索引，不删除文件
# -r 删除文件夹
```

### mv

重命名文件。

```bash
$ git mv <file1> <file2> # 重命名文件
```

### commit

将暂存区的文件提交，提交务必带有提交信息。

```bash
$ git commit -m "..." # 提交暂存区的修改

# 选项
# -a 提交所有已追踪的修改，而不只是暂存区
# --amend 本次提交并入上一次提交
```

### checkout

切换分支，或是将工作区的修改撤销。

```bash
$ git checkout <branch> # 切换分支
$ git checkout -b <branch> # 创建并切换分支
$ git checkout -b <branch> <commit> # 基于某一提交创建
$ git checkout -- <file> # 撤销工作区文件修改
$ git checkout -- . # 撤销工作区所有文件修改
```

### reset

将版本回退至某一提交，或是将暂存区的文件放回工作区。

```bash
$ git reset <commit> # 版本回退到某一提交
$ git reset HEAD <file> # 将暂存区文件放回工作区
$ git reset HEAD # 将暂存区所有文件放回工作区
```

也可以将已 commit 但未 push 至 origin 的提交回退。

```bash
$ git reset --hard HEAD^ # 撤销最后一次 commit
$ git reset --soft HEAD^ # 撤销最后一次 commit 并保留修改至工作区
$ git reset --hard HEAD~2 # 撤销最后两个 commit
```

### clean

删除未追踪文件。

```bash
$ git clean -f <file> # 删除未追踪文件
$ git clean -fd <file> # 删除未追踪文件和目录
$ git clean -xfd <file> # 删除未追踪文件和目录和.gitignore的文件
```

### stash

工作现场的保存和恢复。

如果当前有一些未提交的修改是有用的，但是还暂时不想提交一个分支，又要切换其他分支的时候，可以选择把工作现场保存，这样可以先做其他改动，或者切换其他分支，后面可以在任何一个分支将工作现场恢复，继续做修改。

```bash
$ git stash # 保存当前工作现场
$ git stash save "..." # 保存当前工作现场，注释说明
$ git stash list # 查看工作现场
$ git stash apply # 恢复所有工作现场
$ git stash apply stash@{0} # 恢复某个工作现场
$ git stash pop # 恢复所有工作现场，并删除保存
$ git stash pop stash@{0} # 恢复某个工作现场，并删除保存
$ git stash drop # 删除所有工作现场
$ git stash drop stash@{0} # 删除某个工作现场
$ git stash clear # 清空工作现场
```

### blame

显示文件的每一行所对应的最新提交。

```bash
$ git blame <file> # 显示文件每一行的最新提交

$ git blame -L 1,10 <file> # 显示文件指定范围的最新提交
```



## 3.6 标签

### tag

标签。

```bash
$ git tag # 查看所有标签
$ git tag <tag> # 创建标签
$ git tag -d <tag> # 删除标签
```

### show

显示各种类型对象的信息。

```bash
$ git show <commit> # 查看某个提交的信息
$ git show <tag> # 查看某个标签的信息
```

## 3.7 管理

### gc

删除 Git 仓库中一些不需要的仓库管理文件。

```bash
$ git gc
```



# 4. 操作技巧

## 4.1 将分支回退到之前的提交

有时候做了错误的提交，然后想将当前分支回退到之前的某次提交。

这时可以对于想要回退的分支 test，基于之前的某次提交新建一个临时分支 test_tmp，将原本的分支 test 删除掉，再基于临时分支 test_tmp 新建回分支 test，最后将临时分支 test_tmp 删除。这时候分支就相当于回退到之前的某次提交了。



# 参考

- [Git - Reference](https://git-scm.com/docs)
- [《Pro Git》中文版](https://git-scm.com/book/zh/v2)

