# Git 常见使用

git branch 查看所有分支/<>创建一个分支

git checkout  切换分支 

git checkout -b <> 新建并切换到该分支

git merge 在当前分支下合并某个<>分支

git commit -a -m “local 1” 提交所有已追踪的文件

git add 用git追踪新的文件
> 把目标文件快照放入暂存区域，也就是 add file into staged area，同时未曾跟踪过的文件标记为需要跟踪。  
> 后面可以指明要跟踪的文件或目录路径，如果是目录的话，就说明要递归跟踪该目录下的所有文件。  

git status 查看文件状态

git diff 查看修改过了哪些地方

git remote show 查看<>某个远程库信息

git remote -v  查看当前的远程库

git fetch 拉取远程库<>数据到本地仓库中
> 果是克隆了一个仓库，此命令会自动将远程仓库归于 origin 名下。所以，git fetch origin 会抓取从你上次克隆以来别人上传到此远程仓库中的所有更新（或是上次 fetch 以来别人提交的更新）。有一点很重要，需要记住，fetch 命令只是将远端的数据拉到本地仓库，并不自动合并到当前工作分支，只有当你确实准备好了，才能手工合并。  

git pull 拉取远端仓库的数据并合并到指定分支
```
$ git pull <远程主机名> <远程分支名>:<本地分支名>
```
如果远程分支是与当前分支合并，则冒号后面的部分可以省略：
```
$ git pull origin master
```

> 如果设置了某个分支用于跟踪某个远端仓库的分支（参见下节及第三章的内容），可以使用 git pull 命令自动抓取数据下来，然后将远端分支自动合并到本地仓库中当前分支。在日常工作中我们经常这么用，既快且好。实际上，默认情况下 git clone 命令本质上就是自动创建了本地的 master 分支用于跟踪远程仓库中的 master 分支（假设远程仓库确实有 master 分支）。所以一般我们运行 git pull，目的都是要从原始克隆的远端仓库中抓取数据后，合并到工作目录中的当前分支。  

git push origin master 把本地的 master 分支推送到 origin 服务器上

git checkout —track origin/dev 追踪一个远程分支并创建到本地

添加远程仓库
remote set-url origin https://username:password@github.com/Hinsverson/Notes.git

修改远程仓库
git remote set-url origin https://username:password@github.com/Hinsverson/Notes.git

删除rmote设置
git remote remove origin


为当前分支指定一个track的远程分支，默认push和pull会提交到当前分支。当有2个远程仓库时（比如mirror和origin），可以通过这个指定默认的git push操作会提交到哪个远端仓库对应的分支上，否则需要`git push mirror dev`指定具体的仓库和分支。
```
git branch --set-upstream-to=origin/master master
```

bug fixs