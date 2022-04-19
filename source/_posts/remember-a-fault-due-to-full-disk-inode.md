---
title: 记一次由于磁盘 Inode 用满导致的故障
date: 2017-10-01 08:00:00
tags:
- 技术
- Linux
---

>  原文首发于　laravel　china：https://learnku.com/articles/10048/remember-a-fault-due-to-full-disk-inode

## 背景
在一台ubuntu服务器上运行了多个laravel项目

## 发现问题
昨天早上有客户提醒服务不可用。排查后发现是磁盘满了（100G）。删除掉40G的备份文件后，服务恢复正常。

但是今天早上，又有很多客户反应服务挂掉了。

查看日志，发现有大量的 failed to open stream: No space left on device 的错误。使用 df -h 查看，磁盘明明还有40G空间，按道理不应该出现这个错误。

## 查找问题
磁盘空间明明还有，但提示空间不足。遂使用 df -ih 查看了一下 inode 的使用情况。

    ➜  ~ df -ih
    Filesystem     Inodes IUsed IFree IUse% Mounted on
    udev             490K   437  490K    1% /dev
    tmpfs            494K   944  494K    1% /run
    /dev/vda1        2.5M  102K  2.5M    4% /
    tmpfs            494K     1  494K    1% /dev/shm
    tmpfs            494K     2  494K    1% /run/lock
    tmpfs            494K    16  494K    1% /sys/fs/cgroup
    /dev/vdb1        6.3M  6.3M  1.3M. 100% /mnt/vdb

发现 inode 已经用满了，从而导致系统无法创建文件。以至于系统抛出 No space left on device 错误。

inode是什么？

> 理解inode，要从文件储存说起。

> 文件储存在硬盘上，硬盘的最小存储单位叫做"扇区"（Sector）。每个扇区储存512字节（相当于0.5KB）。

> 操作系统读取硬盘的时候，不会一个个扇区地读取，这样效率太低，而是一次性连续读取多个扇区，即一次性读取一个"块"（block）。这种由多个扇区组成的"块"，是文件存取的最小单位。"块"的大小，最常见的是4KB，即连续八个 sector组成一个 block。

> 文件数据都储存在"块"中，那么很显然，我们还必须找到一个地方储存文件的元信息，比如文件的创建者、文件的创建日期、文件的大小等等。这种储存文件元信息的区域就叫做inode，中文译名为"索引节点"。

> 每一个文件都有对应的inode，里面包含了与该文件有关的一些信息。

引用自：http://www.ruanyifeng.com/blog/2011/12/inode.html

因此，为题就是磁盘的 block 空间还有剩余，但是 inode 空间用完，无法继续创建新文件。

## 分析问题

在运行的几个 laravel 项目中，有2个访问量很大，而且有写缓存（CACHE_DRIVER=file)。

进入到laravel项目的 storeage/framework/cache/data 目录下，发现存在大量的文件夹，里面存储的是大量的小文件（一个文件只有几KB）。

这些大量的小文件并没有占用多少 block 空间，却吧 inode 空间占满了。

## 解决问题

1. 删除项目的 Cache 文件
   停止 nginx、 php-fpm，进入 data 目录，执行：rm -rf ./*
   然后悲剧了，系统卡死， 10分钟、半小时、1小时...，没办法，只能先清理一部分出来，晚上再删。

2. 切换缓存到 redis（阿里云的云服务）
3. 搞一个脚本，每晚定时清理。
4. 挂在新的磁盘，软连接到 cache 目录，以缓解单块磁盘的inode空间不足的问题。（计划实施）



