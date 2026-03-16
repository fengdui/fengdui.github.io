---
title: "kprobes防勒索和透明加密"
date: "2025-08-02"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

kprobes 是 Linux 内核的动态插桩工具，允许在不修改源码或重启系统的情况下，在内核函数的关键位置插入自定义处理逻辑。它提供了两种探测方式：

kprobe：在函数执行前（pre_handler）和执行后（post_handler）介入。

kretprobe：在函数返回时介入，可获取或修改返回值。

利用这些能力，我们可以实现对内核行为的监控和干预，从而应用于安全防护和数据加密

读取文件时动态解密数据返回给用户，写入文件时动态加密数据存储到磁盘，但磁盘上始终保存密文。这相当于在用户空间和磁盘之间插入一个实时的加解密层

钩住文件操作相关函数：如 vfs_write、security_file_permission、unlink、rename 等，监控对特定后缀文件（如文档、图片）的批量写入或重命名。

钩住网络通信函数：如 tcp_sendmsg、udp_sendmsg，检测可疑的外连流量。

钩住进程创建函数：如 do_execve，监控异常进程的启动

修改返回值：在 kretprobe 中，将函数返回值改为错误码（如 -EPERM），使系统调用失败