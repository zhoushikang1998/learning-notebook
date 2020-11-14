### 1.安装 git

- 设置名字和邮箱

  ```bash
  注意git config命令的--global参数，用了这个参数，表示你这台机器上所有的Git仓库都会使用这个配置，当然也可以对某个仓库指定不同的用户名和Email地址
  git config --global user.name "Your Name"
  git config --global user.email "email@example.com"
  ```

### 2.创建版本库

- 初始化一个Git仓库，使用`git init`命令。
- 添加文件到Git仓库，分两步：（不用记事本打开文件）
  1. 使用命令`git add <file>`，注意，可反复多次使用，添加多个文件；
  2. 使用命令`git commit -m <message>`，完成。

### 3.版本穿梭

- 要随时掌握工作区的状态，使用`git status`命令。
- 如果`git status`告诉你有文件被修改过，用`git diff <file>`可以查看修改内容。

### 4.版本回退

- `git log`命令显示从最近到最远的提交日志

  ```bash
  git log --pretty=oneline
  ```

- 使用`git reset --hard commit_id`命令进行版本回退

  ```bash
   git reset --hard HEAD^
   
    git reset --hard 1094a
  ```

- `git reflog`用来记录你的每一次命令

### 5.工作区和暂存区

- 工作区：本地

- 版本库：`.git`文件

  - 暂存区：版本库中成为`stage`的暂存区
  - 还有Git为我们自动创建的第一个分支`master`，以及指向`master`的一个指针叫`HEAD`

  > 工作区===>git add ===>暂存区===>git commit ===>版本库 

### 6.**管理修改**

- 每次修改，如果不用`git add`到暂存区，那就不会加入到`commit`中

### 7.撤销修改

- `git checkout -- file`可以丢弃工作区的修改，两种情况（**让这个文件回到最近一次`git commit`或`git add`时的状态**）
  - 一种是`readme.txt`自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态
  - 一种是`readme.txt`已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态
- 用命令`git reset HEAD <file>`可以把暂存区的修改撤销掉（unstage），重新放回工作区
  - `git reset`命令既可以回退版本，也可以把暂存区的修改回退到工作区。当我们用`HEAD`时，表示最新的版本。

### 8.删除文件（修改操作）

- `rm test.txt`：删除文件
- 从版本库删除：命令`git rm`删掉，并且`git commit`
- 误删了：`git checkout -- <file>`

### 9.远程仓库

- 添加远程库
- 从远程库克隆

### 10.分支管理

- 创建`dev`分支，然后切换到`dev`分支：`git checkout -b dev`/`git switch -c dev`========`git branch dev` + `git checkout <name>/git switch <name>`
- `git branch`命令查看当前分支
- `git merge`命令用于合并指定分支到当前分支
- 删除`dev`分支：`git branch -d dev`
- 切换到已有的`master`分支：`git switch <name>/git checkout <name>`