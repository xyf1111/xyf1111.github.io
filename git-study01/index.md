# Git Study01

<!--more-->
## Git配置
### 查看配置
- git config -l : git的配置清单
- git config  --system --list : git的系统配置
- git config --global --list : git的用户配置
### 文件配置
- 系统配置 : E:\git\Git\mingw64\etc\gitconfig
- 用户配置 : C:\Users\Administrator\.gitconfig
### 设置用户名和邮箱
- git config --global user.name "xyf1111" : 设置名称
- git config --global user.email 2219316464@qq.com : 设置邮箱
## Git基本理论
### 工作区域
Git本地有三个工作区域：工作目录(Working Directory)、暂存区(Stage/Index)、资源库(Repository或Git Directory)。如果在加上远程的Git仓库(Remote Directory)就可以分为四个工作区域。文件在这四个区域之间的转换关系如下：
- Workspace : 工作区，就是你平时存放项目代码的地方
- Index/Stage : 暂存区，用于临时存放你的改动，事实上她只是一个文件，保存即将提交到文件列表的信息
- Repository : 仓库区(或本地仓库)，就是安全存放数据的位置，这里面有你提交到所有版本的数据。其中Head指向最新放入仓库的版本
- Rempte : 远程仓库，托管代码的服务器，可以简单的认为是你项目组中的一台电脑用于远程数据交换
### 工作流程
1. 在工作目录中添加、修改文件
2. 将需要进行版本管理的文件放入暂存区
3. 将暂存区的文件提交到git仓库
因此，git管理的文件有三种状态：已修改，已暂存，已提交
## Git项目搭建
### 创建工作目录与常用指令
工作目录(WorkSpace)一般就是你希望Git帮你管理的文件夹，可以是项目的目录，也可以是一个空目录，建议不要有中文。

日常使用只要记住以下6个命令
!["Git日常命令"](/images/git1.png "Git日常命令")
### 本地仓库搭建
1. 创建全新的仓库，需要用GIT管理的项目的根目录执行：
```bash
# 在当前目录新建一个Git代码库
$ git init
```
2. 执行后可以看见，在项目中多出了一个git目录，关于版本等的所有信息都在这个目录里面
### 克隆远程仓库
1. 将远程服务器上的仓库完全镜像一份至本地
```bash
# 克隆一个项目和她的整个代码历史(版本信息)
$ git clone [url]
```
## Git文件操作
### 文件4种状态
版本控制就是对文件的版本控制，要对文件进行修改、提交等操作，首先要知道文件当前在什么状态，不然可能会提交了现在不想提交的文件，或者要提交的文件没提交上
- Untracked : 未跟踪，此文件在文件夹中，但是并没有加入到git库，不参与版本控制，通过 git add 状态变为 Staged
- Unmodify : 文件已经入库，未修改，即版本库中的文件快照内容与文件中完全一致，这种类型的文件有两种去处，如果她被修改，而变为 Modified 。如果使用 git rm 移出版本库，则称为 Untracked 文件
- Modified : 文件已修改，仅仅是修改，并没有进行其他操作，这个文件也有两个去处，通过 git add 可进入暂存 Staged 状态，使用 git checkout 则丢弃修改过的，返回 Unmodify 状态，这个 git checkout 即从库中取出文件，覆盖当前修改
- Staged : 暂存状态，执行 git commit 则将修改同步到库中，这时库中的文件和本地的文件又变为一致，文件为 UNmodify 状态，执行 git reset HEAD filename 取消暂存，文件状态为 Modified
### 查看文件状态
```bash
# 查看指定文件状态
git status [filename]

# 查看所有文件状态
git status
```
### 忽略文件
有些时候不想把某些文件纳入版本控制中，比如数据库文件，临时文件，设计文件等
在主目录下建立".gitingnore"文件，此文件有如下规则
1. 忽略文件中的空行或者以井号(#)开始的行都会被忽略
2. 可以使用Linux通配符。例如：星号(*)代表任意多个字符，问号(?)代表一个字符，方括号([abc])代表可选字符范围，大括号({string1,string2...})代表可选的字符串等
3. 如果名称的最前面有一个感叹号(!)，表示例外规则，将不被忽略
4. 如果名称的最前面是一个路径分隔符(/)，表示要忽略的文件在此目录下，而子目录中的文件不被忽略
5. 如果名称的最后名是一个路径分隔符(/)，表示要忽略的是此目录下该名称的子目录，而非文件(默认文件或目录都被忽略)
```bash
# 为注释

# 忽略所有 .txt 结尾的文件
*.txt

# lib.txt 除外
!lib.txt

# 忽略项目根目录下的temp文件，不包含其他目录
/temp

# 忽略build/目录下的所有文件
build/

# 忽略doc/file.txt但是不包括 doc/xx/aa.txt
doc/*.txt
```
