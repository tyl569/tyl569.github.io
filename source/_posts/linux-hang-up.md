---
title: Linux后台运行脚本
tags: []
id: '121'
categories:
  - - Linux
date: 2017-08-24 19:58:19
---

### Linux后台运行脚本

#### 我们经常会在Linux运行很多脚本，但是有些脚本是没办法因为某些原因没办法放到后台运行，这就需要我们使用一些工具了，我作为linux新手，目前只知道部分工具或者命令：nuhup，supervisor，screen
<!-- more -->
### 测试方法：持续写入内容，然后查看写入的内容结果

#### 为了验证是否把程序挂到后台运行，我们使用一个循环写入的程序，然后查看被写入的文件，时不时刷新，查看是否有数据持续写入。

*   **创建一个test.c和test.txt**
    
    ```bash
    $ touch test.c
    $ touch test.txt
    ```
    
*   **编写代码**
    

```cpp
#include<stdio.h>
#include<string.h>
#include <unistd.h>
int main() {
        FILE *fp = fopen("test.txt", "a+");
        if (fp == 0) {
                printf("Can not open file \n");
                return 0;
        }
        int ix = 0;
        for (;; ix++) {
                fseek(fp, 0, SEEK_END);
                char s_add_arr[10];
                memset(s_add_arr, '\0', 10);
                sprintf(s_add_arr, "%i this is test \n", ix);
                fwrite(s_add_arr, strlen(s_add_arr), 1, fp);
                sleep(1);
        }
        fclose(fp);
        return 0;
}
```

*   **编译c++文件**

```bash
$ gcc -o test test.c
```

*   **此时会生成一个test的可运行程序**

![](/uploads/2017/08/1489635678506.png)

*   **测试脚本是否可行**

```bash
$ ./test
```

*   **打开一个新窗口，使用vim命令查看是否有写入内容** ![](/uploads/2017/08/1489635664287.png) ![](/uploads/2017/08/1489635678506.png)

### nuhup

*   **运行脚本**

```bash
$ nohup ./test > myout.file 2>&1 &
[2] 3172
[1]   Terminated              nohup ./test
```

*   **查看进程**

```bash
$ jobs
[2]+  Running                 nohup ./test > myout.file 2>&1 &
```

*   **查看test.txt是否写进内容**

```bash
$ vim test.txt
```

### supervisor

#### 之前写过supervisor安装配置方法，具体参见 [Linux 安装supervisor (CentOs or RedHat)](http://feilong.tech/?p=118)

### screen

#### 只要Screen本身没有终止，在其内部运行的会话都可以恢复。这一点对于远程登录的用户特别有用——即使网络连接中断，用户也不会失去对已经打开的命令行会话的控制。只要再次登录到主机上执行screen -r就可以恢复会话的运行。同样在暂时离开的时候，也可以执行分离命令detach，在保证里面的程序正常运行的情况下让Screen挂起（切换到后台）。

*   **运行脚本**

```bash
$ screen -S test ./test ##screen -S 会话命名 command，然后Ctrl+a+d放到后台
[detached from 4070.test]
$ screen -ls ## 查看会话
There is a screen on:
        4070.test       (03/15/2017 10:27:19 PM)        (Detached)
1 Socket in /var/run/screen/S-tengyunlong.
```

*   **查看test.txt内容是否写入**