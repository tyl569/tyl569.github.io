---
title: Git Push 出现错误处理
tags: []
id: '63'
categories:
  - - Git
date: 2017-08-24 19:42:45
---

#### 这几次一直使用`git push`出现如下错误，百度一直没有找到好的解决办法

```bash
remote: error: insufficient permission for adding an object to repository database ./objects
remote: fatal: failed to write object
error: unpack failed: unpack-objects abnormal exit
To git@GIT-ADDRESS
 ! [remote rejected] develop -> develop (unpacker error)
error: failed to push some refs to 'git@GIT-ADDRESS'
```
<!-- more -->
#### 基本上每次都是绕开：

```bash
$ git remote set-url origin HTTPS
#HTTPS 为https的项目地址
$ git push
#输入用户名
#输入密码
```

#### 今天终于找到了解决办法：

```bash
#切换为ssh
$ git config --global push.default matching
$ git push
```

#### 拓展：

```bash
$ git push origin master
```

上面命令表示，将本地的master分支推送到origin主机的master分支。如果后者不存在，则会被新建。

```bash
$ git push origin
# git push
```

上面命令表示，将当前分支推送到origin主机的对应分支。

不带任何参数的git push，默认只推送当前分支，这叫做simple方式。此外，还有一种matching方式，会推送所有有对应的远程分支的本地分支。Git 2.0版本之前，默认采用matching方法，现在改为默认采用simple方式。如果要修改这个设置，可以采用git config命令。

```bash
$ git config --global push.default matching
# 或者
$ git config --global push.default simple
```

```bash
$ git push --force origin
```

上面命令使用–force选项，结果导致在远程主机产生一个”非直进式”的合并(non-fast-forward merge)。除非你很确定要这样做，否则应该尽量避免使用–force选项。