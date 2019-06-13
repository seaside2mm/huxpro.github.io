# 6. Git

## 6.1 系统设置

通常情况下，我们只需简单设置用户信息和着色即可。

```
$ git config --global user.name "Q.yuhen"
$ git config --global user.email qyuhen@abc.com
$ git config --global color.ui true
```

可以使用 "--list" 查看当前设置。

```
$ git config --list
```

## 6.2 初始化

创建项目目录，然后执行 git init 初始化。这会在项目目录创建 .git 目录，即为元数据信息所在。

```
$ git init
```

通常我们还需要创建一个忽略配置文件 ".gitignore"，并不是什么都需要加到代码仓库中的。

```
$ cat > .gitignore << end

> *.[oa]
> *.so
> *~
> !a.so
> test
> tmp/
> end
```

如果作为 Server 存在，那么可以忽略工作目录，以纯代码仓库形式存在。

```
$ git --bare init
```

在客户端，我们可以调用 clone 命令克隆整个项目。支持 SSH / HTTP/ GIT 等协议。

```
$ git clone ssh://user@server:3387/git/myproj
$ git clone git://github.com/schacon/grit.git mygrit
```

## 6.3 基本操作

Git 分为 "工作目录"、"暂存区"、"代码仓库" 三个部分。

### 6.3.1 添加

文件通过 "git add " 被添加到暂存区，如此暂存区将拥有一份文件快照。

```
$ git add .
$ git add file1 file2
$ git add *.c
```

"git add" 除了添加新文件到暂存区进行跟踪外，还可以刷新已被跟踪文件的暂存区快照。需要注意的是，被提交到代码仓库的是暂存区的快照，而不是工作目录中的文件。

### 6.3.2 提交

"git commit -m " 命令将暂存区的快照提交到代码仓库。

```
$ git commit -m "message"
```

在执行 commit 提交时，我们通常会直接使用 "-a" 参数。该参数的含义是：刷新暂存区快照，提交时同时移除被删除的文件。但该参数并不会添加未被跟踪的新文件，依然需要执行 "git add "操作。

```
$ git commit -am "message"
```

### 6.3.3 状态

可以使用 "git status" 查看暂存区状态，通常包括 "当前工作分支 (Branch)"、"被修改的已跟踪文件(Changed but not updated)"，以及 "未跟踪的新文件 (Untracked files)" 三部分信息。

```
$ git status

# On branch master

# Changed but not updated:
# (use "git add <file>..." to update what will be committed)
# (use "git checkout -- <file>..." to discard changes in working directory)
#
# modified: readme
#

# Untracked files:
# (use "git add <file>..." to include in what will be committed)
#
# install
no changes added to commit (use "git add" and/or "git commit -a")
```

### 6.3.4 比较

要比较三个区域的文件差别，需要使用 "git diff" 命令。

使用 "git diff [file]" 查看工作目录和暂存区的差异。 使用 "git diff --staged [file]" 或 "git diff --cached [file]" 查看暂存区和代码仓库的差异。

```
$ git diff readme

diff --git a/readme b/readme
index e69de29..df8285e 100644
--- a/readme
+++ b/readme
@@ -0,0 +1,2 @@
+1111111111111111111
+
```

查看当前所有未提交的差异，包括工作目录和暂存区。

```
$ git diff HEAD
```

### 6.3.5 撤销

作为代码管理工作，我们随时可以 "反悔"。

使用 "git reset HEAD " 命令可以取消暂存区的文件快照(即恢复成最后一个提交版本)，这不会影响工作目录的文件修改。

使用 "git checkout -- " 从仓库恢复工作目录文件，暂存区不受影响。

```
$ git chekcout -- readme
```

在 Git 中 "HEAD" 表示仓库中最后一个提交版本，"HEAD^" 是倒数第二个版本，"HEAD~2" 则是更老的版本。

我们可以直接 "签出" 代码仓库中的某个文件版本到工作目录，该操作同时会取消暂存区快照。

```
$ git checkout HEAD^ readme
```

如果想将整个项目回溯到以前的某个版本，可以使用 "git reset"。可以选择的参数包括默认的 "--mixed" 和 "--hard"，前者不会取消工作目录的修改，而后者则放弃全部的修改。该操作会丢失其后的日志。

```
$ git reset --hard HEAD^
```

### 6.3.6 日志

每次提交都会为整个项目创建一个版本，我们可以通过日志来查看相关信息。

参数 "git log -p" 可以查看详细信息，包括修改的内容。 参数 "git log -2" 查看最后两条日志。 参数 "git log --stat" 可以查看统计摘要。

```
$ git log --stat -2 -p

commit c11364da1bde38f55000bc6dea9c1dda426c00f9
Author: Q.yuhen <qyuhen@hotmail.com>
Date: Sun Jul 18 15:53:55 2010 +0800

b
---
0 files changed, 0 insertions(+), 0 deletions(-)

diff --git a/install b/install
new file mode 100644
index 0000000..e69de29

commit 784b289acc8dccd1d2d9742d17f586ccaa56a3f0
Author: Q.yuhen <qyuhen@hotmail.com>
Date: Sun Jul 18 15:33:24 2010 +0800

a
---
0 files changed, 0 insertions(+), 0 deletions(-)

diff --git a/readme b/readme
new file mode 100644
index 0000000..e69de29
```

### 6.3.7 重做

马有失蹄，使用 "git commit --amend" 可以重做最后一次提交。

```
$ git commit --amend -am "b2"

[master 6abac48] b2
    0 files changed, 0 insertions(+), 0 deletions(-)
    create mode 100644 abc
    create mode 100644 install

$ git log

commit 6abac48c014598890c6c4f47b4138f6be020e403
Author: Q.yuhen <qyuhen@hotmail.com>
Date: Sun Jul 18 15:53:55 2010 +0800

    b2

commit 784b289acc8dccd1d2d9742d17f586ccaa56a3f0

Author: Q.yuhen <qyuhen@hotmail.com>
Date: Sun Jul 18 15:33:24 2010 +0800
    a
```

### 6.3.8 查看

使用 "git show" 可以查看日志中文件的变更信息，默认显示最后一个版本 (HEAD)。

```
$ git show readme
$ git show HEAD^ readme
```

### 6.3.9 标签

可以使用标签 (tag) 对最后提交的版本做标记，如此可以方便记忆和操作，这通常也是一个里程碑的标志。

```
$ git tag v0.9

$ git tag
v0.9

$ git show v0.9
commit 3fcdd49fc0f0a45cd283a86bc743b4e5a1dfdf5d
Author: Q.yuhen <qyuhen@hotmail.com>
Date: Sun Jul 18 14:53:55 2010 +0800
...
```

可以直接用标签号代替日志版本号进行操作。

```
$ git log v0.9

commit 3fcdd49fc0f0a45cd283a86bc743b4e5a1dfdf5d
Author: Q.yuhen <qyuhen@hotmail.com>
Date: Sun Jul 18 14:53:55 2010 +0800

    a
```

### 6.3.10 补丁

在不方便共享代码仓库，或者修改一个没有权限的代码时，可以考虑通过补丁文件的方式来分享代码修改。

输出补丁：

```
$ git diff > patch.txt
$ git diff HEAD HEAD~ > patch.txt
```

合并补丁：

```
$ git apply < patch.txt
```

## 6.4 工作分支

用 Git 一定得习惯用分支进行工作。

使用 "git branch " 创建分支，还可以创建不以当前版本为起点的分支 "git branch