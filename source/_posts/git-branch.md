---
title: Git 分支（branch）的使用整理
tags: []
id: '62'
categories:
  - - Git
date: 2017-08-24 19:42:06
---

> 平时git branch用的比较少，大多数用的git add/commit/pull/push用的比较多，不过也特意找了一些资料 完整资料请点击[这里](http://blog.jobbole.com/78960/)或者[这里](http://www.open-open.com/lib/view/open1328069889514.html)
> 
> 每次提交版本的时候，git会形成一个时间线，上面会有各种操作，git管这个“时间线”叫做分支，也就是我们常见的master，我们也可以根据需要创建各种分支，但是git只是识别master分支，所以每次创建分支后，需要合并才行。
<!-- more -->
#### 1）我们在github创建一个test的项目，创建过程自行操作

![](/uploads/2017/08/1469950279275.png)

#### 2）我们把项目`` `git clone` ``到本地

```bash
 $ git clone https://github.com/tyl569/test.git ./
 Cloning into '.'...
 warning: You appear to have cloned an empty repository.
 Checking connectivity... done.
```

#### 3）yes，这里肯定是个empty的库，按照常理操作，我们会把项目放在版本库，咱们就姑且创建一个test.txt，然后在里面写句：hello ,world. 当做项目好了。

```tex
hello ,world.
```

#### 4）ok，接下来就是套路操作了。

```bash
$ git status
On branch master

Initial commit

Untracked files:
  (use "git add <file>..." to include in what will be committed)

        test.txt

nothing added to commit but untracked files present (use "git add" to track)

$ git add .
$ git commit -m 初次提交项目
[master (root-commit) 97b26e5] 初次提交项目
 1 file changed, 1 insertion(+)
 create mode 100644 test.txt

$ git push
git: 'credential-osxkeychain' is not a git command. See 'git --help'.
Username for 'https://github.com': tyl569
Password for 'https://tyl569@github.com':
git: 'credential-osxkeychain' is not a git command. See 'git --help'.
Counting objects: 3, done.
Writing objects: 100% (3/3), 240 bytes  0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To https://github.com/tyl569/test.git
 * [new branch]      master -> master
```

#### 5）可以从`` `git push` ``结果看到，这是一个新的分支。

#### 6）创建一个新的分支，并且切换到新的分支上面。

```bash
$ git branch new
$ git checkout new
Switched to branch 'new'
## 提示已经切换到new分支
```

#### 7）此时在本地项目做一些改动，然后提交到github上面。

```bash
# git add/commit等操作省略
$ git push
fatal: The current branch new has no upstream branch.
To push the current branch and set the remote as upstream, use

    git push --set-upstream origin new
```

#### 在`` `git push` ``的时候会提示新的分支没有添加到分支流中，然后使用提示的命令push一下。然后输入用户名和密码。

```bash
$ git push --set-upstream origin new
git: 'credential-osxkeychain' is not a git command. See 'git --help'.
Username for 'https://github.com': tyl569
Password for 'https://tyl569@github.com':
git: 'credential-osxkeychain' is not a git command. See 'git --help'.
Counting objects: 3, done.
Writing objects: 100% (3/3), 283 bytes  0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To https://github.com/tyl569/test.git
 * [new branch]      new -> new
Branch new set up to track remote branch new from origin.
```

#### 8）查看github，会看到合并分支的请求

![](/uploads/2017/08/1469951539710.png)

#### 查看一下test.txt文件，发现内容并没有改变，这是因为github把master作为主分支，如果两个分支不合并的话，另外的用户``` `pull``或者``clone` ```的时候，只会得到master分支的项目，这样如果用户随意搞new分支的内容，都不会影响master分支。

![](/uploads/2017/08/1469951597308.png) ![](/uploads/2017/08/1469951611981.png)

#### 9）如果功能在new分支上面开发完之后，合并分支。

```bash
$  git checkout master
$ git merge new
Updating 97b26e5..96d3fe8
Fast-forward
 test.txt  5 ++++-
 1 file changed, 4 insertions(+), 1 deletion(-)
```

#### 10）这个时候，就会把new分支的改动，合并到master分支了，然后`` `push` ``

```bash
git push
git: 'credential-osxkeychain' is not a git command. See 'git --help'.
Username for 'https://github.com': tyl569
Password for 'https://tyl569@github.com':
git: 'credential-osxkeychain' is not a git command. See 'git --help'.
Total 0 (delta 0), reused 0 (delta 0)
To https://github.com/tyl569/test.git
   97b26e5..96d3fe8  master -> master
```

![](/uploads/2017/08/1469952187908.png) ![](/uploads/2017/08/1469952201409.png)

* * *

### 分支合并的作用：

> *   可以独立开发某个功能或者模块
> *   如果功能没有搞完，也可以`` `push` ``，对项目没有影响