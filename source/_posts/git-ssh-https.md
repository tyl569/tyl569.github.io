---
title: Git SSH和HTTPS互相切换
tags: []
id: '141'
categories:
  - - Git
date: 2017-08-24 20:19:12
---

#### 先前使用git clone到本地的项目，有一次提交的时候，莫名有个文件不能push了，错误提示是没有权限，也没找到具体的原因，后来想到一个办法，就是切换方式，例如，我是通过SSH clone项目的，能不能通过HTTPS提交呢？查了下资料，发现 git remote 可以实现

<!-- more -->

#### 1) 查看当前remote版本

```bash
$ git remote -v
origin  git@your domain:tylerteng/project.git (fetch)
origin  git@your domain:tylerteng/project.git (push)
```

#### 2) 上面的结果很明显是push 和pull都是使用SSH，使用如下命令改为HTTPS

```bash
$ git remote set-url origin https://your domain/tylerteng/project.git
```

#### 3) 然后你再进行push等操作，就是按照HTTPS进行提交

```bash
$ git push
Username for 'https://your domain':
Password for 'https://tylerteng@your domain':
```