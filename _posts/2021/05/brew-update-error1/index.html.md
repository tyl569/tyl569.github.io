---
layout: post
title: brew update的时候提示：fatal: It seems that there is already a rebase-apply directory, and....
date: 2021-05-16
tags: ["Git"]
---

今天在更新brew update的时候，提示了报错信息：

    $ brew update
    fatal: It seems that there is already a rebase-apply directory, and
    I wonder if you are in the middle of another rebase.  If that is the
    case, please try
        git rebase (--continue ' --abort ' --skip)
    If that is not the case, please
        rm -fr ".git/rebase-apply"
    and run me again.  I am stopping in case you still have something
    valuable there.

    Already up-to-date.

解决这个的办法就是，重新将brew的update reset下，就OK了。

    $ brew update-reset

本文链接： [https://feilong.tech/2021/05/16/brew-update-error1](https://feilong.tech/2021/05/16/brew-update-error1)