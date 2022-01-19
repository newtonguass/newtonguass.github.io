---
title: '[on-going]揭開容器神秘面紗– part I: Kernel Space'
date: 2022-01-19 20:52:47
tags:
---

這是篇針對CNCF容器介紹文章的簡略翻譯
[文章出處](https://www.cncf.io/blog/2019/06/24/demystifying-containers-part-i-kernel-space/)

# Part I: Kernel Space
這篇文章範圍為Linux Kernel主題, 會提供你容器需要的背景知識. 會從UNIX, Linux的歷史出發, 談及相關的容器技術如chroot, namespace與cgroups.

## Introduction
談到容器時大部分的人都會想到大鯨魚或是籃底白舵
但容器到底是什麼, Docker官方說明解釋容器是一種standard unit of software, Kubenetes的官方文件也沒有深度解釋. 單就字面意義上解釋container的話, 會讓人以為是輕量的虛擬機器, 用container來形容container技術不盡然正確, 反之, 用pod來形容container協作亦不精確.
container其實是一台host上的<mark>isolated groups of prcesses</mark>, 而這些processes group有著Linux的常見功能, 這些功能有些跟Linux kernel相關, 且絕大多數有歷史因素.
container必須要滿足四個需求
1. Not negotiable: 要在單一host上執行
1. Clearly: 他們是groups of processes. Linux的process是有樹狀結構的, 所以container會有root process.
1. Okay: 必須要isolated.
1. Not so clearly: 會有常用功能.
![container feature](https://i.imgur.com/ujtK6bL.png)
單論任一個需求都會造成混淆, 先從歷史說起

## chroot
多數的UNIX作業系統都有機會更改root directory of current running process 及其child process. chroot最早出現在UNIX 7 (released 1979), Kerkeley Software Distribution(BSD). 現在的Linux可以用 [chroot(2)](https://man7.org/linux/man-pages/man2/chroot.2.html)當成system call(a kernel API function call). chroot亦被認為 jail, 因為1991年有人將chroot當成honeypot去監測駭客, 所以chroot實際上比Linux更老.
以下列例子來說
- chroot example
    ```bash
    $ j=$HOME/tmp/jail
    $ mkdir -p $j
    $ mkdir -p $j/{bin, lib64}
    $ cp /bin/bash $j/bin
    $ sudo chroot $j /bin/basj
    $ ldd /bin/bash | grep / | rev | cut -d " " -f 2 | rev  | xargs -n1 -i dirname {} | xargs -n1 -i mkdir -p $j/{}
    $ ldd /bin/bash | grep / | rev | cut -d " " -f 2 | rev  | xargs -n1 -i cp {} $j/{}
    $ ldd /bin/ls | grep / | rev | cut -d " " -f 2 | rev  | xargs -n1 -i dirname {} | xargs -n1 -i mkdir -p $j/{}
    $ ldd /bin/ls | grep / | rev | cut -d " " -f 2 | rev  | xargs -n1 -i cp {} $j/{}
    $ sudo chroot $j /bin/bash
    $ ls /
    $ ls /etc
    $ exit
    ```
我們創造了一個新的根目錄給ls, 會發現他無法碰到原來的/etc, 但凡原則必有例外
