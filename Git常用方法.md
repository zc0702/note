### Git

Git 是一个分布式版本控制系统。什么是分布式，什么是版本控制问谷歌好了，😁。其实它就是一个“时间机器”，可以让我们在任何地方，任何时间，找到当时发生的“任何事情”。

这里面只列了常用的 Git 命令。

#### 常用命令

- git diff Readme.md

  用来查看 Readme.md 文件的修改内容

- git status

  用来查看各个文件的状态

- git add test.md

  将文件加入到 git 的索引库

- git commit -m "修改文件内容"

  提交修改或新增的文件内容

- git reset --hard HEAD^

  回退文件到上一个版本

- git log

  查看历史操作记录

- git clone https://xxxx/aaa/xxx.git

  从远程库中复制

- git checkout master

  切换到 master 分支（主分支或主支）上

- git checkout -b dev

  创建 dev 分支，并切换到 dev 分支

- git branch

  查看分支

- git merge dev

  将 dev 分支的内容，合并到 master 主支上

- git branch -d dev

  删除 dev 分支

- git remote -v

  查看远程库信息

- git remote add origin https://xxxx/aaa/xxx.git

  关联远程库

- git push origin master

  推送分支，将该分支上的本地提交到远程库中

- git pull

  抓取分支

- git rm test.md

  删除 test.md 文件

#### 不常用命令

- git init

  将当前目前变成一个 git 仓库。

  执行后，会自动生成一个 .git 的隐藏目录，这个目录是用来配置和管理仓库的，不是必要的时候不用管，也不要管它，当它是空气就好了。