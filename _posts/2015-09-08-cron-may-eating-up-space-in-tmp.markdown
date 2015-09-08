---
layout: post
title: Cron 中运行的任务可能会填满 /tmp 目录
tags:
    - cron
    - tmp
    - file table entry
comments: true
---

最近在一台服务器上遇到了 /tmp 目录（单独挂载了一个分区）空间用尽的问题。`df` 显示该分区的可用空间为 0. 而 `ls` 则显示 /tmp 目录为空。

这种情况出现的原因和 VFS 的设计有关。在文件系统中，实际表示文件的结构是 inode (index node), 而在目录中出现的文件则是结构 dentry (directory entry). 某个文件的 dentry 会引用对应的 inode. 当文件被打开时，内核使用 file table entry 表示打开的文件，file table entry 也会引用对应的 inode.

系统调用 unlink(2) 的名称就很好的说明了这种行为。删除文件的操作只是移除了引用 inode 的 dentry, 如果 inode 在其它地方还有引用，那么 inode 并不会被删除，文件使用的磁盘空间也不会释放。

为了弄清楚 /tmp 目录中的空间是被谁占用的，就需要找到打开了 /tmp 目录中已经被 unlink 的文件的进程。使用 `lsof /tmp` 命令，可以列出打开了 /tmp 文件系统中的进程的名称、PID, 打开的文件路径、对应的文件描述符。

运行这个命令，发现是某个进程的 1 号和 2 号文件描述符（即 stdout 和 stderr）被打开了 /tmp 目录中已经被删除的文件。一步一步地排查那个程序的代码，和调用它的程序代码，并没有发现哪里有把 stdout 和 stderr 重定向到 /tmp 目录中。

最后，发现那个进程最初的祖先是 cron. 于是开始怀疑到 cron 上。我之前也并不熟悉，不指导 cron 中运行的任务是如何定向 stdout 和 stderr 的。通过查 man page, 发现 cron 会运行结果收集起来 mail 给对应任务的 owner. 这下问题就比较明了了，应该是 cron 为了收集结果，把输出重定向到了某个临时文件。然后被运行的进程一层一层 fork 下来，一直继承了这两个输出。

这个事故告诉我们：

1. 让每个系统部件做该做的事。启动 daemon 的任务就该交给 init 系统. 就算使用 cron 来监测进程，在需要重启进程的时候也应该通过 init 系统来启动，不要越俎代庖。

2. 用每个功能之前先把文档看完。
